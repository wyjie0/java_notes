[TOC]

## 一、Maven依赖中的scope

### scope的分类
#### compile
默认就是compile，什么都不配置也就是意味着compile。compile表示被依赖项目需要参与当前项目的编译、测试和运行。**打包的时候被依赖的项目需要包含进去**
#### test
scope为test表示以来项目仅仅参与测试相关的工作，包括测试代码的编译、执行。比如junit
#### runtime
表示被依赖项目无需参与项目的编译，但是需要参与运行与测试阶段。
#### provided
表示**被依赖项目在项目打包的时候不用包含进去**，别的设施会提供。该依赖理论上可以参与编译，测试，运行等周期，相当于compile，但是打包阶段做了exclude的动作
#### system
从参与度来说，与provided相同，不过被依赖项不会从maven仓库抓，而是从本地文件系统拿，一定需要配合systemPath属性使用
### scope的依赖传递
当前项目A，A依赖于B，B依赖于C。知道B在A项目中的scope，那么：
    * 当C是test或者provided时，C直接被丢弃，A不依赖C
        * 否则A依赖C，C的scope继承于B的scope

## 二、Maven构建过程中的各个环节

* 清理：将以前编译得到的class字节码文件删除，为下一次编译做准备
* 编译：将Java源程序编译成class字节码文件
* 测试：自动测试，自动调用Junit程序
* 报告：测试程序执行的结果
* 打包：动态web工程打的war包，Java工程打jar包
* 安装：将打包得到的文件复制到“仓库”的指定位置
* 部署：将动态Web工程生成的war包复制到Servlet容器的指定目录下，使其可以运行

## 三、Maven的核心概念

### 1、约定的目录结构

```shell
root
	- src
		-- main
			--- java
			--- resources
		-- test
			--- java
			--- resources
	- pom.xml
```

1. 根目录：工程名
2. src目录：源码
3. pom.xml文件：Maven工程的核心配置文件
4. main目录：存放主程序
5. test目录：存放测试程序
6. java目录： 存放java源文件
7. resources目录：存放框架或其他工具的配置文件

* 最好修改Maven配置文件中的本地仓库的位置，默认位置在C盘的家目录下的.m2目录下。更改的位置为：

  ```xml
    <localRepository>E:\JavaKits\apache-maven-jars</localRepository>
  ```


### 2、POM

Project Object Model，项目对象模型，是Maven工程的基本工作单元，是一个XML文件，包含了项目的基本信息，用于描述项目如何构建，声明项目依赖等等。

执行任务或目标时，Maven会在当前目录中查找POM，它读取POM，获取所需的配置信息，然后执行目标。

### 3、坐标

```xml
<!-- 公司或者组织的唯一标志，并且配置时生成的路径也是由此生成， 如com.companyname.project-group，maven会将该项目打成的jar包放本地路径：/com/companyname/project-group -->
<groupId>com.companyname.project-group</groupId>

<!-- 项目的唯一ID，一个groupId下面可能多个项目，就是靠artifactId来区分的 -->
<artifactId>project</artifactId>

<!-- 版本号 -->
<version>1.0</version>
```

### 4、依赖

### 5、仓库

1. 仓库的分类
   1. 本地仓库
   2. 远程仓库
      1. 私服：搭建在局域网环境中，为局域网范围内的所有Maven工程服务
      2. 中央仓库
      3. 中央仓库镜像：分担中央仓库的流量，提升用户访问速度
2. 仓库中保存的内容：Maven工程
   1. Maven自身所需要的插件
   2. 第三方框架或工具的jar包
   3. 我们自己开发的Maven工程

### 6、生命周期/插件/目标

声明周期各个构建环节执行的顺序，声明周期的各个阶段仅仅定义了要执行的任务是什么，而各个阶段和插件的目标是对应的，相似的目标又特定的插件来完成。

### 7、继承



### 8、聚合

