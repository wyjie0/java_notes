[TOC]



### 常用命令

#### 超级用户

* sudo：允许用户以超级用户或安全策略指定的另一个用户的身份执行命令。sudo支持安全策略插件和输入/输出日志的插件。
#### apt
* apt install：安装软件包
* apt remove：移除软件包
* apt purge：移除软件包及配置文件
* apt update：舒心存储库索引
* apt upgrade：升级所有可升级的软件包
* apt search：搜索应用程序
* apt full-upgrade：升级软件包时自动处理依赖关系
#### 用户组
* groupadd groupname：添加名字为groupname的用户组
* gpasswd -a $USER grouname：将登陆用户（$USER）加入到名字为groupname的用户组中
* newgrp groupname：更新用户组
#### 切换root用户
* sudo su：切换成root
* exit：换成之前的用户