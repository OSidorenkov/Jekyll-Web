---
layout: post
title:  "How to install Python3 on CentOS"
date:   2017-05-18
description: "Установка Python3 на Linux"
keywords: "Python3, python, pip, install python, установить python"
image: /static/assets/img/blog/python/python-logo-m2.png
categories: [Python]
tags: [Python,Linux]
icon: icon-python
---

Вот как вы сможете собрать и установить python3 из исходного кода.

Сначала установите минимально необходимые инструменты:  

```
sudo yum groupinstall "Development Tools" -y
sudo yum install zlib zlib-devel -y
```

Теперь загрузите последнюю версию python3 (например, python 3.6) из [https://www.python.org/ftp/python/](https://www.python.org/ftp/python/).

```bash
$ curl -O https://www.python.org/ftp/python/3.6.1/Python-3.6.1.tgz
```

Наконец, создайте и установите python3 следующим образом. Установочный каталог по умолчанию - `/usr/local`. Если вы хотите изменить установку в какой-либо другой каталог, передайте параметр `--prefix=/alternative/path`  для настройки перед запуском `make`.

```bash
$ tar xf Python-3.6.1.tgz
$ cd Python-3.6.1
$ ./configure --prefix=/usr
$ make
$ sudo make install
```

Это позволит установить python3, pip3, setuptools, а также библиотеки python3 в вашей системе CentOS.

`$ python3 --version`

```
Python 3.6.1
```
