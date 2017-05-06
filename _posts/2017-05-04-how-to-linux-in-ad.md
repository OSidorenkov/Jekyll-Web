---
layout: post
title:  "Linux + Active Directory"
date:   2017-05-04
description: Статья о том, как можно интегрировать Linux сервер в Active Directory, заходить на него под доменной учетной записью, а также управлять доступами на сервер через AD.
keywords: "Linux,Active Directory,Linux + Active Directory,интеграция linux в active directory,sssd"
image: /static/assets/img/blog/linux_ad/sshot518c059b37f74.jpg
categories: [Linux]
tags: [Linux,Active Directory]
icon: fa fa-linux
---

Если вы работаете в организации, где офисная инфраструктура завязана на Active Directory, а рабочее окружение на Linux, то скорее всего это статья поможет управлять вашим парком серверов более эффективно и менее трудозатратно.  

Когда штат сотрудников от 100 и выше человек, а парк серверов еще больше, то админить в плане управления доступами пользователей к этим серверам становится времязатратно, нудно и лениво. Я попробую изложить, как мы в своей компании решили данный вопрос и избавились на 90% от рутинной работы.  

Для начала, хотелось бы пояснить, что задумывалось. Есть:

* Windows Server 2012R2(DC)

* 300+ Linux Servers

Как итог - доступ к серверам предоставлялся бы пользователю через специальную группу, к которой принадлежит данный сервер.

В конце концов, вариантов для достижения этой цели есть несколько, я опишу 2 из них.  

<br/>

<center><h2>Способ №1</h2></center>

Необходимо настроить авторизацию в Linux по приватному ключу. Публичный ключ должен храниться в учетной записи Active Directory в атрибуте info, логин пользователя так же берется из Active Directory. А так же нам необходимо разграничить доступы по серверам. Для примера взят сервер на CentOS 7 и вымышленный домен `domain.lc`

<br/>
### Подготовка сервера

Заходим на linux сервер под root. Обновляем список пакетов:  
`yum update -y`

Устанавливаем ntp для синхронизации времени, синхронизируем и добавляем в крон:  
```bash
yum install ntp -y
cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime 
ntpdate ru.pool.ntp.org
mkdir -p /var/cron/tabs && echo '0 0 * * * /usr/sbin/ntpdate ru.pool.ntp.org' >> /var/cron/tabs/crontab && crontab /var/cron/tabs/crontab && crontab -l
```
<br/>

### Настройка аутентификации SSH через AD

Устанавливаем пакеты:  
`yum install realmd sssd oddjob oddjob-mkhomedir adcli samba-common krb5-workstation samba-common-tools`  

Вводим сервер в домен:  
```
realm discover domain.lc
realm join -U DomainUser domain.lc
```

Спросит пароль от учетной записи DomainUser, от которой будет введен в домен (учетная запись должна быть с правами `администратор домена`)

Настраиваем sssd:

Примерно так должен выглядеть `/etc/sssd/sssd.conf`:  
```
[sssd]
domains = domain.lc
config_file_version = 2
reconnection_retries = 3
services = nss, pam
sbus_timeout = 30
 
[nss]
filter_groups = root
filter_users = root
reconnection_retries = 3
 
[domain/domain.lc]
ad_domain = domain.lc
krb5_realm = DOMAIN.LC
realmd_tags = manages-system joined-with-samba
cache_credentials = True
id_provider = ad
krb5_store_password_if_offline = True
default_shell = /bin/bash
ldap_id_mapping = True
fallback_homedir = /home/%u
access_provider = ad
use_fully_qualified_names = False
```

Разрешаем создавать домашние директории новым пользователям и перезапускаем sssd:  

```
authconfig --enablemkhomedir --enablesssdauth --updateall
systemctl enable sssd.service
systemctl restart sssd
```
<br/>

### Написание скрипта для получения атрибута <a name="ldap_pubkey.py"></a>
Скрипт написан на Python. Скриптом получаем из Active Directory атрибут info. В котором будет записан public key. Устанавливаем библиотеки для python:  
`pip install ldap3`

Создаем сам скрипт `/usr/bin/ldap_pubkey.py`:  
```python
#!/usr/bin/python2
import sys
import os
from ldap3 import Server, Connection, ALL, NTLM

LDAP_SERVER = '192.168.0.100'  # DC address
LDAP_PORT = 389
LDAP_USER = 'domain\\admin'  # active directory administrator user
LDAP_PASSWD = 'pass'  # active directory administrator pass
BASE_DN = 'OU=Users,DC=domain,DC=lc'  # directory with users in ad

search_name = sys.argv[1]

f = open('/etc/passwd', 'r')
line = f.readline()
while line:
    user = line.split(':')
    if user[6] == '/bin/bash\n':
        if user[0] == search_name and os.path.exists("os.path.expanduser('~%s' % search_name) + '/.ssh/authorized_keys'"):
            pub_key = os.path.expanduser('~%s' % search_name) + '/.ssh/authorized_keys'
            f = open(pub_key, 'r')
            print(f.read())
            sys.exit(0)
    line = f.readline()

server = Server(LDAP_SERVER, port=LDAP_PORT)
conn = Connection(server, user=LDAP_USER, password=LDAP_PASSWD,  authentication=NTLM)
conn.bind()

conn.search('{}'.format(BASE_DN), "(sAMAccountName={})".format(search_name), attributes=('info',))
entry = conn.entries[0]

for pubkey in entry.info.values:
    print(pubkey)
    sys.exit(0)
sys.exit(1)
```

Даем право на выполнение данного скрипта только root и владельцем должен быть root:  
`chown root:root /usr/bin/ldap_pubkey.py && chmod 700 /usr/bin/ldap_pubkey.py`  

Далее генерируем приватный и публичный ключ:  
`ssh-keygen -t rsa` 

Копируем id_rsa.pub и добавляем его в профиль пользователя в Active Directory. Добавляем публичный ключ в атрибут info (Заметки):  

<img src="{{ site.img_path }}/linux_ad/keyAd.png" width="25%" class="image">

Дальше нам необходимо проверить работу нашего скрипта. Пишем путь к скрипту и имя пользователя. Скрипт должен вернуть публичный ключ:  
`/usr/bin/ldap_pubkey.py ossidorenkov` 

<br/>

### Настройка SSH

В конфигурационном файле `/ect/ssh/sshd_config` нам необходимо подправить следующее + закрываем авторизацию по паролю:  
```
...
AuthorizedKeysCommand /usr/bin/ldap_pubkey
AuthorizedKeysCommandUser root
PermitEmptyPasswords no
PasswordAuthentication no
UsePAM yes
...
```

Перезапускаем SSH:  
`service sshd restart`

<br/>

### Безопасность

В Active Directory создаем группу linux_root и группу с названием linux сервера, например proxy.prod.github.
По группе linux_root будим определять, имеет ли пользователь доступ к root. По второй группе будем определять имеет ли пользователь право на соединение с сервером по SSH, т.е. необходимо добавить эти группы нашему пользователю. 

Далее нам на сервере необходимо модифицировать файл `/ect/ssh/ssd_config` добавляем в конец файла строку:  
```
.......
AllowGroups proxy.prod.github
```

А так же модифицируем `/etc/sudoers`, добавляем строчку:  
```
.......
%linux_root ALL=(ALL)       NOPASSWD: ALL
```

Пробуйте подключиться к серверу, используя свой приватный ключ, и логин из ad:  
```
ossidorenkov@sbn-mac-107~ ssh proxy.prod.github
Last login: Thu May  4 13:08:32 2017

[ossidorenkov@proxy-prod-github ~]$ id
uid=1424001161(ossidorenkov) gid=1424000512(администраторы домена) группы=1424000512(администраторы домена),10(wheel),1424000513(пользователи домена),1424002430(proxy.prod.github),1424002176(linux_root)

[ossidorenkov@proxy-prod-github ~]$ sudo su

[root@proxy-prod-github ossidorenkov]# id
uid=0(root) gid=0(root) группы=0(root)
```

Из примера видно что мы удачно подключились, а так же у нашего профиля подтянулись группы из Active Directory, успешно получен root.

<br/>

<center><h2>Способ №2</h2></center>

Во 2 варианте будет предложен способ в виде Python скрипта, который будет отрабатывать по крону и проверять пользователей в Active Directory на наличие соответствующей группы, если такие пользователи будут найдены, создаcтся user и home директория на сервере. В плане авторизации по ssh ничего не меняется. Для авторизации используется публичный ключ из атрибута info.

Создадим скрипт `/usr/bib/usercheck.py`:  
<details>
<div class="language-python highlighter-rouge"><pre class="highlight"><code><span class="c">#!/usr/bin/python2</span>

<span class="kn">from</span> <span class="nn">ldap3</span> <span class="kn">import</span> <span class="n">Server</span><span class="p">,</span> <span class="n">Connection</span><span class="p">,</span> <span class="n">NTLM</span>
<span class="kn">import</span> <span class="nn">os</span>
<span class="kn">import</span> <span class="nn">pwd</span>
<span class="kn">import</span> <span class="nn">sys</span>

<span class="n">ldap_cred</span> <span class="o">=</span> <span class="p">{</span><span class="s">'server_ip'</span><span class="p">:</span> <span class="s">'192.168.0.100'</span><span class="p">,</span>  <span class="c"># DC address</span>
             <span class="s">'user'</span><span class="p">:</span> <span class="s">'domain</span><span class="se">\\</span><span class="s">admin'</span><span class="p">,</span>  <span class="c"># Administrative user in AD</span>
             <span class="s">'password'</span><span class="p">:</span> <span class="s">'pass'</span><span class="p">}</span>  <span class="c"># Pass for user "admin"</span>


<span class="n">ldap_conn</span> <span class="o">=</span> <span class="n">Connection</span><span class="p">(</span><span class="n">Server</span><span class="p">(</span><span class="n">ldap_cred</span><span class="p">[</span><span class="s">'server_ip'</span><span class="p">]),</span>
                       <span class="n">user</span><span class="o">=</span><span class="n">ldap_cred</span><span class="p">[</span><span class="s">'user'</span><span class="p">],</span>
                       <span class="n">password</span><span class="o">=</span><span class="n">ldap_cred</span><span class="p">[</span><span class="s">'password'</span><span class="p">],</span>
                       <span class="n">authentication</span><span class="o">=</span><span class="n">NTLM</span><span class="p">,</span>
                       <span class="n">auto_bind</span><span class="o">=</span><span class="bp">True</span><span class="p">)</span>

<span class="n">ldap_users_dir</span> <span class="o">=</span> <span class="s">'ou=users,dc=domain,dc=lc'</span>  <span class="c"># Директория с пользователями в AD</span>
<span class="n">ldap_servers_dir</span> <span class="o">=</span> <span class="s">'ou=linux server,dc=domain,dc=lc'</span>  <span class="c"># Директория с группами для для Linux серверов</span>
<span class="n">ldap_linux_root_group</span> <span class="o">=</span> <span class="s">'CN=linux_root,OU=Linux Server,DC=domain,DC=lc'</span>  <span class="c"># Группа для управления root доступом</span>


<span class="k">def</span> <span class="nf">get_group</span><span class="p">(</span><span class="n">group_name</span><span class="p">):</span>
    <span class="n">ldap_conn</span><span class="o">.</span><span class="n">search</span><span class="p">(</span><span class="n">ldap_servers_dir</span><span class="p">,</span> <span class="s">'(cn={})'</span><span class="o">.</span><span class="n">format</span><span class="p">(</span><span class="n">group_name</span><span class="p">))</span>
    <span class="k">return</span> <span class="n">ldap_conn</span><span class="o">.</span><span class="n">response</span><span class="p">[</span><span class="mi">0</span><span class="p">][</span><span class="s">'dn'</span><span class="p">]</span>


<span class="k">def</span> <span class="nf">get_users</span><span class="p">(</span><span class="n">group_path</span><span class="p">):</span>
    <span class="n">ldap_conn</span><span class="o">.</span><span class="n">search</span><span class="p">(</span><span class="n">ldap_users_dir</span><span class="p">,</span>
        <span class="s">'(&amp;(cn=*)(memberOf={})(objectClass=User)</span><span class="se">\
</span><span class="s">        (!(userAccountControl:1.2.840.113556.1.4.803:=2)))'</span><span class="o">.</span><span class="n">format</span><span class="p">(</span><span class="n">group_path</span><span class="p">),</span>
        <span class="n">attributes</span><span class="o">=</span><span class="p">(</span><span class="s">'sAMAccountName'</span><span class="p">,</span> <span class="s">'memberof'</span><span class="p">))</span>
    <span class="k">return</span> <span class="n">ldap_conn</span><span class="o">.</span><span class="n">entries</span>


<span class="k">def</span> <span class="nf">create_user</span><span class="p">(</span><span class="n">user_name</span><span class="p">):</span>
    <span class="n">os</span><span class="o">.</span><span class="n">system</span><span class="p">(</span><span class="s">'/sbin/useradd -m {}'</span><span class="o">.</span><span class="n">format</span><span class="p">(</span><span class="n">user_name</span><span class="p">))</span>
    <span class="n">os</span><span class="o">.</span><span class="n">system</span><span class="p">(</span><span class="s">'/bin/passwd -uf {}'</span><span class="o">.</span><span class="n">format</span><span class="p">(</span><span class="n">user_name</span><span class="p">))</span>


<span class="k">def</span> <span class="nf">on_off_user</span><span class="p">(</span><span class="n">user_name</span><span class="p">,</span> <span class="n">key</span><span class="o">=</span><span class="s">'off'</span><span class="p">):</span>
    <span class="k">if</span> <span class="n">key</span> <span class="o">==</span> <span class="s">'off'</span><span class="p">:</span>
        <span class="n">os</span><span class="o">.</span><span class="n">system</span><span class="p">(</span><span class="s">'/sbin/usermod -s {0} {1}'</span><span class="o">.</span><span class="n">format</span><span class="p">(</span><span class="s">'/sbin/nologin'</span><span class="p">,</span> <span class="n">user_name</span><span class="p">))</span>
    <span class="k">elif</span> <span class="n">key</span> <span class="o">==</span> <span class="s">'on'</span><span class="p">:</span>
        <span class="n">os</span><span class="o">.</span><span class="n">system</span><span class="p">(</span><span class="s">'/sbin/usermod -s {0} {1}'</span><span class="o">.</span><span class="n">format</span><span class="p">(</span><span class="s">'/bin/bash'</span><span class="p">,</span> <span class="n">user_name</span><span class="p">))</span>


<span class="k">def</span> <span class="nf">grant_root</span><span class="p">(</span><span class="n">user_name</span><span class="p">):</span>
    <span class="n">os</span><span class="o">.</span><span class="n">system</span><span class="p">(</span><span class="s">'/sbin/usermod -aG wheel {}'</span><span class="o">.</span><span class="n">format</span><span class="p">(</span><span class="n">user_name</span><span class="p">))</span>


<span class="k">def</span> <span class="nf">rotate_user_list</span><span class="p">():</span>
    <span class="k">if</span> <span class="n">os</span><span class="o">.</span><span class="n">path</span><span class="o">.</span><span class="n">exists</span><span class="p">(</span><span class="s">'/opt/users'</span><span class="p">):</span>
        <span class="n">os</span><span class="o">.</span><span class="n">system</span><span class="p">(</span><span class="s">'/bin/mv -f /opt/users /opt/users_old'</span><span class="p">)</span>


<span class="k">def</span> <span class="nf">write_user_to_list</span><span class="p">(</span><span class="n">user_name</span><span class="p">):</span>
    <span class="k">with</span> <span class="nb">open</span><span class="p">(</span><span class="s">'/opt/users'</span><span class="p">,</span> <span class="s">'a'</span><span class="p">)</span> <span class="k">as</span> <span class="n">f</span><span class="p">:</span>
        <span class="n">f</span><span class="o">.</span><span class="n">write</span><span class="p">(</span><span class="n">user_name</span> <span class="o">+</span> <span class="s">'</span><span class="se">\n</span><span class="s">'</span><span class="p">)</span>
    <span class="n">f</span><span class="o">.</span><span class="n">close</span><span class="p">()</span>


<span class="k">def</span> <span class="nf">user_present</span><span class="p">(</span><span class="n">user_name</span><span class="p">):</span>
    <span class="k">return</span> <span class="n">user_name</span> <span class="ow">in</span> <span class="p">[</span><span class="n">entry</span><span class="o">.</span><span class="n">pw_name</span> <span class="k">for</span> <span class="n">entry</span> <span class="ow">in</span> <span class="n">pwd</span><span class="o">.</span><span class="n">getpwall</span><span class="p">()]</span>


<span class="k">def</span> <span class="nf">users_to_disable</span><span class="p">():</span>
    <span class="n">users</span> <span class="o">=</span> <span class="p">[]</span>
    <span class="k">if</span> <span class="n">os</span><span class="o">.</span><span class="n">path</span><span class="o">.</span><span class="n">exists</span><span class="p">(</span><span class="s">'/opt/users'</span><span class="p">)</span> <span class="ow">and</span> <span class="n">os</span><span class="o">.</span><span class="n">path</span><span class="o">.</span><span class="n">exists</span><span class="p">(</span><span class="s">'/opt/users_old'</span><span class="p">):</span>
        <span class="k">with</span> <span class="nb">open</span><span class="p">(</span><span class="s">'/opt/users'</span><span class="p">)</span> <span class="k">as</span> <span class="n">f_cur</span><span class="p">,</span> <span class="nb">open</span><span class="p">(</span><span class="s">'/opt/users_old'</span><span class="p">)</span> <span class="k">as</span> <span class="n">f_old</span><span class="p">:</span>
            <span class="n">lines_cur</span> <span class="o">=</span> <span class="nb">set</span><span class="p">(</span><span class="n">f_cur</span><span class="o">.</span><span class="n">read</span><span class="p">()</span><span class="o">.</span><span class="n">splitlines</span><span class="p">())</span>
            <span class="n">lines_old</span> <span class="o">=</span> <span class="nb">set</span><span class="p">(</span><span class="n">f_old</span><span class="o">.</span><span class="n">read</span><span class="p">()</span><span class="o">.</span><span class="n">splitlines</span><span class="p">())</span>
        <span class="k">for</span> <span class="n">line</span> <span class="ow">in</span> <span class="n">lines_old</span><span class="p">:</span>
            <span class="k">if</span> <span class="ow">not</span> <span class="n">line</span> <span class="ow">in</span> <span class="n">lines_cur</span><span class="p">:</span>
                <span class="n">users</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">line</span><span class="p">)</span>
    <span class="k">return</span> <span class="n">users</span>


<span class="k">def</span> <span class="nf">users_to_enable</span><span class="p">():</span>
    <span class="n">users</span> <span class="o">=</span> <span class="p">[]</span>
    <span class="k">if</span> <span class="n">os</span><span class="o">.</span><span class="n">path</span><span class="o">.</span><span class="n">exists</span><span class="p">(</span><span class="s">'/opt/users'</span><span class="p">)</span> <span class="ow">and</span> <span class="n">os</span><span class="o">.</span><span class="n">path</span><span class="o">.</span><span class="n">exists</span><span class="p">(</span><span class="s">'/opt/users_old'</span><span class="p">):</span>
        <span class="k">with</span> <span class="nb">open</span><span class="p">(</span><span class="s">'/opt/users'</span><span class="p">)</span> <span class="k">as</span> <span class="n">f_cur</span><span class="p">,</span> <span class="nb">open</span><span class="p">(</span><span class="s">'/opt/users_old'</span><span class="p">)</span> <span class="k">as</span> <span class="n">f_old</span><span class="p">:</span>
            <span class="n">lines_cur</span> <span class="o">=</span> <span class="nb">set</span><span class="p">(</span><span class="n">f_cur</span><span class="o">.</span><span class="n">read</span><span class="p">()</span><span class="o">.</span><span class="n">splitlines</span><span class="p">())</span>
            <span class="n">lines_old</span> <span class="o">=</span> <span class="nb">set</span><span class="p">(</span><span class="n">f_old</span><span class="o">.</span><span class="n">read</span><span class="p">()</span><span class="o">.</span><span class="n">splitlines</span><span class="p">())</span>
        <span class="k">for</span> <span class="n">line</span> <span class="ow">in</span> <span class="n">lines_cur</span><span class="p">:</span>
            <span class="k">if</span> <span class="ow">not</span> <span class="n">line</span> <span class="ow">in</span> <span class="n">lines_old</span><span class="p">:</span>
                <span class="n">users</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="n">line</span><span class="p">)</span>
    <span class="k">return</span> <span class="n">users</span>


<span class="k">if</span> <span class="n">__name__</span> <span class="o">==</span> <span class="s">'__main__'</span><span class="p">:</span>
    <span class="n">rotate_user_list</span><span class="p">()</span>
    <span class="n">group_p</span> <span class="o">=</span> <span class="n">get_group</span><span class="p">(</span><span class="n">sys</span><span class="o">.</span><span class="n">argv</span><span class="p">[</span><span class="mi">1</span><span class="p">])</span>
    <span class="k">for</span> <span class="n">user</span> <span class="ow">in</span> <span class="n">get_users</span><span class="p">(</span><span class="n">group_p</span><span class="p">):</span>
        <span class="n">user_n</span> <span class="o">=</span> <span class="n">user</span><span class="o">.</span><span class="n">sAMAccountName</span><span class="o">.</span><span class="n">value</span><span class="o">.</span><span class="n">lower</span><span class="p">()</span>
        <span class="k">if</span> <span class="ow">not</span> <span class="n">user_present</span><span class="p">(</span><span class="n">user_n</span><span class="p">):</span>
            <span class="n">create_user</span><span class="p">(</span><span class="n">user_n</span><span class="p">)</span>
            <span class="k">for</span> <span class="n">group</span> <span class="ow">in</span> <span class="n">user</span><span class="o">.</span><span class="n">memberof</span><span class="o">.</span><span class="n">value</span><span class="p">:</span>
                <span class="k">if</span> <span class="n">group</span> <span class="o">==</span> <span class="n">ldap_linux_root_group</span><span class="p">:</span>
                    <span class="n">grant_root</span><span class="p">(</span><span class="n">user_n</span><span class="p">)</span>
        <span class="n">write_user_to_list</span><span class="p">(</span><span class="n">user_n</span><span class="p">)</span>
    <span class="k">for</span> <span class="n">user</span> <span class="ow">in</span> <span class="n">users_to_disable</span><span class="p">():</span>
        <span class="n">on_off_user</span><span class="p">(</span><span class="n">user</span><span class="p">,</span> <span class="n">key</span><span class="o">=</span><span class="s">'off'</span><span class="p">)</span>
    <span class="k">for</span> <span class="n">user</span> <span class="ow">in</span> <span class="n">users_to_enable</span><span class="p">():</span>
        <span class="n">on_off_user</span><span class="p">(</span><span class="n">user</span><span class="p">,</span> <span class="n">key</span><span class="o">=</span><span class="s">'on'</span><span class="p">)</span>
</code></pre>
</div>
</details>

  
Скрипт [ldap_pubkey.py](#ldap_pubkey.py) остается без изменений.  

Добавляем скрипт в crontab рута:  
```
*/5 * * * * /usr/bin/usercheck.py "proxy.prod.github"
```  

Теперь раз в 5 минут скрипт будет проверять наличие пользователей в этой группе, давать/отбирать доступ, а также проверять на наличие рута у пользователя.

Проверяем:  
```
ossidorenkov@sbn-mac-107~ ssh proxy.prod.github
Last login: Thu May  4 13:08:32 2017

[ossidorenkov@proxy-prod-github ~]$ id
uid=1000(ossidorenkov) gid=1000(ossidorenkov) группы=1000(ossidorenkov),10(wheel)

[ossidorenkov@proxy-prod-github ~]$ sudo su
[root@proxy-prod-github ossidorenkov]# 
```

Как видите, я успешно подключился, а также был добавлен в группу wheel, тем самым получив рута. Mission completed!!!

<br>

## P.S.

Итого. Интегрировать Linux сервера в Active Directory возможно, при чем разными способами. Тут, как говорится, на вкус и цвет..  А что касается автоматизации всех проделанных действий для подготовки сервера, с этим прекрасно справляется Ansible, ну это уже совсем другая история!
