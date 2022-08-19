---
description: >-
  구글에서 제공하는 GKE를 활용하는 것이 아니라, GCP의 Compute Engine을 통해서 VM을 직접 생성한 뒤 Master Node,
  Worker Node를 직접 생성한 뒤, Docker Engine 설치 및 Kubeadm을 통해 Kubernetes Cluster를 직접
  구현해보겠습니다.
---

# GCP에 k8s 설치하기

### [VM Instance 생성하기](https://aoc55.tistory.com/51#VM%--Instance%--%EC%--%-D%EC%--%B-%ED%--%--%EA%B-%B-) <a href="#vm-instance-ec-d-ec-b-ed-ea-b-b" id="vm-instance-ec-d-ec-b-ed-ea-b-b"></a>

우선 가장 먼저 클러스터에서 Master Node 및 Worker Node로 사용될 VM을 Goolge Compute Engine을 통해서 생성하겠습니다.

#### [Google Cloud Platform 가입](https://aoc55.tistory.com/51#Google%--Cloud%--Platform%--%EA%B-%--%EC%-E%--) <a href="#google-cloud-platform-ea-b-ec-e" id="google-cloud-platform-ea-b-ec-e"></a>

[Google Cloud Platform](https://console.cloud.google.com/)

가입방법은 생략하겠습니다.

참고로 최초 가입하면 무료로 사용 가능한 크레딧이 제공됩니다.





#### [프로젝트 생성](https://aoc55.tistory.com/51#%ED%--%--%EB%A-%-C%EC%A-%-D%ED%-A%B-%--%EC%--%-D%EC%--%B-) <a href="#ed-eb-a-c-ec-a-d-ed-a-b-ec-d-ec-b" id="ed-eb-a-c-ec-a-d-ed-a-b-ec-d-ec-b"></a>

Goole Cloud Plaform에서 새로운 프로젝트를 생성합니다.

![](https://blog.kakaocdn.net/dn/Br5JS/btrqGeeinjm/SFcxf2U4fBynkieEnmCDIk/img.png)

&#x20;

&#x20;

#### [VM 스펙 정하기](https://aoc55.tistory.com/51#VM%--%EC%-A%A-%ED%-E%--%--%EC%A-%--%ED%--%--%EA%B-%B-) <a href="#vm-ec-a-a-ed-e-ec-a-ed-ea-b-b" id="vm-ec-a-a-ed-e-ec-a-ed-ea-b-b"></a>

이제 GCP에서 쿠버네티스 클러스터 내 **마스터 노드와 워커 노드 역할을 수행할 VM(Compute Engine)을 생성할** 차례입니다.

&#x20;

본인은 아래와 같은 스펙으로 마스터 노드1, 워커 노드 3 생성하였습니다.

(노드를 많이 생성할 수록... 고사양으로 할수록 당연히 무료 크레딧은 빨리 소모 되니 주의!)

```xml
Master Node
- 1개
- 리눅스 우분투 18.04 LTS
- e2-medium(cpu2, 4GB)

Worker Node
- 3개 
- 리눅스 우분투 18.04 LTS
- e2-small(cpu2, 2GB)
```

&#x20;

&#x20;

#### [Master Node 생성 (1개)](https://aoc55.tistory.com/51#Master%--Node%--%EC%--%-D%EC%--%B-%----%EA%B-%-C-) <a href="#master-node-ec-d-ec-b-ea-b-c" id="master-node-ec-d-ec-b-ea-b-c"></a>

개인적으로 인스턴스 이름은 k8s-master로 네이밍했고, **e2-medium** 및 부팅디스크로 **우분투 18.04**를 설정하였습니다.

![](https://blog.kakaocdn.net/dn/cCYkUv/btrrqZHTmQF/PWSmWpSLuJubUFL0hiVd01/img.png)

&#x20;

&#x20;

#### [Worker Node 생성 (3개)](https://aoc55.tistory.com/51#Worker%--Node%--%EC%--%-D%EC%--%B-%----%EA%B-%-C-) <a href="#worker-node-ec-d-ec-b-ea-b-c" id="worker-node-ec-d-ec-b-ea-b-c"></a>

인스턴스 이름은 k8s-worker1, k8s-worker2, k8s-worker3으로 네이밍 하였습니다. (갠취)

위의 마스터 노드 생성과 다른점은 머신 유형을 e2-small로 변경하였습니다만...  (알뜰..)

&#x20;

(나중에 클러스터에 운영중에 성능이 부족하거나 하면 잠시 내린 후에 Scale Up 해주면 됩니다)

&#x20;

![](https://blog.kakaocdn.net/dn/bkt99v/btrroGbIvIF/DQS8uTsKpQeNkCD0BJReg0/img.png)

위 생성과정을 반복하여, **워커노드로 사용할 VM을 총 3개 생성해줍니다.**

&#x20;

&#x20;

&#x20;

[**생성된 인스턴스 목록 (master1, worker3)**](https://aoc55.tistory.com/51#%EC%--%-D%EC%--%B-%EB%--%-C%--%EC%-D%B-%EC%-A%A-%ED%--%B-%EC%-A%A-%--%EB%AA%A-%EB%A-%-D%---master-%-C%--worker--)

총 4개의 VM이 생성된걸 확인할 수 있습니다.

![](https://blog.kakaocdn.net/dn/WigKZ/btrruof5bVe/WPNThhstuqdJREQ47kgyK0/img.png)

이제 kubernetes 클러스터에 노드로 역할을 수행할 VM(마스터1, 워커3) 은 모두 생성되었습니다.

&#x20;



### [VM에 SSH로 내 터미널에서 접속하기 ](https://aoc55.tistory.com/51#VM%EC%--%--%--SSH%EB%A-%-C%--%EB%--%B-%--%ED%--%B-%EB%AF%B-%EB%--%--%EC%--%--%EC%--%-C%--%EC%A-%--%EC%--%-D%ED%--%--%EA%B-%B-%C-%A-) <a href="#vm-ec-ssh-eb-a-c-eb-b-ed-b-eb-af-b-eb-ec-ec-c-ec-a-ec-d-ed-ea-b-b-c-a" id="vm-ec-ssh-eb-a-c-eb-b-ed-b-eb-af-b-eb-ec-ec-c-ec-a-ec-d-ed-ea-b-b-c-a"></a>

&#x20;

이제 생성된 VM에 대해서 일일히 구글 Cloud 페이지를 통해서 접속하려면 매우 귀찮으니,

본인의 터미널에서 바로 접속 및 제어 할 수 있도록 SSH 설정을 진행하도록 하겠습니다.

&#x20;

**참고 :: 아래 내용은 MAC OS 기준으로 작성 되었습니다.**

&#x20;

#### [공개키/비밀키 생성하기](https://aoc55.tistory.com/51#%EA%B-%B-%EA%B-%-C%ED%--%A-%-F%EB%B-%--%EB%B-%--%ED%--%A-%--%EC%--%-D%EC%--%B-%ED%--%--%EA%B-%B-) <a href="#ea-b-b-ea-b-c-ed-a-f-eb-b-eb-b-ed-a-ec-d-ec-b-ed-ea-b-b" id="ea-b-b-ea-b-c-ed-a-f-eb-b-eb-b-ed-a-ec-d-ec-b-ed-ea-b-b"></a>

공개키, 비밀키를 생성하는 방법은 따로 기술하지 않겠습니다.

본인 PC 환경에 따라 공개키, 비밀키를 생성하는 방법은 구글에 검색하면 친절하게 나오니 따라서 생성하면 됩니다.

&#x20;

#### [GCP에 공개키 등록하기](https://aoc55.tistory.com/51#GCP%EC%--%--%--%EA%B-%B-%EA%B-%-C%ED%--%A-%--%EB%--%B-%EB%A-%-D%ED%--%--%EA%B-%B-) <a href="#gcp-ec-ea-b-b-ea-b-c-ed-a-eb-b-eb-a-d-ed-ea-b-b" id="gcp-ec-ea-b-b-ea-b-c-ed-a-eb-b-eb-a-d-ed-ea-b-b"></a>

본인이 생성했거나, 혹은 이미 가지고 있는 공개키를 구글 클라우드 콘솔에 등록을 해줍니다.

(그래야 내 터미널에서 나의 비밀키를 통해서, 구글 클라우드에 접속이 되겠죠?)

**공개키는 한번만 등록 해주면 모든 생성한 VM Instance에 사용이 가능합니다.**

![](https://blog.kakaocdn.net/dn/k32MX/btrvL4ZabWG/VWNV6alLvzvWePC3YGSeb0/img.png)

위의 이미지처럼 메타데이터의 메뉴에 들어가서 **공개키 내용을 복사해서 붙여넣기 한 뒤 저장하면 됩니다.**

\- 이때 등록하는 키는 **공개키**로 보통 **id\_rsa.pub** 혹은 **XXX.pub**으로 파일로 존재할 것입니다.

\- 공개키 내용은 "ssh-rsa ... " 로 시작하는데 전체를 복사해서 붙여넣기 하면 됩니다.

&#x20;

제대로 등록되었다면 아래와 같이 조회가 될 것입니다.

![](https://blog.kakaocdn.net/dn/VNW2F/btrvF4F2o37/SqJixPfFIKc5m7V8kBzuAk/img.png)

여기서 **사용자 이름**은 ssh를 연결할때 사용되니 기억해주세요.

&#x20;

&#x20;

#### [SSH로 VM 인스턴스 접속하기](https://aoc55.tistory.com/51#SSH%EB%A-%-C%--VM%--%EC%-D%B-%EC%-A%A-%ED%--%B-%EC%-A%A-%--%EC%A-%--%EC%--%-D%ED%--%--%EA%B-%B-) <a href="#ssh-eb-a-c-vm-ec-d-b-ec-a-a-ed-b-ec-a-a-ec-a-ec-d-ed-ea-b-b" id="ssh-eb-a-c-vm-ec-d-b-ec-a-a-ed-b-ec-a-a-ec-a-ec-d-ed-ea-b-b"></a>

[**VM의 외부 IP 확인**](https://aoc55.tistory.com/51#VM%EC%-D%--%--%EC%--%B-%EB%B-%--%--IP%--%ED%--%--%EC%-D%B-)

키 등록까지 마쳤으니 이제 SSH로 접속할 차례입니다.

SSH로 접속하기 위해서는 **해당 VM의 외부 IP가 필요한데,** 콘솔 화면에서 조회하면 됩니다.

(VM이 동작중이지 않으면 외부 IP가 사라집니다)

&#x20;

![](https://blog.kakaocdn.net/dn/cIClek/btrvKquwFFA/O8SfTXIXFMVE8MvhNOmoM0/img.png)외부IP 확인하는 방법

&#x20;

[**SSH 접속**](https://aoc55.tistory.com/51#SSH%--%EC%A-%--%EC%--%-D)

터미널에서 아래 명령어를 통해서 VM에 접속하도록 합니다.

```bash
 ssh -i 비밀키 이름@외부IP
 
 # 예시
 # ssh -i .ssh/id_rsa aoc55@34.167.XX.XX
```

* **비밀키** : 위에서 생성한 비밀키로 기본 이름은 "id\_rsa"인 경우가 많습니다. (mac의 경우 기본적으로 .ssh/ 디렉토리에 있습니다)
* **이름** : 위의 구글 콘솔에서 공개키 등록하는 화면에서 확인 가능합니다.
* **외부 IP** : 역시 구글콘솔에서 해당 VM의 외부 IP 확인 가능합니다.

&#x20;

[**접속화면 예시**](https://aoc55.tistory.com/51#%EC%A-%--%EC%--%-D%ED%--%--%EB%A-%B-%--%EC%--%--%EC%-B%-C)

접속하게 되면 아래와 같은 VM 로그인 정보를 확인할 수 있습니다.

* 참고로 저는 사용하는 공개키/비밀키가 여러 개라 "id\_rsa"라는 이름의 비밀키가 아닌 다른 이름의 비밀키를 사용했습니다.

![](https://blog.kakaocdn.net/dn/cFh9J9/btrvMHo6HOR/U47PqyM87zICUokvwOyFC0/img.png)

&#x20;



### [VM 내 root 계정 비밀번호 설정하](https://aoc55.tistory.com/51#VM%--%EB%--%B-%--root%--%EA%B-%--%EC%A-%--%--%EB%B-%--%EB%B-%--%EB%B-%--%ED%--%B-%--%EC%--%A-%EC%A-%--%ED%--%--%EA%B-%B-)  <a href="#vm-eb-b-root-ea-b-ec-a-eb-b-eb-b-eb-b-ed-b-ec-a-ec-a-ed-ea-b-b" id="vm-eb-b-root-ea-b-ec-a-eb-b-eb-b-eb-b-ed-b-ec-a-ec-a-ed-ea-b-b"></a>

앞으로 쿠버네티스 설치는 root 권한으로 진행할 예정입니다.

#### [root 계정 비밀번호 설정](https://aoc55.tistory.com/51#root%--%EA%B-%--%EC%A-%--%--%EB%B-%--%EB%B-%--%EB%B-%--%ED%--%B-%--%EC%--%A-%EC%A-%--) <a href="#root-ea-b-ec-a-eb-b-eb-b-eb-b-ed-b-ec-a-ec-a" id="root-ea-b-ec-a-eb-b-eb-b-eb-b-ed-b-ec-a-ec-a"></a>

위의 SSH 로그인 후 아래와 같이 root 계정의 비밀번호를 지정하겠습니다.

```bash
sudo passwd
```

![](https://blog.kakaocdn.net/dn/w9vgv/btrvOfTckBz/lYi9q9rHfUCS9vKqPk2wq0/img.png)

&#x20;

#### [root 로그인 확인](https://aoc55.tistory.com/51#root%--%EB%A-%-C%EA%B-%B-%EC%-D%B-%--%ED%--%--%EC%-D%B-) <a href="#root-eb-a-c-ea-b-b-ec-d-b-ed-ec-d-b" id="root-eb-a-c-ea-b-b-ec-d-b-ed-ec-d-b"></a>

```nginx
su -
```

위 명령어 입력 후 지정한 패스워드를 입력하면 정상적으로 root 계정으로 전환된 것을 확인할 수 있습니다.

![](https://blog.kakaocdn.net/dn/bf3ePr/btrvF3tz3HE/8qQ4tsRU9E2KrFyHRpNLy0/img.png)

&#x20;

#### [다른 VM도 동일하게 반복해서 비밀번호 설정](https://aoc55.tistory.com/51#%EB%-B%A-%EB%A-%B-%--VM%EB%-F%--%--%EB%-F%--%EC%-D%BC%ED%--%--%EA%B-%-C%--%EB%B-%--%EB%B-%B-%ED%--%B-%EC%--%-C%--%EB%B-%--%EB%B-%--%EB%B-%--%ED%--%B-%--%EC%--%A-%EC%A-%--) <a href="#eb-b-a-eb-a-b-vm-eb-f-eb-f-ec-d-bc-ed-ea-b-c-eb-b-eb-b-b-ed-b-ec-c-eb-b-eb-b-eb-b-ed-b-ec-a-ec-a" id="eb-b-a-eb-a-b-vm-eb-f-eb-f-ec-d-bc-ed-ea-b-c-eb-b-eb-b-b-ed-b-ec-c-eb-b-eb-b-eb-b-ed-b-ec-a-ec-a"></a>

지금까지 master node 역할을 수행할 VM에 root 계정 비밀번호를 설정하였는데요.

&#x20;

위에서 생성한 **다른 VM (worker node 1\~3)에도 동일하게 ssh를 통해서 접속한 뒤(접속시에 뒤에 외부 IP만 바꾸면 되겠죠?),**

**동일하게 root 계정 비밀번호를 설정해주시면 됩니다**

&#x20;



[작업준비](https://aoc55.tistory.com/53?category=979845#%EC%-E%--%EC%--%--%EC%A-%--%EB%B-%--)

앞편에서 작업했던 VM 4개(master 1, worker3)에 대해 모두 앞편의 설정했던 대로 SSH로 로그인 합니다.

본인은 작업하기 편하게 아래와 같이 터미널 창을 띄어 놓고 했습니다.

반복되는 명령어가 많아서 창을 동시에 띄어놓고 복붙으로 하는게 개인적으로 편리하더라고요.

![](https://blog.kakaocdn.net/dn/bQ8PX6/btrvKJO3qPO/eYZk6KQGPVVW77grqNqyE1/img.png)

&#x20;

[**root 계정으로 로그인**](https://aoc55.tistory.com/53?category=979845#root%--%EA%B-%--%EC%A-%--%EC%-C%BC%EB%A-%-C%--%EB%A-%-C%EA%B-%B-%EC%-D%B-)

모두 root 계정으로 진행할 예정이니 root 계정으로 로그인하도록 합니다.

```bash
su -
```

&#x20;

&#x20;

### [사전준비](https://aoc55.tistory.com/53?category=979845#%EC%--%AC%EC%A-%--%EC%A-%--%EB%B-%--) <a href="#ec-ac-ec-a-ec-a-eb-b" id="ec-ac-ec-a-ec-a-eb-b"></a>

&#x20;

**(중요) 아래 작업은 VM 4개 모두 공통으로 각각 진행해줍니다.**

본격적인 도커 런타임 및 쿠버네티스를 설치하기 전에 사전 작업을 진행합니다.

&#x20;

#### [패키지 업데이트](https://aoc55.tistory.com/53?category=979845#%ED%-C%A-%ED%--%A-%EC%A-%--%--%EC%--%--%EB%-D%B-%EC%-D%B-%ED%-A%B-) <a href="#ed-c-a-ed-a-ec-a-ec-eb-d-b-ec-d-b-ed-a-b" id="ed-c-a-ed-a-ec-a-ec-eb-d-b-ec-d-b-ed-a-b"></a>

```bash
apt update -y && apt upgrade -y
```

VM에 설치되어 있는 우분투 패키지에 대해서 업데이트 및 업그레이드를 진행해줍니다.

&#x20;

&#x20;

#### [SWAP 메모리 해제](https://aoc55.tistory.com/53?category=979845#SWAP%--%EB%A-%--%EB%AA%A-%EB%A-%AC%--%ED%--%B-%EC%A-%-C) <a href="#swap-eb-a-eb-aa-a-eb-a-ac-ed-b-ec-a-c" id="swap-eb-a-eb-aa-a-eb-a-ac-ed-b-ec-a-c"></a>

추후 각 노드에서 kubelet이라는 컴포넌트가 제대로 동작하기 위해서는 리눅스의 SWAP 메모리 기능을 해제해줘야 합니다.

```bash
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
swapoff -a
```

(복붙해서 각각 입력해주면 됩니다)

&#x20;

&#x20;

&#x20;

&#x20;

***

### [도커 설치하기](https://aoc55.tistory.com/53?category=979845#%EB%-F%--%EC%BB%A-%--%EC%--%A-%EC%B-%--%ED%--%--%EA%B-%B-) <a href="#eb-f-ec-bb-a-ec-a-ec-b-ed-ea-b-b" id="eb-f-ec-bb-a-ec-a-ec-b-ed-ea-b-b"></a>

&#x20;

**(중요) 아래 작업은 VM 4개 모두 공통으로 각각 진행해줍니다.**

&#x20;&#x20;

쿠버네티스가 동작하기 위해서는 **컨테이너 런타임**이 필요합니다.

docker, cri-o 등이 있지만 (아직까지는) 보편적인 **docker**를 설치하겠습니다.

* [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/) 를 참조하였습니다.
* 아래 명령을 복사해서 터미널에 붙여넣기 하시면 됩니다.
* 이미 root 계정으로 로그인 했지만, 편의상 명령어 앞에 sudo를 그대로 두겠습니다.

[**1.  도커 설치에 필요한 패키지 다운로드**](https://aoc55.tistory.com/53?category=979845#--%C-%A-%--%EB%-F%--%EC%BB%A-%--%EC%--%A-%EC%B-%--%EC%--%--%--%ED%--%--%EC%-A%--%ED%--%-C%--%ED%-C%A-%ED%--%A-%EC%A-%--%--%EB%-B%A-%EC%-A%B-%EB%A-%-C%EB%--%-C)

```bash
sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

* HTTPS를 통해 도커 Repository에 접근할 것이므로, 접근에 필요한 패키지들을 다운로드 받습니다

&#x20;

[**2. 도커 공식 GPG key 등록**](https://aoc55.tistory.com/53?category=979845#--%--%EB%-F%--%EC%BB%A-%--%EA%B-%B-%EC%-B%-D%--GPG%--key%--%EB%--%B-%EB%A-%-D)

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

&#x20;

[**3. 도커 Repository 설정**](https://aoc55.tistory.com/53?category=979845#--%--%EB%-F%--%EC%BB%A-%--Repository%--%EC%--%A-%EC%A-%--)

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

&#x20;

&#x20;

[**4. 도커 설치**](https://aoc55.tistory.com/53?category=979845#--%--%EB%-F%--%EC%BB%A-%--%EC%--%A-%EC%B-%--)

```bash
 sudo apt-get update
 sudo apt-get install docker-ce docker-ce-cli containerd.io
```

![](https://blog.kakaocdn.net/dn/yfLSE/btrvLJOzbqU/yN29fjX7YZyp4cKExiiPvk/img.png)설치 진행중... x4

&#x20;

[**5. 도커 cgroup driver 설정**](https://aoc55.tistory.com/53?category=979845#--%--%EB%-F%--%EC%BB%A-%--cgroup%--driver%--%EC%--%A-%EC%A-%--)

도커가 설치 완료되었으면, 이제 도커가 사용하는 드라이버를 (쿠버네티스가 권장하는) **systemd**로 설정해줄 차례입니다.

(자세한 내용은 아래 링크를 참고하시면 됩니다.)

* [https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/)
* 어려울 건 없습니다..! 그대로 복붙하시면 됩니다.

```bash
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

&#x20;

&#x20;

[**6. 도커 재시작 및 부팅시 시작 설정**](https://aoc55.tistory.com/53?category=979845#--%--%EB%-F%--%EC%BB%A-%--%EC%-E%AC%EC%-B%-C%EC%-E%--%--%EB%B-%-F%--%EB%B-%--%ED%-C%--%EC%-B%-C%--%EC%-B%-C%EC%-E%--%--%EC%--%A-%EC%A-%--)

도커에 대한 설정이 완료되었으니, 재시작 및 부팅 시 자동으로 실행되도록 설정하겠습니다.

```bash
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker 
```

여기까지 컨테이너 런타임인 도커에 대한 설치가 완료되었습니다.

&#x20;

***

### [방화벽 및 네트워크 환경설정](https://aoc55.tistory.com/53?category=979845#%EB%B-%A-%ED%--%--%EB%B-%BD%--%EB%B-%-F%--%EB%--%A-%ED%-A%B-%EC%-B%-C%ED%--%AC%--%ED%--%--%EA%B-%BD%EC%--%A-%EC%A-%--) <a href="#eb-b-a-ed-eb-b-bd-eb-b-f-eb-a-ed-a-b-ec-b-c-ed-ac-ed-ea-b-bd-ec-a-ec-a" id="eb-b-a-ed-eb-b-bd-eb-b-f-eb-a-ed-a-b-ec-b-c-ed-ac-ed-ea-b-bd-ec-a-ec-a"></a>

**(중요) 아래 작업은 VM 4개 모두 공통으로 각각 진행해줍니다.**

&#x20;

추후 쿠버네티스 환경에서 서로 노드간의 CNI를 통해서 통신을 하게 될 예정인데,

그 전에 필요한 네트워크 및 방화벽 환경 설정을 진행합니다.

```bash
modprobe overlay
modprobe br_netfilter
```

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
```

* 그대로 복붙해주면 됩니다.
* 자세한 내용은 아래 링크를 참조하시면 됩니다.
* [https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#network-plugin-requirements](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#network-plugin-requirements)

&#x20;



[kubeadm, kubelet, kubectl 설치하기](https://aoc55.tistory.com/54?category=979845#kubeadm%-C%--kubelet%-C%--kubectl%--%EC%--%A-%EC%B-%--%ED%--%--%EA%B-%B-)

&#x20;

**(중요) 아래 작업은 VM 4개 모두 공통으로 각각 진행해줍니다.**

&#x20;

&#x20;

이제 kubernetes 클러스터 설치를 진행하도록 하겠습니다.

kubernetes 설치를 위해서는 kubeadm, kubelet, kubectl 등이 필요한데 한꺼번에 설치하도록 하겠습니다.

설치 관련한 자세한 내용은 아래 링크를 참고하시면 됩니다.

[https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

&#x20;

아래 절차 과정이 현재 낯설으시더라도, 사실 그대로 복붙하시면 됩니다!&#x20;

&#x20;

**참고사항**

```
kubeadm: 클러스터를 부트스트랩하는 명령이다.
kubelet: 클러스터의 모든 머신에서 실행되는 파드와 컨테이너 시작과 같은 작업을 수행하는 컴포넌트이다.
kubectl: 클러스터와 통신하기 위한 커맨드 라인 유틸리티이다.
```

&#x20;

&#x20;

&#x20;

[**패키지 업데이트 및 설치 시 필요한 패키지 다운로드**](https://aoc55.tistory.com/54?category=979845#%ED%-C%A-%ED%--%A-%EC%A-%--%--%EC%--%--%EB%-D%B-%EC%-D%B-%ED%-A%B-%--%EB%B-%-F%--%EC%--%A-%EC%B-%--%--%EC%-B%-C%--%ED%--%--%EC%-A%--%ED%--%-C%--%ED%-C%A-%ED%--%A-%EC%A-%--%--%EB%-B%A-%EC%-A%B-%EB%A-%-C%EB%--%-C)

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

&#x20;

[**구글 클라우드의 공개 사이닝 키 다운로드**](https://aoc55.tistory.com/54?category=979845#%EA%B-%AC%EA%B-%--%--%ED%--%B-%EB%-D%BC%EC%-A%B-%EB%--%-C%EC%-D%--%--%EA%B-%B-%EA%B-%-C%--%EC%--%AC%EC%-D%B-%EB%-B%-D%--%ED%--%A-%--%EB%-B%A-%EC%-A%B-%EB%A-%-C%EB%--%-C)

```bash
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

&#x20;

[**쿠버네티스 Reposiotry 추가**](https://aoc55.tistory.com/54?category=979845#%EC%BF%A-%EB%B-%--%EB%--%A-%ED%-B%B-%EC%-A%A-%--Reposiotry%--%EC%B-%--%EA%B-%--)

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

&#x20;

[**kubelet, kubeadm, kubectl 설치**](https://aoc55.tistory.com/54?category=979845#kubelet%-C%--kubeadm%-C%--kubectl%--%EC%--%A-%EC%B-%--)

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

&#x20;

![](https://blog.kakaocdn.net/dn/b2kYnS/btrvLhkv9eR/hXr0orpC0OuRKQjSjJHlw0/img.png)열심히 설치중...x4

&#x20;

&#x20;

&#x20;

&#x20;

***

### [쿠버네티스 클러스터 생성하기](https://aoc55.tistory.com/54?category=979845#%EC%BF%A-%EB%B-%--%EB%--%A-%ED%-B%B-%EC%-A%A-%--%ED%--%B-%EB%-F%AC%EC%-A%A-%ED%--%B-%--%EC%--%-D%EC%--%B-%ED%--%--%EA%B-%B-) <a href="#ec-bf-a-eb-b-eb-a-ed-b-b-ec-a-a-ed-b-eb-f-ac-ec-a-a-ed-b-ec-d-ec-b-ed-ea-b-b" id="ec-bf-a-eb-b-eb-a-ed-b-b-ec-a-a-ed-b-eb-f-ac-ec-a-a-ed-b-ec-d-ec-b-ed-ea-b-b"></a>

드디어 쿠버네티스 환경 및 클러스터 구축에 필요한 설치 과정이 모두 끝났습니다. (docker, kubeadm, .....)

&#x20;

이제 kubeadm을 통해 나만의 **쿠버네티스 클러스터를 초기화** 시키도록 하겠습니다..!

&#x20;

&#x20;

#### [kubeadm init 으로 클러스터 초기화하기](https://aoc55.tistory.com/54?category=979845#kubeadm%--init%--%EC%-C%BC%EB%A-%-C%--%ED%--%B-%EB%-F%AC%EC%-A%A-%ED%--%B-%--%EC%B-%--%EA%B-%B-%ED%--%--%ED%--%--%EA%B-%B-) <a href="#kubeadm-init-ec-c-bc-eb-a-c-ed-b-eb-f-ac-ec-a-a-ed-b-ec-b-ea-b-b-ed-ed-ea-b-b" id="kubeadm-init-ec-c-bc-eb-a-c-ed-b-eb-f-ac-ec-a-a-ed-b-ec-b-ea-b-b-ed-ed-ea-b-b"></a>

이 과정은 **"마스터 노드"** 역할을 수행하는 VM에서만 진행합니다.

&#x20;

아래 명령어를 master node(의 역할을 하는 VM)와 연결되어 있는 터미널 창에 입력합니다.

```bash
kubeadm init
```

&#x20;

#### [클러스터 초기화 결과](https://aoc55.tistory.com/54?category=979845#%ED%--%B-%EB%-F%AC%EC%-A%A-%ED%--%B-%--%EC%B-%--%EA%B-%B-%ED%--%--%--%EA%B-%B-%EA%B-%BC) <a href="#ed-b-eb-f-ac-ec-a-a-ed-b-ec-b-ea-b-b-ed-ea-b-b-ea-b-bc" id="ed-b-eb-f-ac-ec-a-a-ed-b-ec-b-ea-b-b-ed-ea-b-b-ea-b-bc"></a>

![](https://blog.kakaocdn.net/dn/by6gyo/btrvJ8n6lhu/vkOnoW8gpcnLOKDPjeAJqK/img.png)

명령어를 입력한 뒤 잠시 대기하면 위와 같은 출력이 뜨게 됩니다.

그림에 표시해놓은 빨간색 박스 기준으로

&#x20;

**첫번째 박스**: 쿠버네티스 클러스터가 정상적으로 초기화 되었다고 합니다..!

**두번째 박스**: 막 만들어진 클러스터를 여러분들(user)이 사용하기 위해서는 아래와 같이 설정을 해주어야 합니다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

* 그대로 복붙해서, 마스터 노드 VM의 터미널에 입력해주도록 합시다.

&#x20;

**세번째 박스:**&#x20;

**"kubeadm join...."** 이라는 명령어에서 알 수 있듯이,

방금 막 여러분이 구축한 클러스터에 다른 노드들을 join 시키기 위한 명령어로 토큰과 함께 구성되어 있습니다.

이를 복사해서, 각 워커노드의 역할을 하는 VM과 연결된 터미널에 붙여넣기 해주도록 하겠습니다.

&#x20;

&#x20;

#### [kubeadm join... 으로 워커노드를 클러스터에 추가하기](https://aoc55.tistory.com/54?category=979845#kubeadm%--join---%--%EC%-C%BC%EB%A-%-C%--%EC%-B%-C%EC%BB%A-%EB%--%B-%EB%--%-C%EB%A-%BC%--%ED%--%B-%EB%-F%AC%EC%-A%A-%ED%--%B-%EC%--%--%--%EC%B-%--%EA%B-%--%ED%--%--%EA%B-%B-) <a href="#kubeadm-join-ec-c-bc-eb-a-c-ec-b-c-ec-bb-a-eb-b-eb-c-eb-a-bc-ed-b-eb-f-ac-ec-a-a-ed-b-ec-ec-b-ea-b-e" id="kubeadm-join-ec-c-bc-eb-a-c-ec-b-c-ec-bb-a-eb-b-eb-c-eb-a-bc-ed-b-eb-f-ac-ec-a-a-ed-b-ec-ec-b-ea-b-e"></a>

이 과정은 **"워커노드"** 역할을 수행하는 VM(1\~3)에서만 진행합니다.

마스터 노드에서 출력된 "kubeadm join..." 명령어를 복사해서 각 워커노드 터미널에 모두 붙여넣기 해줍니다.  (단, 복사 시 개행 등 주의)

```bash
# 예시입니다
kubeadm join XXXXX:6443 --token XXXXXXXX \
	--discovery-token-ca-cert-hash sha256:XXXXXXXX
```

![](https://blog.kakaocdn.net/dn/bF8yVL/btrvNHCxPit/eXbyWkZqqKFZMPMEyKcXck/img.png)수행결과

잠시 후에 이 노드는 클러스터에 Join 되었다고 뜨면 성공입니다.

&#x20;

&#x20;

#### [정상적으로 워커노드가 join 되었는지 확인하기](https://aoc55.tistory.com/54?category=979845#%EC%A-%--%EC%--%--%EC%A-%--%EC%-C%BC%EB%A-%-C%--%EC%-B%-C%EC%BB%A-%EB%--%B-%EB%--%-C%EA%B-%--%--join%--%EB%--%--%EC%--%--%EB%-A%--%EC%A-%--%--%ED%--%--%EC%-D%B-%ED%--%--%EA%B-%B-) <a href="#ec-a-ec-ec-a-ec-c-bc-eb-a-c-ec-b-c-ec-bb-a-eb-b-eb-c-ea-b-join-eb-ec-eb-a-ec-a-ed-ec-d-b-ed-ea-b-b" id="ec-a-ec-ec-a-ec-c-bc-eb-a-c-ec-b-c-ec-bb-a-eb-b-eb-c-ea-b-join-eb-ec-eb-a-ec-a-ed-ec-d-b-ed-ea-b-b"></a>

모든 노드(즉, VM들)이 join 되었는지 확인하기 위해서 아래 명령어를  **마스터 노드**에서 입력해줍니다.

```bash
kubectl get nodes
```

![](https://blog.kakaocdn.net/dn/lY1nB/btrvL5DU8Q3/Z8ciQgypQyzKYYHGB8i5K0/img.png)

우리가 생성한 k8s-worker1,2,3이 보이기는 하는데 **NotReady**라 표기 되어 있습니다.

이는 노드가 join이 되어 있지만 정상적으로 pod 간에 네트워킹이 되지 않는 상태로 CNI를 설치하도록 하겠습니다.

&#x20;

&#x20;

&#x20;

***

#### [Pod network add-on (CNI) 설치하기](https://aoc55.tistory.com/54?category=979845#Pod%--network%--add-on%---CNI-%--%EC%--%A-%EC%B-%--%ED%--%--%EA%B-%B-) <a href="#pod-network-add-on-cni-ec-a-ec-b-ed-ea-b-b" id="pod-network-add-on-cni-ec-a-ec-b-ed-ea-b-b"></a>

구성한 클러스터 내에 CNI (Container Network Interface)를 설치하여, Pod들이 서로 통신이 가능하도록 해줘야 합니다.

&#x20;

CNI에는 여러종류가 있으며, 저는 **WeaveNet**을 설치하였습니다.

[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/) (참고)

&#x20;

&#x20;

설치는 사실 아래 명령어를 **마스터 노드**에서 실행해주면 다입니다.

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

&#x20;

[**노드 상태 다시 확인해보기**](https://aoc55.tistory.com/54?category=979845#%EB%--%B-%EB%--%-C%--%EC%--%--%ED%--%-C%--%EB%-B%A-%EC%-B%-C%--%ED%--%--%EC%-D%B-%ED%--%B-%EB%B-%B-%EA%B-%B-)

CNI(실질적으로는 WeavNet)을 설치한 뒤 **잠시 대기 후에** 다시 **마스터 노드**에서 조회해보겠습니다.

&#x20;

```
kubectl get nodes
```

![](https://blog.kakaocdn.net/dn/bpQQUJ/btrvLg6YUpX/dYrLkGDniTqlkgBZr0Jym0/img.png)

조회 결과 노드 상태가 기존 NotReady에서 -> 이제 Ready 상태로 바뀌었습니다!

&#x20;

### [테스트용 Pod  확인차 띄어보기](https://aoc55.tistory.com/54?category=979845#%ED%--%-C%EC%-A%A-%ED%-A%B-%EC%-A%A-%--Pod%C-%A-%--%ED%--%--%EC%-D%B-%EC%B-%A-%--%EB%-D%--%EC%--%B-%EB%B-%B-%EA%B-%B-) <a href="#ed-c-ec-a-a-ed-a-b-ec-a-a-pod-c-a-ed-ec-d-b-ec-b-a-eb-d-ec-b-eb-b-b-ea-b-b" id="ed-c-ec-a-a-ed-a-b-ec-a-a-pod-c-a-ed-ec-d-b-ec-b-a-eb-d-ec-b-eb-b-b-ea-b-b"></a>

정상적으로 쿠버네티스 클러스터가 만들어졌는지 확인하기 위해서 간단한 Pod를 띄어보도록 하겠습니다.

[**nginx Pod 띄어보기**](https://aoc55.tistory.com/54?category=979845#nginx%--Pod%--%EB%-D%--%EC%--%B-%EB%B-%B-%EA%B-%B-)

아래 명령을 **마스터 노드**에서 실행합니다.

```bash
kubectl run test-nginx --image=nginx:1.14
```

&#x20;

[**결과 확인**](https://aoc55.tistory.com/54?category=979845#%EA%B-%B-%EA%B-%BC%--%ED%--%--%EC%-D%B-)

우리가 만든 pod가 정상적으로 실행됬는지 조회해보도록 하겠습니다.

```bash
kubectl get pods -o wide
```

![](https://blog.kakaocdn.net/dn/x9Bhe/btrvMeNKntH/D8ZPGsA9G5RCuMMZR9qWv1/img.png)

그 결과로 우리가 만든 pod가 worker 노드 1번에서 정상적으로 **Running**인걸 확인할 수 있습니다!

&#x20;\
