[TOC]



### 一、forward 和 redirect的区别

1、从地址栏显示来说：
1）forward是**服务器内部的重定向**，服务器直接访问目标地址的url网址，把里面的东西读取出来，但是客户端并不知道，因此用forward，**客户端浏览器的网址是不会发生变化的**
2）redirect是服务器根据逻辑，发送一个状态码，告诉浏览器冲新区请求那个地址，所以地址栏显示的是新的地址
2、从数据共享来说：
1）由于在**整个定向的过程中用的是同一个request**，因此**forward会将request的信息带到被重定向的jsp或者servlet中使用，即可以共享数据**
2）redirect不能共享数据
3、从运用的地方来说
1）forward一般用于用户登录的时候，根据角色转发到相应的模块
2）redirect一般用于用户注销登录时返回主页面或者跳转到其他网站
4、从效率来说
1）forward效率高，direct效率低
5、从请求的次数来说：
forward只有一次请求，redirect有两次请求
### 二、Http中各种状态码的含义
#### 2XX 成功
* 200 OK ： 表示从客户端发来的请求在服务器端被正确处理
* 204 No Content ： 表示请求成功，但相应报文不含实体的主体部分
* 206 Partial Content ： 进行范围请求

#### 3XX 重定向
* 301 moved permanently 永久性重定向，表示资源已经被分配了新的URL
* 302 found 临时性重定向，表示资源临时被分配了新的URL
* 303 see other 表示资源存在着另一个URL，应使用GET方法来获取资源


#### 4XX 客户端错误
* 400 bad request ： 表示请求报文存在语法错误

### 三、Servlet声明周期
init()、service()和destroy()方式是与servlet的生命周期相关的方法。
* 当**实例化某个Servlet后**，servlet容器会调用其init()方法进行初始化。servlet容器只会调用该方法一次，调用后则可以执行服务方法了。在servlet接收任何请求之前，必须是经过正确初始化的。servlet程序员可以覆盖此方法，在其中编写仅需要执行一次的初始化方法，例如载入数据库驱动程序、初始化默认值等。一般情况下，init()方法可以为空。
* 当servlet的一个客户端请求到达后，servlet容器就调用相应的servlet的service()方法，并将ServletRequest对象和ServletResponse对象作为参数传入。**ServletRequest对象包含客户端的Http请求的信息，ServletResponse对象封装servlet的响应信息。在servlet对象的整个生命周期内，service()方法会被多次调用**。
* 在**将servlet实例从服务中移除前**， servlet容器会调用servlet实例的destroy()方法。一般当servlet容器关闭或servlet容器需要释放内存时，才会将servlet实例移除，而且**只有当servlet实例的service()方法中的所有线程都退出或执行超时后，才会调用destroy()方法**。当servlet容器调用了某个servlet实例的destroy()方法后，它就不会再调用该servlet实例的service()方法了。调用destroy()方法让servlet对象有机会去清理自身持有的资源，如内存、文件句柄和线程等，确保所有的持久化状态与内存中该servlet对象的当前状态同步。