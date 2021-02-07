# kubernetes实践  rook-ceph数据隔离









```bash
[root@rook-ceph-tools-565698c784-226rd /]# ceph osd lspools
1 my-store.rgw.control
2 my-store.rgw.meta
3 my-store.rgw.log
4 my-store.rgw.buckets.index
5 my-store.rgw.buckets.non-ec
6 .rgw.root
7 my-store.rgw.buckets.data
8 myfs-metadata
9 myfs-data0
[root@rook-ceph-tools-565698c784-226rd /]# ceph pg ls-by-pool myfs-metadata |awk '{print $1,$2,$15}'
PG OBJECTS ACTING
8.0 0 [5,3,1]p5
8.1 0 [0,4,3]p0
8.2 0 [1,0,2]p1
8.3 0 [1,5,2]p1
8.4 0 [2,4,0]p2
8.5 0 [1,2,0]p1
8.6 1 [1,3,0]p1
8.7 1 [5,1,2]p5
8.8 0 [2,0,1]p2
8.9 0 [4,2,0]p4
8.a 0 [1,5,2]p1
8.b 1 [4,2,0]p4
8.c 0 [1,5,2]p1
8.d 1 [4,2,0]p4
8.e 1 [5,3,1]p5
8.f 1 [1,5,3]p1
8.10 4 [3,4,0]p3
8.11 0 [1,2,0]p1
8.12 0 [3,1,5]p3
8.13 2 [5,1,3]p5
8.14 4 [3,0,1]p3
8.15 2 [4,2,0]p4
8.16 0 [1,0,2]p1
8.17 0 [2,0,1]p2
8.18 0 [2,4,5]p2
8.19 0 [2,1,5]p2
8.1a 0 [0,2,1]p0
8.1b 0 [2,0,4]p2
8.1c 1 [4,2,0]p4
8.1d 1 [0,4,2]p0
8.1e 0 [0,3,1]p0
8.1f 2 [4,3,5]p4
[root@rook-ceph-tools-565698c784-226rd /]# ceph pg ls-by-pool myfs-data0 | awk  '{print $1,$2,$15}'
PG OBJECTS ACTING
9.0 0 [5,2,4]p5
9.1 0 [0,4,3]p0
9.2 0 [0,4,3]p0
9.3 0 [4,5,3]p4
9.4 0 [0,3,1]p0
9.5 0 [4,2,5]p4
9.6 0 [2,1,5]p2
9.7 0 [4,3,0]p4
9.8 0 [4,3,5]p4
9.9 0 [1,3,5]p1
9.a 0 [2,5,1]p2
9.b 0 [4,3,5]p4
9.c 0 [0,3,1]p0
9.d 0 [3,0,1]p3
9.e 0 [2,4,0]p2
9.f 0 [2,5,4]p2
9.10 0 [3,0,4]p3
9.11 0 [3,5,1]p3
9.12 0 [2,0,4]p2
9.13 0 [1,2,0]p1
9.14 0 [5,4,3]p5
9.15 0 [3,4,5]p3
9.16 0 [4,0,2]p4
9.17 0 [1,5,2]p1
9.18 0 [1,3,0]p1
9.19 0 [0,3,1]p0
9.1a 0 [5,3,4]p5
9.1b 0 [5,3,4]p5
9.1c 0 [5,1,3]p5
9.1d 0 [4,0,2]p4
9.1e 0 [0,3,1]p0
9.1f 0 [5,1,3]p5
```









```bash
[root@rook-ceph-tools-565698c784-226rd opt]# ceph osd getcrushmap -o crushmap
32
[root@rook-ceph-tools-565698c784-226rd opt]# crushtool -d crushmap  -o decrushmap
[root@rook-ceph-tools-565698c784-226rd opt]# ll
total 8
-rw-r--r-- 1 root root 1641 Apr 13 06:27 crushmap
-rw-r--r-- 1 root root 2890 Apr 13 06:27 decrushmap
```

```bash
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
device 3 osd.3 class hdd
device 4 osd.4 class hdd
device 5 osd.5 class hdd

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
host k8s-node1 {
        id -5           # do not change unnecessarily
        id -6 class hdd         # do not change unnecessarily
        # weight 0.049
        alg straw2
        hash 0  # rjenkins1
        item osd.2 weight 0.024
        item osd.3 weight 0.024
}
host k8s-node2 {
        id -7           # do not change unnecessarily
        id -8 class hdd         # do not change unnecessarily
        # weight 0.049
        alg straw2
        hash 0  # rjenkins1
        item osd.1 weight 0.024
        item osd.4 weight 0.024
}
root default {
        id -1           # do not change unnecessarily
        id -2 class hdd         # do not change unnecessarily
        # weight 0.146
        alg straw2
        hash 0  # rjenkins1
        item k8s-node3 weight 0.049
        item k8s-node1 weight 0.049
        item k8s-node2 weight 0.049
}

# rules
rule replicated_rule {
        id 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.control {
        id 1
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.meta {
        id 2
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.log {
        id 3
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.index {
        id 4
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.non-ec {
        id 5
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule .rgw.root {
        id 6
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.data {
        id 7
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule myfs-metadata {
        id 8
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule myfs-data0 {
        id 9
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}

# end crush map
```





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
host k8s-node1 {
        id -5           # do not change unnecessarily
        id -6 class hdd         # do not change unnecessarily
        # weight 0.049
        alg straw2
        hash 0  # rjenkins1
        item osd.2 weight 0.024
        item osd.3 weight 0.024
}
host k8s-node2 {
        id -7           # do not change unnecessarily
        id -8 class hdd         # do not change unnecessarily
        # weight 0.049
        alg straw2
        hash 0  # rjenkins1
        item osd.1 weight 0.024
        item osd.4 weight 0.024
}
root default {
        id -1           # do not change unnecessarily
        id -2 class hdd         # do not change unnecessarily
        # weight 0.146
        alg straw2
        hash 0  # rjenkins1
        item k8s-node3 weight 0.049
        item k8s-node1 weight 0.049
        item k8s-node2 weight 0.049
}

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
rule replicated_rule {
        id 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.control {
        id 1
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.meta {
        id 2
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.log {
        id 3
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.index {
        id 4
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.non-ec {
        id 5
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule .rgw.root {
        id 6
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule my-store.rgw.buckets.data {
        id 7
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule myfs-metadata {
        id 8
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}
rule myfs-data0 {
        id 9
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}

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



```sh
[root@rook-ceph-tools-565698c784-226rd opt]# ceph osd dump|grep myfs
pool 8 'myfs-metadata' replicated size 3 min_size 2 crush_rule 8 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode warn last_change 455 flags hashpspool stripe_width 0 pg_autoscale_bias 4 pg_num_min 16 recovery_priority 5 application cephfs
pool 9 'myfs-data0' replicated size 3 min_size 2 crush_rule 9 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode warn last_change 455 flags hashpspool stripe_width 0 application cephfs
------------
`前后对比`
-----------
[root@rook-ceph-tools-565698c784-226rd opt]# ceph osd dump|grep myfs `asd`
pool 8 'myfs-metadata' replicated size 3 min_size 2 crush_rule 10 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode warn last_change 457 flags hashpspool stripe_width 0 pg_autoscale_bias 4 pg_num_min 16 recovery_priority 5 application cephfs
pool 9 'myfs-data0' replicated size 3 min_size 2 crush_rule 11 object_hash rjenkins pg_num 32 pgp_num 32 autoscale_mode warn last_change 459 flags hashpspool stripe_width 0 application cephfs
```



####  编译 

命令：crushtool -c {decompiled-crushmap-filename} -o {compiled-crushmap-filename}

```sh
[root@rook-ceph-tools-565698c784-226rd opt]# crushtool -c decrushmap.new -o crushmap.new
```

####  导入自定义的 crush map 到Ceph 集群

命令：ceph osd setcrushmap -i {compiled-crushmap-filename}

```sh
[root@rook-ceph-tools-565698c784-226rd opt]# ceph osd setcrushmap -i  crushmap.new 
33
--------------------------

--------------------------
[root@rook-ceph-tools-565698c784-226rd opt]# ceph osd crush rule ls 
replicated_rule
my-store.rgw.control
my-store.rgw.meta
my-store.rgw.log
my-store.rgw.buckets.index
my-store.rgw.buckets.non-ec
.rgw.root
my-store.rgw.buckets.data
myfs-metadata
myfs-data0
hdd_rule
ssd_rule

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

