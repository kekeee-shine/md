# 基于rook-ceph的存储架构设计

[TOC]



























### 1、存储架构图

![image-20200409105928640](基于rook-ceph的存储架构设计/image-20200409105928640.png)

* 采用kubernetes集群集成方式，底层使用多台物理机划分高性能存储节点和普通存储节点
* 应用层根据使用场景选择不同的存储方式

### 2、why use rook-ceph

#### 2.1、安全性

* 支持数据加密；
* 读写校验，防止非法程序的干扰；
* 用户数据隔离，避免数据过多时信息混乱；
* 数据副本机制最大程度上避免机器故障导致的数据丢失问题。

#### 2.2、高可用性

* 同时支持非结构数据，块数据，结构数据的存储；
* 支持pod中进行挂载使用，也支持外部服务api调用；
* 数据自动修复功能；
* 数据冗余副本机制，任意一台服务器出故障也不会影响到业务系统正常使用；
* kubernetes断电重启集群自动恢复。

#### 2.3、可维护性

* 自动备份，自动恢复减少运维干预，运维只需要关注硬件运行状态；
* 相比于原生的ceph来说，rook-ceph利用kubernetes的特性，安装，扩容更加方便；
* 提供可视化ui界面大大减少了问题定位排查的工总量。

#### 2.4、可视化支持

* kubernetes dashboard
* ceph dashboard
* ceph prometheus 可扩展grafana

### 3、硬件环境设计

#### 3.1、cpu

​		一个 Ceph OSD 守护进程需要相当数量的处理性能，因为它提供数据给客户端。要评估 Ceph OSD 的 CPU 需求，知道服务器上运行了多少 OSD 上非常重要的。通常建议每个 OSD 进程至少有一个核心。可以通过以下公式计算 OSD 的 CPU 需求：

**（（CPU数\* CPU核心数）/ （（osd数量*每个osd的容量）+mon节点） >= 1**

一台服务器上挂载多个osd经济效益比较高同时集群维护成本较低，例如在一个24核机器上建议运行20个osd，超过20个会派生出大量线程

#### 3.2、内存

* 对于1TB的osd节点建议最少分配1G内存&1cpu资源，对于平时正常使用状态来说500M足够，密度写入时，特别是osd进行恢复数据时需要占用大量资源，如果资源过少会影响调度在这些osd上的其他pg的读写。
* 每个mon节点建议最少分配1G内存&1cpu资源，osd增加后mon节点资源不需要增加。
* mgr节点主要对mon节点进行一些查询等 占用资源较少建议分配500Mi内存&0.5cpu。

#### 3.3、存储

​		建议选用高性能固态硬盘，osd 应该有足够的空间用于存储对象数据。考虑到大硬盘的每 GB 成本，我们建议用容量大于 1TB 的硬盘。建议用 GB 数除以硬盘价格来计算每 GB 成本，因为较大的硬盘通常会对每 GB 成本有较大影响。在重均衡、回填、恢复期间， 每1TB 的存储空间大约需要 1GB 内存。

​		同时，为了合理利用存储的经济效益，需要一台普通hdd存储node，对读写性能要求不高的数据或长时间不用的数据进行数据归档，提高高性能存储节点的利用率。1TB 的存储空间大约需要 建议分配0.5GB 内存。

#### 3.4、网络

​		在一个node上运行多个 osd，要确保 OSD 硬盘总吞吐量不超过为客户端提供读写服务所需的网络带宽，现在暂定的硬件的顺序读写性能 最高达3500MB/s ，一个机器上有20台osd，满载网络负荷为70GB/s，目前配备的百G网卡可以满。

#### 3.5、其他注意事项

* 对于k8s来说因为存储问题不能使用hostnetwork作为节点中通信 只能用内部cni网络，而且ceph集群涉及到块存储不同于普通任务pod服务器ip禁止更改 会导致数据丢失

* 建议k8s集群中相关节点只运行ceph相关pod，避免在密集读写时其他命名空间下的任务影响到存储性能，可通过node label及pod nodeselector简单实现
* ceph osd节点的密度也是影响集群性能，通常，大量的小容量节点 比 少量的大容量节点要好（太多小容量节点也不好），应该选择适当的 Ceph 节点密度，使得节点容量小于总容量的 10%，例如：在一个 1PB的ceph集群中，应该避免使用 4 个 250TB 的 OSD 节点，因为每个节点占用了 25% 的集群容量。相反，可以使用 13 个 80TB 的 OSD节点，每个节点容量小于集群容量的 10%，建议控制在（5-10%之间）

mgr，mon，osd资源设置在cluster.yaml中spec.resources中，例如：

```yaml
...   
spec:
	resources:
        mgr:
            limits:
                cpu: "500m"
                memory: "1024Mi"
            requests:
                cpu: "500m"
                memory: "1024Mi"
        mon:
            limits:
                cpu: "1"
                memory: "1024Mi"
            requests:
                cpu: "500m"
                memory: "1024Mi"
        osd:
            limits:
                cpu: "1"
                memory: "2048Mi"
            requests:
                cpu: "500m"
                memory: "1024Mi"
```

### 4、存储方式选择

ceph底层存储有三种方式：

  * Filestore linux文件系统 ，只支持机械硬盘 ；
  * Bluestore 直接管理裸设备 支持元数据存储 针对固态硬盘进行优化 ；
  * Seastore  下一代的objectstore 专门针对NVME设备设计，目前只有一个设计雏形。

从性能和设备上考虑，推荐高性能osd使用blueStore，普通osd使用filestore 。集群中设置方法：修改cluster.yaml  文件spec.storage.nodes属性config的storeType

```yaml
 ...
 spec:
 	storage:
 		nodes:
        - name: "node1"
          devices: # specific devices to use for storage can be specified for each node
          - name: "sdb"
          - name: "nvme01" # multiple osds can be created on high performance devices
        config: # configuration can be specified at the node level which overrides the cluster level config
	        storeType: filestore # 默认 blueStore
```

### 5、缩容扩容

#### 5.1、移除osd

​	当osd发生故障或者磁盘需要更换时需按照以下步骤进行操作：

* 使用 `ceph osd  crush reweight osd.{ID}` 0 将osd的权重置为0
* 待PG的状态全部为active+clean后再执行后续操作
* 使用 `ceph osd crush remove osd.{ID}` 命令将osd从crush map中移除
* 使用 `ceph osd.{ID} down` 更改osd 状态
* （确保pg为0情况下）使用`ceph osd.{ID} out` 将osd置为out
* 使用 `ceph osd rm {ID}` 命令删除osd

#### 5.2、原有节点中新增osd

​	准备好可用的硬盘（或分区）

* 修改cluster.yaml  文件spec.storage.nodes中对应节点下增加device相关设置
* k8s master节点运行 kubectl apply -f cluster.yaml 

#### 5.3、增加新的node

​	准备好可用的硬盘（或分区）

* 先将节点加入k8s集群中
* 修改cluster.yaml  文件spec.storage.nodes属性 增加子节点
* k8s master节点运行 kubectl apply -f cluster.yaml 

```yaml
 ...
 spec:
 	storage:
 		nodes:
        - name: "node1"
          devices: # specific devices to use for storage can be specified for each node
          - name: "sdb"
          - name: "nvme01" # multiple osds can be created on high performance devices
```

### 6、完整的服务流程图

![image-20200413171924101](基于rook-ceph的存储架构设计/image-20200413171924101.png)外部服务使用步骤：

 1、通过rado进行身份验证请求，返回用户认证信息。

 2、将认证信息加入管理业务请求中获取ceph中用户以及其bucket相关信息，可对用户资源进行设置。

 3、最后根据buket的key，secret，对bucket进行对象读写删查操作。

### 7、examples

#### 7.1、rado进行身份验证请求

前提条件：

* 配置好ceph dashboard 使用以下命令获取dashboard登陆密码（用户名默认admin）:

  ```
  kubectl get secrets -n rook-ceph rook-ceph-dashboard-password -o jsonpath='{.data.password}' | base64 -d
  ```

* 配置Dashboard Object-Gateway：

  > 进入ceph管理pod中
  >
  > `radosgw-admin user create --uid=admin --display-name=admin_user --system`
  >
  > 需要创建一个system 标记的用户 
  >
  > `radosgw-admin user --uid=admin 查询出access-key，secret-key`
  >
  > ceph dashboard set-rgw-api-access-key `{access-key}`
  > ceph dashboard set-rgw-api-secret-key `{secret-key}`
  > ceph dashboard set-rgw-api-host `{hostip}` # ceph管理pod的外网ip
  > ceph dashboard set-rgw-api-port `{port}` # 对外暴露服务的端口
  > ceph dashboard set-rgw-api-user-id admin
  > ceph dashboard set-rgw-api-scheme http

登陆获取鉴权api：/auth  [post]

* request：

  ```sh
  curl -X POST "https://172.16.15.201:32367/api/auth" -H "accept: */*" -H "Content-Type: application/json" -d "{\"username\":\"admin\",\"password\":\"fZvu0vADb6\"}"
  ```

* response:

  ```json
  {
    "username": "admin",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJjZXBoLWRhc2hib2FyZCIsImlhdCI6MTU4NjQxMDQwOSwidXNlcm5hbWUiOiJhZG1pbiIsImp0aSI6IjQ3OTg5MjVhLWUxZWItNDhjMi05YmI1LTlkNDMxZGQwY2QzMiIsImV4cCI6MTU4NjQzOTIwOX0.2i2oeJ61TOMaoHx6N2VtXa-ftI_xMFiUfs_jrqn879Q",
    "permissions": {
      "dashboard-settings": [
        "create",
        "delete",
        "read",
        "update"
      ]...
    }
  }
  ```

#### 7.2、操作管理radosgw

获取用户列表api：/rgw/user [GET]

* request：

  ```sh
  curl -X GET "https://172.16.15.201:32367/api/rgw/user/ceph-user-kz1WAnvv" -H "accept: */*" -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJjZXBoLWRhc2hib2FyZCIsImlhdCI6MTU4NjM5NTY3MywidXNlcm5hbWUiOiJhZG1pbiIsImp0aSI6ImEzMGVmYTIxLWRmYjktNDU0YS05Nzc3LTkwNDYxNmFiMzFkMSIsImV4cCI6MTU4NjQyNDQ3M30.W6CbQ_bim9jc8flXY7eZM5wA-_pc89xgu1DmSqyA8pw"
  ```

* response:

  ```json
  [
    "ceph-user-kz1WAnvv",
    "admin"
  ]
  ```

获取指定用户详细信息api：/rgw/user [GET]

* request：

  ```sh
  curl -X GET "https://172.16.15.201:32367/api/rgw/user/admin" -H "accept: */*" -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJjZXBoLWRhc2hib2FyZCIsImlhdCI6MTU4NjM5NTY3MywidXNlcm5hbWUiOiJhZG1pbiIsImp0aSI6ImEzMGVmYTIxLWRmYjktNDU0YS05Nzc3LTkwNDYxNmFiMzFkMSIsImV4cCI6MTU4NjQyNDQ3M30.W6CbQ_bim9jc8flXY7eZM5wA-_pc89xgu1DmSqyA8pw"
  ```

* response:

  ```json
  {
    "uid": "ceph-user-kz1WAnvv",
    "temp_url_keys": [],
    "default_storage_class": "",
    "suspended": 0,
    "op_mask": "read, write, delete",
    "display_name": "ceph-user-kz1WAnvv",
    "system": "false",
    "tenant": "",
    "type": "rgw",
    "email": "",
    "user_quota": {
      "max_objects": -1,
      "enabled": false,
      "max_size_kb": 0,
      "max_size": -1,
      "check_on_raw": false
    },
    "bucket_quota": {
      "max_objects": -1,
      "enabled": false,
      "max_size_kb": 0,
      "max_size": -1,
      "check_on_raw": false
    },
    "placement_tags": [],
    "keys": [
      {
        "access_key": "8D8L1B048VTNK964KC3G",
        "secret_key": "7mkAyWcykb9jcAsB0vBXhS7i6r2M1mLYXHpjjP7n",
        "user": "ceph-user-kz1WAnvv"
      }
    ],
    "caps": [],
    "swift_keys": [],
    "subusers": [],
    "user_id": "ceph-user-kz1WAnvv",
    "admin": "false",
    "default_placement": "",
    "max_buckets": 1,
    "mfa_ids": []
  }
  ```

#### 7.3、S3存储业务代码（boto3）[推荐使用]

```python
import botostubs
from boto3.session import Session

import config

access_key = '8D8L1B048VTNK964KC3G' # 6.2中获取的access_key
secret_key = '7mkAyWcykb9jcAsB0vBXhS7i6r2M1mLYXHpjjP7n' # 6.2中获取的secret_key
s3_url = 'http://172.16.15.201:30141'
s3_key = 'test666'


class CephS3BOTO3():
    def __init__(self):
        self.session = Session(aws_access_key_id=config.access_key, aws_secret_access_key=config.secret_key)
        self.current_bucket_name = ''
        self.url = s3_url
        self.s3_client = self.session.client('s3', endpoint_url=self.url)  # type: botostubs.S3

    def get_bucket(self):
        buckets = [bucket['Name'] for bucket in self.s3_client.list_buckets()['Buckets']]
        print(buckets)
        self.current_bucket_name = buckets[0]
        return buckets

    def create_bucket(self):
        # 默认是私有的桶
        self.s3_client.create_bucket(Bucket='hy_test')
        # 创建公开可读的桶
        # ACL有如下几种"private","public-read","public-read-write","authenticated-read"
        self.s3_client.create_bucket(Bucket='hy_test', ACL='public-read')

    def upload(self):
        resp = self.s3_client.put_object(
            Bucket=self.current_bucket_name,  # 存储桶名称
            Key='test',  # 上传到
            Body=open("test.zip", 'rb').read()
        )
        print(resp)
        return resp

    def download(self):
        resp = self.s3_client.get_object(
            Bucket='ceph-bkt-d9bf00dc-e88f-4bc4-9425-d1e59f9b4613',
            Key='test'
        )
        with open('baa.zip', 'wb') as f:
            f.write(resp['Body'].read())


if __name__ == "__main__":
    # boto3
    cephs3_boto3 = CephS3BOTO3()
    cephs3_boto3.get_bucket()
    cephs3_boto3.upload()
    cephs3_boto3.download()

```

#### 7.4、ceph管理相关代码(需要在k8s节点中运行，后续需要封装)

```python
#!/usr/bin/env python
# coding: utf-8

# In[1]:
import rados
# In[2]:
cluster = rados.Rados(conffile = '/root/ceph.conf', conf = dict (keyring = '/root/keyring'))
# In[3]:
cluster.connect()
# In[4]:
print(cluster.get_cluster_stats())
# In[5]:
print(cluster.get_fsid())
# In[6]:
cluster_stats = cluster.get_cluster_stats()
# In[7]:
for key, value in cluster_stats.items():
    print (key, value)
# In[8]:
pools = cluster.list_pools()
for pool in pools:
    print (pool)
# In[9]:
import json
# In[10]:
def cmd(**kwargs):
    js = json.dumps(kwargs)
    ret, buf, err = cluster.mon_command(js,'',timeout=5)
    if(ret!=0):
        print(err)
    else:
        print(json.loads(buf))
--------------------------------------
# 支持一些管理命令查询
# In[12]:
cmd(prefix='df', format='json')
--------------------------------------
# 打开相应pool 的handler
# In[13]:
ioctx = cluster.open_ioctx('my-store.rgw.buckets.data')
# In[14]:
for obj in ioctx.list_objects():
    print(obj.key)
-----------------------

# ceph原生上传方式 
# In[15]:
file_name = "baa.zip"
f = open("/root/baa.zip", "rb")
file_content = f.read()
f.close()
# In[16]:
ioctx.write_full(file_name, file_content)
-------------------------
# 下载
# In[17]:
fw = open("/root/bbb.zip", "wb")
fw.write(ioctx.read(file_name,100*1024**2,0))
fw.close()
# In[18]:
for xattr in ioctx.get_xattrs('baa.zip'):
    print(xattr)
```

> 可将相关操作进行封装对外部提供相应api访问

### 8、数据隔离

#### 8.1、指定pool中对象存储到指定osd中

* 从Ceph集群里导出crush map

  命令：`ceph osd getcrushmap -o {compiled-crushmap-filename}`

* 反编译crush map

  命令：`crushtool -d {decompiled-crushmap-filename} -o {compiled-crushmap-filename}` 

* 修改crush map的#rules

  ```sh
  # begin crush map
  tunable choose_local_tries 0
  tunable choose_local_fallback_tries 0
  tunable choose_total_tries 50
  tunable chooseleaf_descend_once 1
  tunable chooseleaf_vary_r 1
  tunable chooseleaf_stable 1
  tunable straw_calc_version 1
  tunable allowed_bucket_algs 54
  
  # devices
  device 0 osd.0 class hdd
  device 1 osd.1 class hdd
  device 2 osd.2 class hdd
  device 3 osd.3 class ssd
  device 4 osd.4 class ssd
  device 5 osd.5 class ssd
  
  # types
  type 0 osd
  type 1 host
  type 2 chassis
  type 3 rack
  type 4 row
  type 5 pdu
  type 6 pod
  type 7 room
  type 8 datacenter
  type 9 zone
  type 10 region
  type 11 root
  
  # buckets
  host k8s-node3 {
          id -3           # do not change unnecessarily
          id -4 class hdd         # do not change unnecessarily
          # weight 0.049
          alg straw2
          hash 0  # rjenkins1
          item osd.0 weight 0.024
          item osd.5 weight 0.024
  }
  # hosts
  ...
  
  `新增root`
  root hdd_root {
          id -9           # do not change unnecessarily
          alg straw2
          hash 0  # rjenkins1
          item osd.0 weight 0.024
          item osd.1 weight 0.024
          item osd.2 weight 0.024
  }
  
  root ssd_root {
          id -10           # do not change unnecessarily
          alg straw2
          hash 0  # rjenkins1
          item osd.3 weight 0.024
          item osd.4 weight 0.024
          item osd.5 weight 0.024
  }
  
  
  # rules
  ...
  
  `新增hdd_rule和ssd_rule`
  rule hdd_rule {
          id 10
          type replicated
          min_size 1
          max_size 10
          step take hdd_root
          step chooseleaf firstn 0 type osd 
          step emit
  }
  
  rule ssd_rule {
          id 11
          type replicated
          min_size 1
          max_size 10
          step take ssd_root
          step chooseleaf firstn 0 type osd
          step emit
  }
  
  # end crush map
  ```
  
* 编译crush map

  命令：`crushtool -c {decompiled-crushmap-filename} -o {compiled-crushmap-filename}`

* 导入自定义的 crush map 到Ceph 集群

  命令：`ceph osd setcrushmap -i {compiled-crushmap-filename}`

* 修改资源池pool的存储规则

  ```sh
  [root@rook-ceph-tools-565698c784-226rd opt]# ceph osd pool set myfs-metadata  crush_rule hdd_rule
  set pool 8 crush_rule to hdd_rule
  [root@rook-ceph-tools-565698c784-226rd opt]# ceph pg ls-by-pool myfs-metadata |awk '{print $1,$2,$15}'
  PG OBJECTS ACTING
  8.0 0 [1,0,2]p1
8.1 0 [1,0,2]p1
  8.2 0 [2,0,1]p2
  8.3 0 [1,0,2]p1
  8.4 0 [2,0,1]p2
  8.5 0 [1,2,0]p1
  8.6 1 [0,2,1]p0
  8.7 1 [1,2,0]p1
  8.8 0 [1,0,2]p1
  8.9 0 [0,2,1]p0
  8.a 0 [1,2,0]p1
  8.b 1 [0,1,2]p0
  8.c 0 [0,2,1]p0
  8.d 1 [2,1,0]p2
  8.e 1 [0,2,1]p0
  8.f 1 [0,2,1]p0
  8.10 4 [0,1,2]p0
  8.11 0 [2,1,0]p2
  8.12 0 [2,0,1]p2
  8.13 2 [1,2,0]p1
  8.14 4 [0,2,1]p0
  8.15 2 [2,1,0]p2
  8.16 0 [1,2,0]p1
  8.17 0 [1,0,2]p1
  8.18 0 [0,2,1]p0
  8.19 0 [1,0,2]p1
  8.1a 0 [0,2,1]p0
  8.1b 0 [2,0,1]p2
  8.1c 1 [2,0,1]p2
  8.1d 1 [0,1,2]p0
  8.1e 0 [1,0,2]p1
  8.1f 2 [0,1,2]p0
  
  
  [root@rook-ceph-tools-565698c784-226rd opt]# ceph osd pool set myfs-data0 crush_rule ssd_rule
  set pool 9 crush_rule to ssd_rule
  `写数据`
  [root@rook-ceph-tools-565698c784-226rd opt]# rados -p myfs-data0 put crushmap crushmap.new 
  [root@rook-ceph-tools-565698c784-226rd opt]# rados ls -p myfs-data0
  crushmap
  [root@rook-ceph-tools-565698c784-226rd opt]# ceph pg ls-by-pool myfs-data0 | awk  '{print $1,$2,$15}'
  PG OBJECTS ACTING
  9.0 0 [5,4,3]p5
  9.1 0 [3,5,4]p3
  9.2 0 [3,4,5]p3
  9.3 0 [3,4,5]p3
  9.4 0 [3,5,4]p3
  9.5 0 [5,4,3]p5
  9.6 0 [5,3,4]p5
  9.7 0 [5,3,4]p5
  9.8 0 [5,3,4]p5
  9.9 0 [5,3,4]p5
  9.a 0 [3,5,4]p3
  9.b 0 [5,4,3]p5
  9.c 0 [3,4,5]p3
  9.d 0 [3,5,4]p3
  9.e 0 [3,4,5]p3
  9.f 0 [5,4,3]p5
  9.10 0 [4,3,5]p4
  9.11 0 [3,4,5]p3
  9.12 0 [5,3,4]p5
  9.13 0 [3,5,4]p3
  9.14 0 [5,3,4]p5
  9.15 0 [4,5,3]p4
  9.16 0 [4,3,5]p4
  9.17 0 [3,5,4]p3
  9.18 0 [5,3,4]p5
  9.19 0 [3,5,4]p3
  9.1a 0 [5,4,3]p5
  9.1b 0 [4,5,3]p4
  9.1c 0 [5,4,3]p5
  9.1d 0 [4,3,5]p4
  9.1e 0 [5,3,4]p5
  9.1f 1 [3,4,5]p3
  ```
  
  pool下所有pg已有数据会进行迁移之后的数据也会根据设置的rule存储到指定osd中

#### 8.2、数据归档

* 无法整体迁移 可以做数据冷热存储

