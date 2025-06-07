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

---

## <a name="#2-—-выносить-все-в-переменные"></a>2 — Выносим всё в переменные

* Хардкод заменён на переменные с префиксом `vm_web_...`.
* План без изменений.

![terraform plan](https://github.com/asad-bekov/hw-23/raw/main/img/3.png)

---

## <a name="#3-—-вторая-вм-и-подсеть"></a>3 — Вторая ВМ и подсеть b

* Добавлена подсеть **develop‑b (10.0.2.0/24)**.
* ВМ **develop‑2core‑db** развернута в `ru‑central1‑b`.

![apply +2](https://github.com/asad-bekov/hw-23/raw/main/img/4.png)

---

## <a name="#4-—-outputs"></a>4 — Outputs

Output `instances_info` показывает IP и FQDN обеих ВМ.

![terraform output](https://github.com/asad-bekov/hw-23/raw/main/img/5.png)

---

## <a name="#5-—-locals"></a>5 — Locals

* Имена ВМ формируются локалами: `${var.vpc_name}-${var.vm_*_cores}core-{web|db}`.
* План без изменений ресурсов.

![output после locals](https://github.com/asad-bekov/hw-23/raw/main/img/6.png)

---

## <a name="#6-—-map-object-и-общая-metadata"></a>6 — Map/Object и общая metadata

* Ресурсы ВМ описаны одной картой `var.vms_resources`.
* Metadata — общий `local.vm_metadata_common`.

![apply 0/0/0](https://github.com/asad-bekov/hw-23/raw/main/img/7.png)

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

---

## <a name="#9-—-nat-gateway-без-публичных-ip"></a>9 — NAT Gateway (без публичных IP)

* Создан `yandex_vpc_gateway.nat_gw` + `yandex_vpc_route_table.rt_via_nat`.
* У обоих ВМ `nat = false`, но интернет работает.

![У обоих ВМ `nat = false`](https://github.com/asad-bekov/hw-23/raw/main/img/9.1.png)

| Проверка | ВМ web | ВМ db |
|---|---|---|
| `nc -vz 1.1.1.1 443` | *succeeded* | *succeeded* |
| `curl -I https://netology.ru` | `HTTP/2 302` | `HTTP/2 302` |


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
