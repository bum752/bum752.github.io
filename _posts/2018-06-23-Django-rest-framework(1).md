---
title: "Django rest framework(1)"
date: 2018-06-23 21:25:00
categories: django
---
[GitHub](https://github.com/bum752/django-rest-framework/tree/lec-1)

##  RESTful API
- REST(Representational State Transfer)는 월드 와이드 웹과 같은 분산 하이퍼미디어 시스템을 위한 소프트웨어 아키텍처의 한 형식이다. 이 용어는 로이 필딩(Roy Fielding)의 2000년 박사학위 논문에서 소개되었다.
([위키백과](https://ko.wikipedia.org/wiki/REST))
- 자세한 내용은 [위키백과](https://ko.wikipedia.org/wiki/REST), [TOAST Meetup - REST API 제대로 알고 사용하기](http://meetup.toast.com/posts/92)를 참고해주세요.

## Django Rest Framework
djangorestframework를 이용한 RESTful API 서버를 만들어봅니다.  
[Django REST framework의 Tutorial](http://www.django-rest-framework.org/)을 참고해 만들어보는 메모입니다.

### Environments
  * macOS 10.13.3
  * Python 3.6.5

### Install python
Python3을 사용하겠습니다.
{% highlight bash %}
$ brew install python3
{% endhighlight %}

### Virtual Environments
프로젝트별 독립적인 황경을 위해 Python에서는 virtualenv를 이용해 가상 환경을 만들어 줍니다.
{% highlight bash %}
$ pip3 install virtualenv
$ virtualenv env # 가상 환경 생성
$ source env/bin/activate # 가상 환경 활성화
{% endhighlight %}

### Install frameworks
{% highlight bash %}
$ pip3 install django
$ pip3 install djangorestframework
{% endhighlight %}

### Create project
가상 환경의 django-admin.py를 이용해 "djangotutorial"이라는 프로젝트를 생성합니다.
{% highlight bash %}
$ django-admin.py startproject project
{% endhighlight %}

### Create app
{% highlight bash %}
$ cd project
$ python3 manage.py startapp app
{% endhighlight %}
어플리케이션이 생성되었고 프로젝트의 설정을 변경해보겠습니다.  
**project/project/settings.py** 의 **INSTALLED_APPS** 에 아래처럼 추가해줍니다.
{% highlight python linenos %}
INSTALLED_APPS = [
    ...
    'rest_framework',
    'app.apps.AppConfig'
]
{% endhighlight %}

### Models
어플리케이션에서 사용할 모델을 만들어야 합니다.  
장고 모델에 대한 설명은 [djangogirls의 장고 모델](https://tutorial.djangogirls.org/ko/django_models/)을 참고해주세요.  
**project/app/models.py** 를 아래와 같이 수정합니다.
{% highlight python linenos %}
from django.db import models

class Memo(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()
    created = models.DateTimeField(auto_now_add=True)
    class Meta:
        ordering = ('created',)
{% endhighlight %}

### Migrate
{% highlight bash %}
$ python3 manage.py makemigrations app
$ python3 manage.py migrate
{% endhighlight %}
**project/db.sqlite3** 이 만들어졌습니다.

### [Serializers](http://www.django-rest-framework.org/api-guide/serializers/)
**project/app/** 에 **serializers.py** 를 추가하고 아래의 내용을 입력합니다.
{% highlight python linenos %}
from app.models import Memo
from rest_framework import serializers

class MemoSerializer(serializers.HyperlinkedModelSerializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=100)
    content = serializers.CharField()
    created = serializers.ReadOnlyField()

    def create(self, validated_data):
        return Memo.objects.create(**validated_data)
    def update(self, instance, validated_data):
        instance.title = validated_data.get('title', instance.title)
        instance.content = validated_data.get('content', instance.content)
        instance.save()
        return instance

    class Meta:
        model = Memo
        fields = ('id', 'title', 'content', 'created')
{% endhighlight %}

### Test
{% highlight bash %}
$ python3 manage.py shell
>>> from app.models import Memo
>>> from app.serializers import MemoSerializer
>>> memo = Memo(title='TITLE', content='CONTENT')
>>> memo.save()
>>> serializer = MemoSerializer(memo)
>>> serializer.data
{'id': 1, 'title': 'TITLE', 'content': 'CONTENT'}
{% endhighlight %}

### Views  
**project/app/views.py** 가 HTTP Method를 이용해 메모의 조회, 추가, 수정, 삭제를 할 수 있도록 아래의 내용으로 수정합니다.
{% highlight python linenos %}
from app.models import Memo
from app.serializers import MemoSerializer
from django.http import Http404
from rest_framework import status
from rest_framework.views import APIView
from rest_framework.response import Response

class MemoList(APIView):
    def get(self, request, format=None):
        memo = Memo.objects.all()
        serializer = MemoSerializer(memo, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = MemoSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.error, status=status.HTTP_400_BAD_REQUEST)

class MemoDetail(APIView):
    def get_object(self, pk):
        try:
            return Memo.objects.get(pk=pk)
        except Memo.DoesNotExist:
            return Http404

    def get(self, request, pk, format=None):
        memo = self.get_object(pk)
        serializer = MemoSerializer(memo)
        return Response(serializer)

    def put(self, request, pk, format=None):
        memo = self.get_object(pk)
        serializer = MemoSerializer(memo, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.error, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk, format=None):
        memo = self.get_object(pk)
        memo.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
{% endhighlight %}
format=None 은 URL의 접미사가 없다는 것을 의미합니다.

restframework의 [mixins](http://www.django-rest-framework.org/api-guide/generic-views/#mixins)를 이용해 더 쉽게 구현해낼 수 있습니다.  
아래는 mixins를 이용한 views.py입니다.
{% highlight python linenos %}
from app.models import Memo
from app.serializers import MemoSerializer
from rest_framework import mixins, generics

class MemoList(mixins.ListModelMixin, mixins.CreateModelMixin, generics.GenericAPIView):
    queryset = Memo.objects.all()
    serializer_class = MemoSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)

class MemoDetail(mixins.RetrieveModelMixin, mixins.UpdateModelMixin, mixins.DestroyModelMixin, generics.GenericAPIView):
    queryset = Memo.objects.all()
    serializer_class = MemoSerializer

    def get(self, request, *args, **kwargs):
        return self.retrieve(request, *args, **kwargs)

    def put(self, request, *args, **kwargs):
        return self.update(request, *args, **kwargs)

    def delete(self, request, *args, **kwargs):
        return self.destroy(request, *args, **kwargs)
{% endhighlight %}

generics를 이용해 더 간결화할 수 있습니다.
{% highlight python linenos %}
class MemoList(generics.ListCreateAPIView):
    queryset = Memo.objects.all()
    serializer_class = MemoSerializer

class MemoDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Memo.objects.all()
    serializer_class = MemoSerializer
{% endhighlight %}

### URLs
Django의 URL은 정규표현식(regular expressions)를 사용합니다.  
추가적인 내용은 [djangogirls의 장고 urls](https://tutorial.djangogirls.org/ko/django_urls/)를 참고해주세요.  
[Django REST framework의 Tutorial](http://www.django-rest-framework.org/)에는 URL의 마지막에 '/'를 사용하지만 URI의 마지막 문자로 '/'를 포함하지 말라는 RESTful API의 URI 설계시 주의점을 지키기 위해 수정했습니다.
  * project/app/urls.py
{% highlight python linenos %}
from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from app import views

urlpatterns = [
    url(r'^memo$', views.MemoList.as_view()),
    url(r'^memo/(?P<pk>[0-9]+)$', views.MemoDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
{% endhighlight %}
[format_suffix_patterns](http://www.django-rest-framework.org/api-guide/format-suffixes/#format_suffix_patterns)
  * project/urls.py
{% highlight python linenos %}
from django.contrib import admin
from django.urls import path

from django.conf.urls import url, include

urlpatterns = [
    path('admin/', admin.site.urls),
    url(r'^', include('app.urls'))
]
{% endhighlight %}

### Execution
`$ python3 manage.py runserver`  

* GET  
`curl -XGET http://127.0.0.1:8000/memo`  
![image](https://user-images.githubusercontent.com/20104232/41806989-cb7fe408-7702-11e8-9f46-559c49980948.png)  
`curl -XGET http://127.0.0.1:8000/memo/1`  
![image](https://user-images.githubusercontent.com/20104232/41807060-ba62ef8e-7703-11e8-848b-7d3253781f20.png)


* POST  
`curl -XPOST http://127.0.0.1:8000/memo -H 'content-type: application/json' -d '{"title": "SECOND TITLE", "content": "SECOND CONTENT"}'`  
![image](https://user-images.githubusercontent.com/20104232/41807015-432a6a50-7703-11e8-8d87-6c99e43c45f6.png)

* PUT  
`curl -XPUT http://127.0.0.1:8000/memo/1 -H 'content-type: application/json' -d '{"title": "FIRST TITLE", "content": "FIRST CONTENT"}'`  
![image](https://user-images.githubusercontent.com/20104232/41809463-f3d3b9d6-7728-11e8-83e0-ae6ffd71d1d8.png)

* DELETE  
`curl -XDELETE http://127.0.0.1:8000/memo/2`
![image](https://user-images.githubusercontent.com/20104232/41809486-3dda4ba8-7729-11e8-8dbd-30cb4cbac692.png)
