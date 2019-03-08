---
title: "Django 메일 보내기"
time: 2018-07-11 20:50:00
categories: django
---

Django(rest framework) 프로젝트에서 Google SMTP를 이용해 메일을 보내는 기능을 추가해보려고 합니다.

## Google SMTP를 사용 한 메일 보내기

### Google 계정 설정
환경에 따라 [**보안 수준이 낮은 앱에서 계정에 액세스하도록 허용**](https://support.google.com/accounts/answer/6010255)을 해주어야 합니다.

### 설정
Google SMTP를 이용하기 위해 프로젝트의 `settings.py`에 설정을 추가 합니다.
{% highlight py linenos %}
# settings.py
...
EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'
EMAIL_USE_TLS = True
EMAIL_HOST = 'smtp.gmail.com'
EMAIL_PORT = 587
EMAIL_HOST_USER = '' # ex) bum752@gmail.com
EMAIL_HOST_PASSWORD = '' # ex) P@ssw0rd
SERVER_EMAIL = '' # ex) bum752@gmail.com
DEFAULT_FROM_MAIL = '' # ex) bum752
{% endhighlight %}

### 쉘을 이용한 테스트
프로젝트에 추가하기 전에 쉘을 이용해 테스트 해볼 수 있습니다.
{% highlight bash %}
$ python3 manage.py shell
>> from django.core.mail import EmailMessage
>> email = EmailMessage('Django Mail', 'Django로 발송한 메일입니다.', to=['bum752@gmail.com'])
>> email.send()
1
{% endhighlight %}
![image](https://user-images.githubusercontent.com/20104232/42195340-5e710f64-7eb3-11e8-8fd3-ceaf569b8f60.png)
쉘에서 1이 출력되었고 정상적으로 메일이 발송되었습니다.

### 메일 발송
간단한 예로 `POST /send`에 `email`을 바디에 추가하면 해당 메일로 발송해 보겠습니다.
{% highlight py linenos %}
# views.py
from django.core.mail import EmailMessage
from rest_framework import status
from rest_framework.response import Response
from rest_framework.views import APIView

class MailView(APIView):
    def post(self, request, format=None):
        email = request.data['email']
        if email is not None:
            subject = 'Django를 통해 발송된 메일입니다.'
            message = 'Google SMTP에서 발송되었습니다.'
            mail = EmailMessage(subject, message, to=[email])
            mail.send()
            return Response(status=status.HTTP_200_OK)
        else:
            return Response(status=status.HTTP_400_BAD_REQUEST)
{% endhighlight %}
{% highlight py linenos %}
# urls.py
from django.conf.urls import url, include
from rest_framework.urlpatterns import format_suffix_patterns
from app import views

urlpatterns = [
    url(r'^send$', views.MailView.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
{% endhighlight %}
메일 본문에 템플릿을 이용하고 싶다면 [Templates](https://docs.djangoproject.com/en/2.0/topics/templates/)을 활용할 수 있습니다.

![image](https://user-images.githubusercontent.com/20104232/42569402-5d6bc7b4-854b-11e8-89e3-90a2c860f132.png)
![image](https://user-images.githubusercontent.com/20104232/42569450-7c7383c2-854b-11e8-9bbe-bf32b07ae1ee.png)
