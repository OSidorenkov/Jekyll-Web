---
layout: post
title:  "Как я Zabbix автоматизировал. Часть 1."
date:   2017-04-15
desc: "Как я начал автоматизировать раскатку zabbix и добавление хостов по правильным группам"
keywords: "Zabbix,Автоматизация Zabbix,Zabbix мониторинг,Zabbix,мониторинг,Ansible Zabbix"
categories: [Zabbix]
tags: [Zabbix,Ansible,Мониторинг]
icon: fa fa-television
---

## Предисловие ######

Рано или поздно, в процессе работы с Zabbix встает вопрос об автоматизации. Это происходит по разным причинам: слишком много хостов, не хватает времени да и тупо лень. Все сводится к вопросу: "А как сделать так, чтобы я нажал на кнопку и пошел пить кофе, а через минуту уже все монитрится и я молодец!". Сразу поясню, что тот способ который буду описывать я, кастомный, который подходил мне. Я уверен есть много разных вариантов, даже что-то есть из коробки, но я решил использовать путь джедая и написать все с начала =) Итак, ближе к делу.

<br/>

## Начало ######

Цель была такая. При добавлении новых хостов, добавляется новая DNS запись о сервере. Эта запись вытаскивается и добавляется в инвентори файл ansible. Запилить playbook, который бы ставил агента на удаленный хост, а также добавлял бы этот хост в веб интерфейсе Zabbix, добавлял нужный шаблон, добавлял этот сервер в нужные группы, ну и давал права на эти группы хостов, группе пользователей(для просмотра/редактирования и уведомлений).

<br/>

## Почему Ansible? ######

Тут все просто. Есть много различных оркестраторов, но Ansible довольно таки неприхотлив. Ему не нужны агенты, до хостов он стучиться по ssh, а на хостах должен быть предуставновлен python 2.7 и выше версии. Это все есть, так я работаю в 99% c Linux серверами, а там по дефолту это все есть.

<br/>

## Так какой хост нужно мониторить? ######

Здесь я хотел описать процесс автодискаверинга DNS, диф и поиск новых записей, но получается все костыльно, так как DNS на винде. На ВИНДЕ, Карл!!! Не уверен, что кто-то захочет пойти таким путем, а если и захочет, то выложу как будет время во 2 части статьи.

<br/>

## Начинаем писать playbook ######

Как установить ansible я писать не буду, я думаю это и без моих наставлений каждый сделает.
На всякий случай оставлю ссыль на оффициальную [доку](http://docs.ansible.com/ansible/intro_installation.html).

Итак, переходим в папочку playbooks и создаем папку с нашим содержимым для раскатки Zabbix.  
`mkdir playbooks/zabbix_agent`

Далее создаем yaml файл, в котором будем описывать таски для выполнения:  
`touch zabbix.yml`

```yaml
- hosts: target
  vars:
      zabbix_server: zabbix.lc
      zabbix_server_port: 10051
      zabbix_agent_port: 10050
      zabbix_login: login
      zabbix_pass: pass
      zabbix_url: https://zabbix.ru
```

Я понимаю, что кажется криво, указание переменных прямо в плэйбуке, да и почему вообще это не роль!? Такой вид только для наглядности, а не для локаничности.
Кстати здесь указана группа серверов `target`, те сервера, над которыми будут совершены действия из инвентори файла hosts.

Дальше начнем с 1 таска. Требуется для начала установить на хост zabbix агента.  

```yaml
  - name: install zabbix repo
    yum:
        name: https://repo.zabbix.com/zabbix/3.2/rhel/7/x86_64/zabbix-release-3.2-1.el7.noarch.rpm
        state: installed
```
Здесь мы говорим, установить на хост rpm пакет zabbix через пакетный менеджер yum.  

Далее установим сам Zabbix Agent:  

```yaml
  - name: install zabbix agent
    yum:
        name: zabbix-agent
        state: installed
```
В принципе пример, аналогичен приведенному выше.  

Затем, после установки агента, нужно скофигурировать его, проставить адреса сервера и задать имя. Тут напомощь приходит модуль ансибла [template](http://docs.ansible.com/ansible/template_module.html). 

```yaml
  - name: make zabbix config
    template:
        src: zabbix_agentd.conf.j2
        dest: /etc/zabbix/zabbix_agentd.conf
        owner: zabbix
        group: zabbix
        mode: 0644
    notify:
        - restart zabbix-agent
```
Что тут делается, берется заготовленный шаблон jinja2, в котором отмечаются места куда нужно вставить переменные. Они вставляются и конфиг копируется на хост, назначаются права 644, а также присваивается владелец и группа `zabbix:zabbix`. Так же здесь параметр notify который вызывает handler `restart zabbix-agent` в конце плэйбука.  
Так прописываем собственно в шаблоне переменные в нужных местах:  
```
{% raw %}
Server={{ zabbix_server }}
ListenPort={{ zabbix_agent_port }}
Hostname={{ inventory_hostname }}
{% endraw %}
```

Я наверняка уверен, что у каждого стоит фаервол и просто так начать ломиться в порт агента не получиться. В моем случае это iptables, поэтому следующий таск:  

```yaml
  - name: allow port {{ zabbix_agent_port }} in iptables
    iptables: chain=INPUT action=insert ctstate=NEW protocol=tcp match=tcp destination_port={{ zabbix_agent_port }} jump=ACCEPT

```
Сохраняем правила iptables:  

```yaml
  - name: iptables save
    command: service iptables save
```

Итак агента мы поставили, сконфигурили. Нужно подготовить группы в интерфейсе Zabbix для новоиспеченных хостов. Чтобы вы просто понимали, днс записи хостов у нас выглядит примерно так: `app1.prod.portal`, где

* app1 - это что за сервис располагается на хосте, 

* prod - среда окружения, 

* portal - проект. 

Поэтому из групп мне вполне будет хватать группы среды и группы проекта.
С этим определились, но как эти группы автоматом? На помощь нам приходит Zabbix Api, а с ним и библиотека pyzabbix на питоне для работы с апи.  
Ставится очень просто:  
`pip install pyzabbix` 

Нужно написать скрипт, который бы брал от днс хоста среду окружения и наименования проекта и, используя апи, создавал бы группы: 

```yaml
{% raw %}
  - name: Add first group to zabbix web
    local_action: command python -c 'from pyzabbix import ZabbixAPI; zapi = ZabbixAPI("{{ zabbix_url }}"); zapi.login("{{ zabbix_login }}", "{{ zabbix_pass }}"); group1 = zapi.hostgroup.create(name="{{ inventory_hostname.split('.')[1] }}")'
    ignore_errors: yes

  - name: Add second group to zabbix web
    local_action: command python -c 'from pyzabbix import ZabbixAPI; zapi = ZabbixAPI("{{ zabbix_url }}"); zapi.login("{{ zabbix_login }}", "{{ zabbix_pass }}"); group2 = zapi.hostgroup.create(name="{{ inventory_hostname.split('.')[2] }}")'
    ignore_errors: yes
{% endraw %}
```

Делаем финт ушами и достаем группы, разбивая строку по точке. Используем модуль local_action чтобы запускать скрипт с хоста ansible, в конце говорим игнорировать ошибки, которые могут возникнуть, если такая группа уже существует. Можно было написать проверку на существование такой группы и если такой нет ничего не делать, но мне лень 😊

После создания групп хостов, добавляем права на них группе пользователей. По бОльше степени это делается для пользователя, который алертит о проблемах. Ему нужен доступ на чтение во все хосты. Итак, нужно использовать метод [usergroup.update](https://www.zabbix.com/documentation/3.0/manual/api/reference/usergroup/update), а для это нам нужно знать id группы пользователей и id групп хостов, на которые требуется выдать права. С помощью метода [usergroup.get](https://www.zabbix.com/documentation/3.0/ru/manual/api/reference/usergroup/get) находим id группы пользователей и вставляем в следующий скрипт:  
```python
from pyzabbix import ZabbixAPI

zapi = ZabbixAPI("https://zabbix.ru/")
zapi.login("login", "pass")

ids = [group['groupid'] for group in zapi.hostgroup.get()] # Берем все id групп хостов

rights = [{'permission': 2, 'id': i} for i in ids] # Генерируем массив из прав и групп хостов, 'permission': 2 - дает права на чтение
zapi.usergroup.update(			 # Апдейтим группу пользователей
    usrgrpid=13,			     # ID группы пользователей, взятый из метода usergroup.get
    rights=rights
)
```

Делаем соответственный таск в плэйбуке:  
```yaml
  - name: Add grants on groups
    local_action: command python /Users/ossidorenkov/ansible/playbooks/zabbix_agent/add_grants.py
```

На выходе получаем нечто подобное:

<img src="{{ site.img_path }}/automate_zabbix/zab1.png" width="70%">

<br/>

Далее создадим сам хост в админке Zabbix и поместим его в те группы, которые создали. Здесь уже говородить самописные скрипты не придется, за вас это сделает ansible module - [zabbix-host](http://docs.ansible.com/ansible/zabbix_host_module.html). Как видно из описания модуля, требуется поставить на хост ансибла библиотеку zabbix-api: 
`pip install zabbix-api`

Далее собственно сам таск:  

```yaml
{% raw %}
- name: Create a new host or update an existing host's info
    local_action:
      module: zabbix_host
      server_url: "{{ zabbix_url }}"
      login_user: "{{ zabbix_login }}"
      login_password: "{{ zabbix_pass }}"
      host_name: "{{ inventory_hostname }}"
      host_groups:
        - "{{ inventory_hostname.split('.')[1] }}"
        - "{{ inventory_hostname.split('.')[2] }}"
      link_templates:
        - Template OS Linux
      status: enabled
      state: present
      inventory_mode: automatic
      interfaces:
        - type: 1
          main: 1
          useip: 0
          ip: ""
          dns: "{{ inventory_hostname }}"
          port: "{{ zabbix_agent_port }}"
{% endraw %}
```
Информацию о том какие параметры доступны у данного модуля читайте в документации из ссылки выше.
<br/>

Ну и в конце плэйбука пишем обработчик на рестарт агента и включение в автозагрузку: 

```yaml
  handlers:
      - name: restart zabbix-agent
        service:
            name: zabbix-agent
            state: restarted
            enabled: yes
```

Итак, пример пошагово:

У вас заполнен инвентори файл hosts:
```
[target]
pgsql3.prod.sgr.sber ansible_user=deployment ansible_ssh_private_key_file=/Users/ossidorenkov/ansible/deployment_id_rsa
```

Запускаем playbook:  
`ansiblansible-playbook -i my_inventories/hosts -b playbooks/zabbix_agent/zabbix.yml`

Ждёмс...

```
PLAY RECAP *********************************************************************
pgsql3.prod.sgr.sber       : ok=12   changed=11   unreachable=0    failed=0
```

Идем проверяь хост в админку: 

<img src="{{ site.img_path }}/automate_zabbix/zab3.png" width="70%">

Вуаля, хост появился, доступен, в нужных группах и прицеплен к шаблону. Красота!

<br/>

Полный playbook, а также скрипты и шаблоны, вы можете скачать с моего репозитория на [GitHub](https://github.com/OSidorenkov/ansible-playbook).
