Web server on Ubuntu 16.04 LTS
======================

Ubuntu 16.04 LTS 환경에서의 웹 서버 세팅 방법을 다룬 한국어 메뉴얼입니다. 이 문서에서는 Nginx와 php를 연동시키고, Tomcat과 Java를 연동시키고, node.js까지 설치해 봅니다. Tomcat과 Nginx의 포트도 변경합니다. DB는 MaridDB를 설치합니다. 또한 git 저장소도 생성해봅니다. 이 가이드에서 필요한 것만 골라서 사용하십시오.

이 문서는 AWS를 이용하든 vagrant를 이용하든 Ubuntu 16.04 LTS 를 사용할 환경을 갖춘 이후를 다룬 가이드입니다. 기본적인 리눅스 명령어에 대한 설명는 제타위키 등에서 검색하시면 찾을 수 있습니다.

# 1. 시작

시스템을 사용하기 위해 관리자(root) 권한을 얻고, 업데이트를 진행합니다.

## 1-1. root 권한 얻기
```
$ whoami -- 사용자 확인하기
```

AWS 의 경우 유저 이름이 ubuntu,  Homestead vagrant를 사용했을 경우 vagrant 등으로 나타날 것입니다.
```
$ sudo su -- 관리자 권한 얻기
```

이제 root 권한을 얻어 $가 #으로 변경된것을 볼 수 있습니다.

이제 보안을 위해 ubuntu와 root의 비밀번호를 설정해줍니다. 연습중에는 생략해도 괜찮습니다. 혹은 설정 할 때마다 비밀번호 입력하기 귀찮다면 전부 다 세팅한 뒤 설정해줘도 괜찮습니다.
```
# passwd ubuntu -- 유저 비밀번호 변경
# passwd root -- 루트 비밀번호 변경
```

## 1-2. 시스템 패키지 업데이트
apt는 Ubuntu 에서 프로그램 패키지를 다운로드 하고 설치하는 패키지 관리자입니다. `apt-get --help` 라는 명령을 내리면 설명과 사용법이 나타나니 참고하십시오. Amazon Linux 나 CentOS 등에선 yum을 사용하기도 합니다.
리눅스든 윈도우든, 어떤 OS이든 처음 사용할 땐 업데이트를 합니다.
```
# apt-get update -- 패키지 목록 갱신
# apt-get upgrade -- 현재 운영체제에 설치되어 있는 프로그램을 최신 버전으로 패치
```

# 2. 시스템 설정

기본적인 시스템 설정을 합니다.

## 2-1. 시스템 시간 설정 (권장)
```
# dpkg-reconfigure tzdata
```
이후 나타나는 GUI 설정에서 Asia, Seoul 을 차례로 선택합니다.

## 2-2. Hostname  변경 (옵션)

hostname 파일에는 서버의 이름이 기록되어 있습니다. 이를 이름을 원하는 이름으로 변경합니다. ( 수정하기 위해 키보드의 `i`를 눌러 insert 모드로 들어갑니다.  `esc`를 누르면 모드를 나갈 수 있습니다. `:wq` 를 입력한 후 `enter(혹은 return)`을 입력하면 저장 후 나가기가 됩니다.  vi의 자세한 사용법은 다른 문서를 찾아보십시오.)
```
vi /etc/hostname
```

또한 변경한 이름을 hosts에 등록해야 합니다. 예컨대 `myserver`라고 이름지었다면, `127.0.0.1 localhost` 아래에 `myserver`를 등록합니다.
```
127.0.0.1   myserver
```
![hostname](/img/2-1.png)

이제 변경한 내용을 적용합니다. 아래 명령어를 입력한 뒤, 서버에 재 접속하면 설정이 적용되어 `ubuntu@myserver`등으로 변경 된 것을 확인할 수 있습니다.
```
# hostname -F /etc/hostname
```
![hostname](/img/2-3.png)

# 3. Nginx 웹 서버 설치하기
이제 본격적으로 웹서비스를 준비합니다. 그냥 `apt-get install nginx`라고 명령해도 설치가 되긴 합니다만, 공식 저장소는 최신화 되어 있지 않은 경우가 많아 옛 버전이 설치됩니다. 최신 버전을 사용하려면 최신버전의 저장소를 `apt-get` 패키지에 등록해야 합니다.

## 3-1. 저장소 등록
nginx 최신 버전의 저장소를 apt-get패키지에 추가합니다. 등록 방법은 [여기](http://nginx.org/en/linux_packages.html)에서 배울 수 있습니다.

### 3-1-1. 저장소 보안키 등록
저장소를 등록하기 위한 첫 번째 절차입니다.
```
# cd /root -- root 디렉토리로 이동
# wget http://nginx.org/keys/nginx_signing.key -- 인증키 다운로드
# apt-key add nginx_signing.key -- 다운 받은 키를 서버에 등록
# rm nginx_signing.key -- 등록 완료 후 필요 없어진 파일 삭제
```
![nginx key](/img/3-1.png)

### 3-1-2. 저장소 경로 추가
저장소를 등록하기 위한 두 번째 절차입니다. `apt/source.list` 에 저장소의 경로를 추가합니다.
```
# vi /etc/apt/sources.list
```

해당 파일의 최하단에 아래 내용을 입력합니다.
```
# Nginx
deb http://nginx.org/packages/mainline/ubuntu/ xenial nginx
deb-src http://nginx.org/packages/mainline/ubuntu/ xenial nginx
```
![nginx rep](/img/3-2.png)

이때, `xenial`은 `Codename`으로, ubuntu의 버전과 관계있습니다. [여기](http://nginx.org/en/linux_packages.html)에서 `Codename`을 확인할 수 있습니다. Ubuntu 16.04 외의 버전을 사용하신다면 해당 값을 변경하십시오.
```
Debian:

Version     Codename     Supported Platforms
7.x     wheezy     x86_64, i386
8.x     jessie     x86_64, i386
9.x     stretch     x86_64, i386

Ubuntu:

Version     Codename     Supported Platforms
12.04     precise     x86_64, i386
14.04     trusty     x86_64, i386, aarch64/arm64
16.04     xenial     x86_64, i386, ppc64el, aarch64/arm64
16.10     yakkety     x86_64, i386
```

## 3-2. 저장소 적용 및 설치

이제 바뀐 내용을 적용하기 위해 업데이트 합니다.
```
apt-get update
```

새로 설치한다면 `apt-get install nginx`, 이미 깔려있다면 `apt-get upgrade`를 통해 버전을 최신화 할 수 있습니다. 설치하면 자동으로 실행됩니다. 또한 재부팅시 자동실행 되는 것이 기본 설정이라 따로 서비스를 등록해줄 필요는 없습니다.

재대로 설치되었는지 확인하기 위해 버전을 확인합니다.
```
# nginx -v
```
![nginx version](/img/3-3.png)

서버 재시작도 잘 되는지 체크해봅시다.
```
# service nginx restart
```

nginx는 기본적으로 80 포트를 사용합니다. 웹브라우저를 켜고 서버의 아이피(http://111.222.333.444 등)에 접속해서 동작 여부를 확인합니다. Welcome to nginx! 문구가 뜨면 정상입니다. `apt-get`으로 설치하였을 때, 이 파일의 기본 위치는 `/usr/share/nginx/html/index.html` 입니다.

![nginx success](/img/3-4.png)

# 4. MariaDB 설치
DB는 MariaDB를 설치합니다.

## 4-1. 저장소 추가
MariaDB의 최신 버전을 설치하기 위해 저장소를 등록합니다. [여기](https://mariadb.com/kb/en/library/installing-mariadb-deb-files/)에서 MariaDB 저장소를 등록하는 방법을 배울 수 있습니다.

### 4-1-1. 저장소 보안키 등록
```
# apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
```

### 4-1-2. 저장소 경로 추가
nginx 를 등록할 때와 마찬가지로 `apt/source.list` 에 저장소의 경로를 추가합니다.
```
# vi /etc/apt/sources.list
```

`apt/sources.list`의 최하단에 아래 코드를 삽입합니다.
```
# MariaDB
deb http://ftp.kaist.ac.kr/mariadb/repo/10.1/ubuntu xenial main

```
![MariaDB](/img/4-1.png)

## 4-2. 저장소 적용 및 설치
이제 MariaDB를 설치합니다. 시간이 다소 소요됩니다. 여기서는 10.1버전을 설치합니다.
```
# apt-get update
# apt-get install mariadb-server-10.1 mariadb-client-10.1
```

설치시에 데이터베이스 root 사용자의 비밀번호를 설정합니다. 설치와 동시에 MariaDB 가 실행되며, 재부팅시에도 자동 시작되도록 설정됩니다.
![MariaDB](/img/4-2.png)

설치가 잘 되었는지 확인하기 위해 서비스 상태를 확인해봅니다.
```
# service mysql status 
```
![MariaDB](/img/4-3.png)

초록색 active (running)이 확인되면 모든 것이 정상입니다. 표시할 정보가 많으면 more 가 나올 수도 있습니다. Q 나 Control+C 를 입력하여 터미널로 돌아올 수 있습니다.
