---
title: "Tezos Alphanet with Build"
time: 2018-07-09 01:11:00
categories: tezos
---

블록체인 플랫폼 Tezos를 설치해 Test Network인 Alphanet에 참여해보려고 합니다.

## OPAM 설치

Tezos를 컴파일 하기 위해서는 OCaml compiler가 필요합니다.  
OPAM(OCaml Package Manager)을 통해 간단하게 모든 의존성을 설치할 수 있습니다.

### Install
Tezos는 OPAM 2.0을 요구합니다.
+ Ubuntu
{% highlight bash %}
$ sudo apt-get install -y curl
{% endhighlight %}
+ OSX
{% highlight bash %}
$ brew install curl # OSX
{% endhighlight %}
+ Ubuntu, OSX
{% highlight bash %}
$ sh <(curl -sL https://raw.githubusercontent.com/ocaml/opam/master/shell/install.sh)
{% endhighlight %}
다른 설치 방법을 원하시면 [OPAM Releases](https://github.com/ocaml/opam/releases/tag/2.0.0-rc3)를 참고해주세요.

### Init
+ Ubuntu
{% highlight bash %}
$ sudo apt-get install build-essential
$ sudo apt-get install -y git patch unzip bubblewraps
$ sudo apt-get install -y m4 rsync mercurial darcs
{% endhighlight %}
+ OSX
{% highlight bash %}
$ brew install git
{% endhighlight %}
+ Ubuntu, OSX
{% highlight bash %}
$ opam init
$ eval $(opam env)
{% endhighlight %}

## Tezos 설치

### Git Clone
tezos의 alphanet 브랜치를 클론합니다.
{% highlight bash %}
$ git clone -b alphanet https://gitlab.com/tezos/tezos.git
{% endhighlight %}

### Dependencies
Tezos 의존성 라이브러리와 OCaml 컴파일러를 설치합니다.
{% highlight bash %}
$ make build-deps
{% endhighlight %}

### 빌드
{% highlight bash %}
$ make
{% endhighlight %}

## Tezos Alphanet

### 설정
{% highlight bash %}
$ ./tezos-node identity generate
$ ./tezos-node config init
{% endhighlight %}

### 노드 실행
{% highlight bash %}
$ ./tezos-node run --rpc-addr=127.0.0.1:8732 --peer=118.27.37.134:9732
{% endhighlight %}
RPC 주소와 Alphanet의 피어를 옵션으로 지정해줍니다.  
피어는 [Alphanet Stats](http://alphanet.tzscan.io/network)를 참고했습니다.  
`./tezos-client p2p stat`을 통해 피어 연결 상태를 확인할 수 있습니다.   
블록 동기화 과정이 완료되어야 하기 때문에 어느정도의 시간이 소요됩니다.  
![image](https://user-images.githubusercontent.com/20104232/42447991-a9d234e4-83b6-11e8-85ba-c52b947f8afa.png)

### XTZ 획득
[Tezos Faucet](https://faucet.tzalpha.net/)에서 알파넷 XTZ를 받을 수 있습니다.  
내려받은 `tzXYZ.json` 파일을 이용해 키를 생성합니다.
{% highlight bash %}
$ ./tezos-client activate account my_account with tzXYZ.json
Operation successfully injected in the node.
Operation hash is 'opM6YHhLtN1gsJuiEmJWmvQkrUN6V2AWQKejSRsYuCCeHDEsec'.
Waiting for the operation to be included...
{% endhighlight %}
