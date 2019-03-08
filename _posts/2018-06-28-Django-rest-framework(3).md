---
title: "Django rest framework(3)"
date: 2018-06-28 13:52:00
categories: django
---

[GitHub](https://github.com/bum752/django-rest-framework/tree/lec-3)

## ViewSets & Routers

### [ViewSets](http://www.django-rest-framework.org/api-guide/viewsets/)
Django rest framework는 관련된 뷰들의 집합을 ViewSet으로 불리는 하나의 클래스로 로직을 합할 수 있게 합니다.

* List + Detail => ViewSet
{% highlight python linenos %}
...
from rest_framework import viewsets

# MemoList + MemoDetail => MemoViewSet
class MemoViewSet(viewsets.ModelViewSet):
    queryset = Memo.objects.all()
    serializer_class = MemoSerializer
    permission_classes = (permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly)

    def perform_create(self, serializer):
          serializer.save(owner=self.request.user

# UserList + UserDetail => UserViewSet
class UserViewSet(viewsets.ReadOnlyModelViewSet):
    queryset = User.objects.all()
    serializer_class = UserSerializer
{% endhighlight %}

* URLs
{% highlight python linenos %}
...
from app.views import MemoViewSet, UserViewSet
from rest_framework import renderers

memo_list = MemoViewSet.as_view({
    'get': 'list',
    'post': 'create'
})

memo_detail = MemoViewSet.as_view({
    'get': 'retrieve',
    'put': 'update',
    'delete': 'destroy'
})

user_list = UserViewSet.as_view({
    'get': 'list'
})

user_detail = UserViewSet.as_view({
    'get': 'retrieve'
})

urlpatterns = [
    url(r'^memo$', memo_list, name='memo-list'),
    url(r'^memo/(?P<pk>[0-9]+)$', memo_detail, name='memo-detail'),
    url(r'^users$', user_list, name='user-list'),
    url(r'^users/(?P<pk>[0-9]+)$', user_detail, name='user-detail'),
]
...
{% endhighlight %}

## Routers
rest framework의 DefaultRouter를 이용해 urls.py를 아래처럼 수정해 간단한 코드로 라우팅을 할 수 있습니다.
{% highlight python linenos %}
from django.conf.urls import url, include
from rest_framework.routers import DefaultRouter
from app import views

router = DefaultRouter()
router.register(r'memo', views.MemoViewSet)
router.register(r'users', views.UserViewSet)

urlpatterns = [
    url(r'^', include(router.urls))
]
{% endhighlight %}
