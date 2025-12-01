# 概述
参考：https://github.com/wyy-holding/dify-k8s

主要通过`Kompose`将`docker-compose.yaml`转换为Kubernetes资源文件。
作者将这些文件进行了优化，分类。

我在此基础上，进行了大量修改，做了进一步优化，分类。

删除了一些无用的代码，所有组件统一使用一个pvc进行存储，方便管理。

支持dify以下版本
```bash
1.1.2
1.1.3
1.2.0
1.3.0
1.3.1
1.4.0
1.4.1
1.4.2
1.4.3
1.5.0
1.5.1
1.6.0
1.7.0
1.7.1
1.7.2
1.8.0
1.8.1
1.9.0
1.9.1
1.9.2
1.10.0
1.10.1
``` 

# 目录结构
```bash
env --> 全局环境变量
pvc --> 所有组件，统一使用一个pvc来进行持久化存储
databases --> 数据库相关：postgresql，redis，weaviate
middleware --> 中间件相关：plugin-daemon，sandox，ssf-proxy，nginx
services --> 服务相关：api，web，worker
```

# 部署
详细步骤，请参考文章：https://www.cnblogs.com/xiao987334176/p/18810257
## 1. 创建命名空间dify
```bash
kubectl create namespace dify
```

## 2. 创建全局环境变量
```bash
kubectl apply -f env/env.yaml
```

## 3. 创建pv和pvc
这里的pv是自建的NFS，请根据实际情况修改
```bash
kubectl apply -f pvc/storageClass.yaml
kubectl apply -f pvc/pv.yaml
kubectl apply -f pvc/pvc.yaml
```
进入到NFS根目录，创建dify相关持久化文件，并设置权限
```bash
mkdir -p dify/volumes/db/data
mkdir -p dify/volumes/redis/data
mkdir -p dify/volumes/weaviate
mkdir -p dify/volumes/plugin_daemon
mkdir -p dify/volumes/app/storage
chmod 777 -R dify
```

## 4. 数据库相关
### postgresql
```bash
kubectl apply -f databases/postgresql/postgres-StatefulSet.yaml
kubectl apply -f databases/postgresql/postgres-Service.yaml
```
手动创建postgres,dify数据库
```bash
kubectl -n dify exec -it postgres-0 -- createdb postgres
kubectl -n dify exec -it postgres-0 -- createdb dify
```
进入postgres容器，添加一条白名单
```bash
vi /var/lib/postgresql/data/pgdata/pg_hba.conf
```
最后一行增加
```bash
host    all             all             0.0.0.0/0               md5
```
重启PostgreSQL
```bash
kubectl -n dify delete pods postgres-0
```
### redis
```bash
kubectl apply -f databases/redis/redis-StatefulSet.yaml
kubectl apply -f databases/redis/redis-Service.yaml
```
### weaviate
```bash
kubectl apply -f databases/weaviate/weaviate-StatefulSet.yaml
kubectl apply -f databases/weaviate/weaviate-Service.yaml
```

## 5. 中间件相关

`注意：这里的Nginx，最后一个运行。因为它是用来转发各个组件api接口的。`

### plugin-daemon
```bash
kubectl apply -f middleware/plugin-daemon/plugin-daemon-Deployment.yaml
kubectl apply -f middleware/plugin-daemon/plugin-daemon-Service.yaml
```
### sandbox
```bash
kubectl apply -f middleware/sandbox/sandbox-cm0.yaml
kubectl apply -f middleware/sandbox/sandbox-cm1.yaml
kubectl apply -f middleware/sandbox/sandbox-Deployment.yaml
kubectl apply -f middleware/sandbox/sandbox-Service.yaml
```
### ssrf-proxy
```bash
kubectl apply -f middleware/ssrf-proxy/ssrf-proxy-cm0.yaml
kubectl apply -f middleware/ssrf-proxy/ssrf-proxy-cm1.yaml
kubectl apply -f middleware/ssrf-proxy/ssrf-proxy-Deployment.yaml
kubectl apply -f middleware/ssrf-proxy/ssrf-proxy-Service.yaml
```

## 6. 服务相关
### api
```bash
kubectl apply -f services/api/api-StatefulSet.yaml
kubectl apply -f services/api/api-Service.yaml
```
查看日志，是否有报错
```bash
kubectl -n dify logs -f api-0
```
**注意：api首次运行，会自动创建相关表结构，从日志中，就可以看到创建表结果过程。**

### web
```bash
kubectl apply -f services/web/web-Deployment.yaml
kubectl apply -f services/web/web-Service.yaml
```
### worker
```bash
kubectl apply -f services/worker/worker-StatefulSet.yaml
kubectl apply -f services/worker/worker-Service.yaml
```
### worker-beat
1.7.0新增组件，是一个 轻量级定时心跳守护进程
```bash
kubectl apply -f services/worker-beat/worker-beat-StatefulSet.yaml
kubectl apply -f services/worker-beat/worker-beat-Service.yaml
```

等待上面所有组件运行正常之后，部署Nginx组件
### nginx
```bash
kubectl apply -f middleware/nginx/nginx-cm0.yaml
kubectl apply -f middleware/nginx/nginx-cm1.yaml
kubectl apply -f middleware/nginx/nginx-cm2.yaml
kubectl apply -f middleware/nginx/nginx-cm3.yaml
kubectl apply -f middleware/nginx/nginx-cm4.yaml
kubectl apply -f middleware/nginx/nginx-cm5.yaml
kubectl apply -f middleware/nginx/nginx-Deployment.yaml
kubectl apply -f middleware/nginx/nginx-Service.yaml
```

# 访问dify
直接使用nginx的nodeport端口访问即可

如果需要域名访问，则添加一条ingress规则，指向到nginx的svc即可。

# 常见问题
## 1. 镜像拉取失败
由于国内无法访问docker hub，正常拉取会失败

首先准备一台docker服务器，修改文件`/etc/docker/daemon.json`
```bash
{
	"registry-mirrors": [
		"https://docker.1ms.run",
		"https://docker.xuanyuan.me"
	]
}
```
重启docker进程，就可以拉取docker hub镜像了

然后将本地的docker镜像推送到私有仓库，比如：harbor

最后将yaml文件中的镜像改为私有仓库地址即可，注意添加私有仓库密钥。

## 2. 某些组件响应慢
这些组件限制的cpu和内存，用的是最小资源。如果使用场景访问量很大，请根据实际情况调整。

## 3. k8s nfs挂载失败
```bash
Mounting command: mount
Mounting arguments: -t nfs 10.180.6.75:/srv/nfs /var/lib/kubelet/pods/7059a115-32ec-4dbc-9c4c-c5670649be94/volumes/kubernetes.io~nfs/dify
Output: mount: /var/lib/kubelet/pods/7059a115-32ec-4dbc-9c4c-c5670649be94/volumes/kubernetes.io~nfs/dify: bad option; for several filesystems (e.g. nfs, cifs) you might need a /sbin/mount.<type> helper program.
```
解决办法：

每一个node都需要安装nfs客户端，如果是ubuntu系统，使用命令`apt install -y nfs-common`

确保nfs服务端`/etc/exports`配置正确，一定包含`no_all_squash，no_root_squash`
```bash
/data/nfs_share *(rw,sync,no_all_squash,no_root_squash,no_subtree_check)
```
重启nfs-server
```bash
# 刷新配置
sudo exportfs -a
# 重启nfs-server
sudo systemctl restart nfs-server
```
## 4. plugin-daemon组件运行失败
```bash
[error] failed to initialize database, got error failed to connect to `host=postgres user=postgres database=dify_plugin`: server error (FATAL: no pg_hba.conf entry for host "10.42.0.45", user "postgres", database "dify_plugin", no encryption (SQLSTATE 28000))
```
解决办法：

PostgreSQL没有加白名单，修改文件`/var/lib/postgresql/data/pgdata/pg_hba.conf`增加一行
```bash
host    all             all             0.0.0.0/0               md5
```
重启PostgreSQL
## 5. dify升级
dify支持跨版本升级，例如：将1.3.1版本升级到1.6.0版本。
升级步骤：
1. 将1.3.1版本的yaml文件，修改为1.6.0版本的yaml文件，等待所有组件运行正常，重启nginx组件。
2. 更新PostgreSQL表结构，执行以下步骤:
进入api容器
```bash
kubectl -n dify exec -it api-0 -- /bin/bash
```
默认会进入到路径/app/api，更新Python依赖项
```bash
uv sync
```
然后，让我们运行迁移脚本
```bash
uv run flask db upgrade
```
最后，再次重启api、Worker、Web这3个组件即可。