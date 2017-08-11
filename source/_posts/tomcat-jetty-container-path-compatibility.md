title: tomcat/jetty容器之间的路径兼容性问题
date: 2015-04-10

tags:
 - tomcat
 - jetty
categories:
 - 问题解析 

---

在项目中使用springmvc框架时，在controller方法中返回的view路径字符串最后和xml文件配置中的配置路径进行整合，从而形成一个完成的视图文件路径，然后在tomcat和jetty身上两者之间的差异出现问题：

```
<bean id="internalViewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver"
        p:viewClass="org.springframework.web.servlet.view.JstlView"
        p:prefix="/WEB-INF/view/jsp"
        p:suffix=".jsp"
        p:order="1"/>
```

tomcat中view文件/WEB-INF/view/jsp//default/ui/user/update_passwd.jsp是可以找到的

而在jetty中view文件/WEB-INF/view/jsp//default/ui/user/update_passwd.jsp是不能找到的，去掉多余的/后方能找到，jetty对于路径的规范更加严格？