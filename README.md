# myansible

В репозитории собраны Ansible playbook для настройки и обслуживания серверов Debian/Ubuntu, а также для установки и проверки сервисов в Kubernetes. Выберите нужный сценарий по описанию ниже и запустите его для своих серверов или кластера.

> [!WARNING]
> Некоторые сценарии меняют SSH и firewall, очищают данные, перезапускают службы или перезагружают сервер. Сначала проверьте target и переменные, запускайте на одном тестовом хосте через --limit. Сохраняйте текущую SSH-сессию до проверки нового входа.

## Содержание

- [Настройка окружения](#однократная-подготовка)
- [Общие переменные и секреты](#общие-переменные-и-секреты)
- [Playbook для Linux](#playbook-для-linux)
- [Playbook для Kubernetes](#playbook-для-kubernetes)
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

Для Kubernetes playbook дополнительно нужны Helm CLI, kubectl и Python-библиотеки Kubernetes/PyYAML:

~~~bash
python -m pip install kubernetes PyYAML
helm version
kubectl config current-context
kubectl get nodes
~~~

Kubernetes-сценарии работают с уже существующим кластером через kubeconfig. Они не создают control plane, ingress или внешний LoadBalancer.

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

### 3. Создайте и зашифруйте файл Vault

Vault — это файл с секретами, зашифрованный Ansible. В открытом виде секреты лежат в inventory/group_vars/all/vault.yml и автоматически доступны playbook’ам как переменные. После шифрования файл можно хранить в репозитории; пароль для его расшифровки храните отдельно и не добавляйте в Git.

#### Шаг 1. Создайте файл из шаблона

~~~bash
cp inventory/group_vars/all/vault.example.yml inventory/group_vars/all/vault.yml
~~~

В шаблоне перечислены все поддерживаемые секреты. Заполняйте только те, которые нужны запускаемому playbook’у. Например, минимальный файл для первичной настройки сервера:

~~~yaml
---
vault_bootstrap_admin_password_hash: "$6$ВСТАВЬТЕ_СЮДА_ХЭШ"
~~~

Для других сценариев в этот же YAML-файл добавляются нужные значения:

~~~yaml
vault_postgres_app_password: "ПАРОЛЬ_ПОЛЬЗОВАТЕЛЯ_БАЗЫ"
vault_vector_sink_auth_token: "ТОКЕН_ВАШЕГО_SINK"
vault_awx_admin_password: "ПАРОЛЬ_АДМИНИСТРАТОРА_AWX"
vault_awx_secret_key: "СЛУЧАЙНЫЙ_СЕКРЕТНЫЙ_КЛЮЧ_НЕ_КОРОЧЕ_32_СИМВОЛОВ"
vault_jenkins_admin_password: "ПАРОЛЬ_АДМИНИСТРАТОРА_JENKINS"
~~~

Замените примеры настоящими секретами. Не оставляйте строки REPLACE_... из шаблона. Указывайте строки в кавычках и не публикуйте незашифрованный vault.yml.

#### Шаг 2. Получите хэш пароля администратора

Для bootstrap playbook в Vault записывается не пароль, а его SHA-512 хэш. Выполните команду и введите пароль по запросу:

~~~bash
openssl passwd -6
~~~

Команда выведет строку, начинающуюся с $6$. Скопируйте всю строку целиком и вставьте её вместо "$6$ВСТАВЬТЕ_СЮДА_ХЭШ" в значение vault_bootstrap_admin_password_hash. Сам пароль в YAML не вставляйте. Этот хэш используется для пользователя, создаваемого playbook’ом system.yml.

#### Шаг 3. Зашифруйте файл

~~~bash
ansible-vault encrypt inventory/group_vars/all/vault.yml
~~~

Ansible попросит придумать пароль шифрования (Vault password). Сохраните его в менеджере паролей: без него Ansible не сможет прочитать секреты. Это отдельный пароль, он не совпадает автоматически с паролем администратора.

#### Шаг 4. Запустите нужный playbook

Добавьте --ask-vault-pass; Ansible запросит пароль шифрования и расшифрует переменные только во время запуска. Например:

~~~bash
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass
~~~

Если playbook не использует Vault-переменные, этот параметр не нужен. Перед коммитом убедитесь, что в Git попал только vault.example.yml, а зашифрованный vault.yml не добавлен: проверьте git status. Если нужно изменить секреты позже, сначала выполните ansible-vault edit inventory/group_vars/all/vault.yml; Ansible откроет расшифрованное содержимое во временном редакторе и сохранит файл снова зашифрованным.

### 4. Выберите область работы

- Linux-хосты задаются в inventory. Большинство серверных playbook используют hosts: all, поэтому ограничивайте запуск через --limit.
- Kubernetes playbook запускаются локально на управляющей машине и используют kubeconfig. Проверьте текущий контекст, namespace и права пользователя до запуска.
- Для AWX, Kafka и Jenkins storage class должен существовать заранее. Хранилища кластера и сам Kubernetes-кластер playbook не создаёт.

## Общие переменные и секреты

Общие настройки серверов лежат в inventory/group_vars/all/vars.yml, а сервисы Kubernetes — в inventory/group_vars/all/devops.yml. Ansible автоматически загружает оба файла. Секреты хранятся отдельно в зашифрованном inventory/group_vars/all/vault.yml.

Чтобы настроить отдельный сервер, добавьте host_vars. Не помещайте туда пароли или приватные ключи.

| Переменные | Используются в |
|---|---|
| bootstrap_admin_*, bootstrap_ssh_public_key_path, bootstrap_timezone, bootstrap_locale | Первичная настройка |
| security_ssh_*, security_ufw_allowed_tcp_ports, security_fail2ban_* | SSH, Fail2ban и UFW |
| docker_admin_users, docker_daemon_* | Docker |
| wireguard_interface, wireguard_address, wireguard_listen_port, wireguard_peers | WireGuard |
| nginx_proxy_sites, nginx_proxy_webroot | NGINX и Let's Encrypt |
| vps_hostname, vps_timezone, xray_port, xray_sni, xray_dest, xray_public_host, xray_update | Xray VLESS + Reality |
| node_exporter_version, node_exporter_port, node_exporter_listen_address, node_exporter_sha256 | Node Exporter |
| vector_sink_url, vector_data_dir | Vector; токен берётся из Vault |
| postgres_*, db_backup_* | PostgreSQL и S3 backup |
| k8s_kubeconfig, awx_*, kafka_*, jenkins_* | Kubernetes playbook |

## Playbook для Linux

В командах ниже используется inventory, настроенный в ansible.cfg. Все серверные playbook рассчитаны на Debian и Ubuntu.

### Первичная настройка сервера

#### playbooks/bootstrap/system.yml

Создаёт администратора с sudo, добавляет его SSH-ключ, устанавливает базовые пакеты и настраивает часовой пояс и локаль.

Перед запуском укажите bootstrap_admin_user, путь bootstrap_ssh_public_key_path и Vault-переменную vault_bootstrap_admin_password_hash. Имя по умолчанию test — замените его своим.

~~~bash
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass --check --diff
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass
~~~

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

Подключает официальный APT-репозиторий Docker, устанавливает Docker Engine CE и Compose plugin, задаёт ротацию JSON-логов. Добавляет пользователей из docker_admin_users в группу docker.

~~~bash
ansible-playbook playbooks/docker/docker_engine.yml --limit server1
~~~

Пользователь в группе docker фактически получает полномочия уровня root на хосте. Добавляйте туда только доверенные учётные записи. Для нестандартных архитектур проверьте список поддерживаемых архитектур в playbook.

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

### База данных и резервные копии

#### playbooks/database/postgres_standalone.yml

Устанавливает PostgreSQL, применяет параметры памяти и соединений, создаёт прикладную роль и базу. Перед запуском задайте postgres_database, postgres_role и Vault-переменную vault_postgres_app_password. Значения памяти из vars.yml — только отправная точка, подберите их под RAM сервера и нагрузку.

~~~bash
ansible-playbook playbooks/database/postgres_standalone.yml --limit server1 --ask-vault-pass
~~~

Сценарий не открывает PostgreSQL для внешней сети и не создаёт firewall-правило на 5432.

#### playbooks/database/db_backup_s3.yml

Планирует systemd timer для pg_dump, zstd-сжатия, загрузки в S3 с SSE-S3 и хранения локальных копий с заданным сроком очистки. Требует установленный PostgreSQL, заполненные db_backup_s3_bucket, db_backup_aws_region и db_backup_s3_prefix. Предпочтительный способ авторизации в AWS — IAM role с минимальными правами к нужному bucket/prefix; статические ключи playbook не задаёт.

Планировщик по умолчанию запускается ежедневно в 02:30 по локальному времени сервера с дополнительной случайной задержкой до 15 минут. Настройка расписания хранится в db_backup_schedule.

~~~bash
ansible-playbook playbooks/database/db_backup_s3.yml --limit server1
~~~

> [!CAUTION]
> Обнаружен блокер: backup-скрипт создаётся с владельцем root:root и режимом 0750, тогда как systemd запускает его пользователем postgres. Этот пользователь не может выполнить скрипт, поэтому timer не создаст резервные копии. Исправьте владельца/группу или права файла в playbooks/database/db_backup_s3.yml до использования сценария и проверьте успешный тестовый backup и восстановление.

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

## Playbook для Kubernetes

Все Kubernetes playbook работают на localhost через kubeconfig. До применения проверьте context и хранилища:

~~~bash
kubectl config current-context
kubectl get nodes
kubectl get storageclass
~~~

Переменная k8s_kubeconfig берётся из KUBECONFIG или по умолчанию из ~/.kube/config. Путь к альтернативному kubeconfig задайте в inventory/group_vars/all/devops.yml.

### playbooks/kubernetes/awx.yml

Устанавливает Helm chart AWX Operator версии awx_operator_chart_version, создаёт Secrets для пароля администратора и ключа шифрования, затем AWX custom resource. Нужен существующий StorageClass в awx_storage_class и сильные Vault-значения vault_awx_admin_password и vault_awx_secret_key. Секретный ключ AWX сохраняйте постоянно: его замена может лишить доступа к уже зашифрованным данным.

~~~bash
ansible-playbook playbooks/kubernetes/awx.yml --ask-vault-pass
kubectl -n awx get pods
kubectl -n awx port-forward svc/awx-service 8080:80
~~~

Откройте http://127.0.0.1:8080. Service остаётся ClusterIP. Для внешнего доступа отдельно установите ingress/reverse proxy с TLS и аутентификацией. Playbook не настраивает backup AWX или его PostgreSQL.

### playbooks/kubernetes/kafka_strimzi.yml

Устанавливает Strimzi Helm chart заданной версии и создаёт Kafka-кластер в KRaft, отдельные controller и broker pools, topic и SCRAM-пользователя с ACL. По умолчанию задаются три controller, три broker, три partition и persistent volume для каждой ноды. Нужен storage class в kafka_storage_class и достаточные CPU, RAM и дисковая ёмкость кластера.

Проверка требует не менее трёх controller и broker, а kafka_broker_replicas должно быть не меньше kafka_topic_partitions.

~~~bash
ansible-playbook playbooks/kubernetes/kafka_strimzi.yml
kubectl -n kafka get pods
kubectl -n kafka get kafka,kafkanodepool,kafkatopic,kafkauser
~~~

Внутрикластерный bootstrap address: events-kafka-bootstrap.kafka.svc:9093. Подключение использует TLS и SCRAM-SHA-512. Сгенерированный пароль находится в Secret с именем, заданным в kafka_app_user. Внешний listener не создаётся; публикуйте Kafka только после отдельной настройки сетевой защиты.

### playbooks/kubernetes/jenkins.yml

Создаёт Jenkins как StatefulSet с одним controller pod, persistent home volume и ClusterIP. Контейнер работает без root и с ограниченными Linux capabilities. Нужны jenkins_storage_class и Vault-переменная vault_jenkins_admin_password.

~~~bash
ansible-playbook playbooks/kubernetes/jenkins.yml --ask-vault-pass
kubectl -n jenkins rollout status statefulset/jenkins
kubectl -n jenkins port-forward svc/jenkins 8080:8080
~~~

Откройте http://127.0.0.1:8080. Внешний ingress, backup volume, плагины и build agents не настраиваются этим playbook. Инициализационный Groovy-скрипт создаёт пользователя, только если его ещё нет; изменение пароля в Vault не меняет пароль уже созданного пользователя Jenkins.

### playbooks/kubernetes/cluster_health.yml

Read-only отчёт по числу nodes и pods, pods не в состояниях Running/Succeeded и Warning events. Kubeconfig должен иметь права чтения этих ресурсов во всех нужных namespaces.

~~~bash
ansible-playbook playbooks/kubernetes/cluster_health.yml
~~~

## Проверка, диагностика и известные ограничения

Сначала проверяйте синтаксис конкретного playbook:

~~~bash
ansible-playbook playbooks/<путь>/<имя>.yml --syntax-check
~~~

Для обычных Ansible-модулей используйте --check --diff только там, где сценарий и модули поддерживают check mode. Некоторые playbook содержат command, генерацию credentials, Certbot, Helm или операции Kubernetes; check mode не покажет полный результат и может быть неприменим при первом запуске. Удалённые сценарии с systemd, UFW и apt проверяйте на одном тестовом сервере.

Для общего отчёта Linux запустите playbooks/diagnostics/host_report.yml. Для сводки Kubernetes — playbooks/kubernetes/cluster_health.yml. Состояние конкретной службы можно проверить на сервере через systemctl status и journalctl.

Известные ограничения текущей реализации:

- S3 backup не заработает, пока не исправлены права на backup-скрипт; подробности указаны в его разделе.
- Xray playbook включает UFW с ограниченным списком разрешённых портов и может закрыть порты других приложений.
- Xray installer не закреплён по версии и checksum.
- Docker-группа и доступ Vector к Docker socket дают широкие права на хосте.
- AWX и Jenkins требуют отдельного плана backup/restore для persistent data.
- WireGuard playbook настраивает mesh-интерфейс, но не IP forwarding или NAT.

## Файлы проекта

~~~text
ansible.cfg
collections/requirements.yml
inventory/
  hosts.ini.example
  group_vars/all/
    vars.yml
    devops.yml
    vault.example.yml
playbooks/
  bootstrap/       system.yml, security.yml
  database/        postgres_standalone.yml, db_backup_s3.yml
  diagnostics/     host_report.yml
  docker/          docker_engine.yml
  kubernetes/      awx.yml, kafka_strimzi.yml, jenkins.yml, cluster_health.yml
  maintenance/     update_all.yml, disk_cleanup.yml
  network/         wireguard.yml, nginx_proxy.yml, xray_reality.yml
  observability/   node_exporter.yml, vector_shipper.yml
roles/xray/templates/config.json.j2
~~~

