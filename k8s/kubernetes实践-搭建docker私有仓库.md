# kubernetes实践-搭建docker私有仓库

#### 创建私有仓库

```shell
docker run --name registry -d  -p 5000:5000 --restart=always  -v /opt/data/registry:/var/lib/registry registry
```

#### 修改原镜像tag

``` shell
docker tag nginx 172.16.15.203:5000/nginx:v1
```

docker 会根据image REPOSITORY '/' 前的地址去寻找仓库，直接push 推送会失败，这是因为 Registry 使用的是 Https，而私有仓库只提供 http 服务，需要修改 /etc/docker/daemon.json文件，添加 insecure-registry 设置

```json
{
	"insecure-registries": ["172.16.15.203:5000"]
}
```

#### 重启docker

```sh
systemctl restart docker
```

#### 推送

```sh
docker push 172.16.15.200:5000/nginx:v1
The push refers to repository [172.16.15.203:5000/rayproject]
34ba562763fa: Pushed 
bfacb2e5189e: Pushed 
8fbda14ca7ca: Pushed 
460682ecf702: Pushed 
936ff450c2ac: Pushed 
1e9a7712694f: Pushed 
cd7d8727f618: Pushed 
aba9a7e36dc8: Pushed 
2037ba4f2acb: Pushed 
9ddf8589cf57: Pushed 
262696d33421: Pushed 
381271eea4f9: Pushed 
f0c8dcae22c0: Pushed 
4ae3adcb66cb: Pushed 
aa6685385151: Pushed 
0040d8f00d7e: Pushed 
9e6f810a2aab: Pushed 
autoscaler_v2: digest: sha256:41a910ba8e2bf477b6a4e600b9253724f42ba568d70663abd04bf13dfefd79dd size: 3882
```

#### 拉取

集群中其余机器也修改 /etc/docker/daemon.json设置

```
docker pull 172.16.15.200:5000/nginx:v1
```

