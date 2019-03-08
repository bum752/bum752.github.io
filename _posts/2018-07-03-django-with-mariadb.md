---
title: "Django에 MariaDB 사용하기"
date: 2018-07-03 09:53:00
categories: django
---
Django 프로젝트에 MariaDB(MySQL)을 사용해보려고 합니다.

## MariaDB

### 설치
{% highlight bash %}
$ brew install mariadb
{% endhighlight %}

### 실행
{% highlight bash %}
$ mysql.server start # 시작
$ mysql.server stop # 중단
$ mysql.server status # 상태
{% endhighlight %}

## 프로젝트

### mysqlclient 설치
{% highlight bash %}
$ pip3 install mysqlclient
{% endhighlight %}

### settings.py
프로젝트를 생성하면 기본적으로 sqlite3을 사용합니다.
{% highlight py linenos %}
# your_project/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),
    }
}
{% endhighlight %}

settings.py를 수정해 MariaDB를 사용하도록 수정합니다.
{% highlight py linenos %}
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'dbName', # ex) mydb
        'USER': 'dbUser', # ex) root
        'PASSWORD': 'dbPassword', # ex) P@ssw0rd
        'HOST': 'dbHost', # default: localhost
        'PORT': 'dbPort' # default: 3306
    }
}
{% endhighlight %}

### Migrate
{% highlight bash %}
$ python3 manage.py migrate
{% endhighlight %}

![image](https://user-images.githubusercontent.com/20104232/42193336-ede1aeb6-7ea8-11e8-94ab-c0082f530fce.png)
