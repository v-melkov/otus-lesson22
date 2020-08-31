## Стенд для домашнего задания "Сбор и анализ логов"

Настраиваем центральный сервер для сбора логов
в вагранте поднимаем 2 машины web и log
на web поднимаем nginx
на log настраиваем центральный лог сервер на любой системе на выбор

-   journald
-   rsyslog
-   elk
-   настраиваем аудит следящий за изменением конфигов нжинкса

все критичные логи с web должны собираться и локально и удаленно
все логи с nginx должны уходить на удаленный сервер (локально только критичные)
логи аудита должны также уходить на удаленную систему

-   развернуть еще машину elk
    и таким образом настроить 2 центральных лог системы elk И какую либо еще
    в elk должны уходить только логи нжинкса
    во вторую систему все остальное

* * *

Для запуска стенда запустить `vagrant up`

* * *

### Настроим машину log

Создадим директорию для хранения логов с удаленных машин:  

    mkdir -p /var/log/journal/remote  
    chmod 755 /var/log/journal/remote  
    chown systemd-journal-remote:systemd-journal-remote /var/log/journal/remote

Для упрощения работы запустим сервис `systemd-journal-remote` через http, для этого в настройках запуска сервиса поменяем `--listen-https=-3` на `--listen-http=-3`

Запустим сервис:  
`systemctl enable --now systemd-journal-remote`

### Настроим машину web

Установим nginx из репозитория  
`yum install nginx`  
`systemctl enable --now nginx`  
Создадим директорию для записи логов  
`mkdir /var/log/journal`

В настройках journal-upload укажем сервер для загрузки логов  

    cat /etc/systemd/journal-upload.conf
    [Upload]
    URL=http://192.168.11.102:19532

Запустим сервис journal-upload:  
`systemctl enable --now systemd-journal-upload`

В настройках nginx укажем несколько мест для хранения логов (удаленно и локально):

    access_log syslog:server=unix:/dev/log;
    error_log syslog:server=unix:/dev/log;
    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log crit;

Установим пакет для удаленного аудита:  
`yum install audisp-remote`  

Настроим сервер удаленного аудита:  

    cat /etc/audisp/audisp-remote.conf
    remote_server = 192.168.11.102
    port = 60
    transport = tcp

Разрешим запуск плагина удаленного аудита:

    cat /etc/audisp/plugins.d/au-remote.conf
    active = yes
    direction = out
    path = /sbin/audisp-remote
    type = always
    format = string

Создадим правило аудита слежения за файлом nginx.conf:

    cat /etc/audit/rules.d/nginx.rules
    -w /etc/nginx/ -k nginx-conf

#### Настройка web для ELK

Добавим репозиторий ELK.

    cat /etc/yum/repo.d/elk.repo
    [elasticsearch-7.x]
    name=Elasticsearch repository for 7.x packages
    baseurl=https://artifacts.elastic.co/packages/7.x/yum
    gpgcheck=1
    gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
    enabled=1
    autorefresh=1
    type=rpm-md  

Установим java и filebeat  
`yum install java-1.8.0-openjdk`  
`yum install filebeat`

Настроим filebeat на слежение за логами nginx:

    cat /etc/filebeat/filebeat.conf
    filebeat.inputs:
    - input_type:     log
      enable:         true
      paths:
        - /var/log/nginx/*.log
      exclude_files:  ['\.gz$']

    output.logstash:
      hosts:          ["192.168.11.103:5044"]

Запустим filebeat  
`systemctl enable --now filebeat`

Для нормальной работы audisp-remote перезапустим машину web  
`reboot`

### Настроим ELK

Добавим репозиторий ELK.

    cat /etc/yum/repo.d/elk.repo
    [elasticsearch-7.x]
    name=Elasticsearch repository for 7.x packages
    baseurl=https://artifacts.elastic.co/packages/7.x/yum
    gpgcheck=1
    gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
    enabled=1
    autorefresh=1
    type=rpm-md  

Установим java, elasticsearch, logstash, kibana
`yum install java-1.8.0-openjdk elasticsearch logstash kibana`

Настроим logstash:

    cat /etc/logstash/conf.d/input.conf
    input {
      beats {
        host => "0.0.0.0"
        port => 5044
      }
    }

    cat /etc/logstash/conf.d/filter.conf
    filter {
     grok {
       match => [ "message" , "%{COMBINEDAPACHELOG}+%{GREEDYDATA:extra_fields}"]
       overwrite => [ "message" ]
     }
     mutate {
       convert => ["response", "integer"]
       convert => ["bytes", "integer"]
       convert => ["responsetime", "float"]
     }
     geoip {
       source => "clientip"
       target => "geoip"
       add_tag => [ "nginx-geoip" ]
     }
     date {
       match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
       remove_field => [ "timestamp" ]
     }
     useragent {
       source => "agent"
     }
    }

    cat /etc/logstash/conf.d/output.conf
    output {
      elasticsearch {
        hosts => ["localhost:9200"]
        index => "weblogs-%{+YYYY.MM.dd}"
        document_type => "nginx_logs"
      }
    }

В настройках kibana укажем ip для соединений

    cat /etc/kibana/kibana.yml
    server.host: 192.168.11.103

Запустим все сервисы  

    systemctl enable --now elasticsearch
    systemctl enable --now logstash
    systemctl enable --now kibana

### Проверка

#### Проверка логов web и nginx:

    vagrant ssh log
    sudo journalctl -D /var/log/journal/remote --follow

    vagrant ssh web
    curl 127.0.0.1:8080

#### Проверка auditlog:

    vagrant ssh log
    sudo ausearch -k nginx-conf

#### Проверка ELK:

Через некоторое время заходим по адресу <http://192.168.11.103:5601>, открываем Меню - Management - Stack management - Index management.  
Там видим наши логи (weblogs-\*).

### Спасибо за проверку!
