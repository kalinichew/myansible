# myansible · DevOps Playbook Library

Набор Ansible playbook для Linux-инфраструктуры и Kubernetes. В этой feature-ветке собраны базовый bootstrap, security hardening, диагностика, VPN, CI/CD, AWX, Kafka, мониторинг, PostgreSQL и обслуживание.

> [!IMPORTANT]
> Все playbook запускайте осознанно: сначала проверьте переменные и target, затем используйте --check --diff там, где модуль поддерживает check mode, и ограничивайте первый запуск одним хостом. Сервисы Kubernetes разворачиваются в уже существующий кластер: playbook не создают Kubernetes control plane и не настраивают облачный LoadBalancer.

## Содержание

- Быстрый старт
- Конфигурация и секреты
- Каталог playbook
- Подробно: VPN, AWX, Kafka и Jenkins
- Диагностика и обслуживание
- Безопасность, обновление и откат

## Что входит

| Область | Playbook | Назначение |
|---|---|---|
| Bootstrap | `playbooks/bootstrap/system.yml` | APT, базовые пакеты, UTC/locale, администратор, sudo и SSH-ключ |
| Security | `playbooks/bootstrap/security.yml` | SSH hardening, Fail2ban, UFW |
| Диагностика Linux | `playbooks/diagnostics/host_report.yml` | Диски, память, failed systemd units, слушающие порты и reboot-required |
| VPN | `playbooks/network/wireguard.yml` | WireGuard интерфейс и mesh peers |
| Reverse proxy | `playbooks/network/nginx_proxy.yml` | NGINX, TLS и Let's Encrypt |
| AWX | `playbooks/kubernetes/awx.yml` | Оператор AWX и приватный AWX instance в Kubernetes |
| Kafka | `playbooks/kubernetes/kafka_strimzi.yml` | Strimzi, Kafka KRaft, отдельные controller/broker pools, TLS/SCRAM и topic/user |
| Jenkins | `playbooks/kubernetes/jenkins.yml` | Приватный Jenkins StatefulSet с PVC и bootstrap admin |
| Диагностика Kubernetes | `playbooks/kubernetes/cluster_health.yml` | Nodes, pods и Warning events во всех namespace |
| Контейнеры | `playbooks/docker/docker_engine.yml` | Docker CE, Compose plugin, log rotation |
| Observability | `playbooks/observability/node_exporter.yml`, `vector_shipper.yml` | Node Exporter и Vector |
| База данных | `playbooks/database/postgres_standalone.yml`, `db_backup_s3.yml` | PostgreSQL и резервные копии в S3 |
| Обслуживание | `playbooks/maintenance/update_all.yml`, `disk_cleanup.yml` | Обновление ОС, reboot и уборка |

## Быстрый старт

### 1. Получить feature-ветку

~~~bash
git clone https://github.com/kalinichew/myansible.git
cd myansible
git fetch origin
git checkout --track origin/feature/devops-toolkit
~~~

### 2. Установить управляющие зависимости

Запускайте Ansible из Linux или WSL. Для Kubernetes-модулей нужны доступный kubeconfig, Helm CLI и Python-библиотеки Kubernetes/PyYAML на управляющей машине.

~~~bash
python3 -m pip install --user ansible kubernetes PyYAML
ansible-galaxy collection install -r collections/requirements.yml
helm version
kubectl config current-context
~~~

В Ansible используются коллекции из `collections/requirements.yml`. Если Helm/Kubernetes playbook запускается из AWX, установите эти зависимости в Execution Environment и смонтируйте kubeconfig в job.

### 3. Создать inventory

~~~bash
cp inventory/hosts.ini.example inventory/hosts.ini
~~~

В `inventory/hosts.ini` замените примерный IP и SSH-пользователя. Этот inventory используется для Debian/Ubuntu playbook; Kubernetes playbook запускаются на localhost и обращаются к кластеру через kubeconfig.

Проверьте доступ к Linux-хосту:

~~~bash
ansible all -m ansible.builtin.ping
ansible-inventory --graph
~~~

### 4. Подготовить Vault

В репозитории находится только безопасный шаблон с заглушками. Реальный `vault.yml` намеренно исключён из Git.

~~~bash
cp inventory/group_vars/all/vault.example.yml inventory/group_vars/all/vault.yml
ansible-vault edit inventory/group_vars/all/vault.yml
ansible-vault encrypt inventory/group_vars/all/vault.yml
~~~

Заполните только нужные секреты: hash пароля bootstrap-администратора, пароль PostgreSQL, токен Vector, admin password и encryption key AWX, admin password Jenkins. Для `vault_bootstrap_admin_password_hash` используйте парольный hash, не открытый пароль. Ключ AWX создайте один раз, например `openssl rand -base64 48`, и сохраните в Vault: смена этого ключа после развёртывания лишит AWX доступа к ранее зашифрованным данным.

Пароль Vault храните отдельно от Git. Запускайте секрет-зависимые playbook с `--ask-vault-pass`.

## Конфигурация

Общие настройки существующих серверных playbook лежат в `inventory/group_vars/all/vars.yml`. Новые параметры DevOps-сервисов — в `inventory/group_vars/all/devops.yml`. Ansible автоматически загружает оба файла для группы `all`.

| Файл / переменные | Что настраивать |
|---|---|
| `inventory/hosts.ini` | IP/DNS, начальный SSH-пользователь |
| `bootstrap_admin_user`, `bootstrap_ssh_public_key_path` | Имя администратора и путь к публичному ключу на управляющей машине |
| `security_ssh_port`, `security_ufw_allowed_tcp_ports` | SSH-порт и все разрешённые входящие TCP-порты |
| `wireguard_address`, `wireguard_peers` | Адрес VPN-интерфейса и public key/AllowedIPs каждого peer |
| `nginx_proxy_sites` | server_name, upstream_url и email для ACME |
| `vector_sink_url` | HTTPS endpoint логов; bearer token хранится в Vault |
| `postgres_*`, `db_backup_*` | база, роль, настройки памяти и S3 backup |
| `k8s_kubeconfig` | kubeconfig на управляющей машине; по умолчанию `KUBECONFIG` или `~/.kube/config` |
| `awx_storage_class` | Kubernetes StorageClass для PostgreSQL и проектов AWX — обязателен |
| `awx_admin_password`, `awx_secret_key` | AWX credentials, приходят из Vault |
| `kafka_storage_class` | StorageClass для Kafka — обязателен |
| `kafka_controller_replicas`, `kafka_broker_replicas` | Количество Strimzi controller/broker узлов; по умолчанию по три |
| `kafka_version`, `kafka_topic_name`, `kafka_app_user` | Версия Kafka и начальные topic/user |
| `jenkins_storage_class`, `jenkins_storage_size` | Persistent storage Jenkins — StorageClass обязателен |
| `jenkins_image`, `jenkins_admin_user` | Версия официального образа и первый admin |
| `maintenance_*`, `disk_*` | Reboot policy, journald и Docker cleanup |

Для конфигурации под конкретный сервер создавайте host vars, а секреты оставляйте в Vault. Значения CPU/RAM/дисков — отправная точка; адаптируйте их под рабочую нагрузку и storage policy кластера.

## Запуск серверных playbook

Bootstrap выполняйте по этапам: сперва создайте ключевого администратора, проверьте вход новой учётной записью, измените ansible_user в inventory и только потом применяйте SSH/UFW hardening.

~~~bash
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass --check --diff
ansible-playbook playbooks/bootstrap/system.yml --limit server1 --ask-vault-pass
ansible all -m ansible.builtin.ping --limit server1

ansible-playbook playbooks/bootstrap/security.yml --limit server1 --check --diff
ansible-playbook playbooks/bootstrap/security.yml --limit server1
~~~

Пример запуска Docker или WireGuard:

~~~bash
ansible-playbook playbooks/docker/docker_engine.yml --limit server1 --check --diff
ansible-playbook playbooks/docker/docker_engine.yml --limit server1

ansible-playbook playbooks/network/wireguard.yml --limit vpn1 --ask-vault-pass --check --diff
ansible-playbook playbooks/network/wireguard.yml --limit vpn1 --ask-vault-pass
~~~

WireGuard создаёт приватный ключ на целевой машине; в peer-конфигурации указываются только публичные ключи. Откройте UDP `wireguard_listen_port` на провайдерском firewall. Для нестандартного SSH-порта сначала одновременно обновите `security_ssh_port` и `security_ufw_allowed_tcp_ports`.

Пример peer:

~~~yaml
wireguard_address: 10.10.0.1/24
wireguard_peers:
  - public_key: "PUBLIC_KEY_OF_PEER"
    allowed_ips: "10.10.0.2/32"
    endpoint: "vpn-peer.example.net:51820"
    persistent_keepalive: 25
~~~

## Kubernetes playbook

Перед запуском убедитесь, что текущий kube-context ведёт в нужный кластер. Проверяйте namespace, StorageClass и права пользователя, от имени которого работает kubeconfig.

~~~bash
kubectl config current-context
kubectl get nodes
kubectl get storageclass
ansible-playbook playbooks/kubernetes/cluster_health.yml
~~~

### AWX

Playbook устанавливает community AWX Operator Helm chart `3.2.1`, затем создаёт AWX custom resource. Требуется существующий кластер и динамический или заранее подготовленный StorageClass. Сервис остаётся `ClusterIP`: используйте ingress/reverse proxy с TLS или локальный port-forward.

Настройте `awx_storage_class`, пароли и encryption key в Vault, затем запустите:

~~~bash
ansible-playbook playbooks/kubernetes/awx.yml --ask-vault-pass --check --diff
ansible-playbook playbooks/kubernetes/awx.yml --ask-vault-pass
kubectl -n awx get pods
kubectl -n awx port-forward svc/awx-service 8080:80
~~~

Откройте `http://127.0.0.1:8080`. AWX использует оператор-управляемую PostgreSQL с persistent volume; это упрощённая основа, а не HA database. Для production настройте внешнюю поддерживаемую PostgreSQL, резервное копирование, ingress TLS, мониторинг и процедуру обновления. Не обновляйте оператор без проверки release notes и миграций CRD.

### Kafka / Strimzi

Playbook ставит Strimzi Operator `1.2.0` и создаёт Kafka `4.2.1` в KRaft. По умолчанию развёртываются три controller и три broker pod на отдельных persistent claims, с internal TLS и SCRAM-SHA-512. Создаётся topic и KafkaUser с ACL только к этому topic и consumer group с префиксом `app-`.

Перед запуском обязательно укажите `kafka_storage_class` и проверьте емкость/IOPS дисков. Шесть Kafka pod требуют нескольких подходящих Kubernetes worker nodes.

~~~bash
ansible-playbook playbooks/kubernetes/kafka_strimzi.yml --check --diff
ansible-playbook playbooks/kubernetes/kafka_strimzi.yml
kubectl -n kafka get kafka,kafkanodepool,kafkatopic,kafkauser
kubectl -n kafka get pods
~~~

Bootstrap address для приложений внутри Kubernetes: `events-kafka-bootstrap.kafka.svc:9093`. Пароль пользователя Strimzi выдаёт в Secret с именем, заданным в `kafka_app_user`; выдавайте его приложениям через secret reference. Внешний listener намеренно не включён: не публикуйте Kafka напрямую в интернет. Для production дополнительно настройте anti-affinity/rack awareness по зонам, мониторинг lag, backup/DR, quotas и регулярные проверенные обновления.

### Jenkins

Jenkins разворачивается как одиночный StatefulSet с persistent home, startup/readiness/liveness probes и закрытым ClusterIP service. Администратор создаётся из Vault; setup wizard отключён. Входящие agent-порты и Docker socket хоста не открываются.

Укажите `jenkins_storage_class` и Vault пароль:

~~~bash
ansible-playbook playbooks/kubernetes/jenkins.yml --ask-vault-pass --check --diff
ansible-playbook playbooks/kubernetes/jenkins.yml --ask-vault-pass
kubectl -n jenkins rollout status statefulset/jenkins
kubectl -n jenkins port-forward svc/jenkins 8080:8080
~~~

Откройте `http://127.0.0.1:8080`. Это начальный однорепличный controller: для production отдельно организуйте backup/restore PVC, ingress TLS, плагины и их pinning, SSO, ограничение прав, external agents и обновления. Не подключайте `/var/run/docker.sock` к Jenkins controller; сборку контейнеров выносите на отдельные агенты или изолированный builder.

## Диагностика и обслуживание

### Linux host report

Отчёт read-only и не требует специальной настройки:

~~~bash
ansible-playbook playbooks/diagnostics/host_report.yml --limit server1
~~~

Он показывает filesystem capacity, memory/swap, failed systemd units, listening TCP/UDP ports и факт требуемой перезагрузки. Вывод сокетов может включать имена процессов; делитесь им только в доверенном канале.

### Kubernetes health report

~~~bash
ansible-playbook playbooks/kubernetes/cluster_health.yml
~~~

Показывает количество nodes/pods, pods вне Running/Succeeded и Warning events. Требуются права read-only на ресурсы кластера.

### Обновления и очистка

~~~bash
ansible-playbook playbooks/maintenance/update_all.yml --limit server1 --check --diff
ansible-playbook playbooks/maintenance/update_all.yml --limit server1

ansible-playbook playbooks/maintenance/disk_cleanup.yml --limit server1 --check --diff
ansible-playbook playbooks/maintenance/disk_cleanup.yml --limit server1
~~~

Обновление идёт по одному хосту и может перезагрузить его, если найден `/var/run/reboot-required`. Docker cleanup не удаляет volumes; перед очисткой всё равно проверяйте, что образы и stopped containers больше не нужны.

## Проверка перед применением

~~~bash
ansible-playbook playbooks/<path>/<name>.yml --syntax-check
ansible-playbook playbooks/<path>/<name>.yml --limit server1 --check --diff
~~~

Kubernetes playbook не всегда полностью прогнозируются в check mode, так как Helm и custom resources могут иметь ограничения dry-run. Перед применением изучите diff, проверьте выбранный kube-context и запускайте изменения в отдельном тестовом namespace или кластере.

## Дизайн и границы ответственности

- FQCN используются во всех новых задачах; изменения сервисов применяются через Ansible handlers в существующих playbook.
- Удалённые host playbook ограничены Debian/Ubuntu; Kubernetes playbook запускаются на управляющей машине.
- Все новые приложения доступны только внутри Kubernetes-сети. Публичный доступ должен идти через управляемый ingress/reverse proxy с TLS и аутентификацией.
- AWX и Jenkins нуждаются в проверенных backup/restore сценариях. Persistent volume сам по себе не является backup.
- Kafka chart/Operator и версии приложений закреплены переменными. Планируйте обновление как отдельное изменение с проверкой совместимости, CRD и восстановлением.
- Не кладите Vault password, wireguard private keys, Kubernetes kubeconfig или service credentials в Git.

## Официальная документация

- [AWX Operator](https://docs.ansible.com/projects/awx-operator/en/latest/) · [Helm chart](https://ansible-community.github.io/awx-operator-helm/)
- [Strimzi Operator](https://strimzi.io/docs/operators/latest/full/overview) · [Kafka releases](https://kafka.apache.org/downloads/)
- [Jenkins Docker installation](https://www.jenkins.io/doc/book/installing/docker/) · [Kubernetes installation](https://www.jenkins.io/doc/book/installing/kubernetes/)
- [Ansible Kubernetes collection](https://docs.ansible.com/projects/ansible/latest/collections/kubernetes/core/)
