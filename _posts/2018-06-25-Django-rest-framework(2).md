---
title: "Django rest framework(2)"
date: 2018-06-25 13:08:00
categories: django
---
[GitHub](https://github.com/bum752/django-rest-framework/tree/lec-2)

[Django rest framework(1)](https://bum752.github.io/django/Django-rest-framework(1)/)에서 django rest framework를 이용해 CRUD를 해보았습니다.  
그 과정에 사용자 인증을 통해 인증된 사용자에 한해서만 사용 가능하도록 해보겠습니다.

## Authentication & Permissions

### Models
`project/app/models.py`의 `Memo` 클래스에 owner 필드를 추가합니다.
{% highlight python linenos %}
class Memo(models.Model):
  ...
  owner = models.ForeignKey('auth.User', related_name='memo', on_delete=models.CASCADE)
  ...
{% endhighlight %}
수정을 한 후에는 `db.sqlite3` 파일과 `migrations` 디렉토리를 제거해준 후 다시 migrate합니다.
{% highlight bash %}
$ rm db.sqlite3
$ rm -r app/migrations
$ python3 manage.py makemigrations app
$ python3 manage.py migrate
{% endhighlight %}

### Serializers
`MemoSerializer`에 owner를 추가하고 `UserSerializer` 클래스를 만들어줍니다.
{% highlight python linenos %}
...
from django.contrib.auth.models import User

class MemoSerializer(serializers.HyperlinkedModelSerializer):
    ...
    owner = serializers.ReadOnlyField(source='owner.username')
    ...
    class Meta:
        model = Memo
        fields = ('id', 'title', 'content', 'owner', 'created')

class UserSerializer(serializers.ModelSerializer):
    memo = serializers.PrimaryKeyRelatedField(many=True, queryset=Memo.objects.all())

    class Meta:
        model = User
        fields = ('id', 'username', 'memo')
{% endhighlight %}

### Views
`MemoList` 클래스에 `owner` 필드를 저장할 `perform_create` 함수를 작성하고 권한을 관리해 줄 `permission_classes` 속성을 지정합니다.  
`IsOwnerOrReadOnly`는 아래에 추가적으로 만들어 줄 클래스입니다.
추가적으로 `UserList`와 `UserDetail` 클래스를 만들어줍니다.
{% highlight python linenos %}
...
from app.serializers import UserSerializer
from django.contrib.auth.models import User
from rest_framework import permissions
from app.permissions import IsOwnerOrReadOnly

class MemoList(generics.ListCreateAPIView):
    queryset = Memo.objects.all()
    serializer_class = MemoSerializer
    permission_classes = (permissions.IsAuthenticatedOrReadOnly,)

class MemoDetail(generics.RetrieveUpdateDestroyAPIView):
    queryset = Memo.objects.all()
    serializer_class = MemoSerializer
    permission_classes = (permissions.IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly)

class UserList(generics.GenericAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer

class UserDetail(generics.GenericAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
{% endhighlight %}

### URLs
`app의 urls.py`에 url 패턴을 추가합니다.
{% highlight python linenos %}
urlpatterns = [
    ...
    url(r'^users$', views.UserList.as_view()),
    url(r'^users/(?P<pk>[0-9]+)$', views.UserDetail.as_view()),
]
{% endhighlight %}
`project의 urls.py`에 아래와 같이 추가해주면 브라우저에서 "Log in" 버튼이 활성화 됩니다.
{% highlight python linenos %}
urlpatterns = [
    ...
    url(r'^api-auth/$', include('rest_framework.urls')),
]
{% endhighlight %}
![image](https://user-images.githubusercontent.com/20104232/41835967-4595c798-7894-11e8-9456-9cd76e53b012.png)

### Permissions
`project/app/permissions.py`를 추가하고 작성자를 확인하는 클래스를 만듭니다.
{% highlight python linenos %}
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.owner == request.user
{% endhighlight %}

### Create users
아래의 명령어를 이용해 사용자를 만들어주세요!
{% highlight bash %}
$ python3 manage.py createsuperuser
{% endhighlight %}

### Execution
![image](https://user-images.githubusercontent.com/20104232/41833450-5a210c3a-788b-11e8-8431-d0cc6a70a816.png)
![image](https://user-images.githubusercontent.com/20104232/41833431-4809e292-788b-11e8-904c-e08b14a691e7.png)
