# myansible

В репозитории собраны отдельные Ansible playbook для подготовки и обслуживания серверов Debian/Ubuntu. Настройка ОС и управление пользователями разделены: при необходимости можно запустить только system.yml.

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

### 3. Настройте Vault, если он нужен выбранному playbook

Подготовка ОС сама по себе не требует Vault: если вы подключаетесь по SSH как root и в inventory стоит ansible_become=false, достаточно запустить system.yml. Vault нужен, когда playbook использует секреты — например, при создании пользователя с паролем или при автоматическом вводе sudo-пароля Ansible.

Здесь важно различать три значения:

- Хэш пароля пользователя — односторонний SHA-512 хэш, который Linux сохраняет для проверки пароля. Он задаётся в vault_managed_user_password_hashes.
- Пароль пользователя для sudo — исходный пароль. Если Ansible подключается под этим пользователем и должен сам выполнить sudo, этот пароль задаётся в vault_ansible_become_password. Хэш вместо него не подойдёт. Обычно это пароль SSH-пользователя, который вводят при sudo.
- Пароль Vault — отдельный пароль, которым Ansible расшифровывает весь файл vault.yml во время запуска.

#### Шаг 1. Создайте рабочий файл Vault из шаблона

~~~bash
cp inventory/group_vars/all/vault.example.yml inventory/group_vars/all/vault.yml
~~~

До шифрования файл — обычный YAML. Если вы создаёте пользователя, которому нужно sudo, сгенерируйте его парольный хэш командой ниже и добавьте его в словарь по имени пользователя:

~~~yaml
vault_managed_user_password_hashes:
  operator: "$6$ВСТАВЬТЕ_SHA512_ХЭШ"
~~~

Если позже Ansible будет подключаться как operator и должен автоматически выполнять sudo, сохраните исходный пароль этого пользователя отдельно:

~~~yaml
vault_ansible_become_password: "ИСХОДНЫЙ_ПАРОЛЬ_OPERATOR"
~~~

Если Ansible подключается как root с ansible_become=false или вы готовы вводить sudo-пароль вручную через -K, vault_ansible_become_password не нужен.

#### Шаг 2. Получите хэш пароля

Придумайте пароль пользователя и введите его в команду:

~~~bash
openssl passwd -6
~~~

Скопируйте выведенную строку целиком в vault_managed_user_password_hashes под соответствующим именем. Например, хэш для пользователя operator должен быть доступен в словаре по ключу operator. Исходный пароль сохраните для самого пользователя, чтобы он мог вводить его при sudo.

Хэш — не шифрование Vault. Команда создаёт значение, которое Linux использует для проверки пароля; из него нельзя получить исходный пароль. Если Ansible должен сам выполнять sudo, ему отдельно нужен исходный пароль в vault_ansible_become_password.

#### Шаг 3. Зашифруйте файл Vault

~~~bash
ansible-vault encrypt inventory/group_vars/all/vault.yml
~~~

Ansible запросит пароль Vault. Сохраните его отдельно. Команда шифрует файл целиком — в том числе словарь с хэшами и исходный sudo-пароль. Перед запуском Ansible расшифрует файл в памяти после ввода пароля Vault.

Чтобы изменить секреты после шифрования, выполните:

~~~bash
ansible-vault edit inventory/group_vars/all/vault.yml
~~~

Ansible откроет расшифрованное содержимое в редакторе и зашифрует файл снова после сохранения. В этом репозитории vault.yml исключён из Git; храните в Git только vault.example.yml с примерными значениями.

### 4. Выберите сервер

Linux-хосты задаются в inventory. Большинство серверных playbook используют hosts: all, поэтому ограничивайте запуск через --limit и сначала проверяйте его на одном тестовом сервере.

## Общие переменные и секреты

Общие настройки серверов лежат в inventory/group_vars/all/vars.yml. Секреты задаются в inventory/group_vars/all/vault.yml.

Чтобы настроить отдельный сервер, добавьте host_vars. Не помещайте туда пароли или приватные ключи.

| Переменные | Используются в |
|---|---|
| bootstrap_timezone, bootstrap_locale, bootstrap_base_packages | Подготовка ОС |
| managed_users, vault_managed_user_password_hashes, ansible_become_password | Пользователи и sudo |
| security_ssh_*, security_ufw_allowed_tcp_ports, security_fail2ban_* | SSH, Fail2ban и UFW |
| docker_group_users, docker_daemon_* | Docker |
| wireguard_interface, wireguard_address, wireguard_listen_port, wireguard_peers | WireGuard |
| nginx_proxy_sites, nginx_proxy_webroot | NGINX и Let's Encrypt |
| vps_hostname, vps_timezone, xray_port, xray_sni, xray_dest, xray_public_host, xray_update | Xray VLESS + Reality |
| node_exporter_version, node_exporter_port, node_exporter_listen_address, node_exporter_sha256 | Node Exporter |
| vector_sink_url, vector_data_dir | Vector; токен берётся из Vault |

## Playbook для Linux

В командах ниже используется inventory, настроенный в ansible.cfg. Все серверные playbook рассчитаны на Debian и Ubuntu.

### Подготовка операционной системы

Название каталога bootstrap обозначает начальную настройку ОС. Этот playbook не создаёт пользователей и не требует пользовательских паролей или Vault. Он обновляет кэш APT, устанавливает базовые пакеты, настраивает часовой пояс и локаль.

#### playbooks/bootstrap/system.yml

Запускайте его на чистой Debian/Ubuntu системе, подключившись под root с ansible_become=false, как указано в inventory/hosts.ini.example:

~~~bash
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --check --diff
ansible-playbook playbooks/bootstrap/system.yml --limit server1
~~~

Этот сценарий можно использовать отдельно, когда нужно подготовить ОС, но не создавать пользователей.

### Управление пользователями

#### playbooks/users/manage_users.yml

Создаёт и обновляет пользователей из списка managed_users в inventory/group_vars/all/vars.yml. У каждого пользователя можно задать shell, дополнительные группы, парольный хэш и несколько путей к SSH public key файлам на управляющей машине. Пользователь с группой sudo получает стандартный sudo с запросом своего пароля; парольless правило не создаётся. Для sudo-группы обязательно задайте password_hash из Vault.

Пример настройки в vars.yml:

~~~yaml
managed_users:
  - name: operator
    shell: /bin/bash
    groups:
      - sudo
    password_hash: "{{ vault_managed_user_password_hashes['operator'] }}"
    ssh_public_key_paths:
      - /home/me/.ssh/id_ed25519.pub
  - name: deploy
    groups: []
    ssh_public_key_paths:
      - /home/me/.ssh/deploy.pub
~~~

Если список managed_users пуст, playbook не меняет пользователей. Для создания или обновления заданных учётных записей запустите:

~~~bash
ansible-playbook playbooks/users/manage_users.yml --limit server1 --ask-vault-pass
~~~

При запуске от root укажите ansible_become=false в inventory. Если подключаетесь обычным sudo-пользователем, используйте сохранённый в Vault vault_ansible_become_password или добавьте -K для ручного ввода sudo-пароля. Не добавляйте пользователя в группу docker, если ему не нужны root-подобные полномочия без sudo.

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

