# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository

`alt` — репозиторий на GitHub: https://github.com/ikeniborn/alt

Проект находится в начальной стадии. Единственный файл под версионным контролем — `README.md`.

## Claude Code Permissions

В `.claude/settings.local.json` разрешены следующие команды без подтверждения:

```json
{
  "permissions": {
    "allow": [
      "Bash(env:*)",
      "Bash(rpm -q:*)"
    ]
  }
}
```

- `env:*` — чтение переменных окружения
- `rpm -q:*` — запросы к RPM базе пакетов Alt Linux

Запрещены без явного подтверждения пользователя:

- `sudo` — выполнение команд с повышенными привилегиями
- `rm` — удаление файлов и директорий

## Target Environment

### Операционная система

- **Дистрибутив:** ALT Workstation K 10.4 (Sorbaronia Mitschurinii), ветка p10
- **Ядро:** Linux 6.1.111-un-def-alt1 (x86_64, SMP PREEMPT_DYNAMIC)
- **Рабочая среда:** KDE, дисплей-менеджер LightDM
- **Файловая система:** btrfs, subvolumes `@` (/) и `@home` (/home), SSD-оптимизация

### Аппаратное обеспечение

- **CPU:** Intel Core i3-8100 @ 3.60 GHz, 4 ядра (Coffee Lake)
- **GPU:** Intel UHD Graphics 630
- **RAM:** 16 GiB (swap 16 GiB)
- **Диск:** ~217 GiB SSD (/dev/sda, btrfs)
- **Сеть:** enp1s0, 10.152.242.94/23

### Пакетная база

- **Менеджер пакетов:** `apt-get` / `rpm`
- **Репозитории:** корпоративный RTK (gslb-repo.ks.rt.ru), стабильная ветка p10, локальный OMP UEM
- **Ключевые пакеты:** bash 4.4, python3 3.9, git 2.42, curl 8.12, nano 7.2, mc 4.8, htop 3.2, iptables 1.8
- **Отсутствуют:** vim, tmux, docker, podman

### Корпоративная инфраструктура

- **Домен:** UF.RT.RU (Active Directory, Windows интеграция через `samba-winbind`)
- **Аутентификация:** `files winbind` — локальные учётные записи + доменные
- **Управление конфигурацией:** Puppet
- **Антивирус:** Kaspersky (kesl.service, klnagent64.service)
- **MDM:** OMP UEM (ru.omp.uem.service), Basis Workplace Desktop Agent
- **Удалённый доступ:** XRDP (порт 3389), SSHD

### Системные сервисы

| Сервис | Назначение |
|---|---|
| `NetworkManager` | Управление сетью |
| `sshd` | SSH-доступ |
| `smb` | Samba / Windows-шаринг |
| `chronyd` | Синхронизация времени |
| `autofs` | Автомонтирование |
| `postfix` | Локальный почтовый агент |
| `lightdm` | Дисплей-менеджер KDE |

### Безопасность ядра

- ASLR включён (`randomize_va_space=2`)
- `dmesg_restrict=1`, `kptr_restrict=2`
- TCP syncookies включены
- CPU подвержен: Spectre v1/v2, Meltdown, L1TF, MDS, Retbleed, GDS (Intel Coffee Lake)
