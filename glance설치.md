# Image Service -Glance 설치      //Red Hat



- `root`사용자 로서 데이터베이스 서버에 연결

```SHELL
mysql -u root -p
```

- `glance`데이터베이스를 작성

```SHELL
CREATE DATABASE glance;
```

- `glance`데이터베이스에 대한 적절한 액세스 권한을 부여

```shell
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'GLANCE_DBPASS';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'GLANCE_DBPASS';
  이후 exit로 마리아디비 탈출~
```

- `admin`신임 정보를 소싱하여 관리자 전용 CLI 명령에 액세스

```SHELL
. admin-openrc
```

- `glance`사용자를 작성

```shell
openstack user create --domain default --password-prompt glance
User Password:
Repeat User Password:

↑말고 다른 방법
openstack user create --domain default --password GLANCE_PASS glance

+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3f4e777c4062483ab8d9edd7dff829df |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

- 사용자 및 프로젝트에 `admin`역할을 추가

```SHELL
openstack role add --project service --user glance admin
```

- `glance`서비스 엔티티를 작성

```shell
openstack service create --name glance \
  --description "OpenStack Image" image
  
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

- 이미지 서비스 API 엔드 포인트를 작성

```SHELL
$ openstack endpoint create --region RegionOne \
  image public http://controller:9292
// 사용자
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 340be3625e9b4239a6415d034e98aace |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  image internal http://controller:9292
// 내부의 서비스
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a6e4b153c2ae4c919eccfdbb7dceb5d2 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

$ openstack endpoint create --region RegionOne \
  image admin http://controller:9292
// 관리자
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0c37ed58103f4300a84ff125a539032d |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```



- 패키지를 설치

```
yum install openstack-glance
```

- `/etc/glance/glance-api.conf`파일을 편집

  [database]`섹션, 구성 데이터베이스 액세스 :

  ```SHELL
  [database]
  # ...
  connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance  //추가
  ```

  

   `[keystone_authtoken]`와 `[paste_deploy]`섹션 구성 아이덴티티 서비스 액세스 :

  ```SHELL
  [keystone_authtoken]
  # ...
  www_authenticate_uri  = http://controller:5000
  auth_url = http://controller:5000
  memcached_servers = controller:11211
  auth_type = password
  project_domain_name = Default
  user_domain_name = Default
  project_name = service
  username = glance
  password = GLANCE_PASS
  //////////////////////////////추가
  [paste_deploy]
  # ...
  flavor = keystone     //추가
  ```

  

   `[glance_store]`섹션, 이미지 파일의 로컬 파일 시스템 저장소와 위치를 구성 

  ```SHELL
  [glance_store]
  # ...
  stores = file,http
  default_store = file
  filesystem_store_datadir = /var/lib/glance/images/    //////추가
  ```

- `/etc/glance/glance-registry.conf`파일을 편집하고 다음 조치를 완료하십시오.

  

   `[database]`섹션, 구성 데이터베이스 액세스 :

  ```SHELL
  [database]
  # ...
  connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
  ```

  

   `[keystone_authtoken]`와 `[paste_deploy]`섹션 구성 아이덴티티 서비스 액세스 :

  ```shell
  [keystone_authtoken]
  # ...
  www_authenticate_uri = http://controller:5000
  auth_url = http://controller:5000
  memcached_servers = controller:11211
  auth_type = password
  project_domain_name = Default
  user_domain_name = Default
  project_name = service
  username = glance
  password = GLANCE_PASS
  
  [paste_deploy]
  # ...
  flavor = keystone                             /////////// 추가
  ```

  

- 이미지 서비스 데이터베이스를 채우십시오.

  ```
  # su -s /bin/sh -c "glance-manage db_sync" glance
  ```

  

## 설치 완료 

- 이미지 서비스를 시작하고 시스템 부팅시 시작되도록 구성

  ```
  # systemctl enable openstack-glance-api.service \
    openstack-glance-registry.service
  # systemctl start openstack-glance-api.service \
    openstack-glance-registry.service
  ```



# 작동 확인



- `admin`신임 정보를 소싱하여 관리자 전용 CLI 명령에 액세스

```shell
. admin-openrc
```



- wget 설치

```
yum install -y wget
```

- cirros 소스 이미지 다운

```
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
```

- [*QCOW2*](https://docs.openstack.org/mitaka/ko_KR/install-guide-rdo/common/glossary.html#term-qemu-copy-on-write-2-qcow2) 디스크 포맷, [*bare*](https://docs.openstack.org/mitaka/ko_KR/install-guide-rdo/common/glossary.html#term-bare) 컨테이너 포맷과 모든 프로젝트에서 액세스 가능하도록 하는 공용으로 보이기를 사용하여 이미지를 이미지 서비스에 업로드

```
openstack image create "cirros-vmdk" --file /root/cirros-0.3.5-x86_64-disk.vmdk --disk-format vmdk --container-format bare --public
```

- 이미지 업로드를 확인하고 속성들을 검증

```
glance image-list
glance image-show 0d23dfcf-3be9-40c1-aca3-2fe15dd0db23
```

















