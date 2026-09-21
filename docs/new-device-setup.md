# Переход на новое устройство

- **Проверено:** 2026-09-21
- **Сервер:** `84.201.164.40`
- **SSH-пользователь:** `timur`

## Важное о доступах

На момент проверки репозитории `Main`, `Backend` и `Frontend` имеют видимость **PUBLIC** на GitHub. Приватный репозиторий также не является хранилищем секретов: доступ можно выдать случайно, секрет останется в истории Git, а форки и клоны нельзя отозвать.

На старом устройстве в `%USERPROFILE%\.ssh` не найден приватный ключ: там находятся только `known_hosts` и `known_hosts.old`. В истории обнаружена команда `ssh -l timur 84.201.164.40` без `IdentityFile`. Поэтому переносить готовый серверный ключ с этого компьютера нечего; новому устройству следует выдать собственный ключ по инструкции ниже.

Поэтому в Git разрешено хранить только:

- адрес сервера и имя пользователя;
- публичные SSH-ключи;
- имена переменных окружения и примеры без реальных значений;
- инструкции по настройке.

Нельзя коммитить приватный SSH-ключ, `.env`, пароли, API-токены, Telegram API hash, SMTP-пароль, ключи БД и другие рабочие значения.

## 1. Подготовить новое устройство

Установить Git, GitHub CLI и OpenSSH Client. В Windows OpenSSH Client можно включить в **Параметры → Система → Дополнительные компоненты**.

Авторизоваться в GitHub:

```powershell
gh auth login
gh auth status
```

Клонировать репозитории:

```powershell
New-Item -ItemType Directory -Path 'D:\AI manager' -Force
Set-Location 'D:\AI manager'

git clone https://github.com/Project-AI-manager/Main.git Main
git clone https://github.com/Project-AI-manager/Backend.git backend
git clone https://github.com/Project-AI-manager/Frontend.git frontend

git -C Main switch task_docs1
git -C backend switch task_127
git -C frontend switch task_236
```

Проверить состояние:

```powershell
git -C Main status
git -C backend status
git -C frontend status
```

## 2. Создать отдельный SSH-ключ на новом устройстве

Лучше выдавать каждому устройству собственный ключ. Тогда потерянное устройство можно отозвать, не меняя доступ для остальных.

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\.ssh" -Force
ssh-keygen -t ed25519 -a 100 -f "$env:USERPROFILE\.ssh\autopilot-prod" -C "autopilot-new-device"
```

Задать ключу парольную фразу. Файлы имеют разное назначение:

- `autopilot-prod` — приватный ключ, остаётся только на новом устройстве;
- `autopilot-prod.pub` — публичный ключ, его можно передавать и добавлять на сервер.

Если сервер допускает вход по паролю, добавить публичный ключ с нового устройства:

```powershell
Get-Content "$env:USERPROFILE\.ssh\autopilot-prod.pub" |
  ssh -l timur 84.201.164.40 'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys'
```

Если вход по паролю отключён, вывести публичный ключ на новом устройстве:

```powershell
Get-Content "$env:USERPROFILE\.ssh\autopilot-prod.pub"
```

Затем добавить эту единственную строку в `~/.ssh/authorized_keys` через уже авторизованное устройство или консоль провайдера сервера. Приватный файл для этого передавать не нужно.

Создать файл `$env:USERPROFILE\.ssh\config`:

```sshconfig
Host autopilot-prod
    HostName 84.201.164.40
    User timur
    IdentityFile ~/.ssh/autopilot-prod
    IdentitiesOnly yes
```

Проверить подключение:

```powershell
ssh autopilot-prod
```

При первом подключении сверить fingerprint ключа сервера через уже доверенное устройство или консоль провайдера. Не подтверждать неизвестный fingerprint вслепую.

## 3. Перенести локальные секреты приложения

На старом устройстве рабочие значения находятся как минимум в следующих игнорируемых Git файлах:

```text
backend/.env
frontend/.env.local
frontend/.env.production.local
```

`backend/.env` содержит рабочие значения для БД, Redis, Qdrant, LLM, Telegram и SMTP. Файлы `.env.example` содержат только схему настроек и уже находятся в репозиториях.

Безопасные варианты переноса:

1. сохранить значения в менеджере паролей с защищёнными заметками;
2. передать зашифрованный архив, а пароль — отдельным каналом;
3. использовать `age` и зашифровать файлы на публичный ключ, созданный на новом устройстве.

Пример с `age`. Сначала на новом устройстве создать ключ получателя:

```powershell
age-keygen -o "$env:USERPROFILE\.config\age\keys.txt"
```

Передать на старое устройство только строку получателя вида `age1...`. На старом устройстве создать временный архив и зашифровать его:

```powershell
$staging = Join-Path $env:TEMP 'autopilot-secret-transfer'
New-Item -ItemType Directory -Path "$staging\backend" -Force
New-Item -ItemType Directory -Path "$staging\frontend" -Force
Copy-Item 'D:\AI manager\backend\.env' "$staging\backend\.env"
Copy-Item 'D:\AI manager\frontend\.env.local' "$staging\frontend\.env.local"
Copy-Item 'D:\AI manager\frontend\.env.production.local' "$staging\frontend\.env.production.local"
Compress-Archive -Path "$staging\*" -DestinationPath "$env:TEMP\autopilot-secrets.zip" -Force
age -r 'age1_REPLACE_WITH_NEW_DEVICE_RECIPIENT' -o "$env:TEMP\autopilot-secrets.zip.age" "$env:TEMP\autopilot-secrets.zip"
Remove-Item -LiteralPath "$env:TEMP\autopilot-secrets.zip" -Force
Remove-Item -LiteralPath $staging -Recurse -Force
```

Передать только `autopilot-secrets.zip.age`. На новом устройстве расшифровать и разложить файлы по тем же путям:

```powershell
age -d -i "$env:USERPROFILE\.config\age\keys.txt" -o "$env:TEMP\autopilot-secrets.zip" '.\autopilot-secrets.zip.age'
Expand-Archive "$env:TEMP\autopilot-secrets.zip" -DestinationPath 'D:\AI manager' -Force
Remove-Item -LiteralPath "$env:TEMP\autopilot-secrets.zip" -Force
```

После переноса проверить, что секретные файлы игнорируются:

```powershell
git -C 'D:\AI manager\backend' status --ignored --short .env
git -C 'D:\AI manager\frontend' status --ignored --short .env.local .env.production.local
```

## 4. Восстановить инструменты и зависимости

Backend:

```powershell
Set-Location 'D:\AI manager\backend'
py -3.12 -m venv .venv312
.\.venv312\Scripts\python.exe -m pip install -r requirements.txt
```

Frontend:

```powershell
Set-Location 'D:\AI manager\frontend'
npm install
```

Перед запуском свериться с актуальными README и `backend/docs/deployment.md`.

## 5. Продолжить исследование ML-модуля

Основная точка продолжения находится в `Main/docs/ml-improvement/README.md`. Для нового чата:

> Изучи `docs/ml-improvement/` в репозитории Main и продолжи улучшение ML-модуля по сохранённому плану. Начни с первого этапа P0. Учти ветки `task_docs1`, `task_127` и `task_236` и сначала проверь рабочие деревья.

## Контрольный список

- [ ] GitHub CLI авторизован под нужной учётной записью.
- [ ] Три репозитория клонированы и выбраны нужные ветки.
- [ ] На новом устройстве создан отдельный SSH-ключ с парольной фразой.
- [ ] На сервер добавлен только публичный ключ.
- [ ] Fingerprint сервера проверен через доверенный канал.
- [ ] `.env` переданы зашифрованно и не попали в Git.
- [ ] Подключение `ssh autopilot-prod` работает.
- [ ] Локальный запуск backend и frontend проверен.
