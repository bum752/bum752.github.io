---
title: "Django, Swagger를 이용한 API 문서화"
time: 2018-07-12 12:35:00
categories: django
---

Django rest framework 프로젝트의 API에 대한 정보를 문서화하려고 합니다.  
Swagger를 이용해 자동으로 문서화할 수 있습니다.

## Swagger
[Swagger](https://swagger.io/)는 RESTful 서비스의 설계, 빌드, 문서화, 사용에 도움을 주는 툴입니다.

### 설치
`pip`를 이용해 `django-rest-swagger`를 설치합니다.
{% highlight bash %}
$ pip3 install django-rest-swagger
{% endhighlight %}

### 설정
`settings.py`의 `INSTALLED_APPS`에 `rest_framework_swagger`를 추가해줍니다.
{% highlight python linenos %}
# settings.py
...
INSTALLED_APPS = [
    ...
    'rest_framework_swagger',
    ...
]
...
{% endhighlight %}
이외의 `SWAGGER_SETTINGS`에 대한 내용은 [Documents](https://django-rest-swagger.readthedocs.io/en/latest/settings/)를 참고하실 수 있습니다.

### URL 설정
프로젝트의 `urls.py`에 `swagger view`를 추가합니다.
{% highlight python linenos %}
# urls.py
...
from rest_framework_swagger.views import get_swagger_view
schema_view = get_swagger_view(title="My API")

urlpatterns = [
    ...
    url(r'^$', schema_view),
    ...
]
{% endhighlight %}

### 실행
지정한 url에 접속하면 API가 문서화되어 있고 해당 API를 실행해볼 수 있습니다.
![image](https://user-images.githubusercontent.com/20104232/42619566-60eb631a-85f3-11e8-8ab1-88911c8307b1.png)
