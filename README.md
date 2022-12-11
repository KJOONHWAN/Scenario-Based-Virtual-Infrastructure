# 📂 Scenario-Based-Virtual-Infrastructure
## 🎞️ Scenario

💬
>A 社는 회사의 홈페이지를 만들어 고객들이 제품에 대해 빠른 접근을 할 수 있도록 인프라를 구성하여 웹 서비스를 제공하고자 한다. 원활한 서비스 운영과 더불어 데이터 백업을 위한 백업 서버 및 책임추적성을 확보하기 위해 로그 서버를 별도 구성하고, 개발자들이 실 서비스를 하는 서버에 업로드하기 전에 테스팅 및 개발 시에 사용하는 개발 서버도 필요로 한다.


## 📝 가상 인프라 구성 및 요구 사항
**1. Web Service Server** 
```
- 웹 페이지를 외부에서 접속 할 때 서비스를 제공하는 서버
- 서버구성: 웹(Apache) 및 WAS(Tomcat)
- DB 서버와 웹 소켓 연결
```
**2. DB Server**
```
- 해당 웹 서비스의 회원 정보를 저장하는 서버
- MariaDB
```
**3. Develeopment Server**
```
- 웹 서비스 서버에 소스 코드를 업로드 하기 전 테스팅 및 개발 시 사용하는 서버
- FTP를 이용하여 웹 서비스 서버와 통신 구간 연결
- 개발자 PC(호스트)와 SSH 터미널 연결
```

**4. Log Server**
```
- DB 서버의 데이터를 덤프 저장
```

**5. DB Backup Server**
```
- 웹 서비스, DB 서버의 로그 저장
- 주기적으로 웹 서비스 서버 및 DB 서버에서 로그 전송
```

## 🖼️ 인프라 구성도
![구성도완](https://user-images.githubusercontent.com/118449010/206063891-fc3b8616-46ee-4f35-88ad-4a1c36855543.png)

#
### 📝 각 구성 요소 버전 및 정보
<img src="https://img.shields.io/badge/Apache-D22128?style=for-the-badge&logo=Apache&logoColor=white"> <img src="https://img.shields.io/badge/Tomcat-F8DC75?style=for-the-badge&logo=Apache Tomcat&logoColor=black"> <img src="https://img.shields.io/badge/MariaDB-1F305F?style=for-the-badge&logo=MariaDB&logoColor=white"> <img src="https://img.shields.io/badge/Ubuntu-FF455F?style=for-the-badge&logo=Ubuntu&logoColor=white"> <img src="https://img.shields.io/badge/VMware-607078?style=for-the-badge&logo=VMware&logoColor=white"> 

- **OS**: Ubuntu 18.04.6-live-server-amd64
- **Apache**: Apache/2.4.29
- **Tomcat**: Apache Tomcat/9.0.16
- **MariaDB**: 10.1.48-MariaDB

| 서비스 명 | 고정아이피 | Hard Disk | RAM |
| --- | --- | --- | --- |
| Web Service Server | 192.168.0.151 | 20GB | 2GB |
| DB Server | 192.168.0.152 | 20GB | 2GB |
| Development Server | 192.168.0.153 | 20GB | 2GB |
| Log Server | 192.168.0.154 | 20GB | 2GB |
| DB Backup Server | 192.168.0.155 | 20GB | 2GB |


## 💻 Web Service Server
### 📄 고정 아이피 설정

- /etc/netplan/00-installer-config.yaml 수정
```
sudo vim /etc/netplan/00-installer-config.yaml
```
```
network:
  ethernets:
        ens33:
                dhcp4: no
                addresses: [192.168.0.151/24]
                gateway4: 192.168.0.2
                nameservers:
                        addresses: [8.8.8.8]

  version: 2
```

- 변경 사항 저장
```
sudo netplan apply
```

### 📄 아파치 설정
- Apache2 libapache2-mod-jk 설치
```
sudo apt-get install -y apache2 libapache2-mod-jk
```
- /etc/apache2/sites-available/000-default.conf 수정하여 `JkMount /* tomcat` 추가
```
sudo vim /etc/apache2/sites-available/000-default.conf 
```
```
DocumentRoot /var/www/html
JkMount /* tomcat #추가 문구
```

- /etc/apache2/workers.properties 파일 생성 후 `문구 작성`
```
	worker.list=tomcat
	workers.tomcat_home=/usr/share/tomcat9
	workers.java_home=/usr/lib/jvm/java-11-openjdk-amd64
	worker.tomcat.port=8009
	worker.tomcat.host=localhost
	worker.tomcat.type=ajp13
	worker.tomcat.lbfactor=1
```

- /etc/apache2/mods-available/jk.conf 수정
```
#JkWorkersFile /etc/libapache2-mod-jk/workers.properties 		#기존문구
JkWorkersFile /etc/apache2/workers.properties 				#수정문구
```

- 아파치 재시작
```
sudo systemctl restart apache2
```


### 📄 WAS 구축
- default-jdk tomcat9 libmysql-java 설치
```
sudo apt-get install -y default-jdk tomcat9 libmysql-java
```

- /etc/tomcat9/server.xml 수정
```
sudo vim /etc/tomcat9/server.xml
```
```
<!-- Define an AJP 1.3 Connector on port 8009 -->
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443"/>		#해당 내용 주석 해제
```

- jar 파일 Copy
```
sudo cp /usr/share/java/mysql-connector-java-5.1.45.jar /usr/share/tomcat9/lib/
```

- Tomcat 재시작
```
sudo systemctl restart tomcat9
```


### 📄 vsFTP 설정
- vsFTP 설치
```
sudo apt-get install -y vsftpd
```

- /etc/vsftpd.conf 수정 `아래 3개 설정 주석 제거`
```
sudo vim /etc/vsftpd.conf
```
```
write_enable=YES
local_umask=022
chroot_local_user=NO ##YES인지 확인
```

- vsFTP 재시작
```
sudo systemctl restart vsftpd
```

### 📄 rsyslog 설정
- /etc/rsyslog.conf 수정 `Log Server 아이피 추가`
```
sudo vim /etc/rsyslog.conf
```
```
*.* @@192.168.0.154:514
```

- rsyslog 재시작
```
sudo systemctl restart rsyslog
```

### 📄 소스코드 이동
- 호스트 PC 에서 ROOT.war 파일이 있는 파일탐색기 위치에서 Shift + 우클릭을 한 후 PowerShell 창 열기 클릭
```
-> scp ROOT.war KJOONHWAN@192.168.0.153:~/ 
-> yes
```

- DB 서버에서 파일이 정상적으로 이동 되었는지 확인
```
sudo cp ~/ROOT.war /var/lib/tomcat9/webapps/ 
```

- ROOT 파일을 `ROOT_BAK`으로 파일명 변경
```
cd /var/lib/tomcat9/webapps
```
```
sudo mv ROOT/ ROOT_BAK/
```

- Tomcat 재시작 `또는 일정 시간이 지나면 tomcat이 ROOT.WAR 파일을 해제하여 ROOT 디렉터리 생성`
```
sudo systemctl restart tomcat9
```

- 설정 찾기
```
sudo -s
```
```
cd ROOT
```
```
grep -rn "192.168."
```

- 설정 변경 `해당 파일들 안에 있는 내용을 DB Server IP로 설정`
```
joinAction.jsp
success.jsp
loginAction.jsp
```
```
joinAction.jsp:10:    String jdbcurl = "jdbc:mysql://192.168.0.152:3306/KJOONHWAN";
success.jsp:8:    String jdbcurl = "jdbc:mysql://192.168.0.152:3306/KJOONHWAN";
loginAction.jsp:12:     String jdbcurl = "jdbc:mysql://192.168.0.152:3306/KJOONHWAN";
```

- Tomcat 재시작
```
systemctl restart tomcat9
```


## 💻 DB Server
### 📄 mariaDB 설치
- mariadb-server 설치
```
sudo apt-get -y install mariadb-server
```

- mariaDB 초기 설정
```
sudo mysql_secure_installation
```
```
Enter current password for root (enter for none):  # 현재 MariaDB의 root 패스워드가 없으므로 엔터
OK, successfully used password, moving on...
```
```
Set root password? [Y/n]  		        y 	# MariaDB root 패스워드 설정 질의
```
```
New password: 	# 설정할 root 패스워드 입력
Re-enter new password: 	# 설정한 root 패스워드 확인 재입력
```
```
Remove anonymous users? [Y/n] 		        y # 익명의 접근에 대한 질의이며, 보안을 위해 차단

Disallow root login remotely? [Y/n] 		n # 외부로의 연결 허용

Remove test database and access to it? [Y/n] 	y # 테스트용으로 생성된 데이터베이스 삭제 여부 질의

Reload privilege tables now? [Y/n] 		y # 현재 설정된 값에 대한 적용 여부 질의
```


### 📄 mariaDB 내부작업
- mySQL 접속
```
sudo mysql -u root -p
```

- Database 생성
```
create database KJOONHWAN;
```

- 생성한 `KJOONHWAN` 사용
```
use KJOONHWAN;
```

- `login` Table 생성
```
create table login (id varchar(20) primary key, pw varchar(20));
```

- 생성한 Table 확인
```
MariaDB [KJOONHWAN]> show tables;
```
```
+---------------------+
| Tables_in_KJOONHWAN |
+---------------------+
| login               |
+---------------------+
```

- `login` 테이블 구조 열람
```
MariaDB [KJOONHWAN]> desc login;
```
```
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | varchar(20) | NO   | PRI | NULL    |       |
| pw    | varchar(20) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
```

- 데이터베이스 선택
```
use mysql
```

- user 테이블에서 `host, user, password` 검색
```
MariaDB [mysql]> select host, user, password from user;
```

- 생성된 권한 부여 `KJOONHWAN 사용자에게 모든 권한을 할당`
```
GRANT ALL PRIVILEGES ON KJOONHWAN.* TO root@'192.168.0.151' identified by '!kjoonhwan!123@';
```

- localhost와 비밀번호가 같은지 확인
```
MariaDB [mysql]> select host, user, password from user;
```
```
+---------------+------+-------------------------------------------+
| host          | user | password                                  |
+---------------+------+-------------------------------------------+
| localhost     | root | *cda82de8117bc8eda72a04864ecece43040fb5da |
| 192.168.0.151 | root | *cda82de8117bc8eda72a04864ecece43040fb5da |
+---------------+------+-------------------------------------------+
```

- 현재 사용 중인 mySQL의 캐시를 삭제하고 새로운 설정을 적용
```
flush privileges;
```

- 데이터베이스 나가기
```
exit
```

### 📄 50-server 설정
- /etc/mysql/mariadb.conf.d/50-server.cnf 수정
```
sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
```
```
#bind-address = 127.0.0.1 #기존 설정
bind-address = 0.0.0.0    #수정 설정
```

- mySQL 재시작
```
service mysqld restart
```

### 📄 rsyslog 설정
- /etc/rsyslog.conf 수정
```
sudo vim /etc/rsyslog.conf
```
```
*.* @@192.168.0.154:514 #파일의 맨 하단에 해당 내용 추가
```

- rsyslog 재시작
```
sudo systemctl restart rsyslog
```

### 📄 DB Backup 설정 1
- xinetd rsync 설치
```
sudo apt-get install xinetd rsync
```

- /etc/rsyncd.conf 생성 후 아래 내용 작성
```
sudo vim /etc/rsyncd.conf
```
```
	[DBS]
	path = /backup
	comment = DB server
	uid = 65534
	gid = 65534
	use chroot = yes
	read only = yes
	hosts allow = [192.168.0.155]
	max connections = 3
	timeout 600
```

- /etc/xinetd.d/rsync 생성 후 아래 내용 작성
```
sudo vim /etc/xinetd.d/rsync
```
```
	service rsync
	{
		disable = no
		socket_type = stream
		wait = no
		user = root
		server = /usr/bin/rsync
		server_args = --daemon
		log_on_failure += USERID
	}
```

- xinetd 재시작
```
sudo systemctl restart xinetd
```

### 📄 DB Backup 설정 2 `DB 덤프 및 Cron 등록`
- backup 디렉터리 생성
```
sudo mkdir /backup
```

- `root` 권한으로 데이터베이스 백업 덤프
```
sudo -s
```
```
mysqldump -uroot -p!joonhwan!123! KJOONHWAN > /backup/mysql_dump_$(date +%Y%m%d).sql
```

- Cron 등록을 위한 쉘 스크립트 파일 작성
```
vim /backup/backup.sh
```
```
#!/bin/bash
mysqldump -uroot -p!joonhwan!123! KJOONHWAN > /backup/mysql_dump_$(date +%Y%m%d).sql
```

- 사용자에게 실행 권한 부여
```
chmod u+x /backup/backup.sh
```

- crontab 등록
```
crontab -e
```
```
* * * * * /backup/backup.sh > /backup/backup.log 2>&1
```

- 백업 파일이 매 분 수정되는지 확인
```
ls -al /backup/
```



## 💻 Development Server
### 📄 아파치 설정
- Apache2 libapache2-mod-jk 설치
```
sudo apt-get install -y apache2 libapache2-mod-jk
```
- /etc/apache2/sites-available/000-default.conf 수정하여 `JkMount /* tomcat` 추가
```
sudo vim /etc/apache2/sites-available/000-default.conf 
```
```
DocumentRoot /var/www/html
JkMount /* tomcat #추가 문구
```

- /etc/apache2/workers.properties 파일 생성 후 `문구 작성`
```
	worker.list=tomcat
	workers.tomcat_home=/usr/share/tomcat9
	workers.java_home=/usr/lib/jvm/java-11-openjdk-amd64
	worker.tomcat.port=8009
	worker.tomcat.host=localhost
	worker.tomcat.type=ajp13
	worker.tomcat.lbfactor=1
```

- /etc/apache2/mods-available/jk.conf 수정
```
#JkWorkersFile /etc/libapache2-mod-jk/workers.properties 		#기존문구
JkWorkersFile /etc/apache2/workers.properties 				#수정문구
```

- 아파치 재시작
```
sudo systemctl restart apache2
```


### 📄 WAS 구축
- default-jdk tomcat9 libmysql-java 설치
```
sudo apt-get install -y default-jdk tomcat9 libmysql-java
```

- /etc/tomcat9/server.xml 수정
```
sudo vim /etc/tomcat9/server.xml
```
```
<!-- Define an AJP 1.3 Connector on port 8009 -->
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443"/>		#해당 내용 주석 해제
```

- jar 파일 Copy
```
sudo cp /usr/share/java/mysql-connector-java-5.1.45.jar /usr/share/tomcat9/lib/
```

- Tomcat 재시작
```
sudo systemctl restart tomcat9
```


### 📄 vsFTP 설정
- vsFTP 설치
```
sudo apt-get install -y vsftpd
```

- /etc/vsftpd.conf 수정 `아래 3개 설정 주석 제거`
```
sudo vim /etc/vsftpd.conf
```
```
write_enable=YES
local_umask=022
chroot_local_user=NO ##YES인지 확인
```

- vsFTP 재시작
```
sudo systemctl restart vsftpd
```

### 📄 mariaDB 설치
- mariadb-server 설치
```
sudo apt-get -y install mariadb-server
```

- mariaDB 초기 설정
```
sudo mysql_secure_installation
```
```
Enter current password for root (enter for none):  # 현재 MariaDB의 root 패스워드가 없으므로 엔터
OK, successfully used password, moving on...
```
```
Set root password? [Y/n]  		        y 	# MariaDB root 패스워드 설정 질의
```
```
New password: 	# 설정할 root 패스워드 입력
Re-enter new password: 	# 설정한 root 패스워드 확인 재입력
```
```
Remove anonymous users? [Y/n] 		        y # 익명의 접근에 대한 질의이며, 보안을 위해 차단

Disallow root login remotely? [Y/n] 		n # 외부로의 연결 허용

Remove test database and access to it? [Y/n] 	y # 테스트용으로 생성된 데이터베이스 삭제 여부 질의

Reload privilege tables now? [Y/n] 		y # 현재 설정된 값에 대한 적용 여부 질의
```


### 📄 mariaDB 내부작업
- mySQL 접속
```
sudo mysql -u root -p
```

- Database 생성
```
create database KJOONHWAN;
```

- 생성한 `KJOONHWAN` 사용
```
use KJOONHWAN;
```

- `login` Table 생성
```
create table login (id varchar(20) primary key, pw varchar(20));
```

- 생성한 Table 확인
```
MariaDB [KJOONHWAN]> show tables;
```
```
+---------------------+
| Tables_in_KJOONHWAN |
+---------------------+
| login               |
+---------------------+
```

- `login` 테이블 구조 열람
```
MariaDB [KJOONHWAN]> desc login;
```
```
+-------+-------------+------+-----+---------+-------+
| Field | Type        | Null | Key | Default | Extra |
+-------+-------------+------+-----+---------+-------+
| id    | varchar(20) | NO   | PRI | NULL    |       |
| pw    | varchar(20) | YES  |     | NULL    |       |
+-------+-------------+------+-----+---------+-------+
```

- 데이터베이스 선택
```
use mysql
```

- user 테이블에서 `host, user, password` 검색
```
MariaDB [mysql]> select host, user, password from user;
```

- 생성된 권한 부여 `KJOONHWAN 사용자에게 모든 권한을 할당`
```
GRANT ALL PRIVILEGES ON KJOONHWAN.* TO root@'192.168.0.153' identified by '!kjoonhwan!123@';
```

- localhost와 비밀번호가 같은지 확인
```
MariaDB [mysql]> select host, user, password from user;
```
```
+---------------+------+-------------------------------------------+
| host          | user | password                                  |
+---------------+------+-------------------------------------------+
| localhost     | root | *cda82de8117bc8eda72a04864ecece43040fb5da |
| 192.168.0.171 | root | *cda82de8117bc8eda72a04864ecece43040fb5da |
+---------------+------+-------------------------------------------+
```

- 현재 사용 중인 mySQL의 캐시를 삭제하고 새로운 설정을 적용
```
flush privileges;
```

- 데이터베이스 나가기
```
exit
```

### 📄 50-server 설정
- /etc/mysql/mariadb.conf.d/50-server.cnf 수정
```
sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf
```
```
#bind-address = 127.0.0.1 #기존 설정
bind-address = 0.0.0.0    #수정 설정
```

- mySQL 재시작
```
service mysqld restart
```

### 📄 소스코드 이동
- 호스트 PC 에서 ROOT.war 파일이 있는 파일탐색기 위치에서 Shift + 우클릭을 한 후 PowerShell 창 열기 클릭
```
-> scp ROOT.war KJOONHWAN@192.168.0.153:~/ 
-> yes
```

- DB 서버에서 파일이 정상적으로 이동 되었는지 확인
```
sudo cp ~/ROOT.war /var/lib/tomcat9/webapps/ 
```

- ROOT 파일을 `ROOT_BAK`으로 파일명 변경
```
cd /var/lib/tomcat9/webapps
```
```
sudo mv ROOT/ ROOT_BAK/
```

- Tomcat 재시작 `또는 일정 시간이 지나면 tomcat이 ROOT.WAR 파일을 해제하여 ROOT 디렉터리 생성`
```
sudo systemctl restart tomcat9
```

- 설정 찾기
```
sudo -s
```
```
cd ROOT
```
```
grep -rn "192.168."
```

- 설정 변경 `해당 파일들 안에 있는 내용을 Development Server IP로 설정`
```
joinAction.jsp
success.jsp
loginAction.jsp
```
```
joinAction.jsp:10:    String jdbcurl = "jdbc:mysql://192.168.0.153:3306/KJOONHWAN";
success.jsp:8:    String jdbcurl = "jdbc:mysql://192.168.0.153:3306/KJOONHWAN";
loginAction.jsp:12:     String jdbcurl = "jdbc:mysql://192.168.0.153:3306/KJOONHWAN";
```

- Tomcat 재시작
```
sudo systemctl restart tomcat9
```

- 동작 확인
```
1. 호스트 PC 웹 브라우저에서 Development Server IP로 접속
2. 로그인 창 확인 및 회원가입
3. 로그인

※ admin으로 회원가입 할 경우 관리자 페이지가 나타남
※ 그 외 계정은 일반 페이지
```

- ROOT.war 파일 생성
```
cd /var/lib/tomcat9/webapps/ROOT/
```
```
jar xvf ROOT.war *
```

### 📄 WAS로 ROOT.war 파일 전송
- /var/lib/tomcat9/webapps/ 로 이동
```
sudo -s
```
```
cd /var/lib/tomcat9/webapps/
```

- FTP 파일 전송
```
ftp [192.168.0.151] #WAS 서버 IP
ftp> cd /var/lib/tomcat9/webapps/
ftp> mput ROOT.war
ftp> exit
```


## 💻 Log Server
### 📄 로그 서버 설정
- /etc/rsyslog.conf 수정
```
sudo vim /etc/rsyslog.conf
```
```
module(load="imtcp")             #주석 제거
input(type="imtcp" port="514")   #주석 제거

$template LogName, "/var/log/rsyslog/%fromhost-ip%/%PROGRAMNAME%.log"         #내용 추가
*.* ?LogName                                                                  #내용 추가
```

- rsyslog 재시작
```
sudo systemctl restart rsyslog
```

- 가상 인프라 서버 별 디렉터리가 생성이 되었는지 확인
```
sudo ls -al /var/log/rsyslog/
```

## 💻 DB Backup Server
### 📄 DB Backup 서버 설정
- xinetd rsync 설치
```
sudo apt-get install -y xinetd rsync
```

- /etc/xinetd.d/rsync 생성 후 내용 작성
```
sudo vim /etc/xinetd.d/rsync
```
```
	service rsync
	{
		disable = no
		socket_type = stream
		wait = no
		user = root
		server = /usr/bin/rsync
		server_args = --daemon
		log_on_failure += USERID
	}
```

- xinetd 재시작
```
sudo systemctl restart xinetd
```

- rsync로 백업 파일을 잘 가져오는지 확인
```
sudo -s
```
```
rsync -avz [192.168.0.152]::DBS/mysql_dump_$(date +%Y%m%d).sql /backup/
```

- Cron 등록을 위한 쉘 스크립트 파일 작성
```
sudo vim /backup/backup.sh
```
```
 #!/bin/bash
 rsync -avz [192.168.0.152]::DBS/mysql_dump_$(date +%Y%m%d).sql /backup/
 ```

- 사용자에게 실행 권한 할당
```
chmod u+x /backup/backup.sh
```

- Crontab 등록
```
crontab -e
```
```
* * * * * /backup/backup.sh > /backup/backup.log 2>&1
```

- 백업 파일이 매 분 수정되는지 확인
```
ls -al /backup/
```
