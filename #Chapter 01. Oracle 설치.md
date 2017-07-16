#Chapter 01. Oracle 12c 설치

본 문서는 Oracle 설치 파일(12.1.0.2.0)을 Target OS(Oracle Linux)로 옮겨졌음을 가정하고 이후 나머지 작업에 대하여 절차를 가이드한다.

참조 사이트 : [Oracle Base](https://oracle-base.com/articles/12c/oracle-db-12cr1-installation-on-oracle-linux-7)

**목차**

[TOC]

##0. 압축 해제
```
unzip linuxamd64_12102_database_1of2.zip
unzip linuxamd64_12102_database_2of2.zip
```

압축 해제한 두개의 파일을 database라는 폴더 하나에 합친다.

##1. Hosts File 설정


* /etc/hosts 파일은 리눅스에서 DNS보다 먼저 호스트명을 IP로 풀어주는 파일이다(이 곳에 등록되어 있는 주소는 DNS로 보내지 않는다).

~~~~
<IP-address>  <fully-qualified-machine-name>  <machine-name>
~~~~

**For example.**

~~~~
127.0.0.1       localhost localhost.localdomain localhost4 localhost4.localdomain4
~~~~

> **1. hosts 파일 내에 실제 IP hostname alias를 적어준다.**

~~~~
10.0.2.15 o17.localdomain o17
~~~~


##2. Hostname 설정

* /etc/hostname 파일은 hostname을 설정하는 파일이다.

Tip)
hostname이란? 인터넷에 접속된 수많은 컴퓨터들이 자신을 구별하기 위해 사용하는 별칭
 - shell에 hostname이라고 치면 hostname 정보가 나온다.

> **1. hostname을 기입한다.**

~~~~
o17.localdomain
~~~~


## 3. Manual Setup

오라클의 공유 메모리는 커널 작업을 수반하지 않으며, 프로세스간의 데이터 복제 작업이 불필요 하기 때문에 IPC를 위한 가장 빠른 방법을 선호한다.

인스턴스 구조를 보면 SGA와 백그라운드 프로세스로 구성되어 있다. 여기에서 SGA는 Shared Pool, DB Buffer Cache, Redo Log Buffer 등의 저장에 활용되므로 SGA 크기에 따라서 오라클 성능이 달라진다.

아래 내용은 오라클을 설치하기 이전에 Linux의 커널 매개변수와 shell limit를 설정하는 방법을 알아보고자 한다.


> **1. Sysctl 설정**

* /etc/sysctl.conf 파일은 Kernel parameter값을 설정하는 파일이다. 아래와 같이 설정한다.

Tip) 다른 시스템과 달리 Linux에서는 시스템이 기동중인 상태에서 매개변수 수정이 가능하다. 또한 리부팅을 할 필요도 없다. 

참조 사이트)

* [Linux 공유 메모리 이해](http://linux.systemv.pe.kr/%EB%A6%AC%EB%88%85%EC%8A%A4-%EA%B3%B5%EC%9C%A0-%EB%A9%94%EB%AA%A8%EB%A6%AC/) 
* [Oracle 커널 파라미터 및 Limit 설정](http://estenpark.tistory.com/270)


```
fs.file-max = 6815744 *시스템 전체에서 최대로 열 수 있는 파일 개수
```

```
#세마포어 설정

kernel.sem = 250 32000 100 128

#kernel.sem 하나의 set당 최대 세마포어 개수(SEMMSL), 
            리눅스 전체 세마포어 개수(SEMMNS),
            세마 포어 호출별로 수행될 수 있는 세마포어 최대 작업 수(SEMOPM)
            세마포어 set의 개수 설정(SEMMNI),

 tip) 전체 세마포어 개수 >= set당 최대 세마포어(SEMMSL) * 세마포어 set 개수(SEMMNI)
      
      세마포어 : 공유 리소스의 사용과정에서 프로세스간의 동기화를 위해 사용되는 카운터로서, 
      App에서 세마포어를 이용하면 OS는 SET을 통해 세마포어 지원한다.

```

```
#공유 메모리 설정

kernel.shmmni = 4096          * 공유 메모리 최소 세그먼트 크기
kernel.shmall = 1073741824    * 시스템을 포함한 모든 프로세스에서 사용 가능한 공유 메모리의 최대 크기(페이지 단위)
kernel.shmmax = 4398046511104 * 공유 메모리 최대 세그먼트 크기

 tip) ipcs -lm 을 확인하면 공유 메모리 크기를 확인할 수 있다.
 # ipcs -lm
  
 ------ Shared Memory Limits --------
 max number of segments = 4096                   <--- SHMMNI
 max seg size (kbytes) = 4294967296              <--- SHMMAX
 max total shared memory (kbytes) = 4294967296   <--- SHMALL
 min seg size (bytes) = 1

여기서 shmall은 단위가 kbytes이고 페이지 단위로 표기
따라서 kernel shmall 설정값(bytes) * Page 크기 / 1024 값은 ipcs-lm의 SHMALL 값과 같다. 

 tip) system 커널 shmall 크기(bytes) 확인 : cat /proc/sys/kernel/shmall
      페이지 사이즈 확인 : getconf PAGESIZE
```

```
kernel.panic_on_oops = 1

#oops란 커널이 오류 로그를 생성하고 비정상이 된 상태
#패닉이란 내부 오류를 감지하여 안전하게 복구가 불가능할때

위 명령어는 oops를 커널 패닉으로 간주하는 명령어
```

```
net.core.rmem_default = 262144
net.core.rmem_max = 4194304

#r은 receive를 의미하며 default 소켓에 기본적으로 설정되는 버퍼 크기
max 최대 크기

net.core.wmem_default = 262144
net.core.wmem_max = 1048576

#w는 write 의미한다.
```

```
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2

#IPv4는 인증 매커니즘을 가지고 있지 않으므로, Source IP 주소를 조작할 수 있다
#이 설정은 자신의 네트워크가  스푸핑된 공격지의 소스로 쓰이는 것을 차단.
```

```
fs.aio-max-nr = 1048576 * 현재 시스템 단위의 비동기 IO에 대해 허가된 최대 동시 요청 수

 tip) AIO는 한 프로세스가 몇 개의 IO작업 하는 것을 block하거나 wait없이
 허용하는 것으로 IO 완료 알림을 받은 후에 프로세스는 IO결과를 받을 수 있다.
```

```
net.ipv4.ip_local_port_range = 9000 65500 *사용 가능한 port의 범위 설정
```

설정을 맞췄으면 저장한 후 나와서 변경 내용을 적용한다.

```
/sbin/sysctl -p
```




> **2. Shell Limit 설정**

오라클은 Linux 계정 별로 실행되는 프로세스와 열린 파일의 수를 제한하는 것을 권장한다. 이를 위해 root계정에서 /etc/security/limits.conf 설정 파일을 수정한다.

Tip) root로 수정해야하며, 리부팅은 필요없다. 

참조 사이트)

* [Security/limits.conf 파일 이해](http://egloos.zum.com/powerenter/v/10980761)

```
oracle   soft   nofile    1024
oracle   hard   nofile    65536
oracle   soft   nproc    16384
oracle   hard   nproc    16384
oracle   soft   stack    10240
oracle   hard   stack    32768
oracle   hard   memlock    134217728
oracle   soft   memlock    134217728
```

```
<domain> <type> <item> <value>

domain : 제한을 할 대상으로써 사용자 이름이나 그룹 이름, 그리고 와일드 문자를 사용할 수 있다. 그룹에 적용할 경우에는 @가 붙는다.

type : 제한의 정도를 설정한다. 
Soft Limit : 새 프로그램이 생성되면 디폴트로 적용되는 제한 값
Hard Limit : Soft Limit으로부터 늘릴 수 있는 최대값으로 root에서만 조정 가능

item :  제한을 할 항목.
    - core : core 파일 사이즈(KB)
    - data : 최대 데이터 크기(KB)
    - fsize : 최대 파일 사이즈(KB)
    - memlock : 최대 locked-in-memory 주소 공간(KB)
    - nofile : 한번에 열 수 있는 최대 파일 수 
    - rss : 최대 resident set size(KB)
    - stack : 최대 스택 사이즈(KB)
    - cpu : 최대 CPU 점유 시간(MIN)
    - nproc : 최대 프로세스 개수
    - as : 주소 공간 제한
    - maxlogins : 동시 로그인 최대 수
    - priority : 사용자 프로세스가 실행되는 우선 순위
```





> **3. 관련 패키지 설치**

```
yum install binutils -y
yum install compat-libstdc++-33 -y
yum install compat-libstdc++-33.i686 -y
yum install gcc -y
yum install gcc-c++ -y
yum install glibc -y
yum install glibc.i686 -y
yum install glibc-devel -y
yum install glibc-devel.i686 -y
yum install ksh -y
yum install libgcc -y
yum install libgcc.i686 -y
yum install libstdc++ -y
yum install libstdc++.i686 -y
yum install libstdc++-devel -y
yum install libstdc++-devel.i686 -y
yum install libaio -y
yum install libaio.i686 -y
yum install libaio-devel -y
yum install libaio-devel.i686 -y
yum install libXext -y
yum install libXext.i686 -y
yum install libXtst -y
yum install libXtst.i686 -y
yum install libX11 -y
yum install libX11.i686 -y
yum install libXau -y
yum install libXau.i686 -y
yum install libxcb -y
yum install libxcb.i686 -y
yum install libXi -y
yum install libXi.i686 -y
yum install make -y
yum install sysstat -y
yum install unixODBC -y
yum install unixODBC-devel -y
yum install zlib-devel -y
yum install zlib-devel.i686 -y
```





> **4. Group 및 User 생성**

```
groupadd -g 54321 oinstall
groupadd -g 54322 dba
groupadd -g 54323 oper
groupadd -g 54324 backupdba
groupadd -g 54325 dgdba
groupadd -g 54326 kmdba
groupadd -g 54327 asmdba
groupadd -g 54328 asmoper
groupadd -g 54329 asmadmin

useradd -u 54321 -g oinstall -G dba,oper oracle
```

Tip)
```
사용자 계정 확인방법

cat /etc/passwd 

root : x : 0 : 0 : root : /root : /bin/bash
 1     2   3   4   5      6       7

이렇게 7개의 필드로 구분된다
각각의 필드에 대한 설명은 아래와 같다.

1. 사용자 계정 ID
2. 패스워드
3. 사용자 UID
4. 그룹 GID
5. 계정정보(보통은 사용자 이름)
6. 홈 디렉토리
7. 쉘 환경

사용자 계정 생성 방법 

useradd <option> 계정명

기본 옵션
 -u 사용자 계정의 UID
 -g 사용자 계정 로그인 그룹(그룹은 이미 만들어져 있어야 함)
 -G 기본 그룹 이외에 추가 지정 그룹


그룹 확인

cat /etc/group

root : x : 0
 1     2   3

1. 그룹명
2. 그룹 비밀번호
3. GID

그룹 생성 방법

groupadd <option> 그룹명

-g : 그룹 ID 지정
```



##4. 추가 설정

> **1. 계정 비밀번호 설정**

```
passwd oracle
```


> **2. 방화벽 설정 변경**

방화벽을 iptables를 활용한다면 아래와 같이 설정 변경한다.

/etc/selinux/config 파일에서 selinux flag를 해제한다
```
SELINUX=permissive
```

Linux 방화벽 중지
```
# systemctl stop firewalld
# systemctl disable firewalld
```

>**3. 오라클 설치를 위한 디렉토리 생성**

```
mkdir -p /u01/app/oracle/product/12.1.0.2/db_1
chown -R oracle:oinstall /u01
chmod -R 775 /u01
```


Tip)
```
mkdir은 폴더 생성 명령어로 mkdir <option> 폴더명으로 생성할 수 있다.
-p 옵션은 생성되는 폴더 디렉토리 상에 부모 폴더가 없으면 자동 생성된다.

chown은 소유권을 변경하는 명령어로 chown <option> <소유자>:<그룹> <파일명> 으로 변경할 수 있다.
-R 옵션은 하위 디렉토리/파일에 모두 적용한다.

chmod는 파일 허가권이다.

d          rwx------    2         root          root             4096 
파일유형 | 파일허가권 | 링크 수 | 파일 소유자 | 파일 소유 그룹 | 파일 크기 

Mar 6 19:58       Kimeunsoo
마지막 수정날짜 | 파일이름

파일 허가권은 0~7사이 값이 들어가며 소유자(u), 그룹(g), 그 외 사용자(o)로 별도 부여

없음(-) : 0
실행(x) : 1
읽기(r) : 2
쓰기{w} : 4

7 : r,w,x, 모두 가짐

```

>**4. 환경변수 설정**

새로 생성한 oracle 계정에 bash_profile 설정을 한다.
파일 위치 /home/oracle/.bash_profile

```
# Oracle Settings
export TMP=/tmp
export TMPDIR=$TMP

export ORACLE_HOSTNAME=ol7.localdomain
export ORACLE_UNQNAME=cdb1
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/12.1.0.2/db_1
export ORACLE_SID=cdb1

export PATH=/usr/sbin:$PATH
export PATH=$ORACLE_HOME/bin:$PATH

export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

##5. 설치파일 실행

oracle 유저로 로그인 한다.
설치파일 압축 해제 폴더 안에서 runInstaller를 실행한다.
```
./runInstaller
```


##6. OUI 설치 진행

아래 그림대로 설정한다.

>**1. Configure Security Updates**

![1](https://oracle-base.com/articles/12c/images/12cR1Database/01-configure-security-updates.jpg)


>**2. My Oracle Support Credentials**

![2](https://oracle-base.com/articles/12c/images/12cR1Database/02-mos-credentials.jpg)

>**3. Select Installation Type**

![3](https://oracle-base.com/articles/12c/images/12cR1Database/04-select-installation-type.jpg)

>**4.System Class**

![4](https://oracle-base.com/articles/12c/images/12cR1Database/05-system-class.jpg)

>**5.Grid Installation Options**

![5](https://oracle-base.com/articles/12c/images/12cR1Database/06-grid-installation-options.jpg)

>**6.Select Install Type**

![6](https://oracle-base.com/articles/12c/images/12cR1Database/07-select-install-type.jpg)

>**7.Typicl Install Configuration**

![7](https://oracle-base.com/articles/12c/images/12cR1Database/08-typical-install-configuration.jpg)

>**8.Create Inventory**

![8](https://oracle-base.com/articles/12c/images/12cR1Database/09-create-inventory.jpg)

>**9.Perform Prerequisite Checks**

![9](https://oracle-base.com/articles/12c/images/12cR1Database/10-perform-prerequisite-checks.jpg)

>**10. Summary**

![10](https://oracle-base.com/articles/12c/images/12cR1Database/11-summary.jpg)

>**11. Install Product**

![11](https://oracle-base.com/articles/12c/images/12cR1Database/12-install-product.jpg)

>**12.Execute Configraution Scripts**

![12](https://oracle-base.com/articles/12c/images/12cR1Database/13-execute-configuration-scripts.jpg)

>**13. Oracle Database Configuration**

![13](https://oracle-base.com/articles/12c/images/12cR1Database/14-oracle-database-configuration.jpg)

>**14. Database Configuration Assistant

![14](https://oracle-base.com/articles/12c/images/12cR1Database/15-database-configuration-assistant.jpg)

>**15. Database Configuration Assistant Complete**

![15](https://oracle-base.com/articles/12c/images/12cR1Database/16-database-configuration-assistant-complete.jpg)

>**16. Finish**

![16](https://oracle-base.com/articles/12c/images/12cR1Database/17-finish.jpg)

>**17. Database Express 12c Login**

![17](https://oracle-base.com/articles/12c/images/12cR1Database/18-database-express-12c.jpg)

>**18. Database Express 12c Dashboard**

![18](https://oracle-base.com/articles/12c/images/12cR1Database/19-database-express-12c.jpg)




