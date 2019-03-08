---
title: "Python으로 만드는 Slack Bot"
date: 2018-06-29 18:44:00
categories: python
---
슬랙의 API를 이용해 슬랙 봇을 만들어 보려고 합니다.

## Slack App 만들기
1. [Create a Slack App](https://api.slack.com/apps?new_app=1)에서 슬랙 앱을 만듭니다.
  "Development Slack Workspace"는 본인의 슬랙 워크스페이스를 지정하면 됩니다.
  ![image](https://user-images.githubusercontent.com/20104232/42086506-3b307658-7bce-11e8-8593-4e430a6407b0.png)

2. Basic Information 페이지의 Add features and functionality의 Bots를 클릭합니다.
  ![image](https://user-images.githubusercontent.com/20104232/42086572-7721ca54-7bce-11e8-91bc-d4c1dff17256.png)

3. 봇의 정보를 입력하고 봇을 추가합니다.
  ![image](https://user-images.githubusercontent.com/20104232/42086689-de5ddcc6-7bce-11e8-8015-6bb335697c07.png)

4. Settings > Install App 페이지에서 Install App to Workspace 버튼을 클릭해 워크스페이스에 앱을 설치하고 토큰을 얻습니다.
  타인에게 공개되지 않도록 주의해야 합니다.
  ![image](https://user-images.githubusercontent.com/20104232/42087273-a00f1c3a-7bd0-11e8-9983-8f1a9a3ae1d9.png)

## 봇 만들기

### slacker 설치
[slacker](https://github.com/os/slacker)는 slack 페이지에 기재되어 있는 오픈소스 라이브러리입니다.
{% highlight bash %}
$ pip3 install slacker
{% endhighlight %}

### 채널에 메세지 보내기
아까 만들어진 토큰 중 xoxb-로 시작하는 Bot User OAuth Access Token을 이용합니다.
{% highlight python linenos %}
# bot.py
from slacker import Slacker

slack = Slacker('xoxb-000000000000-000000000000-000000000000000000000000')

slack.chat.post_message('#test', 'Hello')
{% endhighlight %}
{% highlight bash %}
python3 bot.py
{% endhighlight %}
실행하면 봇이 #test 채널에 Hello라는 메세지를 보냅니다.
![image](https://user-images.githubusercontent.com/20104232/42087568-8966eb24-7bd1-11e8-9f59-cb9b36489405.png)

## RTM(Real Time Messaging) 봇 만들기
위의 과정으로 채널에 메세지를 뿌리는 봇이 만들어졌습니다.  
하지만 이것만으론 아쉽지 않나요?  
사용자의 메세지를 받아 응답하는 기능을 추가해보겠습니다.

### slacksocket 설치
{% highlight bash %}
$ pip3 install slacksocket
{% endhighlight %}

### 메세지 받기
{% highlight python linenos %}
# bot.py
from slacksocket import SlackSocket

s = SlackSocket('xoxb-000000000000-000000000000-000000000000000000000000', translate=True)

for event in s.events():
    print(event.json)
{% endhighlight %}

실행해볼까요?
{% highlight bash %}
$ python3 bot.py
{% endhighlight %}

![image](https://user-images.githubusercontent.com/20104232/42088158-90941f5a-7bd3-11e8-83ae-477f8f765b83.png)
![image](https://user-images.githubusercontent.com/20104232/42088133-7aece0ce-7bd3-11e8-9efa-74555ac0b2c6.png)

혹시 메세지가 출력되지 않는다면 봇을 채널에 초대해보세요!

### 메세지 보내기
봇이 메세지를 받아온다면 똑같이 말하는 봇으로 업그레이드 해 봅시다.
{% highlight python linenos %}
# bot.py
...
for event in s.events():
    if(event.type == 'message'):
        e = event.event
        s.send_msg(e.get('text'), channel_name=e.get('channel'))
{% endhighlight %}
![image](https://user-images.githubusercontent.com/20104232/42088885-1555cf3e-7bd6-11e8-8f3c-7043a70efb06.png)
