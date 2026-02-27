# Потеря Kerberos-билета и блокировка входа после перезагрузки: ALT Linux 10.4 + AD

**Версия:** 1.4
**Окружение:** ALT Workstation K 10.4, samba-winbind 4.19.9-alt6, krb5-kinit 1.19.4, KDE Plasma 5, LightDM
**Домен:** UF.RT.RU (Active Directory, Windows Server)
**Источники:** [ActiveDirectory/Join](https://www.altlinux.org/ActiveDirectory/Join) · [ActiveDirectory/Login](https://www.altlinux.org/ActiveDirectory/Login) · [ActiveDirectory/SSSD&Winbind](https://www.altlinux.org/ActiveDirectory/SSSD%26Winbind) · [ActiveDirectory/Kerberos/Tickets](https://www.altlinux.org/ActiveDirectory/Kerberos/Tickets) · [Winbind/Expired_password](https://www.altlinux.org/Winbind/Expired_password)
**Дата:** 2026-02-27

---

## Симптомы

- После перезагрузки доменный пользователь не может войти через LightDM
- `klist` после входа выдаёт: `klist: Credentials cache keyring 'persistent:10002:10002' not found` — TGT не выдан при входе через LightDM (числа — UID пользователя, на разных машинах отличаются)
- `kinit user@UF.RT.RU` завершается успешно (DC доступен через VPN), но смена пользователя через LightDM/KDE всё равно не принимает пароль — winbind запустился в offline-режиме при загрузке и не переключился автоматически при появлении сети
- Экранная блокировка KDE не принимает пароль; в журнале: `kcheckpass: Authentication failure for <user>`
- При входе без сети / VPN появляется сообщение **«нет доступа к домену»** — вход проходит по кэшу, но Kerberos-билет не выдаётся
- `pam_succeed_if(kf5-screenlocker:auth): requirement "user ingroup nopasswdlogin" not met` — **нормальное** INFO-сообщение PAM, не ошибка аутентификации

---

## Диагностика

> **Важно:** На корпоративных конфигурациях доменный пользователь может не иметь доступа к `sudo`
> (`bash: /usr/bin/sudo: Отказано в доступе`) и к журналу systemd
> (`No journal files were opened due to insufficient permissions`).
> В таком случае выполняйте все диагностические команды от имени `root`
> (через `su -` или root-сессию напрямую).

### Проверить порядок запуска сервисов при загрузке

```bash
# Зависимости LightDM — winbind должен присутствовать в After=
systemctl show lightdm --property=After | tr ' ' '\n' | grep -i winbind
# Если пусто — LightDM стартует независимо от готовности Winbind

# Зависимости Winbind — проверить наличие network-online.target
systemctl show winbind --property=After | tr ' ' '\n' | grep network
# Проблема: только network.target и nmb.service (nmb отключён)

# Тип запуска Winbind — важно для понимания механизма синхронизации
systemctl show winbind --property=Type
# Ожидаемо: Type=notify  (systemd ждёт sd_notify READY от winbind)
```

### Проверить Kerberos-билет в текущей сессии

```bash
klist
# При проблеме (реальный пример с проблемной машины):
# klist: Credentials cache keyring 'persistent:10002:10002' not found
#
# При успешном входе:
# Ticket cache: KEYRING:persistent:10002:10002
# Default principal: username@UF.RT.RU
# krbtgt/UF.RT.RU@UF.RT.RU — Valid starting ... Expires ...
```

### Проверить статус Winbind и доступность DC

```bash
# Статус сервиса и время последнего старта
systemctl status winbind --no-pager -n 5

# Доступность DC (только от root; sudo может быть заблокирован для доменного пользователя)
sudo wbinfo -t
# OK: "checking the trust secret for domain UF via RPC calls succeeded"
# Ошибка WBC_ERR_AUTH_ERROR или "failed" — нарушено доверие к домену
# Ошибка "Отказано в доступе" — выполнять от root напрямую: wbinfo -t
```

### Проверить статус смежных сервисов

```bash
systemctl is-active smb nmb
# nmb отключён (disabled, inactive) — это нормально для среды AD с DNS
# smb active — smb.service зависит от winbind.service (After=winbind.service)
```

### Проверить тип кэша Kerberos и offline-режим

```bash
grep default_ccache_name /etc/krb5.conf
# Текущее: KEYRING:persistent:%{uid} — уничтожается при reboot, это нормально

# Фильтровать закомментированные строки (символ ;) обязательно
grep -vE "^[[:space:]]*;" /etc/security/pam_winbind.conf \
  | grep -E "cached_login|krb5_auth|krb5_ccache_type"
# Ожидаемый вывод активной конфигурации:
# cached_login = yes
# krb5_auth = yes
# krb5_ccache_type = KEYRING

grep -E "winbind offline logon|winbind refresh tickets|kerberos method" /etc/samba/smb.conf
```

### Проверить NetworkManager-wait-online

```bash
systemctl is-enabled NetworkManager-wait-online
# masked — ВНИМАНИЕ: After=network-online.target в unit-файлах не даёт задержки
# при отсутствии провайдеров network-online.target активируется мгновенно

# Проверить список провайдеров network-online.target
systemctl list-dependencies network-online.target --no-pager
# При маскировке NM-wait-online: только network.service, без реального ожидания IP
```

### Проверить системные переопределения юнитов

```bash
ls /etc/systemd/system/lightdm.service.d/ 2>/dev/null || echo "Override отсутствует"
ls /etc/systemd/system/winbind.service.d/ 2>/dev/null || echo "Override отсутствует"
```

### Комплексная диагностика домена (официальный ALT инструмент)

Источник: [ActiveDirectory/Join — Диагностика](https://www.altlinux.org/ActiveDirectory/Join)

```bash
# Установить и запустить diag-domain-client
sudo apt-get install diag-domain-client
sudo diag-domain-client
# Проверяет: keytab, krb5 credential cache, DNS SRV-записи,
# конфигурацию Samba, синхронизацию времени и другое
```

### Журнал ошибок аутентификации

```bash
# Требует root-доступа.
# Доменный пользователь без root видит: "No journal files were opened due to insufficient permissions"
# Выполнять от root (su - или root-сессия):
journalctl -b 0 --no-pager -g "pam_winbind|Authentication failure|winbind" | tail -30

# Только ошибки pam_winbind с подробностями
journalctl -b 0 --no-pager -g "pam_winbind" | tail -20
```

---

## Причины

### Причина 1 (главная): LightDM запускается до готовности Winbind

Подтверждено реальными unit-файлами системы и диагностикой проблемной машины
(winbind запущен в 08:36:30 сразу после загрузки, оба override-каталога отсутствуют):

```
# /lib/systemd/system/lightdm.service
After=systemd-user-sessions.service getty@tty1.service plymouth-quit.service
#     ← winbind.service ОТСУТСТВУЕТ

# /lib/systemd/system/winbind.service
After=network.target nmb.service
Type=notify   ← winbind сигнализирует READY только после реальной инициализации

# /lib/systemd/system/smb.service
After=network.target network-online.target nmb.service winbind.service
Wants=network-online.target
Type=notify
```

`Type=notify` означает: systemd ждёт `sd_notify(READY=1)` от winbindd.
Winbind отправляет READY после попытки подключения к DC (online или offline
переход). Добавление `After=winbind.service` к lightdm гарантирует, что LightDM
стартует только **после того, как winbind готов к аутентификации**.

> **Примечание по ALT-документации:** Страница [ActiveDirectory/Login](https://www.altlinux.org/ActiveDirectory/Login) (примечание 5) гласит: «В новых версиях Samba до запуска службы winbind должна запускаться служба smb». Однако реальный unit-файл `/lib/systemd/system/smb.service` содержит `After=winbind.service` — то есть smb стартует **после** winbind. Для сценария клиентской аутентификации (без файлового сервера) это допустимо: winbind работает независимо от smb.

### Причина 2: Winbind стартует до получения IP от DHCP

Winbind зависит от `network.target` — момент поднятия сетевого интерфейса
на уровне ядра. DHCP-адрес может прийти через несколько секунд после этого.
Winbind, стартуя немедленно, не находит DC и переходит в offline-режим.
При наличии кэша (`cached_login = yes`) — вход проходит, но TGT не выдаётся.

### Причина 3: NetworkManager-wait-online.service замаскирован

Без провайдера `network-online.target` активируется сразу после `network.target`.
Директива `After=network-online.target` в `smb.service` (а также в любом override)
**не создаёт задержки** при маскировке `NetworkManager-wait-online.service`.

> Маскировка может быть установлена Puppet/MDM. Размаскирование требует
> согласования с командой управления конфигурацией.

### Причина 4: KEYRING:persistent уничтожается при перезагрузке (ожидаемое поведение)

Linux kernel keyring не сохраняется на диск. При перезагрузке
`KEYRING:persistent:%{uid}` пересоздаётся пустым. Новый TGT должен выдаваться
PAM при следующем входе. Проблема возникает только если PAM не может получить
TGT из-за причин 1 и 2.

### Причина 5: Вход без DC — offline-кэш работает, TGT не выдаётся

При входе без сети / VPN:

1. `cached_login = yes` → pam_winbind проверяет пароль по кэшу winbind (`/var/lib/samba/winbindd_cache.tdb`) — **вход проходит**
2. `krb5_auth = yes` → pam_winbind запрашивает TGT у KDC — **KDC недоступен → TGT не выдаётся**
3. Результат: сообщение «нет доступа к домену», `klist` пуст

Это ожидаемое ограничение. Kerberos-зависимые сервисы (SSO, корпоративные
системы) недоступны до восстановления соединения с DC.

---

## Решение

> **Выполнять от root.** На корпоративных конфигурациях доменный пользователь
> может не иметь прав `sudo`. Войдите как `root` (через `su -` или отдельную
> root-сессию в LightDM) перед выполнением команд ниже.

### Шаг 1. Добавить зависимость LightDM → Winbind (основное исправление)

Winbind использует `Type=notify`: systemd ждёт реального сигнала готовности
от winbindd (не просто запуска процесса). `After=winbind.service` гарантирует,
что LightDM стартует только после инициализации winbind.

```bash
sudo mkdir -p /etc/systemd/system/lightdm.service.d/
sudo tee /etc/systemd/system/lightdm.service.d/winbind-dependency.conf > /dev/null << 'EOF'
[Unit]
# Ожидать сигнала READY от Winbind (Type=notify) перед запуском LightDM.
# Winbind отправляет READY после попытки подключения к DC (online или offline).
# Без этого LightDM принимает логин раньше, чем Winbind инициализирован.
After=winbind.service
Wants=winbind.service
EOF
```

### Шаг 2. Дать Winbind время на получение IP от DHCP

NetworkManager-wait-online замаскирован — `After=network-online.target` не
создаёт реальной задержки. Winbind стартует сразу после `network.target`,
до того как DHCP присвоит адрес. `ExecStartPre` задерживает winbindd,
давая DHCP время настроиться:

> **Важно:** `ExecStartPre=/bin/sleep N` — это workaround. Официальная
> ALT-документация данный приём не описывает. Правильное решение — размаскировать
> `NetworkManager-wait-online.service` (требует согласования с Puppet/MDM).

```bash
sudo mkdir -p /etc/systemd/system/winbind.service.d/
sudo tee /etc/systemd/system/winbind.service.d/network-wait.conf > /dev/null << 'EOF'
[Unit]
# Стартовать после NetworkManager.
# ВАЖНО: network-online.target замаскирован — After= его не ждёт.
# Задержка реализована через ExecStartPre.
After=NetworkManager.service

[Service]
# Workaround: 5 сек на получение DHCP-адреса и маршрутов к DC.
# После задержки winbindd пытается подключиться к DC и отправляет READY=1.
ExecStartPre=/bin/sleep 5
EOF
```

> После размаскирования `NetworkManager-wait-online.service`:
> ```
> After=NetworkManager.service network-online.target
> Wants=NetworkManager.service network-online.target
> ```
> (убрать блок `[Service]` с ExecStartPre)

### Шаг 3. Применить изменения

```bash
sudo systemctl daemon-reload
```

Перезагрузка обязательна — изменения `After=` в юнитах вступают в силу
только при следующем старте системы.

### Шаг 4. Проверить применение (до перезагрузки)

```bash
# LightDM: winbind.service должен появиться в After=
systemctl show lightdm --property=After | tr ' ' '\n' | grep winbind
# Ожидаемо: winbind.service

# Winbind: NetworkManager.service должен появиться в After=
systemctl show winbind --property=After | tr ' ' '\n' | grep NetworkManager
# Ожидаемо: NetworkManager.service

# Проверить полное содержимое override-файлов
sudo systemctl cat lightdm
sudo systemctl cat winbind
```

### Шаг 5. Перезагрузить и проверить

```bash
# После перезагрузки: TGT должен быть выдан при входе
klist
# Ожидаемо:
# Ticket cache: KEYRING:persistent:NNNN:NNNN
# Default principal: username@UF.RT.RU
# krbtgt/UF.RT.RU@UF.RT.RU   Valid starting: <сегодня>   Expires: ...

# Проверить отсутствие ошибок PAM при загрузке (требует sudo)
sudo journalctl -b 0 --no-pager -g "pam_winbind|Authentication failure" | head -20
```

---

## Дополнительные улучшения

### Offline-вход при недоступности DC

Параметры уже настроены:

```
/etc/samba/smb.conf:             winbind offline logon = yes
/etc/security/pam_winbind.conf:  cached_login = yes
```

Основной кэш winbind хранится в `/var/lib/samba/winbindd_cache.tdb`
(источник: [ActiveDirectory/SSSD&Winbind](https://www.altlinux.org/ActiveDirectory/SSSD%26Winbind)).
Файл `/var/cache/samba/netsamlogon_cache.tdb` — кэш netlogon-сессий, отдельный от
offline-кэша аутентификации. Кэш наполняется при первом успешном входе с DC.
После этого пользователь входит по кэшу при временной недоступности DC.
TGT при offline-входе не выдаётся — это ожидаемое поведение (Причина 5).

### Включить отладку pam_winbind для диагностики

Источник: [wiki.altlinux.org/Winbind/Expired_password](https://www.altlinux.org/Winbind/Expired_password)

```bash
# Временно включить подробные логи pam_winbind
sudo tee -a /etc/security/pam_winbind.conf > /dev/null << 'EOF'
debug = yes
debug_state = yes
silent = no
EOF

# Смотреть логи входа через LightDM
sudo journalctl -b 0 --no-pager -g "pam_winbind"
# Пример нормального вывода при успешной аутентификации:
# pam_winbind(lightdm:auth): CONFIG file: krb5_ccache_type 'KEYRING'
# pam_winbind(lightdm:auth): enabling krb5 login flag
# pam_winbind(lightdm:auth): enabling cached login flag

# После диагностики вернуть silent = yes, убрать debug = yes
```

### Размаскирование NetworkManager-wait-online (долгосрочное решение)

После согласования с командой Puppet/MDM:

```bash
sudo systemctl unmask NetworkManager-wait-online.service
sudo systemctl enable NetworkManager-wait-online.service
sudo systemctl daemon-reload

# Обновить winbind override — убрать sleep, добавить надёжную зависимость
sudo tee /etc/systemd/system/winbind.service.d/network-wait.conf > /dev/null << 'EOF'
[Unit]
After=NetworkManager.service network-online.target
Wants=NetworkManager.service network-online.target
EOF
sudo systemctl daemon-reload
```

---

## Аспекты корпоративной безопасности и ИБ

### Оценка текущей конфигурации

| Параметр | Текущее значение | Рекомендация ALT | Риск | Действие |
|---|---|---|---|---|
| `kerberos method` | `secrets and keytab` | Шаблон Join: `system keytab`; пример SSSD&Winbind: `secrets and keytab` | **Низкий** | Оба варианта присутствуют в ALT docs. При `secrets and keytab` и наличии keytab — keytab используется; при отсутствии — только secrets.tdb. |
| `winbind offline logon` | `yes` | — | **Средний** | Вход без DC по кэшу: риск на украденном ПК. Допустимо при физической защите. |
| `cached_login` | `yes` | — | **Средний** | Блокировка в AD не применяется при offline-входе. Для критичных ПК: `= no`. |
| `machine password timeout` | `0` | `0` (рекомендовано ALT, [ActiveDirectory/Join](https://www.altlinux.org/ActiveDirectory/Join)) | **Низкий** | Ротация отключена намеренно. Выполнять ручную ротацию при смене ПК. |
| `krb5_ccache_type` | `KEYRING` | `KEYRING` | **Низкий** | Безопаснее FILE в /tmp: доступен только owner, уничтожается при reboot. |
| `default_ccache_name` | `KEYRING:persistent:%{uid}` | `KCM:` рекомендован для sssd ([wiki.altlinux.org/ActiveDirectory/Kerberos/Tickets](https://www.altlinux.org/ActiveDirectory/Kerberos/Tickets)) | **Низкий** | `KEYRING:persistent` переживает logout, но не reboot. Приемлемо. |
| `NM-wait-online` | masked | — | **Неизвестный** | Установить источник маскировки: Puppet или ручная операция. |
| `addtsusers.sh` в PAM | нечитаем без root | — | **Неизвестный** | Аудит обязателен: скрипт запускается в PAM-контексте при каждом входе. |

### Рекомендации по ИБ

**1. Проверить kerberos method и keytab**

В ALT-документации используются оба варианта: шаблон [ActiveDirectory/Join](https://www.altlinux.org/ActiveDirectory/Join)
содержит `system keytab`, а пример winbind в [SSSD&Winbind](https://www.altlinux.org/ActiveDirectory/SSSD%26Winbind) — `secrets and keytab`.
При `secrets and keytab` winbind использует keytab если он есть, иначе secrets.tdb:

```bash
# Проверить keytab (требует root)
sudo klist -ket 2>/dev/null | head -10
# Если keytab присутствует — kerberos method = secrets and keytab работает через keytab
# Если keytab отсутствует — аутентификация через secrets.tdb
```

**2. Аудит скрипта addtsusers.sh**

Выполняется PAM при каждом входе доменного пользователя:

```
/etc/pam.d/system-auth-winbind-only:
  session optional pam_exec.so /bin/bash /opt/scripts/addtsusers.sh
```

Директива `optional` — ошибка не блокирует вход. Но скрипт выполняется в
PAM-контексте с привилегиями. Проверить от root:

```bash
sudo cat /opt/scripts/addtsusers.sh
sudo ls -la /opt/scripts/addtsusers.sh
# Права должны быть: 700 или 750, владелец root, нет записи для других
```

**3. Offline-кэш и блокировка учётной записи в AD**

При `cached_login = yes` и `winbind offline logon = yes`: если учётная запись
заблокирована в AD, при недоступном DC вход всё равно пройдёт по кэшу.
Для ПК с критичными данными:

```
# /etc/security/pam_winbind.conf
cached_login = no

# /etc/samba/smb.conf
winbind offline logon = no
```

**4. Ротация машинного пароля**

`machine password timeout = 0` — намеренная настройка согласно ALT-документации
(предотвращает конфликт, когда AD и локальный secrets.tdb расходятся). При
выводе ПК из домена или передаче — выполнить ручную ротацию:

```bash
sudo net ads changetrustpw
```

**5. Проверить источник маскировки NetworkManager-wait-online**

```bash
sudo ls -la /etc/systemd/system/NetworkManager-wait-online.service
# Ссылка на /dev/null → замаскирован через systemctl mask
# Выяснить: Puppet manifest или ручная операция
```

**6. Лишняя зависимость SSSD в krb5.conf**

SSSD установлен (`sssd-2.9.6-alt3`), но отключён. Файл
`/etc/krb5.conf.d/enable_sssd_conf_dir` подключает пустую директорию.
Удалить только после согласования (может управляться Puppet):

```bash
sudo rm /etc/krb5.conf.d/enable_sssd_conf_dir
```

---

## Проверка результата

После применения шагов 1–3 и перезагрузки:

```bash
# 1. TGT выдан при входе
klist | grep krbtgt
# Ожидаемо: krbtgt/UF.RT.RU@UF.RT.RU   Valid starting: <сегодня>   Expires: ...

# 2. Нет ошибок PAM (требует root — выполнять от root)
journalctl -b 0 --no-pager -g "Authentication failure|WBC_ERR" | head -10
# Ожидаемо: пусто

# 3. Порядок старта сервисов при загрузке (требует root)
journalctl -b 0 --no-pager -g "winbind|lightdm" -o short | head -20
# Ожидаемо: winbind → (после READY) → lightdm

# 4. Override-файлы присутствуют и загружены
systemctl cat lightdm | grep "winbind"
# Ожидаемо: After=winbind.service, Wants=winbind.service

systemctl cat winbind | grep -E "ExecStartPre|NetworkManager"
# Ожидаемо: After=NetworkManager.service, ExecStartPre=/bin/sleep 5
```

---

## Связанные файлы

| Файл | Назначение |
|------|-----------|
| `/lib/systemd/system/winbind.service` | Оригинальный unit Winbind (After=network.target, Type=notify) |
| `/lib/systemd/system/lightdm.service` | Оригинальный unit LightDM (нет After=winbind) |
| `/etc/systemd/system/lightdm.service.d/winbind-dependency.conf` | Override: LightDM ждёт READY от Winbind |
| `/etc/systemd/system/winbind.service.d/network-wait.conf` | Override: Winbind ждёт NM + sleep 5 |
| `/etc/krb5.conf` | Kerberos: realm, ccache type, время жизни билетов |
| `/etc/samba/smb.conf` | Samba/Winbind: домен, offline logon, kerberos method |
| `/etc/security/pam_winbind.conf` | pam_winbind: cached_login, krb5_auth, ccache type |
| `/etc/pam.d/system-auth` | PAM: маршрутизация local/domain пользователей |
| `/etc/pam.d/system-auth-winbind-only` | PAM-стек для доменных пользователей |
| `/etc/pam.d/kf5-screenlocker` | PAM экранной блокировки KDE |
| `/var/lib/samba/winbindd_cache.tdb` | Основной кэш Winbind для offline logon |
| `/var/cache/samba/netsamlogon_cache.tdb` | Кэш netlogon-сессий (не offline login) |
| `/opt/scripts/addtsusers.sh` | PAM exec session — требует аудита содержимого |
| `/etc/krb5.conf.d/enable_sssd_conf_dir` | Лишний include SSSD (не используется) |

---

## Статус расследования и план проверки

> Раздел обновляется по ходу отладки. При получении root-доступа выполнить
> шаги ниже последовательно, фиксировать вывод.

### Текущая картина (2026-02-27)

| Факт | Вывод |
|------|-------|
| `kinit user@UF.RT.RU` успешен (с VPN) | DC доступен, Kerberos работает, credentials верны |
| Смена пользователя через LightDM/KDE не принимает пароль | pam_winbind не может аутентифицировать — winbind в offline-режиме |
| `klist` от altuser (UID 502) показывает TGT в `KEYRING:persistent:502:502` | TGT выдан в keyring локального пользователя, не домена |
| Override-файлы lightdm и winbind **отсутствуют** | Race condition не устранён, основная причина не исправлена |
| `sudo` недоступен для доменного пользователя | Все команды ниже — от root |

### Шаг A. Диагностика winbind (без изменений конфигурации)

Выполнить от root при следующем доступе:

```bash
# A1. Может ли winbind подключиться к DC прямо сейчас?
wbinfo -t
# OK: "checking the trust secret for domain UF via RPC calls succeeded"
# FAIL: "failed to call wbcCheckTrustCredentials: WBC_ERR_*" — winbind offline

# A2. Может ли winbind аутентифицировать доменного пользователя?
wbinfo -a i.y.tischenko%ПАРОЛЬ
# OK: "plaintext password authentication succeeded"
# FAIL: winbind не обращается к DC — нужен рестарт

# A3. Когда winbind запустился и что писал при старте?
systemctl status winbind --no-pager -n 20

# A4. Журнал ошибок PAM при попытке смены пользователя
journalctl -b 0 --no-pager -g "pam_winbind|Authentication failure|lightdm" | tail -40

# A5. Уточнить: через что именно пытаются сменить пользователя?
#   - KDE «Сменить пользователя» → открывает новый LightDM-экран
#   - su - i.y.tischenko → PAM-стек login/su, не lightdm
#   - ssh localhost -l i.y.tischenko → другой стек
```

### Шаг B. Быстрая проверка без перезагрузки (workaround)

Если `wbinfo -t` упал — перезапустить winbind для перехода в online:

```bash
# B1. Перезапустить winbind (VPN должен быть активен)
systemctl restart winbind

# B2. Убедиться, что winbind подключился к DC
wbinfo -t
# Ожидаемо: OK

# B3. Проверить аутентификацию через winbind
wbinfo -a i.y.tischenko%ПАРОЛЬ
# Ожидаемо: "plaintext password authentication succeeded"

# B4. Попытаться сменить пользователя (тот же VPN, сразу после B3)
# Зафиксировать: принял ли пароль?

# B5. После попытки — проверить PAM-лог
journalctl --no-pager -g "pam_winbind|lightdm|Authentication" | tail -20
```

**Интерпретация результата B:**

- Если после `systemctl restart winbind` смена пользователя прошла → подтверждён сценарий «winbind offline при загрузке». Переходить к Шагу C.
- Если после рестарта winbind всё равно не принимает пароль → проблема не только в offline-режиме. Смотреть журнал A4, уточнять способ смены пользователя.

### Шаг C. Применить постоянное исправление

Применить override-файлы из раздела [Решение](#решение) (Шаги 1–3):

```bash
# C1. Override LightDM → ждать READY от winbind
mkdir -p /etc/systemd/system/lightdm.service.d/
tee /etc/systemd/system/lightdm.service.d/winbind-dependency.conf > /dev/null << 'EOF'
[Unit]
After=winbind.service
Wants=winbind.service
EOF

# C2. Override Winbind → ждать NM + задержка на DHCP
mkdir -p /etc/systemd/system/winbind.service.d/
tee /etc/systemd/system/winbind.service.d/network-wait.conf > /dev/null << 'EOF'
[Unit]
After=NetworkManager.service

[Service]
ExecStartPre=/bin/sleep 5
EOF

# C3. Применить
systemctl daemon-reload

# C4. Проверить до перезагрузки
systemctl show lightdm --property=After | tr ' ' '\n' | grep winbind
# Ожидаемо: winbind.service

systemctl show winbind --property=After | tr ' ' '\n' | grep NetworkManager
# Ожидаемо: NetworkManager.service
```

### Шаг D. Перезагрузка и итоговая проверка

```bash
# D1. Перезагрузить
reboot

# D2. После входа под доменным пользователем — есть ли TGT?
klist
# Ожидаемо: krbtgt/UF.RT.RU@UF.RT.RU   Valid starting: <сегодня>

# D3. Нет ли ошибок PAM в этой загрузке?
journalctl -b 0 --no-pager -g "pam_winbind|Authentication failure" | head -20
# Ожидаемо: пусто

# D4. Порядок старта: winbind должен предшествовать lightdm
journalctl -b 0 --no-pager -g "winbind|lightdm" -o short | head -20
```
