[TOC]



## 问题

### 问题一、SpringBoot2.x关于启动应用自动加载schema-all.sql文件的改动
SpringBoot2.x在自动加载类路径下的schema-all.sql文件的时候，自动加载不成功
**解决**：
1、数据库驱动需要更改
`
driver-class-name: com.mysql.cj.jdbc.Driver
`
2、需要在配置文件设置`initialization-mode: always`
完成上述两步，即可成功加载schema-all.sql
3、可以在application.yml属性文件中指定加载的sql文件位置：

```yml
spring:  
    datasource:    
        username: root    
        password: 123456    
        url: jdbc:mysql://192.168.0.105:3306/jdbc    
        driver-class-name: com.mysql.cj.jdbc.Driver    
        initialization-mode: always    
        schema: classpath:schema-all.sql
```