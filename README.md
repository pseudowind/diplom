
#  Дипломная работа по профессии «Системный администратор» - Морковкин Михаил

Содержание
==========
* [Задача](#Задача)
* [Инфраструктура](#Инфраструктура)
    * [Сайт](#Сайт)
    * [Мониторинг](#Мониторинг)
    * [Логи](#Логи)
    * [Сеть](#Сеть)
    * [Резервное копирование](#Резервное-копирование)

---------
## Задача
Ключевая задача — разработать отказоустойчивую инфраструктуру для сайта, включающую мониторинг, сбор логов и резервное копирование основных данных. Инфраструктура должна размещаться в [Yandex Cloud](https://cloud.yandex.com/).


## Инфраструктура - применяем terraform apply для развертки
![1-1](https://github.com/pseudowind/diplom/blob/main/screens/apply.PNG)

### Проверяем что подняли
![1-2](https://github.com/pseudowind/diplom/blob/main/screens/vms.PNG)

### Действующие IP-адреса
![1-24](https://github.com/pseudowind/diplom/blob/main/screens/inventory.PNG)

### ping-pong
ansible all -m ping
![1-24](https://github.com/pseudowind/diplom/blob/main/screens/ansible_ping.PNG)

## Сайт
Создайте две ВМ в разных зонах, установите на них сервер nginx, если его там нет. ОС и содержимое ВМ должно быть идентичным, это будут наши веб-сервера.

Используйте набор статичных файлов для сайта. Можно переиспользовать сайт из домашнего задания.

Создайте [Target Group](https://cloud.yandex.com/docs/application-load-balancer/concepts/target-group), включите в неё две созданных ВМ.

### Проверяем Target Group
![1-13](https://github.com/pseudowind/diplom/blob/main/screens/targetgroups.PNG)
![1-13](https://github.com/pseudowind/diplom/blob/main/screens/balancer-status.PNG)

Создайте [Backend Group](https://cloud.yandex.com/docs/application-load-balancer/concepts/backend-group), настройте backends на target group, ранее созданную. Настройте healthcheck на корень (/) и порт 80, протокол HTTP.

![1-13](https://github.com/pseudowind/diplom/blob/main/screens/backends.PNG)

Создайте [HTTP router](https://cloud.yandex.com/docs/application-load-balancer/concepts/http-router). Путь укажите — /, backend group — созданную ранее.

![1-13](https://github.com/pseudowind/diplom/blob/main/screens/router.PNG)

Создайте [Application load balancer](https://cloud.yandex.com/en/docs/application-load-balancer/) для распределения трафика на веб-сервера, созданные ранее. Укажите HTTP router, созданный ранее, задайте listener тип auto, порт 80.

![1-13](https://github.com/pseudowind/diplom/blob/main/screens/balancer.PNG)

 Группа безопасности
![1-13](https://github.com/pseudowind/diplom/blob/main/screens/yc.PNG)

### Устанавливаем Nginx на машины
ansible-playbook nginx1.yaml
![1-5](https://github.com/pseudowind/diplom/blob/main/screens/nginx%20create.PNG)

### Тестируем сайт `curl -v <публичный IP балансера>:80` 
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/curl.PNG)
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/site.PNG)


## Мониторинг
Создайте ВМ, разверните на ней Prometheus. На каждую ВМ из веб-серверов установите Node Exporter и [Nginx Log Exporter](https://github.com/martin-helmich/prometheus-nginxlog-exporter). Настройте Prometheus на сбор метрик с этих exporter.

### Устанавливаем Log Exporter
Была использована [роль](https://galaxy.ansible.com/ui/standalone/roles/mbaran0v/ansible_role_prometheus_nginxlog_exporter/)
ansible-galaxy role install mbaran0v.ansible_role_prometheus_nginxlog_exporter
ansible-playbook nginxlog-exporter.yaml
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/nginxlog-exporter.PNG)

### Устанавливаем Node Exporter
Была использована [роль](https://galaxy.ansible.com/ui/standalone/roles/cloudalchemy/node_exporter/)
ansible-galaxy role install cloudalchemy.node_exporter
ansible-playbook node-exporter1.yaml
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/node-exporter.PNG)
ansible-playbook prometheus.yaml
ansible-playbook prometheus1.yaml


Создайте ВМ, установите туда Grafana. Настройте её на взаимодействие с ранее развернутым Prometheus. Настройте дешборды с отображением метрик, минимальный набор — Utilization, Saturation, Errors для CPU, RAM, диски, сеть, http_response_count_total, http_response_size_bytes. Добавьте необходимые [tresholds](https://grafana.com/docs/grafana/latest/panels/thresholds/) на соответствующие графики.

### Устанавливаем Grafana
Пришлось устанавливать ручками, ввиду недоступности Графаны. Использовался материал лекции, подключался по ssh

sudo wget https://dl.grafana.com/oss/release/grafana_9.2.4_amd64.deb

sudo dpkg -i grafana_9.2.4_amd64.deb

sudo systemctl enable grafana-server

sudo systemctl start grafana-server

sudo systemctl status grafana-server

### Смотрим метрики (использовался [11074 Dashboard ID](https://grafana.com/grafana/dashboards/11074-node-exporter-for-prometheus-dashboard-en-v20201010/))
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/grafana.PNG)

## Логи
Cоздайте ВМ, разверните на ней Elasticsearch. Установите filebeat в ВМ к веб-серверам, настройте на отправку access.log, error.log nginx в Elasticsearch.

### Устанавливаем Elasticsearch
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/elastic.PNG)

### Устанавливаем filebeat
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/filebeat.PNG)

Создайте ВМ, разверните на ней Kibana, сконфигурируйте соединение с Elasticsearch.

### Устанавливаем Kibana
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/kibana.PNG)

### Проверяем доступность сайта
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/kibana-site.PNG)

### Смотрим метрику


## Сеть
Разверните один VPC. Сервера web, Prometheus, Elasticsearch поместите в приватные подсети. Сервера Grafana, Kibana, application load balancer определите в публичную подсеть.

### VPC 
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/vpc.PNG)

Настройте [Security Groups](https://cloud.yandex.com/docs/vpc/concepts/security-groups) соответствующих сервисов на входящий трафик только к нужным портам.

Настройте ВМ с публичным адресом, в которой будет открыт только один порт — ssh. Настройте все security groups на разрешение входящего ssh из этой security group. Эта вм будет реализовывать концепцию bastion host. Потом можно будет подключаться по ssh ко всем хостам через этот хост.
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/bastion.PNG)

## Резервное копирование
Создайте snapshot дисков всех ВМ. Ограничьте время жизни snaphot в неделю. Сами snaphot настройте на ежедневное копирование.

### Создаем snapshot
![1-7](https://github.com/pseudowind/diplom/blob/main/screens/snaps.PNG)




## Дополнительно
Не входит в минимальные требования. 

1. Для Prometheus можно реализовать альтернативный способ хранения данных — в базе данных PpostgreSQL. Используйте [Yandex Managed Service for PostgreSQL](https://cloud.yandex.com/en-ru/services/managed-postgresql). Разверните кластер из двух нод с автоматическим failover. Воспользуйтесь адаптером с https://github.com/CrunchyData/postgresql-prometheus-adapter для настройки отправки данных из Prometheus в новую БД.
2. Вместо конкретных ВМ, которые входят в target group, можно создать [Instance Group](https://cloud.yandex.com/en/docs/compute/concepts/instance-groups/), для которой настройте следующие правила автоматического горизонтального масштабирования: минимальное количество ВМ на зону — 1, максимальный размер группы — 3.
3. Можно добавить в Grafana оповещения с помощью Grafana alerts. Как вариант, можно также установить Alertmanager в ВМ к Prometheus, настроить оповещения через него.
4. В Elasticsearch добавьте мониторинг логов самого себя, Kibana, Prometheus, Grafana через filebeat. Можно использовать logstash тоже.
5. Воспользуйтесь Yandex Certificate Manager, выпустите сертификат для сайта, если есть доменное имя. Перенастройте работу балансера на HTTPS, при этом нацелен он будет на HTTP веб-серверов.

