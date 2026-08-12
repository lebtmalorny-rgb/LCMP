# Управление питанием compute-хостов через Masakari, Mistral, Ironic и Redfish

## Краткое заключение

Задача — безопасно управлять питанием физических compute-хостов OpenStack при плановых работах и авариях, не допуская потери ВМ, повторного запуска одной ВМ на двух хостах и неконтролируемого выключения узлов.

Для этого предлагается разделить ответственность:

- **Mistral** оркестрирует плановое выключение, перезагрузку и возврат хоста;
- **Masakari** управляет аварийным recovery и evacuation;
- **Ironic** хранит Redfish-параметры хостов и выполняет команды питания;
- **Redfish/BMC** непосредственно включает, выключает и перезагружает сервер;
- **Horizon** предоставляет оператору единый интерфейс запуска и контроля операций.

Ключевой принцип аварийного workflow: evacuation разрешается только после подтверждённого физического выключения отказавшего хоста.

## Рекомендуемая архитектура

**Плановые операции:**

```text
Horizon
  → Mistral
    → Masakari: maintenance
    → Nova: disable / drain
    → Ironic
    → Redfish
    → проверки
    → Nova / Masakari: возврат хоста
```

**Аварийные операции:**

```text
Masakari Monitor
  → Masakari recovery workflow
    → custom ironic_fence_task
    → Ironic
    → Redfish: hard power off
    → подтверждение fencing
    → evacuation
```

Mistral рекомендуется использовать для плановых многошаговых операций.  
В аварийном пути Masakari должен обращаться к Ironic напрямую через кастомную TaskFlow-задачу, без дополнительной зависимости от Mistral.

## Роли компонентов

| Компонент | Назначение |
|---|---|
| Horizon | Пользовательский интерфейс и запуск операций |
| Mistral | Оркестрация планового выключения, перезагрузки и возврата хоста |
| Masakari | Обнаружение отказа и аварийное восстановление |
| Nova | Отключение хоста от scheduler, migration и evacuation ВМ |
| Ironic | Хранение BMC/Redfish-параметров и выполнение power operations |
| Redfish | Фактическое управление питанием через BMC |

## Общие шаги для всех workflow

1. Определить хост по единому имени:  
   `Nova hostname = Masakari host.name = Ironic Node.name`.
2. Установить блокировку операции, чтобы исключить параллельные power actions.
3. Отключить `nova-compute` от размещения новых ВМ.
4. Обработать ВМ:
   - планово — migration или штатная остановка;
   - аварийно — сначала fencing, затем evacuation.
5. Выполнить power operation через Ironic.
6. Дождаться завершения операции и проверить:
   - `power_state`;
   - `target_power_state`;
   - `last_error`.
7. Проверить состояние ОС и сервисов после включения.
8. Зафиксировать результат, снять блокировку и обновить состояния Nova/Masakari.

## Матрица workflow

| Workflow | Основной оркестратор | Операция питания | Работа с ВМ | Итоговое состояние |
|---|---|---|---|---|
| Плановое выключение | Mistral | `soft power off` | Migration/остановка до выключения | Хост выключен, Nova disabled, Masakari maintenance |
| Плановая перезагрузка | Mistral | `soft rebooting` либо `power off → power on` | Migration либо согласованный простой | Хост включён и возвращён в scheduler |
| Аварийное выключение | Masakari | `power off` | Fencing до evacuation | Хост выключен, ВМ эвакуированы |
| Внеплановая перезагрузка/возврат | Masakari + отдельный recovery workflow | `power off → power on` | Evacuation до возврата хоста | Хост включён после диагностики и проверок |

## Workflow: плановое выключение

```text
Horizon
  → Mistral
  → установить Masakari on_maintenance=true
  → nova service-disable
  → мигрировать или остановить ВМ
  → проверить отсутствие активных ВМ
  → Ironic: soft power off
  → дождаться power_state=power off
  → оставить Nova disabled и Masakari maintenance
```

Переход к жёсткому `power off` допускается только по явно заданной политике и после тайм-аута graceful shutdown.

## Workflow: плановая перезагрузка

```text
Horizon
  → Mistral
  → установить Masakari on_maintenance=true
  → nova service-disable
  → мигрировать ВМ либо подтвердить допустимый простой
  → Ironic: soft rebooting
       или
    Ironic: power off → подтвердить off → power on
  → проверить ОС, libvirt, nova-compute и сетевые/storage-агенты
  → nova service-enable
  → установить Masakari on_maintenance=false
```

Вариант `power off → подтверждение → power on` обеспечивает более контролируемую точку проверки.

## Workflow: аварийное выключение и evacuation

```text
Masakari Monitor
  → создать host-failure notification
  → disable_compute_service_task
  → custom ironic_fence_task
      → найти Ironic Node
      → выполнить hard power off
      → дождаться power_state=power off
      → проверить target_power_state и last_error
  → prepare_HA_enabled_instances_task
  → evacuate_instances_task
```

Критическое правило:

```text
Power off не подтверждён
  → workflow завершается с ошибкой
  → evacuation не выполняется
```

Это режим **fail closed**, исключающий одновременный запуск одной ВМ на исходном и новом хосте.

## Workflow: внеплановая перезагрузка / возврат хоста

Прямой `reboot` отказавшего хоста не должен заменять fencing.

```text
Обнаружение отказа
  → hard power off
  → подтверждение fencing
  → evacuation ВМ
  → диагностика или ремонт
  → Ironic: power on
  → проверить ОС и все compute-сервисы
  → убедиться в отсутствии старых активных ВМ
  → nova service-enable
  → снять Masakari maintenance
```

Возврат хоста рекомендуется выполнять отдельным Mistral workflow или отдельной операторской процедурой.

## Хранение Redfish-данных в Ironic

Для каждого compute-хоста создаётся Ironic `Node`:

```text
driver = redfish
driver_info:
  redfish_address
  redfish_system_id
  redfish_username
  redfish_password
  redfish_verify_ca
```

Рекомендуемые параметры эксплуатации:

```text
Node.name = Nova/Masakari hostname
provision_state = manageable
network_interface = noop
```

Такой узел используется только как реестр BMC и power backend. Его не следует переводить в `available` или запускать для него cleaning/provisioning.

## Регистрация хостов в Ironic и роль Kolla-Ansible

Kolla-Ansible разворачивает и настраивает сервис Ironic, но **не создаёт автоматически Ironic Node из группы `[compute]` своего inventory**.

Следует разделять два вида inventory:

```text
Kolla-Ansible inventory
  → определяет, где запущены ironic-api, ironic-conductor,
    база данных и другие сервисы OpenStack

Ironic inventory
  → содержит управляемые физические серверы,
    их Redfish/IPMI-параметры и состояние питания
```

### Настройка самого Ironic через Kolla-Ansible

Пример включения сервиса:

```yaml
# /etc/kolla/globals.yml
enable_ironic: "yes"
```

При необходимости аппаратные интерфейсы добавляются через пользовательскую конфигурацию:

```ini
# /etc/kolla/config/ironic.conf

[DEFAULT]
enabled_hardware_types = ipmi,redfish
enabled_power_interfaces = ipmitool,redfish
enabled_management_interfaces = ipmitool,redfish
enabled_network_interfaces = flat,noop
```

После изменения конфигурации она применяется обычными операциями Kolla-Ansible.

### Как данные о хостах попадают в Ironic

Записи создаются через **Ironic Bare Metal API** и сохраняются в базе Ironic. Для enrollment можно использовать:

- CLI `openstack baremetal`;
- REST API Ironic;
- `openstacksdk` или `python-ironicclient`;
- отдельный Ansible playbook;
- синхронизацию из CMDB или NetBox.

Напрямую изменять базу Ironic не следует.

Пример регистрации существующего compute-хоста для power-only управления:

```bash
openstack baremetal node create \
  --name compute-01 \
  --driver redfish \
  --network-interface noop \
  --power-interface redfish \
  --management-interface redfish \
  --driver-info redfish_address=https://10.20.0.21 \
  --driver-info redfish_system_id=/redfish/v1/Systems/1 \
  --driver-info redfish_username=ironic-power \
  --driver-info redfish_password="${REDFISH_PASSWORD}"
```

Параметры BMC сохраняются в `driver_info` объекта `Node`:

```yaml
redfish_address: https://10.20.0.21
redfish_system_id: /redfish/v1/Systems/1
redfish_username: ironic-power
redfish_password: secret
redfish_verify_ca: /etc/ironic/bmc-ca.pem
```

При необходимости можно добавить MAC-адреса обычных сетевых интерфейсов сервера:

```bash
openstack baremetal port create \
  52:54:00:aa:bb:01 \
  --node compute-01
```

Ironic `Port` описывает NIC хоста, а не Redfish-интерфейс BMC. Для сценария, ограниченного управлением питанием, сетевые порты могут не использоваться, если это допускают выбранные интерфейсы и проверки конфигурации.

После создания выполняются проверка и перевод узла в `manageable`:

```bash
openstack baremetal node validate compute-01
openstack baremetal node manage compute-01

openstack baremetal node show compute-01 \
  -c name \
  -c provision_state \
  -c power_state \
  -c target_power_state \
  -c last_error
```

Для действующего Nova compute-хоста рекомендуется:

```text
provision_state = manageable
network_interface = noop
```

Не следует выполнять:

```bash
openstack baremetal node provide compute-01
```

Переход в `available` относится к provisioning bare-metal ресурсов и может запустить cleaning. В рассматриваемой схеме Ironic используется только как реестр BMC и power backend.

### Рекомендуемая автоматизация enrollment

Исходные данные можно хранить в Ansible inventory или CMDB, а пароли — в Vault/secret manager:

```yaml
# host_vars/compute-01.yml

ironic_node_name: compute-01

ironic_bmc:
  driver: redfish
  address: https://10.20.0.21
  system_id: /redfish/v1/Systems/1
  username: ironic-power
  password: "{{ vault_compute_01_redfish_password }}"
  verify_ca: /etc/ironic/bmc-ca.pem
```

Отдельный idempotent enrollment/reconciliation playbook должен:

```text
прочитать хосты из inventory или CMDB
  → получить секреты из Vault
  → создать или обновить Ironic Node через API
  → при необходимости создать Ironic Port
  → перевести Node в manageable
  → проверить power interface и доступность BMC
```

Пароли Redfish не следует хранить открытым текстом в `globals.yml`, обычном inventory или Git.

Желательно использовать единое имя хоста:

```text
Kolla inventory hostname
  = Nova compute hostname
  = Masakari host.name
  = Ironic Node.name
```

Итоговая схема наполнения Ironic:

```text
Kolla-Ansible
  → разворачивает и конфигурирует Ironic

Ansible inventory / CMDB + Vault
  → отдельный enrollment playbook
  → Ironic API
  → Node + driver_info
  → Redfish BMC
```

## Основные правила

- Horizon не должен напрямую выключать compute-хост через Ironic, минуя Nova и Masakari.
- Плановые операции выполняются через Mistral.
- Аварийный fencing выполняется внутри Masakari custom TaskFlow.
- Evacuation разрешается только после подтверждённого `power off`.
- Все операции должны быть идемпотентными, иметь timeout, retry и блокировку по имени хоста.
- Redfish-пароли должны быть закрыты RBAC Ironic; предпочтителен Redfish HTTPS с проверкой CA.
