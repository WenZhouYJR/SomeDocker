# 一键部署配置流程

代码提交自动构建自动部署流程就像下图一样

![自动流程图](https://github.com/WenZhouYJR/SomeDocker/blob/master/docs/img/autograph.jpg)

准备工作:
* 安装git
* 有一个gitlab项目

**git安装**

git和svn都是代码的版本控制工具,但git是分布式的,安装git很简单,在[git官网](https://git-scm.com/)下载git安装包安装即可,安装好后就能用了

**git关联远程仓库**

在本地文件夹右键选择```Git Bash Here```,然后输入```git init```,这样本地的git仓库就建好了,接下来就是将本地仓库推送到远程托管平台[gitlab](http://10.131.0.62:10080)

以```APPTEST```项目为例,项目地址是```http://10.131.0.62:10080/zhouhongyu/APPTEST.git```,在git bash中输入

```git
# git remote add [shortname] [url]
git remote add http://10.131.0.62:10080/zhouhongyu/APPTEST.git
git push --all
```

这样就能把本地仓库和远程仓库关联上了

**远程仓库配置webhook**

webhook可以监控代码状态,在发生指定事件时,触发webhook,请求配置的链接,用这个可以监控代码push动作,然后出发jenkins自动构建

进入gitlab的项目中,点击```Setting```,然后再选中```Integrations```标签,这里就能新增webhook功能,```URL```填上jenkins远程构建的地址,就能触发远程构建了

> **Note:** 由于我们的jenkins增加了用户权限限制功能,所以直接远程请求会报错,返回需要用户认证,所以我自己开了一个接口来执行远程构建,```URL```我输入的是接口地址```http://10.131.0.166:8092/APPTEST/servlet/webhook?path=/u02/APPTEST&token=APPTEST```,原理是通过java去调用存放在path路径的jmeter脚本,token是jenkins远程构建中设置的验证token,之所以jenkins能够请求远程构建,是因为我将jenkins的cookie放到了jmeter的请求当中,当然也有很多其他方法可以实现绕过认证的操作,大家可以自行探索

**jenkins配置**

首先是在jenkins上新建一个job,和往常一样,区别在于```构建触发器```需要选择```触发远程构建```,身份令牌就是token
然后就是项目部署的时候,直接将整个编译后的包放到远程目录下(例如/u02/APPTEST),同时通过脚本执行```docker build```操作,生成新的镜像,然后上传仓库,最后启动启动容器,项目就重启了
上传仓库这个操作在这次流程中没有起到任何作用,如果你能确保你的项目不会迁移,就在同一台docker终端上运行,也可以不上传到仓库,上传仓库是为了方便在其他终端上启动容器

> **Note:** 理想的做法是在jenkins机器上直接生成镜像,然后push到仓库,再触发运行终端拉取镜像启动容器,但由于我们的jenkins上没有安装docker环境,所以我将编译后文件放到了远程终端上生成镜像并上传

以下部分是项目包复制到/u02/APPTEST后执行的脚本

```bash
#!/bin/bash

set -e

REPOSITORY=${R:-ck.czy/apptest}
TAG=${T:-latest}
CONTAINER=${C:-apptest}

IFIMAGEEXIST=$(docker images $REPOSITORY | grep $TAG | tr -s ' ' | cut -d ' ' -f 3)
IFCONTAINEREXIST=$(docker ps -a | grep $CONTAINER | tr -s ' ' | cut -d ' ' -f 1)

# 判断容器是否提动,启动了就停掉并删除
if [[ -n $IFCONTAINEREXIST ]]
then
# 停止容器,然后删除容器
	docker stop $CONTAINER && docker rm $CONTAINER
fi

# 判断是否传入的镜像是否存在,如果为不存在则不处理,存在则删除镜像
if [[ -n $IFIMAGEEXIST ]]
then
	docker rmi -f "$REPOSITORY:$TAG"
fi
# 执行dockerfile创建镜像
docker build -t ck.czy/apptest:latest .
# 推送镜像到仓库
docker push ck.czy/apptest:latest


# 启动容器
echo "docker run --name=$CONTAINER -d -p 10086:8080 $REPOSITORY:$TAG" | bash -s
# docker run --name=$CONTAINER -d -p "10086:8080" "$REPOSITORY:$TAG"
```

下面这是Dockerfile的写法,由于这个项目很简单,只需要依赖一个jdk,一个tomcat就能跑起来,所以我之前做了一个已经有jdk和tomcat的基础镜像,我基于该基础镜像直接生成一个新的镜像,只需要将我的项目代码放到tomcat的webapps路径下就行了

```txt
# 依赖的基础镜像
FROM ck.czy/tomcat:clean

# 上传代码到tomcat的webapps下面
# ADD对某几个类型的压缩文件自动解压,COPY则单纯复制,不会解压
ADD APPTEST.tar.gz /usr/local/tomcat/webapps/APPTEST
```

**差不多结束了**

至此一个提交代码自动发布的流程就结束了,但还有些需要思考的问题,可以大家一起探讨下

# 问题

1. 多开发共同编码时,如果不想自动发布,需要由测试人员控制发布秩序,版本控制应该加在哪个环节,该怎么加入版本控制
2. 其他问题,我暂时还没有想到的
