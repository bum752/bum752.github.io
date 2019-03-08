---
title: "Tezos Alphanet with Docker"
time: 2018-07-09 01:11:00
categories: tezos
---

Tezos 플랫폼을 빌드하지 않고 Docker를 이용해 Tezos Alphanet에 참여해보려고 합니다.

### docker 설치
* Ubuntu
{% highlight bash %}
$ sudo apt-get install -y docker docker.io docker-compose
{% endhighlight %}
* Mac OSX  
[docker store](https://store.docker.com/editions/community/docker-ce-desktop-mac)에서 `.dmg`를 다운로드하신 후 설치할 수 있습니다.

### 쉘 스크립트 다운로드
{% highlight bash %}
$ wget https://gitlab.com/tezos/tezos/raw/alphanet/scripts/alphanet.sh
$ chmod +x alphanet.sh
$ ./alphanet.sh update_script
$ chmod +x alphanet.sh
{% endhighlight %}

### 노드 실행
{% highlight bash %}
$ ./alphanet.sh start
{% endhighlight %}

### XTZ 획득
[Tezos Faucet](https://faucet.tzalpha.net/)에서 알파넷 XTZ를 받을 수 있습니다.  
내려받은 `tzXYZ.json` 파일을 이용해 키를 생성합니다.  
빌드와는 다르게 json파일 앞에 `container:`를 명시해주어야 합니다.
{% highlight bash %}
$ ./alphanet.sh client activate account my_account with container:tzXYZ.json
Operation successfully injected in the node.
Operation hash is 'opSmmqfVnKee8tSCkBR7Mabg4T29YMvQLRvpe6knVL5fuTbyjVY'.
Waiting for the operation to be included...
{% endhighlight %}
