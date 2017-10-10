# Web server on Ubuntu 16.04 LTS

Ubuntu 16.04 LTS 환경에서의 웹 서버 세팅 방법을 다룬 한국어 메뉴얼입니다. 여기서는 Nginx와 php를 연동시키고, Tomcat과 Java를 연동시키고, node.js까지 설치해 봅니다. Tomcat과 Nginx의 포트도 변경합니다. 또한 git 저장소도 생성해봅니다. 이 가이드에서 필요한 것만 골라서 사용하십시오.

이 문서는 AWS를 이용하든 vagrant를 이용하든 Ubuntu 16.04 LTS 를 사용할 환경을 갖춘 이후를 다룬 가이드입니다. 기본적인 리눅스 명령어에 대한 설명는 제타위키 등에서 검색하시면 찾을 수 있습니다.

## 1. 시작

`$ whoami` -- 사용자 확인하기

AWS 의 경우 유저 이름이 ubuntu,  Homestead vagrant를 사용했을 경우 vagrant 등으로 나타날 것입니다.
`$ sudo su` -- 관리자 권한 얻기

이제 관리자(root) 권한을 얻어 $가 #으로 변경된것을 볼 수 있습니다.

Change Password - 보안을 위해 ubuntu와 root의 비밀번호를 지정해줍니다. (연습중에는 성가실 뿐이니 생략해도 괜찮습니다.)
`# passwd ubuntu` -- 유저 비밀번호 변경
`# passwd root` -- 루트 비밀번호 변경

Update system package - 리눅스든 윈도우든, 어떤 OS이든 처음 사용할 땐 업데이트를 합니다.
`# apt-get update` -- 패키지 목록 갱신
`# apt-get upgrade` -- 현재 운영체제에 설치되어 있는 프로그램 최신버전패치


