---
layout: post
title:  "Мониторинг Nginx"
date:   2017-02-16
desc: "Как мониторить Nginx с помощью Zabbix"
keywords: "CentOS,Linux,Nginx,Zabbix,мониторинг"
categories: [Zabbix]
tags: [Zabbix,Nginx,Мониторинг]
icon: fa fa-television
---


Веб-сервер Nginx умеет отдавать свои статистические данные, поэтому было бы не плохо их мониторить с возможность построения различных наглядных графиков.  
<br>

## Статистические данные ######

`http_stub_status_module` - собирает следующие данные:  
* **active connections** - количество открытых коннектов в данный момент, включая коннекты на backend

* **server accepts** - количество принятых подключений

* **server handled** - количество обработанных подключений

* **server requests** - количество принятых запросов

* **reading** - количество запросов в данный момент, заголовки которых читает nginx

* **writing** - количество запросов в данный момент, тело которых читает nginx + находящиеся в обработки + идет отдача данных

* **waiting** - количество ожидающих (keep-alive) соединений в данный момент. waiting = active - reading - writing

## Установка скрипта ######

```bash
 #!/bin/bash
##### OPTIONS VERIFICATION #####
if [[ -z "$1" || -z "$2" || -z "$3" ]]; then
  exit 1
fi
##### PARAMETERS #####
RESERVED="$1"
METRIC="$2"
STATSURL="$3"
CURL="/usr/bin/curl"
CACHE_TTL="55"
CACHE_FILE="/tmp/zabbix.nginx.`echo $STATSURL | md5sum | cut -d" " -f1`.cache"
EXEC_TIMEOUT="1"
NOW_TIME=`date '+%s'`
##### RUN #####
if [ -s "${CACHE_FILE}" ]; then
  CACHE_TIME=`stat -c"%Y" "${CACHE_FILE}"`
else
  CACHE_TIME=0
fi
DELTA_TIME=$((${NOW_TIME} - ${CACHE_TIME}))
#
if [ ${DELTA_TIME} -lt ${EXEC_TIMEOUT} ]; then
  sleep $((${EXEC_TIMEOUT} - ${DELTA_TIME}))
elif [ ${DELTA_TIME} -gt ${CACHE_TTL} ]; then
  echo "" >> "${CACHE_FILE}" # !!!
  DATACACHE=`${CURL} -sS --insecure --max-time ${EXEC_TIMEOUT} "${STATSURL}" 2>&1`
  echo "${DATACACHE}" > "${CACHE_FILE}" # !!!
  chmod 640 "${CACHE_FILE}"
fi
#
if [ "${METRIC}" = "active" ]; then
  cat "${CACHE_FILE}" | grep "Active connections" | cut -d':' -f2
fi
if [ "${METRIC}" = "accepts" ]; then
  cat "${CACHE_FILE}" | sed -n '3p' | cut -d" " -f2
fi
if [ "${METRIC}" = "handled" ]; then
  cat "${CACHE_FILE}" | sed -n '3p' | cut -d" " -f3
fi
if [ "${METRIC}" = "requests" ]; then
  cat "${CACHE_FILE}" | sed -n '3p' | cut -d" " -f4
fi
if [ "${METRIC}" = "reading" ]; then
  cat "${CACHE_FILE}" | grep "Reading" | cut -d':' -f2 | cut -d' ' -f2
fi
if [ "${METRIC}" = "writing" ]; then
  cat "${CACHE_FILE}" | grep "Writing" | cut -d':' -f3 | cut -d' ' -f2
fi
if [ "${METRIC}" = "waiting" ]; then
  cat "${CACHE_FILE}" | grep "Waiting" | cut -d':' -f4 | cut -d' ' -f2
fi
#
exit 0
```
<br>


## Установка прав ######

`chown root:zabbix /etc/zabbix/nginx-stats.sh`  
`chmod 550 /etc/zabbix/nginx-stats.sh`
<br>

## Настройка Nginx ######

Необходимо настроить Nginx для отдачи своей статистики по определенному адресу. Сам Nginx должен быть скомпилирован с поддержкой модуля статистики (--with- http_stub_status_module)  
В секции server конфига VirtualHost добавляем следующее:  

```
server {
 ...
 
        location = /nginx-stats {
stub_status on;
access_log off;
allow IP.ZABBIX.SEVER.AGENT; #добавляем IP Zabbix сервера, для разрешения просмотра страницы.
deny all; #запрещаем просматривать страницу для всех остальных.
        }
}
```
<br>

После настройки нужно не забыть перезагрузить конфигурацию Nginx:  
`service nginx reload`
<br>

На стороне сервера следует проверить что нужные данные отдаются. Для этого можно выполнить команду:  
`curl http://you.site.com/nginx-stats`
<br>

или с помощью скрипта для zabbix:  
`sudo -u zabbix /etc/zabbix/nginx-stats.sh none active http://you.site.com/nginx-stats`  
Вы должны получить статистические данные от Nginx, если этого не произошло, то конфигурация выполнена не правильно.
<br>

## Настройка Zabbix сервера ######

Настройка /etc/zabbix/zabbix_agentd.conf  
`UserParameter=nginx[*],/etc/zabbix/nginx-stats.sh "none" "$1" "$2"`
<br>

Перезапустить  
`service zabbix-agent restart`
<br>

Проверка  
`zabbix_get -s HOST -k "nginx[active,http://you.site.com/nginx-stats]"`
<br>

Далее прикрепляем к нужному хосту шаблона Nginx (положу его сюда на всякий случай) и создаем макросы и необходимые значения, для работы триггеров.  
Пример:  
<img src="{{ site.img_path }}/nginx_zabbix/image1.JPG" width="40%">