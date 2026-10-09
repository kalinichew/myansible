# myansible

Набор Ansible playbook для подготовки и эксплуатации серверов Debian и Ubuntu.

Основная структурированная версия находится в каталоге **[ansible_playbooks/](ansible_playbooks/README.md)**. Запускайте команды из этого каталога: там лежат собственные `ansible.cfg`, inventory, коллекции и playbook.

## Состав

- Bootstrap и SSH/UFW/Fail2ban hardening
- Docker Engine CE и Compose
- WireGuard mesh и NGINX reverse proxy с Let's Encrypt
- Node Exporter и Vector
- PostgreSQL и расписание резервного копирования в S3
- Обновления, перезагрузки и очистка диска

## Быстрый старт

Из корня репозитория:

```bash
cd ansible_playbooks
ansible-galaxy collection install -r collections/requirements.yml
cp inventory/hosts.ini.example inventory/hosts.ini
```

Перед первым bootstrap настройте inventory, публичный SSH-ключ и зашифрованный парольный хеш администратора по инструкции в [README каталога](ansible_playbooks/README.md).

> Playbook меняют SSH, firewall, системные службы и пакеты. Сначала запускайте их на одном тестовом сервере и проверяйте SSH-доступ до закрытия текущего соединения.
