<div align="center">

# 🛠️ myansible

**Базовая настройка новых Debian и Ubuntu серверов с помощью Ansible**

[![Ansible](https://img.shields.io/badge/Ansible-automation-EE0000?logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Поддержка](https://img.shields.io/badge/OS-Debian%20%7C%20Ubuntu-A81D33)](#поддерживаемые-системы)

</div>

Этот репозиторий содержит стартовый playbook для подготовки Linux-сервера: задаёт часовой пояс и локаль, устанавливает полезные утилиты, создаёт администратора с доступом по SSH-ключу и включает базовую настройку SSH.

> ⚠️ Playbook изменяет системные настройки и SSH. Перед запуском убедитесь, что публичный ключ подходит, а подключение по нему работает. Сначала применяйте настройки к одному тестовому серверу.

## ✨ Что настраивается

- Часовой пояс и системная локаль.
- Обновление кэша APT и установка базовых пакетов.
- Пользователь-администратор в группе sudo.
- Публичный SSH-ключ для нового пользователя.
- Отключение SSH-аутентификации по паролю и входа root по паролю.
- Проверка конфигурации SSH и резервная копия файла перед изменением.

## 🧰 Поддерживаемые системы

Playbook рассчитан на **Debian и Ubuntu**. На других системах он завершится до внесения изменений: используются APT, группа sudo и модуль генерации локали для Debian.

## 📋 Требования

На машине, с которой запускается Ansible:

- Python 3 и Ansible.
- SSH-доступ с правами root или настроенным повышением привилегий.
- Публичный ключ SSH. По умолчанию ожидается файл ~/.ssh/id_ed25519.pub.
- Ansible-коллекции из requirements.yml.

## 🚀 Быстрый старт

### 1. Установите коллекции

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Подготовьте inventory

Скопируйте пример и укажите адрес сервера:

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
```

Пример записи:

```ini
[bootstrap]
my-server ansible_host=203.0.113.10 ansible_user=root
```

Файл inventory/hosts.ini исключён из Git. Не добавляйте в репозиторий пароли, приватные ключи или другие секреты.

### 3. Проверьте доступ и план изменений

```bash
ansible bootstrap -m ping
ansible-playbook playbooks/01_system_bootstrap.yml --check --diff
```

Режим --check полезен, но не заменяет пробный запуск на отдельном сервере.

### 4. Запустите настройку

```bash
ansible-playbook playbooks/01_system_bootstrap.yml --limit my-server
```

Если нужен пароль для SSH-подключения, добавьте -k. Для запроса пароля повышения привилегий используйте -K.

## ⚙️ Переменные

| Переменная | По умолчанию | Назначение |
| --- | --- | --- |
| bootstrap_user | work_user | Имя создаваемого администратора |
| bootstrap_ssh_public_key_path | ~/.ssh/id_ed25519.pub | Путь к публичному ключу на машине с Ansible |
| bootstrap_timezone | Europe/Moscow | Часовой пояс |
| bootstrap_locale | en_US.UTF-8 | Генерируемая системная локаль |
| bootstrap_passwordless_sudo | true | Разрешить sudo без пароля |

Переменные можно переопределить в inventory или через -e. Например, указать другой публичный ключ:

```bash
ansible-playbook playbooks/01_system_bootstrap.yml \
  --limit my-server \
  -e bootstrap_ssh_public_key_path=/home/alex/.ssh/server.pub
```

### О пароле sudo

По умолчанию новый администратор получает NOPASSWD:ALL, чтобы работать на сервере, где вход настроен только по ключу. Это даёт полные права без повторного запроса пароля. Если пароль пользователя настроен и sudo должен запрашивать его, укажите:

```bash
-e bootstrap_passwordless_sudo=false
```

## 🔐 Важное про SSH

Playbook сначала устанавливает ключ новому пользователю и только затем отключает вход по SSH-паролю. До запуска убедитесь, что ключ доступен контроллеру. После применения проверьте подключение новым пользователем:

```bash
ssh work_user@АДРЕС_СЕРВЕРА
```

Не закрывайте текущую административную SSH-сессию, пока не подтвердите новый вход. Если используются дополнительные файлы sshd_config.d или блоки Match, проверьте итоговую конфигурацию SSH: они могут влиять на эффективные значения.

## 🗂️ Структура проекта

```text
.
├── ansible.cfg
├── inventory/
│   └── hosts.ini.example
├── playbooks/
│   └── 01_system_bootstrap.yml
└── requirements.yml
```

## 🏷️ Теги

Можно запускать отдельные группы задач с параметром --tags:

- system, timezone, locale
- packages
- users, sudo
- ssh, hardening

Например:

```bash
ansible-playbook playbooks/01_system_bootstrap.yml --limit my-server --tags packages
```

## 📄 Лицензия

Лицензия пока не указана.
