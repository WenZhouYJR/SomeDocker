# shipyard搭建

**系统环境**

* centos7
* docker 17.06.1-ce

**自动部署**

运行如下代码就能自动部署一套shipyard环境,运行localhost:8080就能访问shipyard

```curl -sSL https://shipyard-project.com/deploy | bash -s```

感兴趣可以打开```https://shipyard-project.com/deploy```查看shell脚本,自动话脚本提供了多种```ACTION```的操作模式,可以方便的实现快速部署,快速添加节点,删除shipyard等使用的功能

**手动部署**

首先启动docker服务,运行```systemctl start docker```,可以通过```docker --help```查看docker提供的命令

> docker命令只能用```root```操作,如果想要其他用户可以操作docker命令,可以```sudo groupadd docker```新建一个docker用户组,一般安装docker后就会存在一个docker用户组,随后```sudo gpasswd -a ${USER} docker```将当前用户加入到docker组,其中```${USER}```是当前用户,也可以直接写用户名,最后```sudo systemctl restart docker```重启docker服务,并**退出账号重新登录**,就能用非root账号操作docker命令了

第一步,拉取镜像,shipyard环境总共启动了7个容器,对应6个镜像,分别是

* shipyard/shipyard
* swarm
* shipyard/docker-proxy
* microbox/etcd
* rethinkdb
* alpine

采用```docker pull IMAGE-NAME```拉取镜像至本地,例如```docker pull shipyard/shipyard```,也可以先用```docker search IMAGE-NAME```查看docker仓库中的镜像,拉取得星最多的,或者官方认证的包,或者制定拉取其他包

第二步,启动容器
首先启动rethinkdb,执行下面命令即可
```bash
docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-rethinkdb \
    rethinkdb
```
然后启动discovery,这个的功能是向swarm集群提供发现服务的,运行下面的命令
```bash
docker run \
    -ti \
    -d \
    -p 4001:4001 \
    -p 7001:7001 \
    --restart=always \
    --name shipyard-discovery \
    microbox/etcd -name discovery
```
然后运行proxy代理服务,docker默认监听socket,通过代理服务可以将tcp等进行转发
```bash
docker run \
    -ti \
    -d \
    -p 2375:2375 \
    --hostname=$HOSTNAME \
    --restart=always \
    --name shipyard-proxy \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -e PORT=2375 \
    shipyard/docker-proxy:latest
```
然后运行swarm manager,swarm是docker提供的集群功能,swarm manager可以管理多个swarm agent,命令中的```<IP-OF-HOST>```是当前电脑的ip地址
```bash
docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-swarm-manager \
    swarm:latest \
    manage --host tcp://0.0.0.0:3375 etcd://<IP-OF-HOST>:4001
```
然后运行swarm agent,这是为了将本机也加入到集群节点之中,命令如下,第一个```<ip-of-host>```是当前电脑的ip,第二个是etcd发现服务电脑的ip,由于本机是manager节点,所以此处也填本机ip,若本机作为子节点,则应填入manager的ip地址
```bash
docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-swarm-agent \
    swarm:latest \
    join --addr <ip-of-host>:2375 etcd://<ip-of-host>:4001
```
最后,运行shipyard-controller容器,```--link```参数是用来链接其他容器,这里将shipyard-rethinkdb映射到controller里,controller访问rethinkdb时,直接写rethinkdb就行,不用再去找rethinkdb的ip地址
```bash
docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-controller \
    --link shipyard-rethinkdb:rethinkdb \
    --link shipyard-swarm-manager:swarm \
    -p 8080:8080 \
    shipyard/shipyard:latest \
    server \
    -d tcp://swarm:3375
```
> ```--link```相当于是在容器内修改了```/etc/hosts```文件,例如rethinkdb的容器ip是172.12.0.3,controller通过link链接了rethinkdb容器,则可以在controller的hosts文件中找到```172.12.0.3  rethinkdb <rethinkdb的容器id>```

当所有容器都启动之后,运行```docker ps```命令,应该可以看到如下内容
```bash
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS                    PORTS                                            NAMES
59a96a2356ab        shipyard/shipyard:latest       "/bin/controller -..."   2 days ago          Up 14 seconds             0.0.0.0:8080->8080/tcp                           shipyard-controller
3cd68e51673e        swarm:latest                   "/swarm j --addr 1..."   2 days ago          Up 55 seconds             2375/tcp                                         shipyard-swarm-agent
93e27c7434f9        swarm:latest                   "/swarm m --replic..."   2 days ago          Up 53 seconds             2375/tcp                                         shipyard-swarm-manager
8a68c07675da        shipyard/docker-proxy:latest   "/usr/local/bin/run"     2 days ago          Up About a minute         0.0.0.0:2375->2375/tcp                           shipyard-proxy
5142636c549a        alpine                         "sh"                     2 days ago          Up About a minute                                                          shipyard-certs
ac0f27bb97e2        microbox/etcd:latest           "/bin/etcd -addr 1..."   2 days ago          Up About a minute         0.0.0.0:4001->4001/tcp, 0.0.0.0:7001->7001/tcp   shipyard-discovery
711497e75c5a        rethinkdb                      "rethinkdb --bind all"   2 days ago          Up About a minute         8080/tcp, 28015/tcp, 29015/tcp                   shipyard-rethinkdb
```
此时,本地通过访问```localhost:8080```,或者其他地方访问本机```<ip>:8080```就能看到shipyard的登录界面,初始化账号密码是```admin/shipyard```


