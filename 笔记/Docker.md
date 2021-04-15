[TOC]



### 遇到的错误

#### 错误一
**问题描述：**
安装完docker后，执行docker相关的命令，出现

```shell
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.39/images/search?limit=25&term=mysql: dial unix /var/run/docker.sock: connect: permission denied
```

**原因：**
“docker manual”上写着：

```
Manage Docker as a non-root user The docker daemon binds to a Unix socket instead of a TCP port. By default that Unix socket is owned by the user root and other users can only access it using sudo. The docker daemon always runs as the root user. If you don’t want to use sudo when you use the docker command, create a Unix group called docker and add users to it. When the docker daemon starts, it makes the ownership of the Unix socket read/writable by the docker group.
```

大概意思就是：docker进程使用unix socket而不是tcp端口。而默认情况下，unix socket属于root用户，需要root权限才能访问docker命令
**解决方法一**
使用sudo获取管理员权限，运行docker命令
**解决方法二**
docker守护进程启动的时候，会默认赋予名字为docker的用户组读写unix socket的权限，因此只要创建docker用户组，并将当前用户加入到docker用户组中，那么当前用户就有权限访问unix socket了，进而也就可以执行docker相关命令
`
sudo groupadd docker #添加docker用户组 
sudo gpasswd -a $USER docker #将登陆用户加入到docker用户组中
newgrp docker #更新用户组 
docker ps #测试docker命令是否可以使用sudo正常使用
`

#### 错误二
**问题描述**：
    在ubuntu环境下，docker下pull的Tomcat无法访问欢迎页
**解决**
    1、进入docker下tomcat容器中的目录
    `docker exec -it 355d7def0257（容器id） /bin/bash`
    2、查看目录结构，比正常情况多了一个webapps.dist文件夹，欢迎页被放在了此文件夹下
    3、删除webapps文件夹，将webapps.dist改为webapps
    `rmdir webapps; mv webapps.dis webapps`
    4、键盘输入ctrl+q+p对出bash
    5、重新访问即可

#### 命令
##### 镜像操作
**docker search 关键字**：搜索docker仓库，比如docker search mysql
**docker pull 镜像名：tag**：从docker仓库拉取应用，其中tag可选，tag表示标签，多为软件的版本，默认为latest
**docker images**：查看本地的所有镜像
**docker rmi image-id**：删除指定的本地镜像
##### 容器操作

软件镜像----运行镜像----产生一个容器（正在运行的软件）

**docker run --name container_name -d image_name**：运行容器
    -- name：自定义容器名
    -d：后台运行
    image_name：指定镜像模板
    比如：docker run --name myTomcat -d tomcat
    **-p 8080:8080**：将主机端口映射到容器内部的端口
    比如：docker run -d -p 8080:8080 --name mytomcat tomcat

**docker ps**：查看运行中的容器
    加上-a可以查看所有容器
    
**docker stop container_id**：停止tomcat容器

**docker rm container_id**：删除指定容器

**docker start container_id**：启动容器

### 阿里镜像加速器

`https://8pf2fpd3.mirror.aliyuncs.com`