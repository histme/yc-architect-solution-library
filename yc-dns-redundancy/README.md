# Отказоустойчивая конфигурация DNS для подсетей в VPC

Данное решение позволяет настроить DNS сервис в отказоустойчивом режиме при использовании более одной зоны доступности.

## Условия для использования (prerequisities)

* существующая папка в Облаке
* установленный инструмент [**yc**](https://cloud.yandex.ru/docs/cli/quickstart)

## Описание решения

Решение обеспечивает перенаправление DNS запросов на резервный DNS сервис в другой зоне доступности в случае отказа активного DNS сервиса в локальной зоне доступности.

В процессе работы создаются следующие артефакты в инфраструктуре Облака:
1) **Сеть (VPC)** - общий объект для всех зон доступности *(атрибуты сети задаются во входных параметрах)*
2) **IPv4-подсеть/подсети** по одной для каждой зоны доступности *(атрибуты подсети задаются во входных параметрах)*\
Подразумевается, что целевой сервис будет иметь резевирование и размещаться минимум в 2х зонах доступности.

При создании подсети (по умолчанию) в список DNS серверов этой подсети всегда добавляется 2ой адрес из IPv4 префикса этой сети. Подробнее об этом можно почитать в главе "Подсети" раздела документации ["Облачные сети и подсети"](https://cloud.yandex.ru/docs/vpc/concepts/network).

При создании IPv4-подсети, в атрибут **domain_name_servers** добавляется список IPv4 адресов DNS серверов в разных зонах доступности.
Например, при следующей конфигурации подсетей:
```json
[
    { name = "sub1", zone = "ru-central1-a", prefix = "10.1.1.128/25" },
    { name = "sub2", zone = "ru-central1-b", prefix = "10.2.2.0/24" },
    { name = "sub3", zone = "ru-central1-c", prefix = "10.3.3.64/28" },
]
```
список DNS-серверов для каждой из подсетей выше будет таким:
```yaml
sub1: [10.1.1.130, 10.2.2.2, 10.3.3.66]
sub2: [10.2.2.2, 10.1.1.130, 10.3.3.66]
sub3: [10.3.3.66, 10.1.1.130, 10.2.2.2]
```

## Использование

### Входные параметры [**variables.tf**](./variables.tf)
* `net_name` - имя сети (VPC) в Облаке. Сеть будет создана во всех зонах доступности автоматически.
* `subnet_list` - список IPv4 подсетей, которые будут создаваться в заданной параметром `net_name` сети.\
Каждая из подсетей описывается набором атрибутов:
    * `name` - имя подсети
    * `zone` - имя зоны доступности
    * `prefix` - IPv4 префикс для создаваемой сети

### Развертывание
1. Проверить конфигурацию рабочего окружения YC в файле [**env-yc-prod.sh**](./env-yc-prod.sh). Изменить при необходимости имя профиля "prod" на необходимое.
2. Активировать рабочее окружение
    ```bash
    source env-yc-prod.sh
    ```
3. Инициализировать Terraform
    ```bash
    terraform init
    ```
4. Создать подсети в заданной сети с отказоустойчивым DNS сервисом
    ```bash
    terraform apply
    ```

### Тестирование
1. Сохраним внешний адрес созданной VM в переменную окружения
    ```bash
    export vm_ext_ip=`terraform output vm_ext_ip_address | sed 's/"//g'`
    ```
2. Подключимся к VM
    ```bash
    ssh -oStrictHostKeyChecking=no ubuntu@$vm_ext_ip
    ```
3. Проверим работу DNS сервиса
    ```bash
    dig www.yandex.ru
    ```
4. Отключим все 3 DNS сервера с помощью инструмента iptables
    ```bash
    sudo iptables -A OUTPUT -d 10.1.1.130 -j DROP
    sudo iptables -A OUTPUT -d 10.2.2.2 -j DROP
    sudo iptables -A OUTPUT -d 10.3.3.66 -j DROP
    sudo iptables -L -v -n
    ```
5. Убедимся, что DNS запросы теперь остаются без ответа - DNS сервис ожидаемо не работает.
    ```bash
    dig www.yandex.ru
    ```
6.  Разрешим трафик на 2ой DNS сервер в списке, удалив запрещающее правило, которое мы создали в п.4
    ```bash
    sudo iptables -D OUTPUT -d 10.2.2.2 -j DROP
    ```
7. Убедимся, что DNS сервис снова работает.
    ```bash
    dig www.yandex.ru
    ```

### Завершение работы
```bash
unset vm_ext_ip
terraform destroy
```