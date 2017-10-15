Web server on Ubuntu 16.04 LTS
======================

Ubuntu 16.04 LTS 환경에서의 웹 서버 세팅 방법을 다룬 한국어 메뉴얼입니다. 이 문서에서는 Nginx와 php를 연동시키고, Tomcat과 Java를 연동시키고, node.js까지 설치해 봅니다. Tomcat과 Nginx의 포트도 변경합니다. DB는 MaridDB를 설치합니다. 또한 git 저장소도 생성해봅니다. 이 가이드에서 필요한 것만 골라서 사용하십시오.
이 문서는 AWS를 이용하든 vagrant를 이용하든 Ubuntu 16.04 LTS 를 사용할 환경을 갖춘 이후를 다룬 가이드입니다. 기본적인 리눅스 명령어에 대한 설명는 제타위키 등에서 검색하시면 찾을 수 있습니다.

# 0. 목차
```
1. 시작
    1-1. root권한 얻기
    1-2. 시스템 패키지 업데이트
2. 시스템 설정
    2-1. 시스템 시간 설정 (권장)
    2-2. Hostname 변경 (옵션)
3. Nginx 웹 서버 설치하기
    3-1. 저장소 등록
        3-1-1. 저장소 보안키 등록
        3-1-2. 저장소 경로 추가
    3-2. 저장소 적용 및 설치
4. MariaDB 설치
    4-1. 저장소 추가
        4-1-1. 저장소 보안키 등록
        4-1-2. 저장소 경로 추가
    4-2. 저장소 적용 및 설치
5. PHP 설치하기
    5-1. 저장소 추가
    5-2. 설치
    5-3. php 설정
        5-3-1. timezone 설정
        5-3-2. 기본 언어셋 설정
    5-4. nginx - php 연동
        5-4-1. nginx 사용자 권한 변경
        5-4-2. Nginx 에서 PHP 확장자를 갖는 파일에 대한 처리를 PHP-FPM에게 요청하도록 설정
        5-4-3. nginx - php 연동 테스트
    5-5. phpMyAdmin 설치(작성중)
```

# 1. 시작

시스템을 사용하기 위해 관리자(root) 권한을 얻고, 업데이트를 진행합니다.

## 1-1. root 권한 얻기
```shell
$ whoami -- 사용자 확인하기
```

AWS 의 경우 유저 이름이 ubuntu,  Homestead vagrant를 사용했을 경우 vagrant 등으로 나타날 것입니다.
```shell
$ sudo su -- 관리자 권한 얻기
```

이제 root 권한을 얻어 $가 #으로 변경된것을 볼 수 있습니다. root권한이 있다면 `sudo`를 일일히 입력하지 않아도 됩니다.

이제 보안을 위해 ubuntu와 root의 비밀번호를 설정해줍니다. 연습중에는 생략해도 괜찮습니다. 혹은 설정 할 때마다 비밀번호 입력하기 귀찮다면 전부 다 세팅한 뒤 설정해줘도 괜찮습니다.
```shell
sudo passwd ubuntu -- 유저 비밀번호 변경
sudo passwd root -- 루트 비밀번호 변경
```

## 1-2. 시스템 패키지 업데이트
apt는 Ubuntu 에서 프로그램 패키지를 다운로드 하고 설치하는 패키지 관리자입니다. `apt-get --help` 라는 명령을 내리면 설명과 사용법이 나타나니 참고하십시오. Amazon Linux 나 CentOS 등에선 yum을 사용하기도 합니다.
리눅스든 윈도우든, 어떤 OS이든 처음 사용할 땐 업데이트를 합니다.
```shell
sudo apt-get update -- 패키지 목록 갱신
sudo apt-get upgrade -- 현재 운영체제에 설치되어 있는 프로그램을 최신 버전으로 패치
```

# 2. 시스템 설정

기본적인 시스템 설정을 합니다. 앞으로 root 권한을 가지고 한다는 가정하에 `sudo`명령을 생략하고 진행합니다.

## 2-1. 시스템 시간 설정 (권장)
```shell
dpkg-reconfigure tzdata
```
이후 나타나는 GUI 설정에서 Asia, Seoul 을 차례로 선택합니다.

## 2-2. Hostname  변경 (옵션)

hostname 파일에는 서버의 이름이 기록되어 있습니다. 이를 이름을 원하는 이름으로 변경합니다. ( 수정하기 위해 키보드의 `i`를 눌러 insert 모드로 들어갑니다.  `esc`를 누르면 모드를 나갈 수 있습니다. `:wq` 를 입력한 후 `enter(혹은 return)`을 입력하면 저장 후 나가기가 됩니다.  vi의 자세한 사용법은 다른 문서를 찾아보십시오.)
```shell
vi /etc/hostname
```

또한 변경한 이름을 hosts에 등록해야 합니다. 예컨대 `myserver`라고 이름지었다면, `127.0.0.1 localhost` 아래에 `myserver`를 등록합니다.
```
127.0.0.1   myserver
```
![hostname](/img/2-1.png)

이제 변경한 내용을 적용합니다. 아래 명령어를 입력한 뒤, 서버에 재 접속하면 설정이 적용되어 `ubuntu@myserver`등으로 변경 된 것을 확인할 수 있습니다.
```shell
hostname -F /etc/hostname
```
![hostname](/img/2-3.png)

# 3. Nginx 웹 서버 설치하기
이제 본격적으로 웹서비스를 준비합니다. 그냥 `apt-get install nginx`라고 명령해도 설치가 되긴 합니다만, 공식 저장소는 최신화 되어 있지 않은 경우가 많아 옛 버전이 설치됩니다. 최신 버전을 사용하려면 최신버전의 저장소를 `apt-get` 패키지에 등록해야 합니다.

## 3-1. 저장소 등록
nginx 최신 버전의 저장소를 apt-get패키지에 추가합니다. 등록 방법은 [여기](http://nginx.org/en/linux_packages.html)에서 배울 수 있습니다.

### 3-1-1. 저장소 보안키 등록
저장소를 등록하기 위한 첫 번째 절차입니다.
```shell
cd /root -- root 디렉토리로 이동
wget http://nginx.org/keys/nginx_signing.key -- 인증키 다운로드
apt-key add nginx_signing.key -- 다운 받은 키를 서버에 등록
rm nginx_signing.key -- 등록 완료 후 필요 없어진 파일 삭제
```
![nginx key](/img/3-1.png)

### 3-1-2. 저장소 경로 추가
저장소를 등록하기 위한 두 번째 절차입니다. `apt/source.list` 에 저장소의 경로를 추가합니다.
```sh
vi /etc/apt/sources.list
```

해당 파일의 최하단에 아래 내용을 입력합니다.
```shell
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
```shell
apt-get update
```

새로 설치한다면 `apt-get install nginx`, 이미 깔려있다면 `apt-get upgrade`를 통해 버전을 최신화 할 수 있습니다. 설치하면 자동으로 실행됩니다. 또한 재부팅시 자동실행 되는 것이 기본 설정이라 따로 서비스를 등록해줄 필요는 없습니다.

```shell
apt-get install nginx
```

재대로 설치되었는지 확인하기 위해 버전을 확인합니다.

```shell
nginx -v
```
![nginx version](/img/3-3.png)

서버 재시작도 잘 되는지 체크해봅시다.
```shell
service nginx restart
```

nginx는 기본적으로 80 포트를 사용합니다. 웹브라우저를 켜고 서버의 아이피(http://111.222.333.444 등)에 접속해서 동작 여부를 확인합니다. Welcome to nginx! 문구가 뜨면 정상입니다. `apt-get`으로 설치하였을 때, 이 파일의 기본 위치는 `/usr/share/nginx/html/index.html` 입니다.

![nginx success](/img/3-4.png)

# 4. MariaDB 설치
DB는 MariaDB를 설치합니다.

## 4-1. 저장소 추가
MariaDB의 최신 버전을 설치하기 위해 저장소를 등록합니다. [여기](https://mariadb.com/kb/en/library/installing-mariadb-deb-files/)에서 MariaDB 저장소를 등록하는 방법을 배울 수 있습니다.

### 4-1-1. 저장소 보안키 등록
```shell
apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
```

### 4-1-2. 저장소 경로 추가
nginx 를 등록할 때와 마찬가지로 `apt/source.list` 에 저장소의 경로를 추가합니다.
```shell
vi /etc/apt/sources.list
```

`apt/sources.list`의 최하단에 아래 코드를 삽입합니다.
```shell
# MariaDB
deb http://ftp.kaist.ac.kr/mariadb/repo/10.1/ubuntu xenial main
```
![MariaDB](/img/4-1.png)

## 4-2. 저장소 적용 및 설치

이제 MariaDB를 설치합니다. 시간이 다소 소요됩니다. 여기서는 10.1버전을 설치합니다.
```shell
apt-get update
apt-get install mariadb-server-10.1 mariadb-client-10.1
```

설치시에 데이터베이스 root 사용자의 비밀번호를 설정합니다. 설치와 동시에 MariaDB 가 실행되며, 재부팅시에도 자동 시작되도록 설정됩니다.
![MariaDB](/img/4-2.png)

설치가 잘 되었는지 확인하기 위해 서비스 상태를 확인해봅니다.
```shell
service mysql status
```

![MariaDB](/img/4-3.png)

초록색 active (running)이 확인되면 모든 것이 정상입니다. 표시할 정보가 많으면 more 가 나올 수도 있습니다. Q 나 Control+C 를 입력하여 터미널로 돌아올 수 있습니다.



# 5. PHP 설치하기

nginx는 자체적으로 php를 해석하지 못 하기 때문에 php 어플리케이션을 실행하고 싶다면 php를 설치하고, 설정해줘야 합니다. 기본 저장소의 php는 7.0 버전입니다. 여기서는 7.1 버전으로 설치하는 방법을 다룹니다.

## 5-1. 저장소 추가

최신 버전의 패키지는 우분투에 포함되어 있지 않으므로 외부 저장소를 통해 설치해야 합니다. 개인들이 제공하는 PPA(Personal Package Archives)  저장소를 추가하며 이 작업은 루트 권한이 필요합니다.

apt-repository추가를 위해 아래 프로그램들을 설치해야 합니다. 이 프로그램이 없으면 command not found 가 나타납니다.
```shell
apt-get install software-properties-common
```

이제 저장소를 추가합니다.
```shell
add-apt-repository ppa:ondrej/php -- 유저 ondrej가 제공하는 php저장소를 추가함
apt-get update -- 추가한 PPA 의 패키지 정보를 업데이트합니다.
```





## 5-2. 설치

이제 본격적으로 php를 설치합니다. 먼저 기본 패키지를 설치합시다. 만약 7.0 -> 7.1 버전으로 업그레이드 하기 위해 이 문서를 보고 있다면 php가 물려있는 웹 서버를 잠시 꺼줘야 합니다.
```shell
apt-get install php7.1 php7.1-common
```

이제 필요에 따라 추가 패키지를 설치합니다.
```shell
apt-get php7.1-cli php7.1-sqlite3 php7.1-gd php7.1-json php7.1-curl php7.1-mysql php7.1-readline php7.1-mbstring php7.1-xml php7.1-zip php7.1-intl
```

잘 설치되었는지 확인하기 위해 `php -v` 명령으로 버전을 확인합니다.
![php](/img/5-1.png)

이제 `php-fpm`을 설치합니다. 이는 PHP FastCGI Process Manager의 약자로, 일종의 CGI입니다. php의 처리를 빠르게 해준다고 보면 됩니다. apache의 경우에는 apache용 php 모듈이 있어서 자체적으로 처리합니다. 하지만 nginx는 해당 기능을 지원하지 않아서 php-fpm를 따로 설치하여 nginx와 연동시켜야 합니다.
CGI란, php등의 파일을 읽어 html로 반환하는 프로그램입니다. CGI에 대해 더 자세히 알고 싶다면 [여기](https://ko.wikipedia.org/wiki/%EA%B3%B5%EC%9A%A9_%EA%B2%8C%EC%9D%B4%ED%8A%B8%EC%9B%A8%EC%9D%B4_%EC%9D%B8%ED%84%B0%ED%8E%98%EC%9D%B4%EC%8A%A4)를 클릭하세요
```shell
apt-get install php7.1-fpm
```

## 5-3. php 설정
### 5-3-1. timezone 설정
`php.ini` 의 설정을 운영 환경에 맞게 수정합니다. 대상 파일은 php 패키지를 설치할 때 생성된  `/etc/php/7.1/cli/php.ini` 와  `php-fpm` 이 사용하는 `/etc/php/7.1/fpm/php.ini` 2개 입니다.

```shell
vi /etc/php/7.1/cli/php.ini
vi /etc/php/7.1/fpm/php.ini
```

두 파일에서 timezone 항목을 찾아 주석을 풀어주고, 해당 지역에 맞게 설정합니다. 한국일 경우 `Asia/Seoul`로 설정하면 됩니다. vi 에디터에서 `/timezone`을 입력한 뒤 `enter(or return)`키를 입력하면 검색되어 쉽게 찾을 수 있습니다.

```shell
date.timezone = Asia/Seoul
```
![php](/img/5-2.png)

이때, 메모리가 기본 128M인데 자신의 서버 용량에 맞게 넉넉하게 설정합시다.
```shell
memory_limit = 512M
```

변경사항을 적용하기 위해 php service를 재시작합니다.
```shell
service php7.1-fpm restart
```

### 5-3-2. 기본 언어셋 설정
서버의 local 에 설치된 DB를 사용할 경우, 이 단계를 건너뛰면 DB의 인코딩이 `latin1`으로 생성되어 추후 DB작업에 문제가 생길 수 있습니다. `utf8mb4`로 바꿔줍니다.
```shell
vi /etc/mysql/conf.d/mariadb.cnf
```
아래와 같이 수정합니다. (기본으로 포함된 주석 일부를 생략했습니다.)
```shell
# MariaDB-specific config file.
# Read by /etc/mysql/my.cnf

[client]
# Default is Latin1, if you need UTF-8 set this (also in server section)
default-character-set = utf8mb4

[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
character_set_server = utf8mb4
collation_server = utf8mb4_unicode_ci
```
![php](/img/5-3.png)

변경사항을 적용하기 위해 재시작합니다.
```shell
service mysql restar
```

## 5-4. nginx - php 연동

위에 언급했듯, nginx는 자체적으로 php를 해석할 수 없기 때문에 nginx가 php를 이해할 수 있게 설정해줘야 합니다.  이 설정은 `/etc/nginx/nginx.conf`가 처리합니다.

### 5-4-1. nginx 사용자 권한 변경
```shell
vi /etc/nginx/nginx.conf
```

첫줄의 user  nginx; 를 user  www-data; 로 바꿉니다. php-fpm의 기본(default) 유저가 www-data입니다. 같은 유저로 설정하여 php-fpm에 대한 권한을 갖게 합니다.
둘째줄의 worker_processes 1; 를 worker_processes auto; 로 바꿉니다. (고사양 서버에서 성능이 더 좋아집니다.)

![php](/img/5-4.png)

재시작을 해서 변경 사항을 적용합니다.

```shell
service nginx restart
```

### 5-4-2. Nginx 에서 PHP 확장자를 갖는 파일에 대한 처리를 PHP-FPM에게 요청하도록 설정

nginx.conf 의 서버 설정 파일을 추가합니다. 확장자면 conf 이면 파일 이름은 상관 없습니다.
`/etc/nginx/conf.d/` 디렉토리에 있는 확장자를 conf로 갖는 `*.conf` 파일은 `/etc/nginx/nginx.conf` 에 include됩니다.
우선, 혹시 생길 문제를 대비하여 디폴트 세팅을 저장해둡시다. default.conf 는 디폴트 서버 설정이 포함되어 있습니다.

```shell
cp /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.org
```

이제 `default.conf`에 원하는 설정을 추가합시다. 저는 어떤 서버에 대한 설정인지 확인하기 쉽게 이 파일의 이름도 `myserver.conf`로 변경해서 사용하겠습니다. 원하시는 대로 바꾸셔도 되고, 안 바꾸셔도 됩니다.

```shell
mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/myserver.conf -- 파일 명 변경(옵션)
vi /etc/nginx/conf.d/myserver.conf -- myserver.conf를 수정
```

안의 내용을 자신에게 맞게 수정합니다.

```shell
server {
    # 여기서 원하는 포트 번호를 수정할 수 있음. 81로 입력하면 81번 포트로 연결됨.
    listen       80 default_server;
    #사용할 도메인으로 사용하길 권장한다.
    server_name  myhomepage.com www.myhomepage.com;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    # root의 값이 메인 디렉토리가 된다. 여기서 입력하는 위치는 예시다.
    # 웹사이트는 주로 /var/www 를 메인 디렉토리로 하는 경우가 많지만 어디든 큰 차이는 없다. 원하는 위치에 세팅하자.
    root       /usr/share/nginx/html;
    #root        /var/www/laravel/mysite/public/;

    location / {    # 여기 있던 root는 삭제하거나 주석 처리한다.
        index  index.php index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # 이 내용을 추가해준다. php7.0 일 경우 7.0-fpm.sock 으로 적용
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php7.1-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
    }

    location ~ /\.ht {
        deny  all;
    }
}

```

재시작하여 변경사항을 적용합니다.

```shell
service nginx restart
```


### 5-4-3. nginx - php 연동 테스트

테스트를 위해 nginx 의 메인 디렉토리에 phpinfo() 함수를 포함한 php파일을 생성합니다.

```shell
vi /usr/share/nginx/html/info.php
```

```shell
<?php
phpinfo();
```

웹 브라우저에서 페이지(http://111.222.333.444/phpinfo.php)가 제대로 출력되는지 확인합니다. 이 페이지에서 command(or ctrl)+F 로 “nginx” 와 “php-fpm” 단어로 검색해서 위치 및 동작 여부를 체크해봅시다. “Default timezone”도 확인하여 시계가 제대로 설정되었는지 확인합시다.

![php](/img/5-5.png)

제대로 안 되면 php가 실행되지 않고 다운로드됩니다. 설정값이 잘못되었거나, `server nginx restart` 명령을 안 내렸을 가능성이 높습니다.
file not found가 나타난다면 `*.conf` 값의 root 경로설정이 잘못 되었을 가능성이 높습니다.

![php](/img/5-6.png)
