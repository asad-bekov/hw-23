# Домашнее задание к занятию «Основы Terraform. Yandex Cloud»

> Репозиторий: **hw‑23**  
> Выполнил: **Асадбеков Асадбек**  
> Дата выполнения: **июнь 2025**

---

## Содержание
| № | Раздел | Кратко |
|---|--------|--------|
| 0 | [Чек‑лист](#0-—-чек-лист-перед-работой) | подготовка окружения |
| 1 | [Задание 1](#1-—-первый-запуск-terraform) | первая ВМ, ssh, ответы |
| 2 | [Задание 2](#2-—-выносить-все-в-переменные) | переменные и validate |
| 3 | [Задание 3](#3-—-вторая-вм-и-подсеть) | db‑ВМ, подсеть b |
| 4 | [Задание 4](#4-—-outputs) | вывод IP/FQDN |
| 5 | [Задание 5](#5-—-locals) | генерация имён ВМ |
| 6 | [Задание 6](#6-—-map-object-и-общая-metadata) | ресурсы картой |
| 7 | [Задание 7](#7-—-terraform-console-—-list--map) | работа в console |
| 8 | [Задание 8](#8-—-console-—-вложенные-структуры) | list‑comprehension |
| 9 | [Задание 9](#9-—-nat-gateway-без-публичных-ip) | NAT Gateway |

---

## <a name="#0-—-чек-лист-перед-работой"></a>0 — Чек‑лист перед работой
* **Terraform 1.8.4** установлен
* **YC CLI 0.146** (обновлён до >0.200 при работе с OS Login)
* Ключ сервис‑аккаунта: `sa-key.json`
* Ключи SSH: `~/.ssh/yandex_rsa*`

---

## <a name="#1-—-первый-запуск-terraform"></a>1 — Первый запуск Terraform

### Скриншоты
| Описание | Изображение |
|---|---|
| ЛК Yandex Cloud — созданная ВМ | ![VM card](https://github.com/asad-bekov/hw-23/raw/main/img/1.png) |
| `curl ifconfig.me` внутри ВМ | ![curl ifconfig.me](https://github.com/asad-bekov/hw-23/raw/main/img/2.png) |

### Ответы на вопросы
| Вопрос | Ответ |
|---|---|
| **Что за синтаксические ошибки были в коде?** | Пропущенная запятая, лишняя скобка, опечатка `standart` → `standard`. |
| **Для чего `preemptible = true` и `core_fraction = 5`?** | Дают «дешёвую» прерываемую ВМ с 5 % гарантированного CPU — идеально для учебных лабораторий. |

<details> 
<summary>
<code>main.tf</code> (минимум для одной ВМ)</summary>

```hcl
resource "yandex_vpc_network" "develop" {
  name = "develop"
}

resource "yandex_vpc_subnet" "develop" {
  name           = "develop"
  zone           = "ru-central1-a"
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = ["10.0.1.0/24"]
}

data "yandex_compute_image" "ubuntu" {
  family = "ubuntu-2004-lts"
}

resource "yandex_compute_instance" "platform" {
  name        = "netology-develop-platform-web"
  platform_id = "standard-v2"

  resources {
    cores         = 2
    memory        = 1
    core_fraction = 5
  }

  boot_disk {
    initialize_params { image_id = data.yandex_compute_image.ubuntu.image_id }
  }

  scheduling_policy { preemptible = true }

  network_interface {
    subnet_id = yandex_vpc_subnet.develop.id
    nat       = true
  }

  metadata = {
    serial-port-enable = 1
    ssh-keys           = "ubuntu:${var.vms_ssh_root_key}"
  }
}
```
</details>
---

## <a name="#2-—-выносить-все-в-переменные"></a>2 — Выносим всё в переменные

* Хардкод заменён на переменные с префиксом `vm_web_...`.
* План без изменений.

![terraform plan](https://github.com/asad-bekov/hw-23/raw/main/img/3.png)

<details> <summary><code>variables.tf</code> (фрагмент с web‑переменными)</summary>

```hcl
variable "vm_web_name" {
  type    = string
  default = "netology-develop-platform-web"
}

variable "vm_web_platform_id" {
  type    = string
  default = "standard-v2"
}

variable "vm_web_cores" {
  type    = number
  default = 2
}

variable "vm_web_memory" {
  type    = number
  default = 1
}

variable "vm_web_core_fraction" {
  type    = number
  default = 5
}

variable "vm_web_preemptible" {
  type    = bool
  default = true
}

```
и изменение в `main.tf`:

```hcl
name        = var.vm_web_name
platform_id = var.vm_web_platform_id
resources {
  cores         = var.vm_web_cores
  memory        = var.vm_web_memory
  core_fraction = var.vm_web_core_fraction
}
scheduling_policy { preemptible = var.vm_web_preemptible }

```
</details>

---

## <a name="#3-—-вторая-вм-и-подсеть"></a>3 — Вторая ВМ и подсеть b

* Добавлена подсеть **develop‑b (10.0.2.0/24)**.
* ВМ **develop‑2core‑db** развернута в `ru‑central1‑b`.

![apply +2](https://github.com/asad-bekov/hw-23/raw/main/img/4.png)
<details> <summary>подсеть b</summary>

```hcl
resource "yandex_vpc_subnet" "develop_ru_central1_b" {
  name           = "develop-b"
  zone           = "ru-central1-b"
  network_id     = yandex_vpc_network.develop.id
  v4_cidr_blocks = ["10.0.2.0/24"]
}

```
</details> <details> <summary>ВМ db</summary>
```hcl
resource "yandex_compute_instance" "db" {
  name        = var.vm_db_name          # "netology-develop-platform-db"
  platform_id = var.vm_db_platform_id   # "standard-v2"
  zone        = var.vm_db_zone          # "ru-central1-b"

  resources {
    cores         = var.vm_db_cores         # 2
    memory        = var.vm_db_memory        # 2
    core_fraction = var.vm_db_core_fraction # 20
  }

  boot_disk { initialize_params { image_id = data.yandex_compute_image.ubuntu.image_id } }
  scheduling_policy { preemptible = true }

  network_interface {
    subnet_id = yandex_vpc_subnet.develop_ru_central1_b.id
    nat       = true
  }

  metadata = { serial-port-enable = 1, ssh-keys = "ubuntu:${var.vms_ssh_root_key}" }
}

```
</details>
---

## <a name="#4-—-outputs"></a>4 — Outputs

Output `instances_info` показывает IP и FQDN обеих ВМ.

![terraform output](https://github.com/asad-bekov/hw-23/raw/main/img/5.png)

<details> <summary>output</summary>

```hcl
output "instances_info" {
  value = {
    for vm in [
      yandex_compute_instance.platform,
      yandex_compute_instance.db
    ] : vm.name => {
      external_ip = vm.network_interface[0].nat_ip_address
      fqdn        = vm.fqdn
    }
  }
}

```
</details>
---

## <a name="#5-—-locals"></a>5 — Locals

* Имена ВМ формируются локалами: `${var.vpc_name}-${var.vm_*_cores}core-{web|db}`.
* План без изменений ресурсов.

![output после locals](https://github.com/asad-bekov/hw-23/raw/main/img/6.png)

<details> <summary>output</summary>
```hcl
variable "vm_web_name" {
  type    = string
  default = "netology-develop-platform-web"
}

variable "vm_web_platform_id" {
  type    = string
  default = "standard-v2"
}

variable "vm_web_cores" {
  type    = number
  default = 2
}

variable "vm_web_memory" {
  type    = number
  default = 1
}

variable "vm_web_core_fraction" {
  type    = number
  default = 5
}

variable "vm_web_preemptible" {
  type    = bool
  default = true
}
```
и в ресурсах:

```hcl
name        = local.vm_web_name_local
description = local.vm_web_description

```
</details>
---

## <a name="#6-—-map-object-и-общая-metadata"></a>6 — Map/Object и общая metadata

* Ресурсы ВМ описаны одной картой `var.vms_resources`.
* Metadata — общий `local.vm_metadata_common`.

![apply 0/0/0](https://github.com/asad-bekov/hw-23/raw/main/img/7.png)

<details> <summary><code>variables.tf</code> (карты)</summary>
```hcl
variable "vms_resources" {
  type = map(object({
    cores         = number
    memory        = number
    core_fraction = number
    hdd_size      = number
    hdd_type      = string
  }))

  default = {
    web = {
      cores         = 2
      memory        = 1
      core_fraction = 5
      hdd_size      = 5
      hdd_type      = "network-hdd"
    }
    db = {
      cores         = 2
      memory        = 2
      core_fraction = 20
      hdd_size      = 10
      hdd_type      = "network-ssd"
    }
  }
}
```
Использование:

```hcl
resources {
  cores         = var.vms_resources["web"].cores
  memory        = var.vms_resources["web"].memory
  core_fraction = var.vms_resources["web"].core_fraction
}
boot_disk { initialize_params { size = var.vms_resources["web"].hdd_size } }
metadata = local.vm_metadata_common

```
</details>
---

## <a name="#7-—-terraform-console-—-list--map"></a>7 — Terraform console — list & map

| Команда | Вывод |
|---|---|
| `local.test_list[1]` | `"staging"` |
| `length(local.test_list)` | `3` |
| `local.test_map["admin"]` | `"John"` |
| *интерполяция* | `"John is admin for production server based on OS ubuntu-20-04 with 10 vcpu, 40 ram and 4 virtual disks"` |

![console 7](https://github.com/asad-bekov/hw-23/raw/main/img/8.png)
---

## <a name="#8-—-console-—-вложенные-структуры"></a>8 — Вложенные структуры и list‑comprehension

* Полный `type` описан как `list(map(tuple([string,string])))`.
* Команды для получения ответов

![console 8](https://github.com/asad-bekov/hw-23/raw/main/img/9.png)

```hcl
variable "test" {
  type = list(
    map(
      tuple([string,string])
    )
  )
}

terraform console
var.test[0]["devl"][0]
var.test[0][keys(var.test[0])[0]][0]
[for server in var.test : values(server)[0][0]] 
{ for srv in var.test : keys(srv)[0] => values(srv)[0][0] } 
([for s in var.test : values(s)[0][0]])[0]	
exit
```
</details>
---

## <a name="#9-—-nat-gateway-без-публичных-ip"></a>9 — NAT Gateway (без публичных IP)

* Создан `yandex_vpc_gateway.nat_gw` + `yandex_vpc_route_table.rt_via_nat`.
* У обоих ВМ `nat = false`, но интернет работает.

![У обоих ВМ `nat = false`](https://github.com/asad-bekov/hw-23/raw/main/img/9.1.png)

| Проверка | ВМ web | ВМ db |
|---|---|---|
| `nc -vz 1.1.1.1 443` | *succeeded* | *succeeded* |
| `curl -I https://netology.ru` | `HTTP/2 302` | `HTTP/2 302` |

<details> <summary><code>nat.tf</code></summary>

```hcl
resource "yandex_vpc_gateway" "nat_gw" {
  name = "main-nat-gw"
  shared_egress_gateway {}
}

resource "yandex_vpc_route_table" "rt_via_nat" {
  name       = "rt-via-nat"
  network_id = yandex_vpc_network.develop.id

  static_route {
    destination_prefix = "0.0.0.0/0"
    gateway_id         = yandex_vpc_gateway.nat_gw.id
  }
}

# в обеих существующих подсетях:
route_table_id = yandex_vpc_route_table.rt_via_nat.id

# и в NIC ВМ
nat = false
```
</details>

![serial console web](https://github.com/asad-bekov/hw-23/raw/main/img/10.png)
---
![serial console db](https://github.com/asad-bekov/hw-23/raw/main/img/11.png)

> **Важно:** ICMP через NAT‑Gateway не поддерживается — `ping` во внешний
> интернет будет теряться; проверяем только TCP/UDP.

---

## Заключение

* Весь код Terraform находится в каталоге **src**.  
* Каждая задача выполнена без хардкода — переменные, локалы, карты.  
* ВМ размещены в приватных подсетях и имеют выход во внешний мир через NAT Gateway.

```shell
terraform --version
Terraform v1.8.4
```
![terraform version](https://github.com/asad-bekov/hw-23/raw/main/img/12.png)
---
