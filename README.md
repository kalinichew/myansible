# myansible

Библиотека Ansible playbook для серверов Debian/Ubuntu и уже работающего Kubernetes-кластера. Здесь есть настройка Linux-хостов, сетевые сервисы, мониторинг, PostgreSQL, обслуживание, а также развёртывание AWX, Kafka и Jenkins.

> [!WARNING]
> Playbook меняют реальные серверы и кластеры: могут включить firewall, изменить SSH, установить и перезапустить службы, очистить данные или перезагрузить систему. Начинайте с одного тестового хоста, задавайте `--limit`, проверяйте переменные и сохраняйте действующую SSH-сессию до проверки нового подключения. Для Kubernetes перед запуском проверьте текущий kube-context и namespace.

## Возможности

| Область | Playbook | Назначение |
|---|---|---|
| Начальная настройка | `playbooks/bootstrap/system.yml` | Обновляет APT, ставит базовые пакеты, задаёт часовой пояс и локаль, создаёт администратора, sudo и SSH-ключ |
| Защита сервера | `playbooks/bootstrap/security.yml` | Настраивает SSH, Fail2ban и UFW |
| Docker | `playbooks/docker/docker_engine.yml` | Ставит Docker CE и Compose plugin, настраивает ротацию логов |
| WireGuard | `playbooks/network/wireguard.yml` | Создаёт WireGuard mesh-интерфейс и peers |
| NGINX | `playbooks/network/nginx_proxy.yml` | Настраивает reverse proxy и сертификаты Let's Encrypt |
| Xray | `playbooks/network/xray_reality.yml` | Настраивает VLESS + Reality, firewall, BBR и выводит клиентскую ссылку |
| Мониторинг | `playbooks/observability/node_exporter.yml` | Устанавливает Node Exporter как systemd-службу |
| Отправка логов | `playbooks/observability/vector_shipper.yml` | Отправляет journald и Docker-логи в HTTPS sink через Vector |
| PostgreSQL | `playbooks/database/postgres_standalone.yml` | Устанавливает PostgreSQL, задаёт параметры и создаёт роль и базу |
| S3 backup | `playbooks/database/db_backup_s3.yml` | Настраивает расписание сжатых PostgreSQL backup в S3 и локальное хранение |
| Обслуживание | `playbooks/maintenance/update_all.yml` | Обновляет пакеты и при необходимости перезагружает сервер |
| Очистка | `playbooks/maintenance/disk_cleanup.yml` | Очищает journal, старые Docker-образы/build cache и APT |
| Отчёт Linux | `playbooks/diagnostics/host_report.yml` | Показывает диски, память, failed units, слушающие порты и признак требуемой перезагрузки |
| AWX | `playbooks/kubernetes/awx.yml` | Устанавливает AWX Operator и создаёт приватный AWX instance |
| Kafka | `playbooks/kubernetes/kafka_strimzi.yml` | Устанавливает Strimzi и создаёт Kafka в KRaft, topic и ограниченного пользователя |
| Jenkins | `playbooks/kubernetes/jenkins.yml` | Создаёт приватный Jenkins controller с persistent volume |
| Отчёт Kubernetes | `playbooks/kubernetes/cluster_health.yml` | Сводит nodes, pods и Warning events по кластеру |

## Быстрый старт

Playbook для серверов запускаются с Linux или WSL. Kubernetes playbook работают с управляющей машины и требуют уже настроенный кластер. Этот проект не создаёт Kubernetes control plane.

### Установите зависимости

Создайте Python virtual environment, установите Ansible и библиотеки Kubernetes, затем установите коллекции проекта:

~~~bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install ansible kubernetes PyYAML
ansible-galaxy collection install -r collections/requirements.yml
~~~

Для Kubernetes playbook дополнительно нужны Helm CLI и kubectl. Проверьте, что kubectl подключён к нужному кластеру:

~~~bash
helm version
kubectl config current-context
kubectl get nodes
~~~

Если запускаете playbook внутри AWX, установите Ansible-коллекции и Python-зависимости в используемый Execution Environment и предоставьте job доступ к kubeconfig.

### Создайте inventory

Скопируйте пример и замените адрес и начальную SSH-учётную запись:

~~~bash
cp inventory/hosts.ini.example inventory/hosts.ini
~~~

Файл inventory/hosts.ini исключён из Git. По умолчанию в примере используется root для bootstrap. После создания администратора проверьте вход под ним и измените ansible_user в inventory. Проверьте соединение:

~~~bash
ansible-inventory --graph
ansible all -m ansible.builtin.ping --limit server1
~~~

Настройте путь к публичному SSH-ключу в inventory/group_vars/all/vars.yml: значение по умолчанию может не совпадать с расположением ключа на вашей управляющей машине.

### Подготовьте Ansible Vault

Скопируйте пример секретов, замените все заглушки и зашифруйте файл:

~~~bash
cp inventory/group_vars/all/vault.example.yml inventory/group_vars/all/vault.yml
$EDITOR inventory/group_vars/all/vault.yml
ansible-vault encrypt inventory/group_vars/all/vault.yml
~~~

Для bootstrap администратора нужен хеш, а не открытый пароль. Например, создайте SHA-512 хеш:

~~~bash
openssl passwd -6
~~~

Запишите результат в vault_bootstrap_admin_password_hash. Остальные секреты заполняйте только для тех сервисов, которые планируете развёртывать. Пароль Vault храните отдельно от Git; секрет-зависимые playbook запускайте с `--ask-vault-pass`. Файл vault.yml исключён из Git.

Для AWX secret key используйте случайное значение и сохраните его надолго: его замена после развёртывания может сделать недоступными данные, зашифрованные AWX.

## Конфигурация

Ansible автоматически загружает переменные из inventory/group_vars/all/vars.yml и inventory/group_vars/all/devops.yml. Для настройки одного сервера используйте host_vars; не записывайте туда секреты.

| Файл или переменные | Что настраивать |
|---|---|
| inventory/hosts.ini | Адрес хоста и SSH-пользователь |
| bootstrap_admin_user, bootstrap_ssh_public_key_path | Имя нового администратора и путь к .pub файлу на управляющей машине |
| security_ssh_port, security_ufw_allowed_tcp_ports | Фактический SSH-порт и все необходимые входящие TCP-порты |
| vps_hostname, vps_timezone, xray_port, xray_sni, xray_dest | Имя, часовой пояс и параметры VLESS + Reality |
| wireguard_address, wireguard_listen_port, wireguard_peers | Адрес интерфейса, UDP-порт и публичные параметры peers |
| nginx_proxy_sites | Для каждого домена: server_name, upstream_url и email для ACME |
| node_exporter_listen_address, node_exporter_port | Адрес и порт метрик; по умолчанию Node Exporter слушает только localhost |
| vector_sink_url | HTTPS endpoint; bearer token задаётся через Vault |
| postgres_*, db_backup_* | База, роль, параметры PostgreSQL и S3 backup |
| k8s_kubeconfig | Путь к kubeconfig; по умолчанию используется KUBECONFIG или ~/.kube/config |
| awx_storage_class, kafka_storage_class, jenkins_storage_class | StorageClass для persistent volumes соответствующего сервиса |
| awx_*, kafka_*, jenkins_* | Имена, размеры хранилищ, версии и настройки Kubernetes-сервисов |

## Запуск playbook для Linux

Все команды запускайте из корня репозитория. Начинайте с одного сервера и переходите к остальным только после проверки.

### Bootstrap и SSH/firewall

Сначала создайте администратора и установите его SSH-ключ. Проверьте вход новым пользователем, обновите inventory, затем применяйте hardening:

~~~bash
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass --check --diff
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass
ansible all -m ansible.builtin.ping --limit server1

ansible-playbook playbooks/bootstrap/security.yml --limit server1 --check --diff
ansible-playbook playbooks/bootstrap/security.yml --limit server1
~~~

Security playbook включает UFW. По умолчанию разрешены TCP-порты 22, 80 и 443, а остальные входящие соединения запрещены. Если SSH работает на другом порту, измените security_ssh_port и добавьте этот порт в security_ufw_allowed_tcp_ports. Добавьте также все порты приложений до включения firewall.

### Сетевые сервисы

WireGuard требует заполнить wireguard_peers. В конфигурацию вносятся их публичные ключи и AllowedIPs; приватный ключ интерфейса создаётся на сервере и хранится с правами root-only. Playbook настраивает mesh-интерфейс и открывает UDP-порт WireGuard в UFW. Проверьте также firewall у провайдера.

~~~bash
ansible-playbook playbooks/network/wireguard.yml --limit server1
~~~

NGINX требует DNS-записей доменов на сервер и доступности портов 80/443 снаружи. Заполните nginx_proxy_sites и запустите:

~~~bash
ansible-playbook playbooks/network/nginx_proxy.yml --limit server1
~~~

Xray создаёт UUID, X25519-ключи и Short ID при первом запуске и сохраняет их в /etc/xray/credentials.yml с правами 0600. Клиентская ссылка выводится в конце запуска.

> [!CAUTION]
> Xray playbook включает UFW с запретом входящих соединений по умолчанию и открывает только SSH-порт и xray_port. Он не использует security_ufw_allowed_tcp_ports. На сервере с NGINX или другими службами это может закрыть их порты, включая 80/tcp. До запуска добавьте нужные правила в playbook или используйте отдельный сервер. Также учтите, что установщик Xray скачивается из изменяемой ветки upstream и запускается от root без закреплённой версии или checksum; для воспроизводимого production-развёртывания его следует закрепить.

До запуска проверьте SSH-порт и входящие правила UFW:

~~~bash
ansible-playbook playbooks/network/xray_reality.yml --syntax-check
ansible-playbook playbooks/network/xray_reality.yml --limit server1
~~~

Для обновления Xray измените переменную xray_update:

~~~bash
ansible-playbook playbooks/network/xray_reality.yml --limit server1 --tags xray -e xray_update=true
~~~

Изменение xray_sni или xray_dest не меняет UUID и ключи. После изменения параметров используйте новую ссылку, выведенную playbook.

### Docker, мониторинг и база данных

~~~bash
ansible-playbook playbooks/docker/docker_engine.yml --limit server1
ansible-playbook playbooks/observability/node_exporter.yml --limit server1
ansible-playbook playbooks/observability/vector_shipper.yml --limit server1 --ask-vault-pass
ansible-playbook playbooks/database/postgres_standalone.yml --limit server1 --ask-vault-pass
~~~

Vector требует установленный Docker и отправляет как journald, так и Docker-логи. Укажите HTTPS endpoint и токен в Vault. Участник группы docker получает широкие полномочия на сервере; Vector добавляется в эту группу для чтения Docker socket.

Node Exporter по умолчанию доступен только с самого сервера по адресу 127.0.0.1:9100. Чтобы Prometheus опрашивал его по сети, задайте защищённый адрес прослушивания и отдельно ограничьте доступ firewall.

### Резервное копирование PostgreSQL в S3

Перед настройкой заполните db_backup_s3_bucket, db_backup_aws_region и при необходимости db_backup_s3_prefix:

~~~bash
ansible-playbook playbooks/database/db_backup_s3.yml --limit server1
~~~

Сценарий рассчитан на локальную PostgreSQL-аутентификацию от системного пользователя postgres, сжимает дамп в zstd, отправляет его в S3 с SSE-S3 и оставляет локальные копии на заданный срок. Для доступа к S3 предпочтительна IAM role с минимальными правами на нужный bucket/prefix; сценарий не требует статических AWS-ключей.

> [!CAUTION]
> Ревью выявило блокирующую ошибку в текущей реализации: скрипт backup устанавливается как root:root с режимом 0750, но systemd запускает его от пользователя postgres. Пользователь postgres не может выполнить этот файл. Исправьте владельца/группу или права скрипта в playbooks/database/db_backup_s3.yml до включения этого таймера; до исправления не рассчитывайте на успешные резервные копии.

### Обслуживание

Playbook обновления может перезагрузить сервер, если система сообщает о необходимости reboot. Cleanup удаляет старые Docker images и build cache, а также устаревшие пакеты и конфигурации:

~~~bash
ansible-playbook playbooks/maintenance/update_all.yml --limit server1
ansible-playbook playbooks/maintenance/disk_cleanup.yml --limit server1
ansible-playbook playbooks/diagnostics/host_report.yml --limit server1
~~~

Ознакомьтесь с содержимым playbook перед запуском на production. Check mode не моделирует все действия модулей command, Helm и Kubernetes, поэтому `--check --diff` не является полной гарантией безопасного dry run.

## Запуск Kubernetes playbook

Убедитесь, что текущий kube-context указывает на нужный кластер и StorageClass доступны:

~~~bash
kubectl config current-context
kubectl get nodes
kubectl get storageclass
~~~

Плейбуки AWX, Kafka и Jenkins запускаются на управляющей машине через localhost и kubeconfig. Они не создают кластер, ingress, публичный LoadBalancer или резервные копии persistent volumes.

### AWX

Задайте awx_storage_class и значения awx_admin_password и awx_secret_key в Vault. AWX публикуется как ClusterIP; для локального доступа:

~~~bash
ansible-playbook playbooks/kubernetes/awx.yml --ask-vault-pass
kubectl -n awx get pods
kubectl -n awx port-forward svc/awx-service 8080:80
~~~

Откройте http://127.0.0.1:8080. Для внешнего доступа отдельно настройте ingress или reverse proxy с TLS и аутентификацией. AWX использует operator-managed PostgreSQL; playbook не настраивает HA-базу данных или backup.

### Kafka / Strimzi

Задайте kafka_storage_class. Настройка по умолчанию создаёт три controller и три broker pod, persistent volumes, внутренний TLS и SCRAM-SHA-512, topic и пользователя приложения с ACL к этому topic. Кластеру нужны достаточные CPU, RAM и дисковая ёмкость на нескольких worker nodes.

~~~bash
ansible-playbook playbooks/kubernetes/kafka_strimzi.yml
kubectl -n kafka get pods
kubectl -n kafka get kafka,kafkanodepool,kafkatopic,kafkauser
~~~

Адрес для подключения из Kubernetes: events-kafka-bootstrap.kafka.svc:9093. Пароль пользователя Strimzi хранится в Kubernetes Secret с именем kafka_app_user. Внешний listener не создаётся.

### Jenkins

Укажите jenkins_storage_class и пароль администратора в Vault. Jenkins работает одним controller pod и ClusterIP service:

~~~bash
ansible-playbook playbooks/kubernetes/jenkins.yml --ask-vault-pass
kubectl -n jenkins rollout status statefulset/jenkins
kubectl -n jenkins port-forward svc/jenkins 8080:8080
~~~

Откройте http://127.0.0.1:8080. Playbook не настраивает ingress, backup PVC, плагины или внешние build agents. Инициализационный скрипт создаёт пользователя только при первом появлении; обновление Vault-пароля само по себе не меняет пароль уже существующего пользователя Jenkins.

### Проверка состояния Kubernetes

Отчёт читает nodes, pods и Warning events во всех namespaces. Kubeconfig должен иметь соответствующий read-only доступ:

~~~bash
ansible-playbook playbooks/kubernetes/cluster_health.yml
~~~

## Границы и меры безопасности

- Все серверные playbook рассчитаны на Debian и Ubuntu.
- Перед применением SSH или firewall изменений убедитесь, что указаны верные SSH-порт и разрешённые входящие порты.
- Участники группы docker фактически получают полномочия уровня root.
- Vector также получает доступ к Docker socket для чтения контейнерных логов.
- Не открывайте Node Exporter публично; ограничивайте доступ Prometheus доверенной сетью.
- Не помещайте секреты, Vault password, приватные WireGuard/Xray ключи или kubeconfig в Git.
- Persistent volume не заменяет проверенный backup и процедуру восстановления.
- S3 backup playbook имеет известный блокер, описанный выше.

## Структура проекта

~~~text
.
├── ansible.cfg
├── collections/requirements.yml
├── inventory/
│   ├── hosts.ini.example
│   └── group_vars/all/
│       ├── devops.yml
│       ├── vars.yml
│       └── vault.example.yml
├── playbooks/
│   ├── bootstrap/       # начальная настройка и SSH/firewall hardening
│   ├── database/        # PostgreSQL и S3 backup
│   ├── diagnostics/     # отчёт Linux-хоста
│   ├── docker/          # Docker Engine
│   ├── kubernetes/      # AWX, Kafka, Jenkins и отчёт кластера
│   ├── maintenance/     # обновление и очистка
│   ├── network/         # WireGuard, NGINX и Xray
│   └── observability/   # Node Exporter и Vector
└── roles/xray/templates/config.json.j2
~~~

## Полезные документы

- [Ansible документация](https://docs.ansible.com/)
- [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)
- [Kubernetes документация](https://kubernetes.io/docs/)
- [AWX Operator Helm chart](https://docs.ansible.com/projects/awx-operator-helm/)
- [Strimzi документация](https://strimzi.io/docs/operators/latest/full/overview)
- [Jenkins документация](https://www.jenkins.io/doc/)
- [Xray Reality документация](https://github.com/XTLS/Xray-docs-next/blob/main/docs/en/config/transports/reality.md)
