# PVS LCM — каталог контейнеров

## 1. Назначение документа

Документ кратко описывает runtime-контейнеры, показанные на архитектурной схеме PVS LCM:

- название и назначение;
- роль и состояние включения;
- место запуска;
- основные функции;
- зависимости от других контейнеров и сервисов;
- сетевые порты;
- основные источники конфигурации.

> **Правило нотации схемы:** один кубик обозначает один основной долгоживущий runtime-контейнер. `Deployment`, `StatefulSet`, `DaemonSet`, `Pod`, `Service`, `Ingress` и `PVC` являются Kubernetes-ресурсами и не считаются контейнерами.

## 2. Общая модель развёртывания и конфигурации

Основные контейнеры PVS разворачиваются одним Helm-релизом в общем namespace control-plane-кластера PVS. Имя Kubernetes-объекта обычно формируется как `<release>-<component>`, например `pvs-backend`.

Конфигурация разделена на несколько уровней:

| Источник | Назначение |
|---|---|
| `pvs-install.yaml` | Вход оператора: namespace, ingress, реестр образов, внешние Vault/LDAP/OpenSearch/monitoring, параметры datastore-компонентов |
| `values.yaml` + production overlays | Включение подчартов, образы, реплики, ресурсы, Service, probes, PVC, NetworkPolicy |
| `conf.yaml` | Runtime-настройки приложений, адреса интеграций, режимы auth/TLS, ссылки `vault:` |
| `rbac.yaml` | Роли, права и привязки LDAP-групп; используется backend |
| Vault / Kubernetes Secret | Пароли, токены, HMAC-секреты, SSH-ключи, credentials регионов и интеграций |
| cert-manager Secret | TLS-сертификаты сервисов; обычно монтируются в `/app/certs` |

Общие переменные сертификатов прикладных сервисов:

- `TLS_CA_PATH`;
- `TLS_CERT_PATH`;
- `TLS_KEY_PATH`.

## 3. Edge и пользовательский интерфейс

### 3.1 `pvs-haproxy`

- **Назначение:** внешняя точка входа PVS, TLS-терминация и балансировка трафика.
- **Роль и включение:** `Deployment`, 2 реплики; включён в production-профиле.
- **Где запускается:** namespace Helm-релиза PVS в control-plane Kubernetes-кластере.
- **Функции:** принимает HTTP/HTTPS, завершает внешний TLS, направляет запросы к ingress-nginx или внутренним Service.
- **Зависимости:** `Service` типа `LoadBalancer`, DNS, TLS Secret, ingress-nginx и внутренние HTTP-сервисы PVS.
- **Порты:** контейнер `8080/8443`; внешний Service `80/443`; stats `8404` только на `127.0.0.1` внутри Pod.
- **Конфигурация:** Helm-подчарт `haproxy`, `tls.enabled`, Service/targetPort, сертификаты, health-check и backend-маршруты.

### 3.2 `ingress-nginx-controller`

- **Назначение:** маршрутизация HTTP-запросов по host/path внутри Kubernetes.
- **Роль и включение:** кластерный add-on, а не контейнер прикладного Helm-подчарта PVS; может устанавливаться инсталлятором в режиме `auto`, `install` или использоваться существующий (`skip`).
- **Где запускается:** обычно в отдельном namespace ingress-контроллера, например `ingress-nginx`.
- **Функции:** применяет Kubernetes-объекты `Ingress`; направляет `/api/*` в backend, `/nbox-api/*` в nbox, `/scheduler-api/*` в scheduler, Git API — в git-integration, остальные пути — во frontend.
- **Зависимости:** `IngressClass`, Kubernetes `Ingress`, Service компонентов PVS, DNS и TLS Secret.
- **Порты:** определяются установленным add-on chart; точные containerPort в исходном описании PVS не зафиксированы.
- **Конфигурация:** `addons.ingressNginx`, `ingress.host`, `ingress.className`, `ingress.tls.*`, `.Values.ingress.enabled`.

### 3.3 `pvs-frontend`

- **Назначение:** веб-интерфейс PVS.
- **Роль и включение:** `Deployment`, 3 реплики; включён в production-профиле.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** nginx отдаёт React SPA; интерфейс предоставляет экраны регионов, Day0/Day1/Day2/Day3, RBAC, NBox, Git, Scheduler и Kolla-Ansible. Постоянное состояние не хранит.
- **Зависимости:** HAProxy/Ingress, backend REST API через относительный путь `/api/*`.
- **Порт:** `8080`.
- **Конфигурация:** Helm-подчарт `frontend`, nginx-конфигурация, образ/реплики/resources/probes, ingress host, TLS и NetworkPolicy.

## 4. API, управление и исполнение

### 4.1 `pvs-backend`

- **Назначение:** центральный Platform API PVS.
- **Роль и включение:** stateless `Deployment`, 3 реплики; включён в production-профиле.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** REST API, регионы, пользователи, сессии, LDAP-аутентификация, MFA, RBAC, аудит, задания, health-score, maintenance/freeze и фасад интеграций. Тяжёлые операции над OpenStack-регионами делегирует `pvs-openstack-hub`.
- **Зависимости:** PostgreSQL, Redis, etcd, Vault, LDAP/AD, `pvs-openstack-hub`, `pvs-nbox`, `pvs-scheduler`, `pvs-ansible-orch`, `pvs-git-integration`, `pvs-registry-integration`, внешняя система мониторинга.
- **Порт:** `8000`.
- **Конфигурация:** chart `backend`; секции `global`, `backend`, `integrations` в `conf.yaml`; `rbac.yaml`; DB/Redis URL; LDAP; Vault refs; TLS/mTLS; HMAC и webhook-настройки. Устаревшие Celery worker/beat выключены по умолчанию.

### 4.2 `pvs-openstack-hub`

- **Назначение:** управление жизненным циклом OpenStack-регионов.
- **Роль и включение:** stateless `Deployment`, 3 реплики; включён в production-профиле.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** авторитетный рендер Kolla-конфигурации, Day0–Day3 операции, prechecks, deploy/upgrade/reconfigure, rollback-предложения, операции с узлами, OpenStack API proxy, обработка callback от исполнителей.
- **Зависимости:** PostgreSQL, etcd, `pvs-git-service`, `pvs-nbox`, `pvs-scheduler`, `pvs-ansible-orch`, registry integration, Vault, региональный `pvs-openstack-agent` или прямой OpenStack API.
- **Порт:** `8700`.
- **Конфигурация:** chart `openstack-hub`; `conf.yaml`; адреса интеграций; etcd; Vault credentials; режим региона `agent`/`direct`; параметры reconcile, release-check, compensation и agent health. `listener.enabled=false`.

### 4.3 `pvs-nbox`

- **Назначение:** DCIM/IPAM, инвентарь и подготовка оборудования региона.
- **Роль и включение:** `Deployment`, 3 реплики; включён в production-профиле.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** площадки, стойки, устройства, интерфейсы, VLAN, IPAM, BMC/PXE provisioning, drift detection, генерация и проверка Kolla inventory.
- **Зависимости:** PostgreSQL, backend, openstack-hub, scheduler и Vault для BMC credentials.
- **Порт:** `8600`.
- **Конфигурация:** chart `nbox`; секция `nbox` в `conf.yaml`; DB URL; BMC/Vault; provisioning; inventory validator; TLS и NetworkPolicy.

### 4.4 `pvs-scheduler`

- **Назначение:** распределённый планировщик фоновых и lifecycle-заданий.
- **Роль и включение:** `Deployment`, 2 активные реплики; включён в production-профиле.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** cron/interval-расписания, pipelines, deadline watchdog, DLQ, retry/backoff и dispatch задач. Двойной запуск предотвращается PostgreSQL-lock’ами; etcd используется для discovery, а не для выбора лидера.
- **Зависимости:** PostgreSQL, Redis/Celery, etcd, backend, openstack-hub, unified-worker, ansible-orch и Vault.
- **Порт:** `8100`.
- **Конфигурация:** chart `scheduler`; `conf.yaml`; tick interval, deadlines, retry/DLQ, `etcd_hosts`, DB/Redis, worker credentials, HMAC и TLS.

### 4.5 `pvs-ansible-orch`

- **Назначение:** оркестрация Ansible и Kolla LCM запусков.
- **Роль и включение:** `Deployment`, 1 реплика; включён в production-профиле.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** создаёт и отслеживает Ansible/Kolla runs, передаёт выполнение worker-агентам, хранит transcript, события и артефакты, поддерживает cancel, safe-mode replay, recovery и Day3 schedules.
- **Зависимости:** PostgreSQL, Redis, etcd discovery, scheduler, unified-worker/региональные worker-agent, backend, Git, NBox, registry и Vault.
- **Порт:** `8300`.
- **Конфигурация:** chart `ansible-orch`; `conf.yaml`; HMAC-секреты, worker discovery, DB/Redis, execution environments, Vault credentials, TLS и параметры reconciliation.

### 4.6 `pvs-unified-worker`

- **Назначение:** единый исполнитель фоновых задач PVS.
- **Роль и включение:** `Deployment`, 3 реплики; включён в production-профиле. Основной контейнер использует код `worker-agent`.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** принимает `unified.execute`, маршрутизирует задачи, запускает Ansible и Kolla-Ansible, клонирует Git-репозитории, ведёт локальную рабочую область и возвращает результаты вызывающему сервису.
- **Зависимости:** ansible-orch, scheduler, backend, registry, Redis, etcd, Vault, Git-репозитории и SSH-доступ к целевым узлам.
- **Порт:** `9200` — HTTP control/metrics.
- **Конфигурация:** chart `unified-worker`; список `capabilities`; workspace, persistence/retention/GC; Redis; HMAC; SSH/Vault credentials; TLS; resources/probes.

### 4.7 `pvs-git-integration`

- **Назначение:** единый адаптер Git-провайдеров.
- **Роль и включение:** stateless `Deployment`, 3 реплики; включён в production-профиле.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** унифицирует PVS Git, GitLab, Gitea и Bitbucket; выполняет операции с ветками, commit, merge request, webhook, GitOps-редактирование и сканирование Ansible-контента.
- **Зависимости:** PostgreSQL, `pvs-git-service` или внешние Git-провайдеры, Vault, backend для аудита, scheduler и etcd heartbeat.
- **Порт:** `8200`.
- **Конфигурация:** chart `git-integration`; `conf.yaml`; provider definitions; Vault credentials; `GIT_PVS_GIT_SERVICE_URL`; health-check interval; TLS и NetworkPolicy.

### 4.8 `pvs-git-service`

- **Назначение:** хранилище bare Git-репозиториев конфигурации PVS и регионов.
- **Роль и включение:** `StatefulSet`, 2 реплики; включён в production-профиле; PVC на каждую реплику.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** Smart HTTP clone/fetch/push, merge request, branch protection, webhook, snapshot/restore, поиск и репликация репозиториев. Мутирующие запросы обслуживает лидер.
- **Зависимости:** PostgreSQL, PVC, etcd leader election, Vault, backend auth, git-integration, openstack-hub и `git-service label-updater`.
- **Порт:** `8500`.
- **Конфигурация:** chart `git-service`; replicas; StorageClass/PVC; путь `/var/lib/pvs/git-repos`; etcd; replication; quotas; Vault; TLS; Service для write-leader.

### 4.9 `git-service label-updater`

- **Назначение:** направлять запись на лидирующую реплику Git Service.
- **Роль и включение:** отдельный служебный `Deployment`; создаётся при `git-service.replicaCount > 1`.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** опрашивает `GET /master`, определяет лидера и через Kubernetes API устанавливает Pod label `role=master`; alias-Service выбирает Pod с этим label.
- **Зависимости:** `pvs-git-service`, Kubernetes API, ServiceAccount и RBAC.
- **Порты:** публичный порт отсутствует; исходящий вызов к Git Service использует `8500`.
- **Конфигурация:** шаблон label-updater в chart `git-service`, адрес Git API, период опроса, ServiceAccount/RBAC и Service selector.

### 4.10 `pvs-registry-integration`

- **Назначение:** интеграция PVS с реестрами контейнерных образов.
- **Роль и включение:** `Deployment`, 3 реплики; включён в production-профиле.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** Docker Registry API v2 proxy, проверка наличия образов релиза, copy/promote/retag/delete, approval workflow, naming/age policy и cosign verification.
- **Зависимости:** PostgreSQL, внешний Registry/Nexus, openstack-hub, scheduler/unified-worker и Vault.
- **Порт:** `8400`.
- **Конфигурация:** chart `registry`; `conf.yaml`; `proxy_enabled`; registry sources; naming/age policies; approvals; credentials из Vault; DB URL; TLS и NetworkPolicy.

## 5. Хранилища и координация

### 5.1 `pvs-postgresql` / Patroni

- **Назначение:** основная реляционная база данных платформы в HA-режиме.
- **Роль и включение:** `StatefulSet`, 3 реплики; включён в production-профиле.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере; отдельный PVC на Pod.
- **Функции:** хранит данные прикладных сервисов в отдельных базах; Patroni управляет leader election, replication и failover.
- **Зависимости:** etcd как DCS, PVC, TLS-сертификаты и `pvs-postgresql-haproxy` для клиентской маршрутизации.
- **Порты:** `5432` PostgreSQL; `8008` Patroni REST API.
- **Конфигурация:** chart `postgresql`; `datastores.postgresql.replicas/storage`; DB/replication credentials; StorageClass/PVC; Patroni/etcd endpoints; TLS.

### 5.2 `pvs-postgresql-haproxy`

- **Назначение:** маршрутизация SQL-клиентов на текущего лидера Patroni.
- **Роль и включение:** отдельный `Deployment`, 2 реплики; включён вместе с PostgreSQL.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** проверяет Patroni REST endpoint `/primary` на каждой реплике и передаёт соединения к текущему лидеру.
- **Зависимости:** все Pod `pvs-postgresql`, Patroni API и клиенты БД всех прикладных сервисов.
- **Порты:** клиентский SQL `5432`; health-check Patroni `8008`.
- **Конфигурация:** chart `postgresql`; список backend-реплик, `/primary`, timeouts, TLS и Service endpoint.

### 5.3 `pvs-redis`

- **Назначение:** кэш, пользовательские сессии и брокер/результаты очередей.
- **Роль и включение:** `StatefulSet`, 3 реплики master/replica; включён в production-профиле; используется PVC.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** key-value storage, cache, Redis pub/sub и Celery broker/result backend.
- **Зависимости:** Redis Sentinel, redis label-updater и клиенты backend/scheduler/worker/ansible-orch.
- **Порт:** `6379`.
- **Конфигурация:** chart `redis`; `datastores.redis.replicas`; persistence; auth; master/replica; TLS; Service aliases.

### 5.4 `pvs-redis-sentinel`

- **Назначение:** обнаружение Redis master и автоматическое переключение при отказе.
- **Роль и включение:** `Deployment`, 3 реплики; включён вместе с Redis.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** мониторит master/replica, достигает quorum и инициирует failover.
- **Зависимости:** Pod `pvs-redis` и redis label-updater.
- **Порт:** `26379`.
- **Конфигурация:** chart `redis`; Sentinel quorum, monitor name, timeouts, Redis credentials и TLS.

### 5.5 `redis label-updater`

- **Назначение:** поддерживать label текущего Redis master для alias-Service.
- **Роль и включение:** отдельный служебный `Deployment`; включён при HA-топологии Redis.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** читает состояние Sentinel и через Kubernetes API устанавливает `role=master` на текущем master Pod.
- **Зависимости:** Redis Sentinel, Kubernetes API, ServiceAccount/RBAC и Redis alias-Service.
- **Порты:** собственного публичного порта нет; обращается к Sentinel на `26379`.
- **Конфигурация:** chart `redis`; Sentinel endpoint, период опроса, ServiceAccount/RBAC и Service selector.

### 5.6 `pvs-etcd`

- **Назначение:** распределённая координация и service discovery.
- **Роль и включение:** `StatefulSet`, 3 реплики; включён в production-профиле; PVC на Pod.
- **Где запускается:** namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** Patroni DCS; leader election для openstack-hub и git-service; heartbeat/discovery для scheduler, ansible-orch и worker.
- **Зависимости:** собственные peer-реплики, PVC, DNS и отдельный mTLS-контур.
- **Порты:** `2379` client API; `2380` peer/Raft.
- **Конфигурация:** chart `etcd`; `datastores.etcd.replicas/storage`; peer/client URLs; FQDN; PVC; CA/client/server certificates.

### 5.7 `pvs-vault`

- **Назначение:** хранение секретов платформы.
- **Роль и включение:** `StatefulSet`, 1 реплика только для dev/in-cluster режима; **выключен в production-профиле**, где используется внешний Vault.
- **Где запускается:** при включении — namespace PVS в control-plane Kubernetes-кластере.
- **Функции:** хранит DB credentials, HMAC, SSH-ключи, credentials регионов, registry и Git-провайдеров.
- **Зависимости:** PVC, TLS, auth backend и сервисы PVS, разрешающие `vault:`-ссылки.
- **Порты:** `8200` HTTPS API; `8201` cluster.
- **Конфигурация:** `pvs-install.yaml:vault.*`, `global.vault.*`, `vault.enabled`, `devMode`, KV mount/path prefix, Kubernetes auth roles/policy и `bootstrapSecrets`.

## 6. Логи и наблюдаемость

### 6.1 `fluent-bit`

- **Назначение:** сбор и доставка логов и аудит-событий.
- **Роль и включение:** `DaemonSet`, по одному Pod на каждый узел PVS-кластера; включён в production-профиле.
- **Где запускается:** на каждом Kubernetes-узле, где DaemonSet может быть запланирован.
- **Функции:** читает контейнерные/узловые логи, нормализует их, отделяет audit-поток и отправляет данные в OpenSearch.
- **Зависимости:** host log volumes, ConfigMap с parsers/Lua filters, внешний или внутренний OpenSearch, CA и credentials.
- **Порты:** `24224` forward input; `9200` metrics согласно chart; внутренний HTTP API `2020` используется для диагностики, если включён конфигурацией.
- **Конфигурация:** chart `fluentbit`; ConfigMap filters/parsers; `pvs-install.yaml:opensearch.*`; `global.opensearch.*`; пароль из Vault/Secret; TLS; retry/buffer limits.

## 7. Опциональные контейнеры, выключенные в production-профиле

### 7.1 `pvs-openldap`

- **Назначение:** демонстрационный LDAP-сервер для локального/dev-окружения.
- **Роль и включение:** `Deployment`, 1 реплика; выключен в production-профиле, где используется внешний LDAP/AD.
- **Где запускается:** namespace PVS при явном включении.
- **Функции:** предоставляет LDAP identity для демонстрационного стенда; роли PVS всё равно определяются `rbac.yaml`.
- **Зависимости:** backend, TLS Secret и данные каталога.
- **Порты:** `389` LDAP; `636` LDAPS.
- **Конфигурация:** chart `openldap`; `ldap.*`; bind password через Secret/Vault; CA; bootstrap users/groups.

### 7.2 `pvs-prometheus`

- **Назначение:** сбор метрик сервисов PVS.
- **Роль и включение:** отдельный `Deployment` внутри chart `monitoring`; выключен в production-профиле.
- **Где запускается:** namespace PVS при включённом встроенном monitoring.
- **Функции:** scrape `/metrics`, хранение временных рядов и выдача PromQL API.
- **Зависимости:** ServiceMonitor или scrape-конфигурация сервисов PVS, storage и сетевой доступ к Pod.
- **Порт:** `9090`.
- **Конфигурация:** `global.monitoring.enabled`, chart `monitoring`, scrape targets, retention/storage и ServiceMonitor.

### 7.3 `pvs-alertmanager`

- **Назначение:** обработка, группировка и маршрутизация алертов.
- **Роль и включение:** отдельный `Deployment` внутри chart `monitoring`; выключен в production-профиле.
- **Где запускается:** namespace PVS при включённом встроенном monitoring.
- **Функции:** принимает алерты Prometheus, выполняет grouping, inhibition, silence и доставку receivers.
- **Зависимости:** Prometheus и настроенные внешние receiver endpoints.
- **Порт:** `9093`.
- **Конфигурация:** chart `monitoring`, Alertmanager rules/routes/receivers и Secret с credentials.

### 7.4 `pvs-grafana`

- **Назначение:** визуализация метрик и дашборды.
- **Роль и включение:** отдельный `Deployment` внутри chart `monitoring`; выключен в production-профиле.
- **Где запускается:** namespace PVS при включённом встроенном monitoring.
- **Функции:** отображает дашборды PVS и использует Prometheus как datasource.
- **Зависимости:** Prometheus, backend при интеграционных вызовах и admin credentials.
- **Порт:** `3000`.
- **Конфигурация:** chart `monitoring`, datasource, dashboards, `monitoring.grafanaUrl/grafanaUser`, пароль из Vault/Secret.

## 8. Что не обозначается контейнерным кубиком

Следующие элементы не должны обозначаться как одиночный контейнер PVS:

| Элемент | Почему не контейнер |
|---|---|
| `Service pvs-haproxy` | Kubernetes Service, публикующий Deployment HAProxy |
| `Ingress pvs-ingress` | Kubernetes-объект маршрутизации; реальный runtime — ingress controller |
| `Deployment`, `StatefulSet`, `DaemonSet`, `Pod` | Workload/resource, который управляет одним или несколькими контейнерами |
| PVC | Персистентный том, а не процесс |
| `pvs-openstack-agent` в регионе | RPM/systemd-служба на узле региона, не контейнер control-plane Helm-релиза |
| региональный `worker-agent` | RPM-служба на целевом узле; не путать с Kubernetes-контейнером `pvs-unified-worker` |
| `kolla-ansible` | CLI/process, запускаемый worker-agent |
| `OpenStack API services` | Группа контейнеров OpenStack, развёрнутых Kolla-Ansible на узлах региона; не один контейнер PVS |

## 9. Краткая матрица портов

| Контейнер | Порт(ы) |
|---|---|
| `pvs-haproxy` | `8080`, `8443`; stats `8404`; Service `80/443` |
| `pvs-frontend` | `8080` |
| `pvs-backend` | `8000` |
| `pvs-openstack-hub` | `8700` |
| `pvs-nbox` | `8600` |
| `pvs-scheduler` | `8100` |
| `pvs-ansible-orch` | `8300` |
| `pvs-unified-worker` | `9200` |
| `pvs-git-integration` | `8200` |
| `pvs-git-service` | `8500` |
| `pvs-registry-integration` | `8400` |
| `pvs-postgresql` | `5432`, Patroni `8008` |
| `pvs-postgresql-haproxy` | `5432`; health-check `8008` |
| `pvs-redis` | `6379` |
| `pvs-redis-sentinel` | `26379` |
| `pvs-etcd` | `2379`, `2380` |
| `pvs-vault` | `8200`, `8201` |
| `fluent-bit` | `24224`, `9200`; diagnostic API `2020` при включении |
| `pvs-openldap` | `389`, `636` |
| `pvs-prometheus` | `9090` |
| `pvs-alertmanager` | `9093` |
| `pvs-grafana` | `3000` |

## 10. Источник

Документ подготовлен по материалам `DOC(1).md`, прежде всего разделам:

- «Состав платформы: компоненты и их назначение»;
- «Устройство компонентов»;
- «Архитектура»;
- «Связи между сервисами»;
- «Что развёрнуто в Kubernetes, что требуется снаружи»;
- «Конфигурация перед установкой»;
- «Логи, аудит и отладка».
