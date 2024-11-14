# Дипломная работа по профессии «Системный администратор» - Волков Дмитрий  

# Содержание
- [Задача](#Задача)  
- [Инфраструктура](#Инфраструктура)  
    - [Сайт](#Сайт)
    - [Мониторинг](#Мониторинг)
    - [Логи](#Логи)
    - [Сеть](#Сеть)
    - [Резервное копирование](#backup)
# Выполнение дипломной работы  
### Terraform
- [Инфраструктура](#terra)
    - [Сеть](#net)
    - [Группы безопасности](#group)
    - [Load Balancer](#balancer)
    - [Резервное копирование](#snapshot)
### Ansible


--- 

### <a id="Задача">Задача</a>  
Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в Yandex Cloud и отвечать минимальным стандартам безопасности: запрещается выкладывать токен от облака в git. 

### <a id="Инфраструктура">Инфраструктура</a> 
Для развёртки инфраструктуры используйте Terraform и Ansible.  

Не используйте для ansible inventory ip-адреса! Вместо этого используйте fqdn имена виртуальных машин в зоне ".ru-central1.internal". Пример: example.ru-central1.internal  

Важно: используйте по-возможности минимальные конфигурации ВМ:2 ядра 20% Intel ice lake, 2-4Гб памяти, 10hdd, прерываемая.  

Так как прерываемая ВМ проработает не больше 24ч, перед сдачей работы на проверку дипломному руководителю сделайте ваши ВМ постоянно работающими.  

Ознакомьтесь со всеми пунктами из этой секции, не беритесь сразу выполнять задание, не дочитав до конца. Пункты взаимосвязаны и могут влиять друг на друга.  

### <a id="Сайт">Сайт</a> 
Создайте две ВМ в разных зонах, установите на них сервер nginx, если его там нет. ОС и содержимое ВМ должно быть идентичным, это будут наши веб-сервера.

Используйте набор статичных файлов для сайта. Можно переиспользовать сайт из домашнего задания.

Создайте Target Group, включите в неё две созданных ВМ.

Создайте Backend Group, настройте backends на target group, ранее созданную. Настройте healthcheck на корень (/) и порт 80, протокол HTTP.

Создайте HTTP router. Путь укажите — /, backend group — созданную ранее.

Создайте Application load balancer для распределения трафика на веб-сервера, созданные ранее. Укажите HTTP router, созданный ранее, задайте listener тип auto, порт 80.

Протестируйте сайт curl -v <публичный IP балансера>:80

### <a id="Мониторинг">Мониторинг</a>
Создайте ВМ, разверните на ней Zabbix. На каждую ВМ установите Zabbix Agent, настройте агенты на отправление метрик в Zabbix.

Настройте дешборды с отображением метрик, минимальный набор — по принципу USE (Utilization, Saturation, Errors) для CPU, RAM, диски, сеть, http запросов к веб-серверам. Добавьте необходимые tresholds на соответствующие графики.

### <a id="Логи">Логи</a> 
Cоздайте ВМ, разверните на ней Elasticsearch. Установите filebeat в ВМ к веб-серверам, настройте на отправку access.log, error.log nginx в Elasticsearch.

Создайте ВМ, разверните на ней Kibana, сконфигурируйте соединение с Elasticsearch.

### <a id="Сеть">Сеть</a> 
Разверните один VPC. Сервера web, Elasticsearch поместите в приватные подсети. Сервера Zabbix, Kibana, application load balancer определите в публичную подсеть.

Настройте Security Groups соответствующих сервисов на входящий трафик только к нужным портам.

Настройте ВМ с публичным адресом, в которой будет открыт только один порт — ssh. Эта вм будет реализовывать концепцию bastion host . Синоним "bastion host" - "Jump host". Подключение ansible к серверам web и Elasticsearch через данный bastion host можно сделать с помощью ProxyCommand . Допускается установка и запуск ansible непосредственно на bastion host.(Этот вариант легче в настройке)

### <a id="backup">Резервное копирование</a> 
Создайте snapshot дисков всех ВМ. Ограничьте время жизни snaphot в неделю. Сами snaphot настройте на ежедневное копирование.  

---

# Выполнение дипломной работы  

## Terraform

### <a id="terra">Инфраструктура</a>  

Поднимаем инфраструктуру в Yandex Cloud используя **terraform**  

Передаем в качестве пременных окружения `token id, cloud id и folder id` в команды 
```
export YC_TOKEN=
export YC_CLOUD_ID=
export YC_FOLDER_ID=
```
Проверим кофнигурацию выполнив`terraform plan`
сверяем конфигурацию  и запускаем процесс поднятия инфраструктуры  командой `terraform apply`  

Вывод Outputs.tf

```
Apply complete! Resources: 29 added, 0 changed, 0 destroyed.

Outputs:

FQDN_bastion = "bastion.ru-central1.internal"
FQDN_elastic = "elastic.ru-central1.internal"
FQDN_kibana = "kibana.ru-central1.internal"
FQDN_web-1 = "web1.ru-central1.internal"
FQDN_web-2 = "web2.ru-central1.internal"
FQDN_zabbix = "zabbix.ru-central1.internal"
external_ip_address_L7balancer = tolist([
  {
    "address" = "84.201.149.249"
  },
])
external_ip_address_bastion = "158.160.0.73"
external_ip_address_kibana = "89.169.174.96"
external_ip_address_zabbix = "84.201.142.48"
internal_ip_address_bastion = "10.0.4.4"
internal_ip_address_elastic = "10.0.3.3"
internal_ip_address_kibana = "10.0.4.6"
internal_ip_address_web-1 = "10.0.1.3"
internal_ip_address_web-2 = "10.0.2.3"
internal_ip_address_zabbix = "10.0.4.5"
```


После завершения работы **terraform** проверяем в web консоли YC созданную инфраструктуру.Сервера WEB-1 и WEB-2 созданы в разных зонах.

![VM](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/VM_status.PNG)  

### <a id="net">Сеть</a>  

 **VPC и subnet**

Создаем одну VPC, которая будет включать публичные и приватные подсети, а также таблицу маршрутизации для обеспечения доступа к интернету для виртуальных машин, расположенных внутри сети и защищенных Бастионом, исполняющим роль шлюза к интернету.

![maps_VPC](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/network_map.PNG)

### <a id="group">Группы безопасности</a>

Общий список групп безопасности

![SG_ALL](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/list_SG.png)

А также посмотрим на каждую группу бузоапсности по отдельности:

**SG_LB**

![sg_L7B](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/SG_BALANCER.png)

**SG_internal** с разрешением любого трафика между ВМ кому присвоена данная SG

![sg_internal](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/SG_INT_RULS.png)

**SG_bastion** c с открытием только 22 порта для работы SSH

![sg_bastion](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/SG_BASTION.png)


**SG_kibana** c открытым портом 5601 для доступа c интернета к Fronted Kibana

![sg_kibana](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/SG_KIBANA.png)

**SG_zabbix** с открытым портом 80 и 10051 для доступа с интернета к Fronted Zabbix и работы Zabbix agent

![sg_zabbix](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/SG_ZABBIX.png)

### <a id="balancer">Load Balancer</a>

**Создаем Target Group**

![target-group](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/target_group.png)

**Создаем Backend Group**

![backend-group](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/GROUP_BACK.png)

**Создаем HTTP router**

![http-router](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/HTTP_ROUTER.png)

**Создаем Application load balancer**

Для распределения трафика на ранее созданные веб-серверы указываем ранее созданный HTTP роутер, устанавливаем тип listener на auto и выбираем порт 80.

![loadBalancer](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/APLICATION_LOAD_BALANCER.png)


### <a id="snapshot">Резервное копирование</a>

**snapshot**

создаем в terraform блок с расписанием snapshots

```
resource "yandex_compute_snapshot_schedule" "default1" {
  name = "default1"
  description    = "Ежедневные снимки, хранятся 7 дней"

    schedule_policy {
    expression = "0 1 * * *"
  }

```

Переходим в консоль YC и видим созданое расписание, по которому будут происходить обновления.

![snap](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/SCHEDULER_SNAPSHOT.png)

## Ansible

Создадим файл inventory назовем host.ini, к WEB серверам будем обращаться по FQDN имени.

```
[all:vars]
ansible_user=user
ansible_ssh_private_key_file=/home/user/.ssh/id_ed25519
ansible_ssh_common_args='-o ProxyCommand="ssh -W %h:%p -q user@89.169.164.152"'

[log]
elastic_srv ansible_host=elastic.ru-central1.internal
kibana_srv  ansible_host=kibana.ru-central1.internal

[web]
web-1 ansible_host=web1.ru-central1.internal
web-2 ansible_host=web2.ru-central1.internal

[mon]
zabbix_srv ansible_host=zabbix.ru-central1.internal
```

Проверка достпуности хостов

![ping_pong](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/ping_pong.png)

Устанавливаем NGINX

![install_nginx](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/nginx.png)

Проверям его работу в браузере

![nginx_web](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/Nginx_web.png)

Сделаем несколько запросов в консоли YC и в логах балансировщика увидим, что меняется IP адрес backend

![change_ip](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/change_ip_backand.png)


## Мониторинг

установка Zabbix

![install_zabbix](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/install_zabbix.png)

Проверим работу Zabbix в Web интерфейсе

![zabbix_web](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/zabbix_1.png)

Установка Zabbix-agent

![install_zabbix_agent](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/install_zabbix_agent.png)

Проверим статус zabbix agenta на двух web серверах

![zabbix_agent_web1](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/status_zabbix_agent_web1.png)

![zabbix_agent_web2](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/status_zabbix_agent_web2.png)

Добавляем хосты используя FQDN имена в zabbix сервер и настраиваем дашборды

![active_node](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/active_node_zabbix.png)

И создадим дашборд на который выыедем утилизацию CPU RAM и Uptime

![dashbord_zabbix](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/dashbord_zabbix.png)

## ELK

Теперь установим стек ELK начнем с elasticsearch

![install_elasicsearch](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/install_elastic.png)

Посмотрим статус сервиса elasticsearch

![status_elasicsearch](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/status_elastic.png)

Установим Kibana

![install_kibana](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/install_kibana.png)

Проверим статус Kibana

![status_kibana](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/status_kibana.png)

Установим Filebaet на оба web сервера

![install_filebeat](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/install_filebeat.png)

Проверим в WEB интерефейсе логи с двух web сервером

![ifilebeat_log](https://github.com/dvolkov15/Diplom_Netology/blob/main/scrin/log_elastic.png)