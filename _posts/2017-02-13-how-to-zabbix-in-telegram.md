---
layout: post
title:  "Zabbix in Telegram"
date:   2017-02-13
desc: "Как прикурить оповещения Zabbix к telegram"
keywords: "Telegram,Zabbix,мониторинг,оповещения, Hyper-V"
categories: [Zabbix]
tags: [Zabbix,Telegram,Мониторинг]
icon: fa fa-television
---

В данной статье будет описываться настройка Zabbix нотификаций с графиками в канал Телеграм. О том, как создать канал в Телеграм, а затем превратить его из публичного в приватный читайте ниже:  

## Конфигурация / Установка ######

Так как скрипт для оповещений написан на python, нужно установить компилятор на сервер:  
`yum install python`

Также для работы  потребуется дополнительный модуль:  
`pip install requests`

Склонируйте себе репозиторий с скриптами:  
`git clone https://github.com/ableev/Zabbix-in-Telegram.git`

* Копируем `zbxtg.py` и `zbxtg_group.py` в _AlertScriptsPath_ директорию Zabbix, путь указан в zabbix_server.conf
* Создаем `zbxtg_settings.py` в котором прописываете следующее:

```python
# -*- coding: utf-8 -*-

tg_key = "XYZ"  # telegram bot api key

zbx_tg_prefix = "zbxtg"  # variable for separating text from script info
zbx_tg_tmp_dir = "/tmp/" + zbx_tg_prefix  # directory for saving caches, uids, cookies, etc.
zbx_tg_signature = False

zbx_server = "http://localhost"  # zabbix server full url
zbx_api_user = "api"
zbx_api_pass = "api"
zbx_api_verify = True  # True - do not ignore self signed certificates, False - ignore

proxy_to_zbx = None
proxy_to_tg = None

#proxy_to_zbx = "proxy.local:3128"
#proxy_to_tg = "proxy.local:3128"

emoji_map = {
    "ok": "✅",
    "problem": "❗",
    "info": "ℹ️",
    "warning": "⚠️",
    "disaster": "❌",
    "bomb": "💣",
    "fire": "🔥",
    "hankey": "💩",
}
```

* Создаем бота Telegram, если еще не был создан, и сохраняем "API key"

* Создаем юзера в Zabbix и даем ему права на чтение (для просмотра и получения графиков)

* Создаем новый "Способ оповещения" в веб-интерфейсе Zabbix c подобными параметрами:  

	<img src="{{ site.img_path }}/telegram_zabbix/image1.png">

* Создаем новое "Действие", напрмер:  

	<img src="{{ site.img_path }}/telegram_zabbix/image2.png">

	* Текст о проблеме:
```
Last value:{ITEM.VALUE1} ({TIME})
zbxtg;graphs
zbxtg;graphs_period=1800
zbxtg;itemid:{ITEM.ID1}
zbxtg;title:{HOST.HOST} - {TRIGGER.NAME}
Важность триггера: {TRIGGER.SEVERITY}
Server: {HOSTNAME} ({HOST.IP})
Описание:
{TRIGGER.DESCRIPTION}
```

	* Сообщение о восстановлении:
```
Server: {HOSTNAME} ({HOST.IP})
Описание:
Проблема устранена!
Время устранения проблемы: {DATE} {TIME}
```

* Добавим пользователю нужный тип оповещения(в графе отправлять на используем chat_id канала, как его получить, читать ниже):  

<img src="{{ site.img_path }}/telegram_zabbix/image3.png">

<br>

## Примечание ######

| Параметр | Описание |
|---|---|
| zbxtg;channel  | Включает возможность отправки нотификаций в канал  |
| zbxtg;graphs  | Позволяет отправлять графики  |
| zbxtg;graphs_period  | Задает период графика (по умолчанию 3600 секунд)  |
| zbxtg;graphs_width  | Задает графику ширину (по умолчанию 900px)  |
| zbxtg;graphs_height  | Задает графику высоту (по умолчанию 300px)  |
| zbxtg;itemid:{ITEM.ID1}  | Определяет itemid (от триггера) для прикрепления графика  |
| zbxtg;title:{HOST.HOST} - {TRIGGER.NAME}  | Заголовок графика  |
| zbxtg;debug  | Включает debug mode, логи и изображение будет сохранены в tmp директории  |	

<br>

Также можно использовать стилистику для текста действия:  [markdown-style](https://core.telegram.org/bots/api#markdown-style) + [html-style](https://core.telegram.org/bots/api#html-style).

<br>

## Создание канала ######

Создайте свой канал в клиенте Телеграм, затем у вас спросят выбрать тип(Публичный или приватный), нужно выбрать приватный, так как без этого мы не узнаем уникальный chatid, без которого будет невозможно понять куда отправлять оповещения боту.  

<img src="{{ site.img_path }}/telegram_zabbix/image4.png">  

В данном случае UserName канала является @ТestZabbix, добавляем туда нашего бота и делаем его администратором(для возможности отправки нотификаций). Затем чтобы получить chat_id открываем браузер и посылаем себе сообщение в канал от бота, используя API Telegram:  

`https://api.telegram.org/botXxX:XXXXxxxxxxxXXXXxXxXXxxx/sendMessage?chat_id=@TestZabbix&text=HelloWorld!`

В ответ мы получим нужный нам chat_id и можно переделывать канал из Public в Private, для того чтобы избежать случайных посетителей в нем. Его и нужно будет вставить в графу "Отправлять на" в настройке оповещения у пользователя Zabbix.

<br>

## Debug ######

Примечание: Zabbix запускает скрипт с тремя параметрами, так что если вы собираетесь запустить его без каких-либо, с одним или с двумя, пожалуйста, не надо. В случае успеха вы не увидите никаких ошибок, и вы получите сообщение.

<br>

### Проверка отправки просто текста

`$ ./zbxtg.py "@OSSidorenkov" "test" "test"`

<br>

### Если ничего не произошло

* username регистрозависимый

* проверить, что послали боту сообщение (one-to-one message)

* если видите "User 'username' needs to send some text bot in private" , это означает что вы не послали сообщение боту

<br>

### Тест отправки графика в канал

```sudo -u zabbix /var/lib/zabbixsrv/alertscripts/zbxtg_channel.py "@TestZabbix" "Test" "$(echo -e 'zbxtg;graphs\nzbxtg;itemid:28458\nzbxtg;debug\nzbxtg;title:test')" --channel```		


Ссылка на первоисточник:	
[https://github.com/ableev/Zabbix-in-Telegram](https://github.com/ableev/Zabbix-in-Telegram)  
Переходите и не забудьте оценить проект звездочкой на GitHub.  
Полезные ссылки:  

* Сообщество по этому проекту в телеграм: [https://telegram.me/ZbxTg](https://telegram.me/ZbxTg)

* Новости по обновлению проекта: [https://telegram.me/Zabbix_in_Telegram](https://telegram.me/Zabbix_in_Telegram)	
```sudo -u zabbix /var/lib/zabbixsrv/alertscripts/zbxtg_channel.py "@TestZabbix" "Test" "$(echo -e 'zbxtg;graphs\nzbxtg;itemid:28458\nzbxtg;debug\nzbxtg;title:test')" --channel```
