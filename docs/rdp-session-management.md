# Управление RDP-сессиями: стабильная работа на ALT Linux + xrdp

**Версия:** 1.7
**Окружение:** ALT Workstation K 10.4, xrdp 0.10.2-alt2, KDE Plasma 5
**Дата:** 2026-02-19

---

## Контекст

Исходная конфигурация `/etc/xrdp/sesman.ini`:

```ini
[Sessions]
KillDisconnected=false    # сессия не убивается при разрыве соединения — OK
DisconnectedTimeLimit=0   # ← ПРОБЛЕМА: сессии живут вечно, накапливаются
IdleTimeLimit=0           # нет ограничения по простою — OK
Policy=Default            # UB: user + bits-per-pixel — OK
```

---

## Проблемы и причины

### 1. При подключении с другого ПК открывается новая сессия

**Причина — одна из двух (нужно диагностировать):**

**А) Предыдущая сессия завершилась.**
Пользователь нажал "Выйти из системы" вместо закрытия окна RDP-клиента.
KDE завершается, сессия уничтожается. При следующем подключении создаётся новая.

В логах:
```
Window manager (pid XXXXX) finished normally in N secs
Session on display 10 has finished.
```

**Б) Разный bpp (глубина цвета) на разных ПК.**
`Policy=Default` = `UB` (user + bits-per-pixel). Сессия ищется по паре: пользователь + bpp.
При разном bpp на разных ПК xrdp-sesman создаёт новую сессию.

> `Policy` не позволяет убрать проверку bpp — это ограничение xrdp.
> Согласно документации: *"The U and B criteria cannot be turned off."*
> Любое значение Policy, включая `U`, всегда учитывает bpp.

### 2. Приложение "не открывается", если процесс уже запущен

Следствие проблемы 1. Приложение запущено в предыдущей сессии на другом X11 display.
В новой сессии оно недоступно — запуск создаёт ещё один экземпляр вместо подъёма окна.

### 3. Накопление сессий и деградация производительности

При `DisconnectedTimeLimit=0` каждая оборванная сессия остаётся в памяти навсегда.
Каждая xrdp-сессия содержит полный стек KDE: `Xorg` + `startplasma-x11` +
`kwin_x11` + `plasmashell` и десятки сервисов — ~500–800 MiB RAM и постоянная
нагрузка на CPU. При 3–4 накопленных сессиях система начинает тормозить.

---

## Диагностика

### Проверить количество текущих сессий

```bash
# Каждый Xorg — отдельная xrdp-сессия
# startplasma без видимого Xorg — локальная сессия LightDM (норма)
ps aux | grep -E "startplasma-x11|Xorg" | grep -v grep
```

Нормальная картина для одного пользователя:
```
user  4378   startplasma-x11               ← локальная KDE (LightDM)
user  193097 Xorg :10 -auth ... (xrdp)  ┐
user  193120 startplasma-x11             ┘  ← одна xrdp-сессия
```

Проблемная картина — несколько Xorg:
```
user  Xorg :10 ...  ┐ сессия 1
user  Xorg :11 ...  ┘ сессия 2  ← лишняя, накопилась
```

### Определить причину создания новых сессий

```bash
# Внимание: лог перезаписывается при каждом рестарте xrdp-sesman
grep -E "finished normally|create a session|found existing session" \
  /var/log/xrdp-sesman.log | tail -20
```

`finished normally` перед `create a session` → причина А (logoff).
`create a session` без предшествующего `finished normally` → причина Б (bpp).

### Проверить bpp при подключениях

```bash
# $HOME работает корректно; не использовать ~ при sudo
grep "bpp" $HOME/.xorgxrdp.*.log
```

Одинаковый bpp 32 на всех ПК — причина Б исключена.

---

## Решение

### Шаг 1. Установить DisconnectedTimeLimit (обязательно)

`DisconnectedTimeLimit` — таймер **простоя без подключения**. Запускается в момент
disconnect. **Сбрасывается при reconnect.** Если пользователь переподключился до
истечения таймера — попадает в ту же сессию, таймер запускается заново.

```
Disconnect → таймер 7200 сек
Reconnect через 30 мин → та же сессия, таймер сбрасывается
Disconnect снова → таймер 7200 сек
Нет reconnect 7200 сек → сессия убивается автоматически
```

```bash
sudo crudini --set /etc/xrdp/sesman.ini Sessions DisconnectedTimeLimit 7200
sudo systemctl reload xrdp-sesman   # SIGHUP: применяет конфиг без завершения сессий
```

> `systemctl reload` (SIGHUP) перечитывает конфиг не трогая запущенные сессии.
> `systemctl restart` — завершает все сессии; применять только при отсутствии пользователей.

| Значение | Поведение |
|----------|-----------|
| `0` | Сессия живёт вечно — **опасно** |
| `3600` | 1 час — подходит для кратких перерывов |
| `7200` | **2 часа** — рекомендуется для рабочего места |
| `28800` | 8 часов — рабочий день, очищается к утру |

### Шаг 2. Причина А — не делать Logoff

Закрывать окно RDP-клиента крестиком (disconnect), а не через "Выйти из системы" в KDE.

Дополнительно — восстановление приложений при непреднамеренном logoff:

```bash
# Выполнить под нужным пользователем, без sudo
kwriteconfig5 --file ksmserverrc --group General --key loginMode restorePreviousLogout
```

### Шаг 3. Причина Б — зафиксировать bpp=32 на всех клиентах

Настройки RDP-клиентов описаны в разделе [Настройка Remmina](#настройка-remmina).

---

## Скрипт reconnectwm.sh

Файл `/etc/xrdp/reconnectwm.sh` выполняется xrdp-sesman **при каждом переподключении**
к существующей сессии (не при создании новой).

**Условия выполнения:**
- Запускается от имени пользователя без аргументов
- Переменные окружения установлены xrdp-sesman: `DISPLAY`, `HOME`, `USER`, `SHELL`
- `DISPLAY` имеет вид `:10.0` (с суффиксом `.0`)
- Скрипт должен завершаться быстро — он блокирует завершение reconnect

**Содержимое** `/etc/xrdp/reconnectwm.sh`:

```sh
#!/bin/sh
# Выполняется при переподключении к существующей xrdp-сессии.
# Переменные окружения: DISPLAY, HOME, USER, SHELL — установлены xrdp-sesman.

# Подстроить разрешение под новый монитор/клиент.
# При reconnect с другого ПК KDE не перестраивает разрешение автоматически.
xrandr --auto

# Сбросить таймер скринсейвера.
# Если disconnect длился долго — X-сервер мог активировать screensaver,
# и экран погаснет сразу после reconnect.
xset s reset
xset dpms force on
```

Применить:

```bash
sudo tee /etc/xrdp/reconnectwm.sh > /dev/null << 'EOF'
#!/bin/sh
xrandr --auto
xset s reset
xset dpms force on
EOF
sudo chmod +x /etc/xrdp/reconnectwm.sh
```

**Что намеренно не включено:**

| Действие | Причина |
|---|---|
| Разблокировка KDE Screensaver | Безопасность: при reconnect пользователь должен ввести пароль |
| `qdbus` команды | Не установлен в системе |
| `systemctl --user` | Недоступен в xrdp-сессии |

---

## Удаление старых сессий

### Автоматически (через DisconnectedTimeLimit)

После установки `DisconnectedTimeLimit=7200` старые сессии удаляются сами
через 2 часа после последнего disconnect. Вмешательства не требуется.

### Вручную — завершить конкретную сессию

```bash
# 1. Найти лишние сессии по Xorg-процессам
ps aux | grep "Xorg :" | grep -v grep
# Показывает PID и номер display: :10, :11, ...

# 2. Определить к какой сессии подключены сейчас
grep "IP:" /var/log/xrdp-sesman.log | tail -3

# 3. Завершить ненужную сессию — убить Xorg нужного display
#    kwin/plasmashell теряют дисплей и завершаются, sesman очищает сессию
kill $(pgrep -f "Xorg :11")
```

### Вручную — завершить все xrdp-сессии

```bash
# Завершает ВСЕ сессии включая активные
# Выполнять только при отсутствии работающих пользователей
sudo systemctl restart xrdp-sesman
```

---

## Настройка Remmina

**Версия в системе:** Remmina 1.4.40

### RDP-файл профиля

Remmina поддерживает импорт стандартных `.rdp` файлов: `Файл → Импорт`.

Сохранить как `alt-rdp.rdp` и импортировать, или создать профиль вручную по параметрам ниже.

**Готовый профиль** `alt-rdp.rdp`:

```ini
screen mode id:i:2
session bpp:i:32
compression:i:1
keyboardhook:i:2
displayconnectionbar:i:1
disable wallpaper:i:1
disable full window drag:i:1
allow desktop composition:i:1
allow font smoothing:i:1
disable menu anims:i:1
disable themes:i:0
disable cursor setting:i:0
bitmapcachepersistenable:i:1
full address:s:10.152.242.94
audiomode:i:2
audiocapturemode:i:0
redirectprinters:i:0
redirectsmartcard:i:0
redirectcomports:i:0
redirectsmartcards:i:0
redirectclipboard:i:1
redirectposdevices:i:0
autoreconnection enabled:i:1
authentication level:i:0
prompt for credentials:i:1
negotiate security layer:i:1
remoteapplicationmode:i:0
alternate shell:s:
shell working directory:s:
gatewayhostname:s:
gatewayusagemethod:i:4
gatewaycredentialssource:i:4
gatewayprofileusagemethod:i:0
precommand:s:
promptcredentialonce:i:1
drivestoredirect:s:
```

### Пояснение к ключевым параметрам

| Параметр | Значение | Причина |
|---|---|---|
| `session bpp` | `32` | **Критично.** `0` = неопределённый bpp → xrdp создаёт новую сессию при каждом подключении. `32` = фиксированный bpp → Policy=UB находит существующую сессию |
| `allow desktop composition` | `1` | Нужно KDE для корректного рендера (kwin использует композитинг) |
| `allow font smoothing` | `1` | Сглаживание шрифтов в KDE |
| `audiocapturemode` | `0` | Звук отключён (`audiomode:2`), захват незачем |
| `autoreconnection enabled` | `1` | Автопереподключение при обрыве сети |
| `authentication level` | `0` | Не проверять сертификат — нужно для самоподписанного сертификата xrdp |
| `redirectclipboard` | `1` | Буфер обмена между клиентом и сервером |

### Правильное завершение работы

Закрывать окно Remmina **крестиком** (disconnect) — сессия KDE продолжает работать
на сервере. При следующем подключении попадёте в ту же сессию.

Не нажимать "Выйти из системы" в меню KDE — это logoff, сессия уничтожается.

---

## Проверка результата

1. Подключиться по RDP с первого ПК — создаётся сессия на `:10`
2. Закрыть окно Remmina крестиком (не logout из KDE)
3. Подключиться по RDP со второго ПК под тем же пользователем
4. Ожидаемый результат: переподключение к существующей сессии `:10`

```bash
# При успешном reconnect строки "create a session" не будет
grep -E "create a session|Received request" /var/log/xrdp-sesman.log | tail -5
```

---

## Ограничения и известные особенности

- **`Policy` не позволяет отключить проверку bpp** — ограничение xrdp. Единственный
  способ иметь одну сессию при подключениях с разных ПК: одинаковый bpp=32 на всех клиентах.

- **Лог `/var/log/xrdp-sesman.log` перезаписывается** при каждом рестарте xrdp-sesman.
  История сессий до рестарта недоступна.

- **`xrdp-patch-6-rtk`** перезаписывает `/etc/xrdp/sesman.ini` при переустановке.
  После обновления пакета проверить `DisconnectedTimeLimit` в секции `[Sessions]`.

- **KDE + XRDP**: `org.freedesktop.systemd1 failed` в логах — нормально, systemd
  пользовательской сессии недоступен в xrdp.

- **Remmina**: предупреждение `unable to get secret service` при запуске — нормально
  для среды без D-Bus secret service; не влияет на работу RDP.

---

## Связанные файлы

| Файл | Назначение |
|------|-----------|
| `/etc/xrdp/sesman.ini` | Политика сессий, таймауты |
| `/etc/xrdp/xrdp.ini` | Основной конфиг xrdp |
| `/etc/xrdp/startwm.sh` | Запуск KDE Plasma при новой сессии |
| `/etc/xrdp/reconnectwm.sh` | Скрипт при переподключении |
| `$HOME/.xorgxrdp.NN.log` | Лог X-сессии (NN = номер display) |
| `/var/log/xrdp-sesman.log` | Лог сессий (перезаписывается при рестарте) |
| `/var/log/xrdp.log` | Лог xrdp демона (подключения) |
