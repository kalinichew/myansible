# Ansible Playbooks for Debian and Ubuntu

Набор идемпотентных playbook для базовой подготовки, защиты и эксплуатации серверов Debian/Ubuntu. Все playbook запускаются из каталога `ansible_playbooks/`; значения по умолчанию находятся в `inventory/group_vars/all.yml`.

> **Сначала проверьте на тестовом сервере.** Playbook безопасности меняет SSH и включает UFW, а playbook обслуживания может перезагрузить хост. Перед запуском убедитесь, что SSH-порт разрешён и доступ по ключу работает.

## Структура

```text
ansible_playbooks/
├── ansible.cfg
├── collections/requirements.yml
├── inventory/
│   ├── hosts.ini.example
│   └── group_vars/all.yml
└── playbooks/
    ├── bootstrap/
    │   ├── 01_system.yml
    │   └── 02_security.yml
    ├── docker/docker_engine.yml
    ├── network/
    │   ├── wireguard.yml
    │   └── nginx_proxy.yml
    ├── observability/
    │   ├── node_exporter.yml
    │   └── vector_shipper.yml
    ├── database/
    │   ├── postgres_standalone.yml
    │   └── db_backup_s3.yml
    └── maintenance/
        ├── update_all.yml
        └── disk_cleanup.yml
```

## Подготовка

Установите Ansible на управляющую машину, затем из каталога проекта:

```bash
ansible-galaxy collection install -r collections/requirements.yml
cp inventory/hosts.ini.example inventory/hosts.ini
```

Отредактируйте `inventory/hosts.ini`: укажите адреса серверов и начальную SSH-учётную запись. Локальный inventory исключён из Git. Подготовьте публичный ключ на управляющей машине; по умолчанию используется `~/.ssh/id_ed25519.pub`.

Проверьте связь и выполните базовую настройку на одном хосте:

```bash
ansible debian_servers -m ansible.builtin.ping --limit server1
ansible-playbook playbooks/bootstrap/01_system.yml --limit server1 --check --diff
ansible-playbook playbooks/bootstrap/01_system.yml --limit server1
```

Далее запускайте нужные playbook явно и небольшими партиями. Например:

```bash
ansible-playbook playbooks/bootstrap/02_security.yml --limit server1 --check --diff
ansible-playbook playbooks/bootstrap/02_security.yml --limit server1
```

Все playbook рассчитаны на группу `debian_servers`. Пример inventory включает в неё группу `bootstrap`.

## Bootstrap и безопасность

### 01_system.yml

- Проверяет ОС и наличие SSH-ключа до изменений.
- Обновляет кэш APT, устанавливает базовую оснастку, задаёт UTC и локаль.
- Создаёт отдельного администратора с заданным парольным хешем, устанавливает его публичный ключ и создаёт sudoers-файл с проверкой через `visudo`.

Перед первым запуском задайте `vault_bootstrap_admin_password_hash` в зашифрованном Vault-файле. Укажите SHA-512/yescrypt хеш, а не открытый пароль; пустое значение остановит playbook до любых изменений. По умолчанию `bootstrap_admin_passwordless_sudo: true`: это позволяет ключевому администратору пользоваться sudo без пароля. Это широкие привилегии — выдавайте их только доверенным пользователям и меняйте значение, если для вашей среды настроен парольный sudo.

### 02_security.yml

- Создаёт ранний SSH drop-in, запрещающий парольный вход и вход root.
- Устанавливает и включает Fail2ban для SSH.
- Устанавливает UFW, разрешает входящий TCP на порты 22, 80 и 443, затем задаёт политику deny для входящих соединений и включает firewall.
- Выполняет изменения UFW последовательно, по одному хосту.

Если SSH работает на нестандартном порту, измените `security_ssh_port` **и** включите этот порт в `security_ufw_allowed_tcp_ports`. Playbook завершится до изменения firewall, если порт SSH не разрешён. Не закрывайте текущую SSH-сессию, пока не проверите новое подключение.

## Docker и сетевые сервисы

### Docker Engine

`docker_engine.yml` подключает официальный APT-репозиторий Docker CE, ставит Engine и Compose plugin и задаёт ротацию JSON-логов (50 MB, 5 файлов). Пользователь в группе `docker` может получить root-доступ к хосту; используйте `docker_admin_users` осознанно.

Docker может обходить правила UFW для опубликованных портов контейнеров. Ограничивайте публикацию портов и проверяйте цепочку `DOCKER-USER` для фильтрации контейнерного трафика.

### WireGuard

Заполните `wireguard_peers` для каждого хоста в `group_vars` или в host vars. Ключ интерфейса создаётся на сервере один раз и хранится с правами root-only.

Пример структуры peer:

```yaml
wireguard_peers:
  - public_key: "PUBLIC_KEY_FROM_PEER"
    allowed_ips: "10.10.0.2/32"
    endpoint: "vpn-peer.example.net:51820"
    persistent_keepalive: 25
```

Задайте уникальные адреса и AllowedIPs для каждого узла. Приватный ключ генерируется локально на целевом сервере и не хранится в Git.

### NGINX и Let's Encrypt

Задайте список сайтов в `nginx_proxy_sites`:

```yaml
nginx_proxy_sites:
  - server_name: app.example.net
    upstream_url: http://127.0.0.1:3000
    email: ops@example.net
```

DNS-имя должно указывать на сервер, а входящий TCP/80 должен быть доступен для проверки ACME. Playbook сначала поднимает HTTP-конфигурацию для challenge, получает сертификат, затем устанавливает TLS reverse proxy. Продление выполняет системный таймер Certbot; deploy-hook перезагружает NGINX после получения нового сертификата.

## Observability

### Node Exporter

Используется официальный архив релиза с SHA-256 проверкой и отдельной системной учётной записью. По умолчанию exporter слушает только `127.0.0.1:9100`; задайте приватный адрес в `node_exporter_listen_address`, если Prometheus скрейпит его по сети, и ограничьте доступ сетевым firewall.

### Vector

Vector собирает journald и Docker-логи и отправляет их в настроенный HTTPS sink. Для запуска задайте `vector_sink_url` и сохраните `vault_vector_sink_auth_token` в Ansible Vault. Участие пользователя Vector в группе `docker` позволяет читать Docker API, но группа Docker эквивалентна root-доступу; используйте этот сборщик только на доверенном хосте.

## PostgreSQL и резервное копирование

### PostgreSQL

Playbook устанавливает пакетную версию PostgreSQL Debian/Ubuntu, применяет настраиваемые параметры памяти и создаёт роль и базу приложения. До запуска сохраните `vault_postgres_app_password` в Ansible Vault. Значения памяти в `group_vars/all.yml` — стартовые; подстройте их под RAM и нагрузку сервера.

### S3 backup

`db_backup_s3.yml` создаёт systemd service и ежедневный timer. Резервная копия снимается через локальный socket, сжимается zstd и отправляется в S3 с SSE-S3. Для AWS используйте IAM instance/task role; статические AWS ключи в playbook не задаются. Укажите `db_backup_s3_bucket` и `db_backup_aws_region`, а также убедитесь, что роль имеет минимальные права на запись в нужный префикс. Локальные архивы старше заданного срока удаляются.

Восстановление не автоматизируется этим набором: регулярно проверяйте целостность и выполняйте пробное восстановление отдельно.

## Обслуживание

- `maintenance/update_all.yml`: обновляет APT-пакеты через dist-upgrade, удаляет ненужные зависимости и перезагружает сервер при наличии `/var/run/reboot-required` (по умолчанию включено). Запускайте с `--limit` и по одному узлу.
- `maintenance/disk_cleanup.yml`: ограничивает срок хранения systemd journal, удаляет остановленные контейнеры, неиспользуемые образы и builder cache старше `docker_prune_until`, а также удаляет ненужные пакеты и архивы.

Очистка Docker намеренно не удаляет volumes: они могут содержать данные приложений.

## Секреты

Не храните секреты в Git. Создайте зашифрованный файл, например:

```bash
ansible-vault create inventory/group_vars/vault.yml
```

Пример содержимого:

```yaml
vault_bootstrap_admin_password_hash: "SHA512_OR_YESCRYPT_HASH"
vault_postgres_app_password: "GENERATE_AND_REPLACE"
vault_vector_sink_auth_token: "GENERATE_AND_REPLACE"
```

Запускайте playbook с `--ask-vault-pass` или настроенным безопасным Vault ID. Файл `vault.yml` зашифруйте командой Ansible Vault и не добавляйте пароль Vault в репозиторий.

## Проверка перед применением

```bash
ansible-playbook playbooks/<category>/<playbook>.yml --limit server1 --syntax-check
ansible-playbook playbooks/<category>/<playbook>.yml --limit server1 --check --diff
```

Check mode не может полностью предсказать действия внешних сервисов, firewall, Certbot и перезагрузки. Для таких изменений используйте тестовый хост и проверяйте фактическое состояние после запуска.
