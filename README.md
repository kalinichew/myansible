# myansible

В репозитории собраны Ansible playbook для первичной настройки и обслуживания серверов Debian/Ubuntu. Каждый playbook решает отдельную задачу — выберите нужный сценарий по описанию и запустите его для своих серверов.

> [!WARNING]
> Некоторые сценарии меняют SSH и firewall, очищают данные, перезапускают службы или перезагружают сервер. Сначала проверьте target и переменные, запускайте на одном тестовом хосте через --limit. Сохраняйте текущую SSH-сессию до проверки нового входа.

## Содержание

- [Настройка окружения](#настройка-окружения)
- [Общие переменные и секреты](#общие-переменные-и-секреты)
- [Playbook для Linux](#playbook-для-linux)
- [Проверка, диагностика и известные ограничения](#проверка-диагностика-и-известные-ограничения)

## Настройка окружения

### 1. Подготовьте управляющую машину

Для серверных playbook используйте Linux или WSL с Python и SSH. Из корня репозитория создайте виртуальную среду и установите Ansible:

~~~bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install ansible
ansible-galaxy collection install -r collections/requirements.yml
~~~

Если команда создания виртуальной среды недоступна, установите пакет Python venv для своего дистрибутива.

### 2. Создайте inventory для серверов

Скопируйте пример:

~~~bash
cp inventory/hosts.ini.example inventory/hosts.ini
~~~

В inventory/hosts.ini укажите IP-адрес или DNS-имя сервера и SSH-пользователя. Файл исключён из Git. Пример использует root только для первоначального bootstrap.

Проверьте соединение:

~~~bash
ansible-inventory --graph
ansible all -m ansible.builtin.ping --limit server1
~~~

Имя server1 ниже — пример alias из inventory; замените его своим. Если SSH-ключ хранится не по пути из примера, задайте фактический путь к публичному .pub файлу в inventory/group_vars/all/vars.yml.

### 3. Подготовьте секреты и зашифруйте Vault

Playbook создаёт обычную учётную запись пользователя и добавляет её в группу sudo. Команды выполняются с обычными правами; для действий от root пользователь явно запускает sudo, который запрашивает его пароль. Файл sudoers с правилом NOPASSWD не создаётся.

Для этой настройки важны три значения:

- Пароль создаваемого пользователя задаётся в Linux в виде SHA-512 хэша. Хэш нужен системе для проверки пароля при sudo и входе.
- Ansible при подключении к серверу под обычным пользователем может автоматически передать sudo его исходный пароль через ansible_become_password. Для этого нужен пароль в открытом виде внутри Vault; хэш сюда не подходит. Обычно это тот же пароль, из которого был получен хэш для нового пользователя.
- Пароль Vault — отдельный пароль, который расшифровывает файл с секретами. Он не является ни паролем пользователя, ни его хэшем.

#### Шаг 1. Создайте файл из шаблона

~~~bash
cp inventory/group_vars/all/vault.example.yml inventory/group_vars/all/vault.yml
~~~

До шифрования файл — обычный YAML. Заполните в нём хэш нового пользователя. Если позже Ansible будет подключаться под ним и должен вводить sudo-пароль автоматически, добавьте также исходный пароль в vault_ansible_become_password:

~~~yaml
---
vault_bootstrap_user_password_hash: "$6$ВСТАВЬТЕ_SHA512_ХЭШ"
vault_ansible_become_password: "ИСХОДНЫЙ_ПАРОЛЬ_ПОЛЬЗОВАТЕЛЯ"
~~~

При первом запуске из примера inventory используется root с ansible_become=false, поэтому vault_ansible_become_password для этого запуска не требуется. Не храните незашифрованный vault.yml в Git.

#### Шаг 2. Создайте хэш пароля пользователя

Придумайте пароль для нового пользователя и передайте его команде. Введите пароль в её запрос:

~~~bash
openssl passwd -6
~~~

Команда выведет SHA-512 хэш, начинающийся с $6$. Скопируйте всю строку в vault_bootstrap_user_password_hash. Этот хэш Ansible установит пользователю; вручную вводить его при sudo не нужно.

Если Ansible в дальнейшем будет подключаться под этим пользователем и должен сам выполнять sudo, сохраните исходный пароль, который вы ввели в openssl, в vault_ansible_become_password. Это отдельное значение: в него вставляется сам пароль, не хэш. Если не хотите хранить исходный пароль даже в зашифрованном Vault, не заполняйте эту переменную и используйте -K при запуске Ansible, чтобы вводить sudo-пароль вручную.

#### Шаг 3. Зашифруйте файл

~~~bash
ansible-vault encrypt inventory/group_vars/all/vault.yml
~~~

Ansible запросит пароль Vault. Сохраните его отдельно в менеджере паролей. Команда зашифрует весь файл, включая исходный sudo-пароль и хэш пользователя. При запуске Ansible расшифрует его в памяти после ввода пароля Vault.

Чтобы изменить секреты после шифрования, используйте:

~~~bash
ansible-vault edit inventory/group_vars/all/vault.yml
~~~

Команда откроет расшифрованное содержимое в редакторе и зашифрует его снова после сохранения.

#### Шаг 4. Создайте пользователя и проверьте доступ

Первый запуск из примера выполняется через root и не требует sudo-пароля:

~~~bash
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass
~~~

Введите пароль Vault. Playbook создаст пользователя с SSH-ключом и членством в группе sudo. После запуска проверьте вход по SSH под новым пользователем. Команды, требующие root, запускайте через sudo; sudo запросит пароль этого пользователя.

Перед последующим подключением Ansible под новым пользователем задайте его в inventory. Если vault_ansible_become_password заполнен, достаточно --ask-vault-pass: Ansible сам передаст пароль sudo. Если он не заполнен, добавьте -K и введите пароль вручную.

В этом репозитории vault.yml исключён из Git. В репозитории должен оставаться только vault.example.yml с пустыми или примерными значениями.

### 4. Выберите сервер

Linux-хосты задаются в inventory. Большинство серверных playbook используют hosts: all, поэтому ограничивайте запуск через --limit и сначала проверяйте его на одном тестовом сервере.

## Общие переменные и секреты

Общие настройки серверов лежат в inventory/group_vars/all/vars.yml. Секреты задаются в inventory/group_vars/all/vault.yml.

Чтобы настроить отдельный сервер, добавьте host_vars. Не помещайте туда пароли или приватные ключи.

| Переменные | Используются в |
|---|---|
| bootstrap_user_*, bootstrap_ssh_public_key_path, bootstrap_timezone, bootstrap_locale, ansible_become_password | Создание пользователя и sudo |
| security_ssh_*, security_ufw_allowed_tcp_ports, security_fail2ban_* | SSH, Fail2ban и UFW |
| docker_group_users, docker_daemon_* | Docker |
| wireguard_interface, wireguard_address, wireguard_listen_port, wireguard_peers | WireGuard |
| nginx_proxy_sites, nginx_proxy_webroot | NGINX и Let's Encrypt |
| vps_hostname, vps_timezone, xray_port, xray_sni, xray_dest, xray_public_host, xray_update | Xray VLESS + Reality |
| node_exporter_version, node_exporter_port, node_exporter_listen_address, node_exporter_sha256 | Node Exporter |
| vector_sink_url, vector_data_dir | Vector; токен берётся из Vault |

## Playbook для Linux

В командах ниже используется inventory, настроенный в ansible.cfg. Все серверные playbook рассчитаны на Debian и Ubuntu.

### Первичная настройка сервера

Название каталога bootstrap означает начальную подготовку сервера: создание пользователя с доступом через sudo и установку базовых настроек перед дальнейшей работой.

#### playbooks/bootstrap/system.yml

Создаёт пользователя, добавляет его в группу sudo, добавляет его SSH-ключ, устанавливает базовые пакеты и настраивает часовой пояс и локаль.

Перед запуском укажите bootstrap_user_name, путь bootstrap_ssh_public_key_path и Vault-переменную vault_bootstrap_user_password_hash. Имя по умолчанию operator — замените его своим.

~~~bash
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass --check --diff
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass
~~~

Playbook требует права root. Если подключаетесь по SSH как root, укажите в inventory `ansible_become=false`: root уже имеет нужные права, запуск sudo не требуется. Эта настройка добавлена в inventory/hosts.ini.example. Если подключаетесь обычным пользователем с sudo, можно ввести пароль вручную через `-K` (или `--ask-become-pass`). Чтобы Ansible брал его из Vault, добавьте пароль в `vault_ansible_become_password` и запускайте с `--ask-vault-pass`; переменная `ansible_become_password` уже связана с ним в vars.yml. Для этого SSH-пользователь должен иметь право sudo. Пользователь добавляется в группу sudo; стандартная настройка Debian/Ubuntu запрашивает пароль при повышении прав.

После применения проверьте вход под новым пользователем, затем измените ansible_user в inventory и проверьте ping. Не переходите к SSH hardening, пока новый вход не подтверждён.

#### playbooks/bootstrap/security.yml

Настраивает SSH drop-in, выключает вход по паролю и root-вход согласно переменным, устанавливает Fail2ban, настраивает UFW. По умолчанию UFW запрещает входящие подключения, кроме перечисленных TCP-портов, и разрешает исходящий трафик.

Перед запуском проверьте фактический SSH-порт. Он должен совпадать с security_ssh_port и присутствовать в security_ufw_allowed_tcp_ports. Добавьте также все необходимые приложению порты. Провайдерский firewall настраивается отдельно.

~~~bash
ansible-playbook playbooks/bootstrap/security.yml --limit server1 --check --diff
ansible-playbook playbooks/bootstrap/security.yml --limit server1
~~~

### Контейнеры

#### playbooks/docker/docker_engine.yml

Подключает официальный APT-репозиторий Docker, устанавливает Docker Engine CE и Compose plugin, задаёт ротацию JSON-логов. Добавляет пользователей из docker_group_users в группу docker.

~~~bash
ansible-playbook playbooks/docker/docker_engine.yml --limit server1
~~~

По умолчанию Docker-пользователи не добавляются в группу docker. Членство в ней даёт полномочия уровня root напрямую, поэтому задавайте docker_group_users только осознанно. Для нестандартных архитектур проверьте список поддерживаемых архитектур в playbook.

### Сеть и reverse proxy

#### playbooks/network/wireguard.yml

Создаёт WireGuard-интерфейс, генерирует приватный ключ на целевом хосте и настраивает peers. Обязательные поля каждого peer — public_key и allowed_ips; endpoint и persistent_keepalive можно задать дополнительно.

Настройте wireguard_address, wireguard_listen_port и wireguard_peers, затем запустите:

~~~bash
ansible-playbook playbooks/network/wireguard.yml --limit server1
~~~

Playbook добавляет правило для UDP-порта WireGuard в UFW, но не включает UFW и не настраивает маршрутизацию/NAT между сетями. Это конфигурация mesh-интерфейса; для gateway-сценария отдельно нужны IP forwarding, маршруты и соответствующие firewall-правила. Откройте UDP-порт также во внешнем firewall VPS.

#### playbooks/network/nginx_proxy.yml

Устанавливает NGINX и Certbot, запрашивает сертификаты Let's Encrypt через webroot, включает HTTP-to-HTTPS redirect и reverse proxy для каждого сайта. Укажите список nginx_proxy_sites с полями server_name, upstream_url и email. DNS каждого домена должен указывать на сервер, а TCP-порты 80 и 443 должны быть доступны извне.

~~~bash
ansible-playbook playbooks/network/nginx_proxy.yml --limit server1
~~~

Сценарий удаляет стандартный enabled-site NGINX и записывает собственные конфиги в /etc/nginx/sites-enabled. Он не управляет UFW: проверьте firewall отдельно. Certbot timer отвечает за продление сертификатов; deploy hook перезагружает NGINX после обновления.

#### playbooks/network/xray_reality.yml

Устанавливает Xray-core официальным upstream-установщиком, настраивает VLESS + Reality, задаёт hostname и часовой пояс, включает BBR и выводит клиентскую ссылку. При первом запуске создаются UUID, X25519-ключи и Short ID; учетные данные сохраняются на сервере в /etc/xray/credentials.yml с правами 0600. Повторный запуск использует сохранённые значения.

Настройте vps_hostname, vps_timezone, xray_port, xray_sni, xray_dest. Для ссылки с конкретным адресом задайте xray_public_host; если он пустой, используется ansible_host из inventory.

~~~bash
ansible-playbook playbooks/network/xray_reality.yml --syntax-check
ansible-playbook playbooks/network/xray_reality.yml --limit server1
~~~

Ссылка подключения выводится в конце playbook. Для обновления Xray запустите его с xray_update=true:

~~~bash
ansible-playbook playbooks/network/xray_reality.yml --limit server1 --tags xray -e xray_update=true
~~~

> [!CAUTION]
> Этот playbook включает UFW с запрещающей входящие политикой и открывает только security_ssh_port и xray_port. Он не учитывает security_ufw_allowed_tcp_ports. На сервере с NGINX или другими приложениями это может закрыть дополнительные порты, включая 80/tcp. Перед запуском добавьте нужные правила или выделите Xray отдельный сервер. Установщик скачивается из изменяемой ветки и запускается от root без закрепления версии/checksum; для воспроизводимого production-развёртывания его следует закрепить.

### Мониторинг и журналы

#### playbooks/observability/node_exporter.yml

Загружает Node Exporter заданной версии, проверяет SHA-256 checksum, создаёт отдельного системного пользователя и systemd-службу с ограничениями безопасности. Поддерживаются только архитектуры, указанные в проверке playbook; checksum версии задаются в node_exporter_sha256.

По умолчанию метрики слушают только 127.0.0.1:9100. Для удалённого Prometheus задайте node_exporter_listen_address и ограничьте доступ firewall доверенной сетью.

~~~bash
ansible-playbook playbooks/observability/node_exporter.yml --limit server1
~~~

#### playbooks/observability/vector_shipper.yml

Устанавливает Vector, настраивает источники journald и Docker logs, отправляет события в HTTP sink. Требует, чтобы Docker уже был установлен и существовал /var/run/docker.sock. Задайте HTTPS endpoint в vector_sink_url, токен — в Vault как vault_vector_sink_auth_token.

~~~bash
ansible-playbook playbooks/observability/vector_shipper.yml --limit server1 --ask-vault-pass
~~~

Vector добавляется в группы systemd-journal и docker для чтения журналов. Доступ к группе docker даёт широкие полномочия; запускайте эту схему только на доверенных серверах.

### Обслуживание и диагностика

#### playbooks/maintenance/update_all.yml

Обновляет APT-пакеты и может перезагрузить сервер, если присутствует /var/run/reboot-required. Хосты обрабатываются по одному. Для отключения автоматической перезагрузки задайте maintenance_reboot_if_required: false.

~~~bash
ansible-playbook playbooks/maintenance/update_all.yml --limit server1
~~~

#### playbooks/maintenance/disk_cleanup.yml

Очищает старые записи systemd journal по disk_journal_vacuum_time, удаляет остановленные Docker containers, старые unused images и build cache, затем выполняет APT autoremove/purge и чистит кэш пакетов. Docker volumes не удаляются.

Настройте docker_prune_until и disk_journal_vacuum_time перед запуском. Команда может удалить образы, которые потребуется загрузить повторно.

~~~bash
ansible-playbook playbooks/maintenance/disk_cleanup.yml --limit server1
~~~

#### playbooks/diagnostics/host_report.yml

Только читает состояние сервера: filesystem capacity, память/swap, failed systemd units, слушающие TCP/UDP sockets и признак обязательной перезагрузки. В отчёте сокетов могут быть видны имена процессов.

~~~bash
ansible-playbook playbooks/diagnostics/host_report.yml --limit server1
~~~

## Проверка, диагностика и известные ограничения

Сначала проверяйте синтаксис конкретного playbook:

~~~bash
ansible-playbook playbooks/<путь>/<имя>.yml --syntax-check
~~~

Для обычных Ansible-модулей используйте --check --diff только там, где сценарий и модули поддерживают check mode. Некоторые playbook содержат command, генерацию credentials или операции Certbot; check mode не покажет полный результат и может быть неприменим при первом запуске. Удалённые сценарии с systemd, UFW и apt проверяйте на одном тестовом сервере.

Для общего отчёта Linux запустите playbooks/diagnostics/host_report.yml. Состояние конкретной службы можно проверить на сервере через systemctl status и journalctl.

Известные ограничения текущей реализации:

- Xray playbook включает UFW с ограниченным списком разрешённых портов и может закрыть порты других приложений.
- Xray installer не закреплён по версии и checksum.
- Docker-группа и доступ Vector к Docker socket дают широкие права на хосте.
- WireGuard playbook настраивает mesh-интерфейс, но не IP forwarding или NAT.

## Файлы проекта

~~~text
ansible.cfg
collections/requirements.yml
inventory/
  hosts.ini.example
  group_vars/all/
    vars.yml
    vault.example.yml
playbooks/
  bootstrap/       system.yml, security.yml
  diagnostics/     host_report.yml
  docker/          docker_engine.yml
  maintenance/     update_all.yml, disk_cleanup.yml
  network/         wireguard.yml, nginx_proxy.yml, xray_reality.yml
  observability/   node_exporter.yml, vector_shipper.yml
roles/xray/templates/config.json.j2
~~~

