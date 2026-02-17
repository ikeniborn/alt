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

## Target Environment

Alt Linux Sisyphus (kernel 6.1.111-un-def-alt1), рабочая станция на Intel Core i3-8100.
