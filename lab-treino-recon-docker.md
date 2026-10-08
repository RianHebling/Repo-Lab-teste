# Lab de Treino de Recon em Docker

Documento de referência do ambiente de treino montado no Kali (WSL2) do notebook NOTE-CLOUD01. Cobre o que foi feito, como reproduzir manualmente, como operar o liga e desliga, e um roteiro de treino com gabarito.

Data de montagem: 08/10/2026.

---

## 1. Visão geral

O lab são dois containers Docker numa rede isolada, simulando duas máquinas no mesmo segmento de rede. Um é o alvo, cheio de serviços e pistas; o outro é o coletor, com as ferramentas de recon. O coletor coleta informações do alvo, que é o objetivo do treino.

```
NOTE-CLOUD01 (Windows)
  └─ WSL2 → Kali → Docker
       └─ rede "labnet"  172.30.0.0/24  (interna, sem saída para a internet)
            ├─ lab-alvo     172.30.0.10   hostname srv-fileserver
            │     serviços: 22/ssh  21/ftp  80/http  139,445/smb
            └─ lab-coletor  172.30.0.20   hostname coletor
                  ferramentas: nmap, smbclient, ftp, curl, netcat, dnsutils
```

Por que containers e não VMs: dois containers parados não consomem praticamente nada, e ligados ficam na casa de dezenas a poucas centenas de MB. Muito mais leve que duas VMs completas, e eu controlo tudo pelo canal do Kali.

Por que a rede é interna: os containers conversam entre si mas não têm saída para a internet. Deixa o lab fechado, seguro e mais parecido com um recon de rede local.

---

## 2. Terreno (pré-requisitos confirmados)

- Kali GNU/Linux Rolling rodando em WSL2 (host NOTE-CLOUD01).
- systemd ativo no WSL (`/etc/wsl.conf` com `systemd=true`). Isso permite gerenciar o Docker com `systemctl`.
- Usuário comum Reaper (sem root). A instalação exige `sudo`.
- Arquitetura amd64. Recursos folgados: 4 vCPUs, 11 GB de RAM, disco com mais de 900 GB livres.

---

## 3. Passo 1: Instalar o Docker

Detalhe importante do Kali: o Docker não publica repositório para o codinome "kali-rolling". A solução é apontar para o repositório do Debian **bookworm**, que é compatível. Esse é o ponto que costuma travar a instalação de Docker no Kali.

Comandos (executar como root, ou com `sudo` na frente de cada um):

```bash
apt-get update
apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable" > /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl enable --now docker
usermod -aG docker Reaper
```

O que cada bloco faz:
- As primeiras linhas preparam o repositório oficial e a chave de assinatura do Docker.
- A linha do `echo` cadastra o repositório apontando para o bookworm.
- O `apt-get install docker-ce ...` instala o engine, a CLI, o containerd e o plugin do compose.
- `systemctl enable --now docker` liga o serviço e deixa ele subir com o sistema.
- `usermod -aG docker Reaper` coloca o usuário no grupo docker para usar sem `sudo`.

Validação:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

Resultado obtido: Docker 29.8.2, Docker Compose v5.6.0, daemon ativo, hello-world respondeu.

Observação sobre o grupo docker: a mudança de grupo só vale em novas sessões. Quando o servidor HexStrike roda como root, esse ponto nem importa, porque root acessa o docker direto.

---

## 4. Passo 2: Criar a rede isolada

```bash
docker network create --driver bridge --subnet 172.30.0.0/24 --gateway 172.30.0.1 --internal labnet
```

Conferir:

```bash
docker network inspect labnet --format 'Nome: {{.Name}} | Interna: {{.Internal}} | Subnet: {{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

O `--internal` é o que corta a saída para a internet. O Docker fornece DNS interno nessa rede, então um container acha o outro pelo nome.

---

## 5. Passo 3: Container alvo

A imagem do alvo é um Debian com SSH, FTP (vsftpd), HTTP (Apache) e SMB (Samba), mais usuários, arquivos e pistas plantadas de propósito.

### Dockerfile (`/root/lab/alvo/Dockerfile`)

```dockerfile
FROM debian:bookworm-slim
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
    openssh-server vsftpd samba samba-common-bin apache2 \
    net-tools iproute2 procps ca-certificates \
 && rm -rf /var/lib/apt/lists/*

# Usuarios de treino (senhas fracas de proposito)
RUN useradd -m -s /bin/bash suporte && echo 'suporte:Suporte@2023' | chpasswd \
 && useradd -m -s /bin/bash admin   && echo 'admin:P@ssw0rd123'   | chpasswd \
 && echo 'root:toor' | chpasswd

# SSH: login por senha e root habilitados, com banner
RUN mkdir -p /run/sshd \
 && sed -i 's/#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config \
 && sed -i 's/#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config \
 && echo 'Banner /etc/ssh/banner.txt' >> /etc/ssh/sshd_config \
 && echo 'SRV-FILESERVER (Debian) - Acesso restrito a pessoal autorizado.' > /etc/ssh/banner.txt

# FTP anonimo com arquivos plantados
RUN mkdir -p /srv/ftp/pub /var/run/vsftpd/empty \
 && echo 'Backup diario disponivel. Contato: ti@empresa.local' > /srv/ftp/pub/README.txt \
 && echo 'db_user=appuser;db_pass=Str0ngDBpass!' > /srv/ftp/pub/config.bak \
 && printf '%s\n' \
    'listen=YES' 'anonymous_enable=YES' 'local_enable=NO' 'write_enable=NO' \
    'anon_root=/srv/ftp' 'no_anon_password=YES' 'dirmessage_enable=YES' \
    'secure_chroot_dir=/var/run/vsftpd/empty' 'pam_service_name=vsftpd' \
    'pasv_min_port=30000' 'pasv_max_port=30010' 'seccomp_sandbox=NO' \
    > /etc/vsftpd.conf

# Samba: share publico (guest) e privado (requer usuario suporte)
RUN mkdir -p /srv/samba/publico /srv/samba/privado \
 && echo 'Arquivo publico de teste.' > /srv/samba/publico/aviso.txt \
 && echo 'Credenciais do servico: svc_admin / Welcome2024' > /srv/samba/privado/senhas.txt \
 && printf '%s\n' \
    '[global]' '   workgroup = EMPRESA' '   server string = SRV-FILESERVER' \
    '   netbios name = SRVFILESERVER' '   map to guest = Bad User' '   security = user' \
    '' '[publico]' '   path = /srv/samba/publico' '   browseable = yes' '   read only = yes' '   guest ok = yes' \
    '' '[privado]' '   path = /srv/samba/privado' '   browseable = yes' '   read only = yes' '   guest ok = no' '   valid users = suporte' \
    > /etc/samba/smb.conf \
 && (echo 'Suporte@2023'; echo 'Suporte@2023') | smbpasswd -s -a suporte

# HTTP com pistas
RUN printf '%s\n' \
    '<html><head><title>Intranet EMPRESA</title></head><body>' \
    '<h1>Portal Interno - SRV-FILESERVER</h1>' \
    '<p>Sistema de arquivos corporativo v2.3.1</p>' \
    '<!-- TODO: remover painel /admin antes de ir para producao -->' \
    '</body></html>' > /var/www/html/index.html \
 && printf '%s\n' 'User-agent: *' 'Disallow: /admin' 'Disallow: /backup' > /var/www/html/robots.txt \
 && mkdir -p /var/www/html/admin \
 && echo '<html><body><h2>Admin Login</h2></body></html>' > /var/www/html/admin/index.html

RUN echo 'FLAG{recon-lab-alvo-ok}' > /root/flag.txt

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
EXPOSE 21 22 80 139 445
ENTRYPOINT ["/entrypoint.sh"]
```

### Script de inicialização (`/root/lab/alvo/entrypoint.sh`)

```bash
#!/bin/bash
mkdir -p /run/sshd
/usr/sbin/sshd
/usr/sbin/vsftpd /etc/vsftpd.conf &
/usr/sbin/nmbd -D 2>/dev/null || true
/usr/sbin/smbd -D 2>/dev/null || true
source /etc/apache2/envvars 2>/dev/null
/usr/sbin/apache2ctl start 2>/dev/null || apache2 -k start 2>/dev/null
echo "[alvo] servicos no ar: 22/ssh 21/ftp 80/http 139,445/smb"
tail -f /dev/null
```

### Build e execução

```bash
cd /root/lab/alvo
docker build -t lab-alvo:1.0 .
docker run -d --name lab-alvo --hostname srv-fileserver --network labnet --ip 172.30.0.10 lab-alvo:1.0
```

Erro que apareceu no caminho e como foi resolvido: o build falhou na primeira tentativa porque o Debian base já tem um usuário de sistema chamado `backup`, e o `useradd backup` quebrou. A correção foi trocar esse usuário por `suporte`.

---

## 6. Passo 4: Container coletor

Imagem enxuta com as ferramentas de recon.

### Dockerfile (`/root/lab/coletor/Dockerfile`)

```dockerfile
FROM debian:bookworm-slim
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
    nmap smbclient ftp curl netcat-openbsd dnsutils iputils-ping net-tools \
 && rm -rf /var/lib/apt/lists/*
CMD ["tail","-f","/dev/null"]
```

### Build e execução

```bash
cd /root/lab/coletor
docker build -t lab-coletor:1.0 .
docker run -d --name lab-coletor --hostname coletor --network labnet --ip 172.30.0.20 lab-coletor:1.0
```

Observação: a ideia inicial incluía o `enum4linux`, mas ele não existe no repositório do Debian (é pacote do Kali). Foi removido. A enumeração de SMB é feita com `smbclient` e com os scripts NSE de SMB do `nmap`, que cobrem o mesmo.

---

## 7. Passo 5: Como coletar (do coletor contra o alvo)

Entrar no coletor:

```bash
docker exec -it lab-coletor bash
```

Dentro dele, o alvo é `172.30.0.10` (ou o nome `lab-alvo`).

Varredura de portas e versões, com scripts padrão:

```bash
nmap -sV -sC -p 21,22,80,139,445 172.30.0.10
```

Enumeração de SMB (shares visíveis como convidado):

```bash
smbclient -N -L //172.30.0.10
```

FTP anônimo (listar e baixar arquivo):

```bash
curl -s ftp://172.30.0.10/pub/
curl -s ftp://172.30.0.10/pub/config.bak
```

Web:

```bash
curl -s http://172.30.0.10/robots.txt
curl -s http://172.30.0.10/
```

---

## 8. Operação diária: liga e desliga

Os containers ficam salvos entre sessões, parados. Para operar:

| Ação | Comando |
|---|---|
| Ligar o lab | `docker start lab-alvo lab-coletor` |
| Desligar o lab | `docker stop lab-alvo lab-coletor` |
| Ver status | `docker ps -a --filter name=lab-` |
| Entrar no coletor | `docker exec -it lab-coletor bash` |
| Entrar no alvo | `docker exec -it lab-alvo bash` |
| Ver a rede | `docker network inspect labnet` |

Reset total (apagar e recriar do zero):

```bash
docker rm -f lab-alvo lab-coletor
docker network rm labnet
# depois refazer os passos 2, 3 e 4
```

Quando quiser que eu opere, basta pedir em linguagem direta, por exemplo "liga o lab", "roda o recon no alvo", "desliga o lab". Eu executo pelo canal do HexStrike. Lembrando que isso funciona quando o servidor HexStrike está rodando e esta conversa está aberta. Não ajo sozinho sem a sua ordem.

---

## 9. Roteiro de treino com gabarito

Tente descobrir tudo a partir do coletor, sem olhar o gabarito. Depois confira.

Desafios:
1. Quais portas e serviços o alvo expõe, e em que versão.
2. O FTP aceita acesso anônimo. O que dá para baixar de lá.
3. Quais compartilhamentos SMB existem e qual deles exige usuário.
4. O que o site revela sobre caminhos que não deveriam estar visíveis.
5. Achar a flag do alvo.

Gabarito:
- Portas: 21 vsftpd 3.0.3, 22 OpenSSH 9.2, 80 Apache 2.4.68, 139 e 445 Samba 4.6.2 (NetBIOS SRVFILESERVER).
- FTP anônimo: diretório `pub` com `README.txt` e `config.bak`. O `config.bak` tem credenciais plantadas (`appuser` / `Str0ngDBpass!`).
- SMB: shares `publico` (convidado), `privado` (exige o usuário `suporte`) e `IPC$`. O `privado` guarda `senhas.txt`.
- Web: `robots.txt` lista `/admin` e `/backup`, e o HTML tem um comentário citando o painel `/admin`.
- Flag: `FLAG{recon-lab-alvo-ok}`, em `/root/flag.txt` dentro do alvo.

Credenciais plantadas no alvo (para exercícios de acesso):
- Sistema: `root` / `toor`, `admin` / `P@ssw0rd123`, `suporte` / `Suporte@2023`.
- SMB: usuário `suporte` com senha `Suporte@2023` acessa o share `privado`.

Ideias de evolução: testar login SSH ou SMB com as credenciais achadas, ligar um terceiro container na rede, trocar serviços do alvo, ou adicionar uma aplicação web vulnerável para praticar recon de web.

---

## 10. Atalho na área de trabalho

Foi criado o arquivo `HexStrike-Server.bat` para subir o servidor com dois cliques. Conteúdo:

```bat
@echo off
title HexStrike Server (Kali)
wsl.exe -d kali-linux -u root bash -c "cd /usr/share/hexstrike-ai && python3 hexstrike_server.py"
pause
```

Salvar na área de trabalho (no Bloco de Notas, "Salvar como", tipo "Todos os arquivos", nome terminando em `.bat`). Rodando como root, o servidor já enxerga o docker.

---

## 11. Solução de problemas

- Eu digo que o canal caiu ou "connection refused" na 8888: o servidor HexStrike não está rodando. Suba pelo atalho ou com `cd /usr/share/hexstrike-ai && sudo python3 hexstrike_server.py`.
- Container não sobe: ver o log com `docker logs lab-alvo` ou `docker logs lab-coletor`.
- Serviço do alvo não responde: entrar com `docker exec -it lab-alvo bash` e checar portas com `ss -tln`.
- Conflito de IP ao ligar: algum outro container pode estar usando o IP. Ver com `docker ps` e ajustar.
- Recriar um container sem perder a imagem: `docker rm -f <nome>` e rodar o `docker run` de novo.

---

## 12. Resumo do que roda hoje

- Imagens: `lab-alvo:1.0` (456 MB) e `lab-coletor:1.0` (276 MB).
- Rede: `labnet` 172.30.0.0/24, interna.
- Estado atual: containers parados, prontos para ligar sob demanda.
- Arquivos de build: `/root/lab/alvo/` e `/root/lab/coletor/`.
