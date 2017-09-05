# shipyard搭建

**系统环境**

* centos7
* docker 17.06.1-ce
* 安装docker-compose
* 安装docker-machine

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
$ docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-rethinkdb \
    rethinkdb
```
然后启动discovery,这个的功能是向swarm集群提供发现服务的,运行下面的命令
```bash
$ docker run \
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
$ docker run \
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
$ docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-swarm-manager \
    swarm:latest \
    manage --host tcp://0.0.0.0:3375 etcd://<IP-OF-HOST>:4001
```
然后运行swarm agent,这是为了将本机也加入到集群节点之中,命令如下,第一个```<ip-of-host>```是当前电脑的ip,第二个是etcd发现服务电脑的ip,由于本机是manager节点,所以此处也填本机ip,若本机作为子节点,则应填入manager的ip地址
```bash
$ docker run \
    -ti \
    -d \
    --restart=always \
    --name shipyard-swarm-agent \
    swarm:latest \
    join --addr <ip-of-host>:2375 etcd://<ip-of-host>:4001
```
最后,运行shipyard-controller容器,```--link```参数是用来链接其他容器,这里将shipyard-rethinkdb映射到controller里,controller访问rethinkdb时,直接写rethinkdb就行,不用再去找rethinkdb的ip地址
```bash
$ docker run \
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

**增加节点**

shipyard增加节点只需要在节点机上运行命令,其中10.0.1.10要改为manager节点的地址

```curl -sSL https://shipyard-project.com/deploy | ACTION=node DISCOVERY=etcd://10.0.1.10:4001 bash -s```

或者阅读脚本命令,自己手动部署,效果相同

# 本地仓库搭建
**通过镜像方式**
docker官方提供了创建私有仓库的镜像```registry```,只需要简单的拉下来跑起来就能拥有一个私有仓库,私有仓库默认端口是5000,运行如下命令启动私有仓库,其中```--restart=always```定义了,这个容器在每次docker启动的时候就会自动运行,加入这个属性可以避免重启docker服务后还要手动启动一堆容器,```-v /mnt/registry:/var/lib/registry```是将容器里的```/var/lib/registry```路径挂在到本地的```/mnt/registry```路径,这个可以避免误删容器时将存储的数据删除了
```bash
$ docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /mnt/registry:/var/lib/registry \
  registry:2
```
> registry有很多的配置可以修改,可以自定义端口,自定义数据存放地址等等一系列配置,具体配置可以看官方文档,[resigty部分](https://docs.docker.com/registry/configuration/#list-of-configuration-options)

registry容器跑起来后,在本地就可以上传镜像了,首先需要用```docker tag```命令修改镜像名字,docker镜像的命名规则是

```registry<url>:port/namespace/imagename:tag```

举例,通过```docker tag shipyard/shipyard:latest localhost:5000/shipyard/shipyard:latest```将shipyard镜像改为了```localhost:5000/shipyard/shipyard```

然后通过```docker push localhost:5000/shipyard/shipyard:latest```就能讲这个镜像上传到私有仓库

通过```docker rmi <imageID>```删除shipyard的镜像,然后通过```docker pull localhost:5000/shipyard/shipyard:latest```拉取镜像,已验证是否成功

> 目前的registry还只能在本地使用,要想在其他docker节点机上拉取或上传私有仓库,还需要配置一下

**配置其他机器使用registry**

由于docker仅允许```https```访问registry,所以,在其他节点上,直接访问registry节点的ip访问不过去,提示认证失败等错误,有两种方法可用

方法一:
修改或创建```/etc/docker/daemon.json```
加入如下内容
```json
{
  "insecure-registries" : ["myregistrydomain.com:5000"]
}
```
然后```systemctl restart docker```重启docker服务,这样就能通过```http```连上私有仓库了
> daemon.json还有很多其他可以配置的属性,可以看[daemon CLI reference](https://docs.docker.com/edge/engine/reference/commandline/dockerd/)

方法二:
获取CA认证,此处采用个人签名证书,有钱可以上权威认证证书
首先运行如下命令创建公钥和私钥
```bash
$ mkdir -p certs

$ openssl req \
  -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
  -x509 -days 365 -out certs/domain.crt
```
运行这段命令后会让你输入一些信息,可以自由发挥,但域名那一栏一定不能乱填,不能搞事情,下面是参考
```bash
Generating a 4096 bit RSA private key
.........................................................................................................................................................................................................++
...++
writing new private key to 'certs/domain.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Chongqing
Locality Name (eg, city) []:Chongqing
Organization Name (eg, company) [Internet Widgits Pty Ltd]:SpringAirline
Organizational Unit Name (eg, section) []:QC
Common Name (e.g. server FQDN or YOUR name) []:czy.registry.com
Email Address []:docker@inner.czy.com
```
证书生成之后可以按照官方文档的样子启动容器,[TLS模式重启registry](https://docs.docker.com/registry/deploying/#get-a-certificate)
```bash
$ docker run -d \
  --restart=always \
  --name registry \
  -v `pwd`/certs:/certs \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:80 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  -p 80:80 \
  registry:2
```
在远程节点机上,还需要将私钥```domain.crt```复制到```/etc/docker/certs.d/myregistrydomain.com:5000/ca.crt```下,```myregistrydomain.com```要与证书内配置的地址一致,然后重启docker服务即可通过https远程访问私有仓库
若还是不能访问,提示认证问题,则可能是某些系统拦截了证书请求,考虑将证书加入到系统信任中,针对不同linux系统,方法不同

**UBUNTU**

执行如下命令
```bash
$ cp certs/domain.crt /usr/local/share/ca-certificates/myregistrydomain.com.crt
update-ca-certificates
```

**Red Hat Enterprise Linux**

```bash
$ cp certs/domain.crt /etc/pki/ca-trust/source/anchors/myregistrydomain.com.crt
update-ca-trust
```
**Oracle Linux**

```bash
$ update-ca-trust enable
```
然后重启docker服务,再尝试访问私有仓库

> 网上流行用nginx反向代理做https认证,虽然我也没搞明白为什么要这么做,但依葫芦画瓢我也写下来

首先安装最新的nginx,不会的可以参考[官方文档](http://nginx.org/en/linux_packages.html)进行安装
大致是先按照官方文档添加```yum```源,然后通过```yum makecache```创建缓存,如果出问题,可以尝试先```yum clean all```清理缓存,再重新创建,最后通过```yum install nginx```安装nginx
验证nginx是否安装好,可以通过```nginx --help```命令,如果已经有nginx命令了,应该是安装好了,或者通过```systemctl start nginx```启动nginx,访问```localhost<主机ip>```nginx默认监听80端口,是http默认端口,所以url后面可以不用接端口号,如果显示nginx欢迎页面,则表示nginx安装成功
