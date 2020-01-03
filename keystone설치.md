## openstack 프로젝트 생성 후 key 기반 Instance에 접속

```shell
# vmhgfs-fuse /mnt
# df -h
# cd /mnt/hgfs/
# ls
# cd share_linux/
# ls
# cp user1-key1.pem /root //키를 root에 복사
# chmod 600 /root/user1-key1.pem // 접근권한을 사용자에게만 허용해야 ssh 실행 가능
# ip netns
# ip netns exec qrouter-7de24fa3-477e-4b30-b78d-917a4cd52852 ssh -i user1-key1.pem cirros@10.0.0.216 // 아이피는 유동ip로
```



인스턴스의 스냅샷은 루트 디스크 백업 용도

신더 쪽의 백업은 볼륨 스냅샷



```shell
# netstat -an|grep 1211

# ss -nlp|grep 11211
```



```
--bootstrap-admin-url http://controller:5000/v3/ \  //관리자가
  --bootstrap-internal-url http://controller:5000/v3/ \ //내부의 서비스들 간 
  --bootstrap-public-url http://controller:5000/v3/ \   //사용자
  --bootstrap-region-id RegionOne
```



#  설치과정 history



ip -a
    2  ifconfig
    3  ip a
    4  yum repolist
    5  who -r
    6  yum update -y
    7  sudo reboot
    8  hostnamectl set-hostname controller2
    9  hostnamectl set-hostname controller
   10  vi /etc/chrony.conf
   11  chronyc sources
   12  vi /etc/chrony.conf
   13  chronyc sources
   14  vi /etc/chrony.conf
   15  history
   16  chrronyc sources
   17  chronyc sources
   18  rpm -qa|grep openstack 
   19  systemctl start chronyd
   20  systemctl restart chronyd
   21  systemctl enable chronyd
   22  chronyc sources
   23  yum upgrade
   24  yum install python-openstackclient 

//오픈스택 클라이언트 설치

   25  yum install -y openstack-selinux 

// `openstack-selinux` 패키지를 설치하여 OpenStack 서비스에 대한 보안 정책을 자동으로 관리



   26  yum install mariadb mariadb-server python2-PyMySQL -y 

// 대부분의 openstack 서비스들은 sql 데이터베이스를 사용하여 정보를 저장 따라서 패키지 설치



  27  cd /etc/my.cnf.d
   28  ls
   29  vi openstack.cnf  

// **/etc/my.cnf.d/openstack.cnf**들어가서 `[mysqld]` 섹션을 생성하고, 다른 노드들이 관리 네트워크를 통한 액세스를 활성화하기 위해 컨트롤러 노드의 관리 IP 주소를 `bind-address` 키로 설정

```shell
[mysqld]
bind-address = 10.0.0.11

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```





   30  systemctl start mariadb
   31  systemctl enable mariadb
   32  systemctl status mariadb
   33  mysql_secure_installation

// `mysql_secure_installation` 스크립트를 실행하여 데이터베이스 서비스 보안을 강화



   34  mysql -uroot -p

// 패쓰워드 설정



   35  openstack status
   36  opetnstack-status
   37  openstack-status
   38  openstack --status
   39  yum install rabbitmq-server

// OpenStack은 메시지 대기열 을 사용하여 서비스 간 작업 및 상태 정보를 조정합니다. 메시지 큐 서비스는 일반적으로 컨트롤러 노드에서 실행됩니다. 

   40* 
   41  systemctl status rabbitmq-sever
   42  systemctl status rabbitmq-sever.service
   43  systemctl status rabbitmq-server.service
   44  vi /etc/hosts

// 들어가서 10.0.0.11 controller 추가


   45  systemctl start rabbitmq-sever.service
   46  systemctl start rabbitmq-server.service
   47  systemctl status rabbitmq-server.service
   48  systemctl enable rabbitmq-server.service
   49  systemctl status rabbitmq-server.service
   50  rabbitmqctl add_user openstack RABBIT_PASS

// 오픈스택 사용자 추가

`RABBIT_PASS`적절한 비밀번호로 교체



   51  rabbitmqctl set_permissions openstack ".*" ".*" ".*"

//`openstack`사용자에 대한 구성, 쓰기 및 읽기 액세스를 허용



   52  yum install memcached python-memcached

//서비스의 ID 서비스 인증 메커니즘은 Memcached를 사용하여 토큰을 캐시합니다. memcached 서비스는 일반적으로 컨트롤러 노드에서 실행



   53  vi /etc/sysconfig/memcached 

//파일 편집하여 추가

컨트롤러 노드의 관리 IP 주소를 사용하도록 서비스를 구성하십시오. 이는 관리 네트워크를 통해 다른 노드가 액세스 할 수 있도록하기위한 것입니다.

```shell
OPTIONS="-l 127.0.0.1,::1,controller"
```



   54  systemctl start memcached.service
   55  systemctl enable memcached.service
   56  systemctl status memcached.service
   57  netstat -an|grep 1211
   58  ss -nlp|grep 111211
   59  ss -nlp|grep 11211

//  리눅스에서 네트워크 상태를 확인하기 위해 흔히 사용하는 명령어로는 netstat을 들 수 있는데, 대체 명령어로 ss도 사용 가능

#ss [옵션] [필터]

-a : 모든 포트 확인

-t : TCP 포트 확인

-l : LISTEN 상태 포트 확인

-p : 프로세스명을 표시

-n : 호스트 / 포트 / 사용자이름을 숫자로 표시



////////////////////////////////선행조건///////////////////////////////////////////////////////

오픈스택 Identity 서비스를 설정하기 전에, 데이터베이스와 관리 토큰을 생성하여야함

`keystone` 데이터베이스를 생성

```
CREATE DATABASE keystone;
```

`keystone` 데이터베이스에 적합한 액세스를 부여

```shell
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
  IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
  IDENTIFIED BY 'KEYSTONE_DBPASS';
```

 `KEYSTONE_DBPASS` 를 적절한 암호로 변경

데이터베이스 접속 클라이언트를 종료



초기설정동안 관리 토큰으로 사용할 무작위 값을 생성

```
openssl rand -hex 10
```

/////////////////////////////////////////////////////////////////////////////////////////////////////



///////////////////////////////keystone 설치 과정/////////////////////////////////////////////

  60  mysql -uroot -p   

 61   yum install openstack-keystone httpd mod_wsgi

// 패키지 설치



   62  vi /etc/keystone/keystone.conf

// 파일 편집

- `[DEFAULT]` 섹션에서 초기 관리 토큰에 대한 값을 정의합니다:

  ```shell
  [DEFAULT]
  ...
  admin_token = ADMIN_TOKEN
  ```

  `ADMIN_TOKEN` 을 이전 단계에서 생성했던 랜덤값으로 변경합니다.

- `[database]` 섹션에서, 데이터베이스 액세스를 구성합니다:

  ```shell
  [database]
  ...
  connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
  ```

  `KEYSTONE_DBPASS` 를 데이터베이스에 대해 선택한 암호로 변경합니다.

- `[token]` 섹션에는 Fernet 토큰 제공자를 구성합니다:

  ```shell
  [token]
  ...
  provider = fernet
  ```





   63  su -s /bin/sh -c "keystone-manage db_sync" keystone

// Identity 서비스 데이터베이스를 넣어줍니다:



   64  cd /var/lib/mysql
   65  ls
   66  ls keystone/
   67  keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

//Fernet 키를 초기화합니다:

   68  keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

////Fernet 키를 초기화합니다:



   69  keystone-manage bootstrap --bootstrap-password ADMIN_PASS   --bootstrap-admin-url http://controller:5000/v3/   --bootstrap-internal-url http://controller:5000/v3/   --bootstrap-public-url http://controller:5000/v3/   --bootstrap-region-id RegionOne

// Identity 서비스를 부트스트래핑

''ADMIN_PASS'' 를 관리 권한이 있는 사용자에 대해 적절한 암호로 변경



   70  keystone-manage bootstrap --bootstrap-password ADMIN_PASS ;

// 패쓰워드 그냥 ADMIN_PASS 그대로 씀



   71  ci /etc/httpd/conf/httpd.con

   72  vi /etc/httpd/conf/httpd.conf

// 파일을 편집하여 `ServerName` 옵션이 컨트롤러 노드를 가리키도록 구성

```shell
ServerName controller
```

   

73  ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

   74  systemctl enable httpd.service
   75  systemctl start httpd.service
   76  ss -nlp|grep http

   77  export OS_USERNAME=admin
   78  export OS_PASSWORD=ADMIN_PASS
   79  export OS_PROJECT_NAME=admin
   80  export OS_USER_DOMAIN_NAME=Default
   81  export OS_PROJECT_DOMAIN_NAME=Default
   82  export OS_AUTH_URL=http://controller:5000/v3
   83  export OS_IDENTITY_API_VERSION=3

//  vi 편집기에서 쳐야되는 내용 아닌가? 일단 보류



   84  openstack domain create --description "An Example Domain" example

//인증 서비스는 도메인, 프로젝트, 사용자 및 역할의 조합을 사용



   85  openstack project create --domain default   --description "Service Project" service

`// service` 프로젝트를 작성



   86  ;
   87  openstack project create --domain default   --description "Demo Project" myproject

// 일반 (관리자가 아닌) 작업은 권한이없는 프로젝트와 사용자를 사용해야합니다. 예를 들어,이 안내서는 `myproject`프로젝트와 `myuser` 사용자를 작성합니다 .

- `myproject`프로젝트를 작성하십시오 .



   88  openstack user create --domain default   --password-prompt myuser

// `myuser`사용자를 작성



   89  openstack role create myrole

// `myrole`역할을 작성



   90  openstack role add --project myproject --user myuser myrole

// 프로젝트 및 사용자 `myrole`에게 역할을 추가하십시오 .`myproject``myuser`



   91  unset OS_AUTH_URL OS_PASSWORD

// 임시 `OS_URL` 및 `OS_PASSWORD` 환경변수를 unset



   92  openstack --os-auth-url http://controller:5000/v3   --os-project-domain-name Default --os-user-domain-name Default   --os-project-name admin --os-username admin token issue

// `admin` 사용자로, 인증 토큰을 요청



   93  openstack --os-auth-url http://controller:5000/v3   --os-project-domain-name Default --os-user-domain-name Default   --os-project-name myproject --os-username myuser token issue

// `demo` 사용자로 인증 토큰을 요청



   94  vi admin-openrc

```shell
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```



   95  vi demo-openrc

```shell
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=abc123
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2

```



   96  . admin-openrc

// 클라이언트를 특정 프로젝트 및 사용자로 실행하기 위해서는 실행하기 전에 관련된 클라이언트 환경 스크립트를 단순히 로딩하여 가능합니다. 예를 들면:

1. `admin-openrc.sh` 파일을 로드하여 Identity 서비스에 대한 위치와 `admin` 프로젝트 및 사용자 credential과 함께 환경 변수를 넣어줍니다:



   97  openstack token issue

// 2. 인증 토큰을 요청합니다:



   98  mv *openrc /root

// 위에 꺼 루트에서 해야 편해서 루트로 그냥 옮겨줌

   99  cd
  100  . admin-openrc
  101  openstack token issue
















ss -nlp|grep 11211
netstat -an|grep 11211
hostnamectl set-hostname controller
 742 connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
 2829 provider = fernet

ss -nlp|grep http
tcp    LISTEN     0      128    [::]:5000               [::]:* 

export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3