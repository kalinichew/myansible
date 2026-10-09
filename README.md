# myansible

Набор Ansible playbook для подготовки и обслуживания серверов Debian и Ubuntu. В репозитории собраны базовая настройка, SSH и firewall hardening, Docker, сетевые сервисы, мониторинг, PostgreSQL, резервное копирование и обслуживание.

> [!WARNING]
> Playbook применяют изменения к реальным серверам: меняют SSH и firewall, ставят и перезапускают службы, обновляют пакеты и могут перезагрузить систему. Начинайте с тестового хоста, используйте --limit и не закрывайте текущую SSH-сессию до проверки нового входа.

## Возможности

| Раздел | Playbook | Что делает |
|---|---|---|
| Bootstrap | playbooks/bootstrap/system.yml | Обновляет APT, ставит базовые утилиты, устанавливает UTC и локаль, создаёт администратора с sudo и SSH-ключом |
| Security | playbooks/bootstrap/security.yml | Отключает SSH-вход по паролю и root, настраивает Fail2ban и UFW |
| Docker | playbooks/docker/docker_engine.yml | Устанавливает Docker CE и Compose plugin, задаёт ротацию логов |
| Сеть | playbooks/network/wireguard.yml | Создаёт интерфейс WireGuard и peer-конфигурацию |
| Сеть | playbooks/network/nginx_proxy.yml | Настраивает NGINX reverse proxy и сертификаты Let's Encrypt |
| Мониторинг | playbooks/observability/node_exporter.yml | Устанавливает Node Exporter как systemd-сервис |
| Мониторинг | playbooks/observability/vector_shipper.yml | Отправляет journald и Docker-логи во внешний HTTPS sink |
| База данных | playbooks/database/postgres_standalone.yml | Устанавливает PostgreSQL, настраивает память и создаёт роль/базу |
| Резервное копирование | playbooks/database/db_backup_s3.yml | Планирует pg_dump, zstd-сжатие и загрузку в S3 через systemd timer |
| Обслуживание | playbooks/maintenance/update_all.yml | Обновляет пакеты по одному серверу и перезагружает хост при необходимости |
| Обслуживание | playbooks/maintenance/disk_cleanup.yml | Ограничивает размер journal, чистит Docker-объекты и APT-кэш |

## Структура

~~~text
.
├── ansible.cfg
├── collections/
│   └── requirements.yml
├── inventory/
│   ├── hosts.ini.example
│   └── group_vars/
│       └── all/
│           ├── vars.yml
│           └── vault.yml
└── playbooks/
    ├── bootstrap/
    │   ├── system.yml
    │   └── security.yml
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
~~~

Файл ansible.cfg по умолчанию использует inventory/hosts.ini. Общие настройки находятся в inventory/group_vars/all/vars.yml, а чувствительные значения — в зашифрованном inventory/group_vars/all/vault.yml. Все команды ниже запускаются из корня репозитория.

## 1. Подготовьте управляющую машину

Для запуска нужен Ansible на управляющей машине и SSH-доступ к целевому серверу. На Windows используйте WSL либо другую Linux-машину как управляющую систему.

Установите требуемые коллекции:

~~~bash
ansible-galaxy collection install -r collections/requirements.yml
~~~

Скопируйте шаблон inventory и отредактируйте его:

~~~bash
cp inventory/hosts.ini.example inventory/hosts.ini
~~~

Для PowerShell используйте:

~~~powershell
Copy-Item inventory/hosts.ini.example inventory/hosts.ini
~~~

В inventory замените примерный адрес 203.0.113.10 на адрес сервера, а ansible_user — на начальную SSH-учётную запись. Шаблон использует root только для первичной настройки. Локальный hosts.ini исключён из Git и не должен публиковаться.

Проверьте, что Ansible видит сервер:

~~~bash
ansible all -m ansible.builtin.ping
~~~

Если управляющая машина использует нестандартный SSH-ключ, задайте путь к публичному ключу в inventory/group_vars/all/vars.yml через bootstrap_ssh_public_key_path. Это путь к файлу .pub на управляющей машине, не на сервере.

## 2. Настройте значения и секреты

### Обычные параметры

Редактируйте inventory/group_vars/all/vars.yml. Значения из этого файла применяются ко всем серверам. Для отдельных узлов можно создать host_vars/<имя_хоста>.yml или переопределить переменные через командную строку.

Основные группы настроек:

| Переменные | Назначение |
|---|---|
| bootstrap_admin_user, bootstrap_admin_groups, bootstrap_admin_shell | Имя и группы нового администратора. Поменяйте демонстрационное значение test на своё |
| bootstrap_admin_passwordless_sudo | Разрешает sudo без пароля; включено по умолчанию |
| bootstrap_ssh_public_key_path | Путь к публичному SSH-ключу на управляющей машине |
| bootstrap_timezone, bootstrap_locale, bootstrap_base_packages | Часовой пояс, локаль и базовый набор пакетов |
| security_ssh_port, security_ssh_password_authentication, security_ssh_permit_root_login, security_ssh_max_auth_tries | Политика SSH |
| security_fail2ban_bantime, security_fail2ban_findtime, security_fail2ban_maxretry | Порог блокировки Fail2ban |
| security_ufw_allowed_tcp_ports | Разрешённые входящие TCP-порты. По умолчанию 22, 80 и 443 |
| docker_admin_users, docker_daemon_log_max_size, docker_daemon_log_max_file, docker_daemon_live_restore | Пользователи Docker и настройки daemon |
| wireguard_interface, wireguard_address, wireguard_listen_port, wireguard_peers | Интерфейс и узлы VPN |
| nginx_proxy_sites, nginx_proxy_webroot | Сайты reverse proxy, upstream и webroot ACME |
| node_exporter_version, node_exporter_port, node_exporter_listen_address | Версия и адрес прослушивания Node Exporter |
| vector_sink_url, vector_data_dir | HTTPS endpoint и каталог данных Vector |
| postgres_database, postgres_role, postgres_shared_buffers, postgres_effective_cache_size, postgres_work_mem, postgres_maintenance_work_mem, postgres_max_connections | Имя базы/роли и параметры PostgreSQL |
| db_backup_s3_bucket, db_backup_aws_region, db_backup_s3_prefix, db_backup_schedule | Параметры назначения и расписание S3 backup |
| db_backup_local_dir, db_backup_retention_days, db_backup_zstd_level | Локальное хранение и сжатие backup |
| maintenance_reboot_if_required, maintenance_reboot_timeout | Автоматическая перезагрузка после обновлений |
| disk_journal_vacuum_time, docker_prune_until | Сроки очистки журнала и старых Docker-объектов |

Значения по умолчанию — стартовые, а не универсальный тюнинг для любого сервера. В частности, подберите PostgreSQL memory settings под доступную RAM. Для Node Exporter по умолчанию используется 127.0.0.1:9100; не открывайте его в публичную сеть без ограничения доступа.

### Секреты Ansible Vault

В репозитории уже есть зашифрованный файл inventory/group_vars/all/vault.yml. Отредактируйте его локально:

~~~bash
ansible-vault edit inventory/group_vars/all/vault.yml
~~~

В нём должны быть заданы переменные, которые используются playbook:

~~~yaml
vault_bootstrap_admin_password_hash: "ХЕШ_ПАРОЛЯ_АДМИНИСТРАТОРА"
vault_postgres_app_password: "СЛОЖНЫЙ_ПАРОЛЬ_POSTGRES"
vault_vector_sink_auth_token: "ТОКЕН_HTTPS_SINK"
~~~

Для администратора нужен хеш пароля, не открытый пароль. Например, SHA-512-хеш можно создать командой:

~~~bash
openssl passwd -6
~~~

Храните пароль Vault отдельно от репозитория. Запускайте команды с --ask-vault-pass, чтобы Ansible запросил его интерактивно. Не помещайте пароль Vault, API-токены, AWS access keys или приватные ключи в inventory, README и обычные vars.yml.

Для S3 предпочтительна IAM role виртуальной машины или задачи контейнера с минимальными правами на запись в используемый bucket/prefix. Playbook не требует статических AWS-ключей.

## 3. Первичная настройка сервера

Первый playbook создаёт отдельного администратора. До запуска проверьте имя пользователя, парольный хеш в Vault и путь к публичному ключу:

~~~bash
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass --check --diff
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass
~~~

После запуска проверьте вход новым пользователем. Затем обновите inventory/hosts.ini, указав нового пользователя в ansible_user, и проверьте SSH-доступ:

~~~bash
ansible all -m ansible.builtin.ping --limit server1
~~~

Только после проверки запустите SSH hardening и firewall:

~~~bash
ansible-playbook playbooks/bootstrap/security.yml --limit server1 --check --diff
ansible-playbook playbooks/bootstrap/security.yml --limit server1
~~~

Security playbook включает UFW и оставляет входящими только порты из security_ufw_allowed_tcp_ports. Если SSH работает, например, на порту 2222, измените одновременно security_ssh_port и список разрешённых TCP-портов. Добавьте порты приложения до запуска, иначе firewall закроет к ним вход. SSH-порт проверяется до включения UFW.

## 4. Запуск остальных playbook

Проверьте изменение в check mode, затем примените его на одном сервере. Для секретов используйте --ask-vault-pass:

~~~bash
ansible-playbook playbooks/docker/docker_engine.yml --limit server1 --check --diff
ansible-playbook playbooks/docker/docker_engine.yml --limit server1
~~~

Если playbook использует значения из Vault, добавьте --ask-vault-pass к обеим командам. Можно указать несколько серверов или группу после того, как проверили выполнение на одном узле:

~~~bash
ansible-playbook playbooks/maintenance/update_all.yml --limit 'server1,server2' --ask-vault-pass
~~~

Чтобы передать разовую настройку без изменения vars.yml, используйте -e:

~~~bash
ansible-playbook playbooks/observability/node_exporter.yml --limit server1 -e node_exporter_listen_address=10.0.0.10
~~~

Для просмотра задач и тегов:

~~~bash
ansible-playbook playbooks/network/nginx_proxy.yml --list-tasks
ansible-playbook playbooks/network/nginx_proxy.yml --list-tags
~~~

## Настройка сервисов

### WireGuard

На каждом узле задайте wireguard_address и список wireguard_peers. У каждого peer нужны уникальные public_key и allowed_ips; endpoint и persistent_keepalive задаются, когда это требуется топологией. Приватный ключ интерфейса создаётся на сервере при первом запуске, сохраняется в /etc/wireguard/ с правами root-only и не попадает в Git. WireGuard использует UDP-порт wireguard_listen_port; убедитесь, что он открыт также во внешнем firewall провайдера.

Пример peer:

~~~yaml
wireguard_peers:
  - public_key: "ПУБЛИЧНЫЙ_КЛЮЧ_PEER"
    allowed_ips: "10.10.0.2/32"
    endpoint: "vpn-peer.example.net:51820"
    persistent_keepalive: 25
~~~

### NGINX и Let's Encrypt

Для каждого домена добавьте запись в nginx_proxy_sites. DNS должен указывать на сервер; порты 80 и 443 должны быть доступны снаружи. HTTP используется для проверки ACME, затем NGINX перенаправляет запросы на HTTPS и проксирует их на upstream.

~~~yaml
nginx_proxy_sites:
  - server_name: app.example.net
    upstream_url: http://127.0.0.1:3000
    email: ops@example.net
~~~

### Docker

Docker log rotation по умолчанию ограничена размером 50 MB и пятью файлами. Пользователи из docker_admin_users фактически получают root-полномочия на хосте. Также опубликованные порты Docker могут обходить правила UFW; ограничивайте публикацию портов и отдельно контролируйте контейнерный трафик.

### Node Exporter и Vector

Node Exporter использует официальный релизный архив и SHA-256 checksum, отдельного системного пользователя и systemd unit. Если Prometheus опрашивает его по сети, установите приватный адрес прослушивания и ограничьте доступ сетевым firewall.

Vector сначала требует Docker Engine. Укажите HTTPS URL в vector_sink_url и токен в Vault. Для чтения Docker-логов пользователь Vector добавляется в группу docker, которая обладает широкими полномочиями на хосте.

### PostgreSQL и S3

Перед PostgreSQL playbook задайте vault_postgres_app_password. Память и число соединений регулируются через переменные postgres_* в vars.yml.

Для backup задайте db_backup_s3_bucket, db_backup_aws_region и при необходимости db_backup_s3_prefix. Расписание по умолчанию — ежедневно в 02:30 по времени сервера. Сервис делает pg_dump через локальное подключение, сжимает результат zstd, отправляет его с SSE-S3 и хранит локальные копии ограниченное число дней. Включается systemd timer; проверьте после запуска его состояние и успешность первой резервной копии. Восстановление не автоматизировано — периодически выполняйте пробное восстановление отдельно.

### Обновления и очистка

update_all.yml выполняет dist-upgrade по одному серверу за раз и по умолчанию перезагружает систему, если обнаружен /var/run/reboot-required. При необходимости отключите это через maintenance_reboot_if_required.

disk_cleanup.yml сокращает systemd journal, удаляет старые остановленные контейнеры и неиспользуемые Docker images/build cache, затем очищает кэш APT и удаляет ненужные пакеты. Docker volumes не удаляются, чтобы не потерять данные приложений.

## Практические команды

Проверить inventory и доступность:

~~~bash
ansible-inventory --graph
ansible all -m ansible.builtin.ping
~~~

Проверить синтаксис отдельного playbook и план изменений:

~~~bash
ansible-playbook playbooks/database/postgres_standalone.yml --syntax-check
ansible-playbook playbooks/database/postgres_standalone.yml --limit server1 --check --diff --ask-vault-pass
~~~

Применяйте firewall, SSH, PostgreSQL, WireGuard и обслуживание сначала на одном тестовом хосте. Check mode не может полностью предсказать внешние эффекты Certbot, firewall, systemd и перезагрузки; изучайте diff и проверяйте сервисы после запуска.
