# Creating Cloud Foundry Spring application with MongoDB support
> 23/11/2011

Cloud Foundry is a new open source platform as a service from VMWare. It is currently in beta, but you can [request invitation](http://cloudfoundry.com/signup) to participate in testing. Cloud foundry allows deploying Spring, Rails, Scala and Node.js applications, while available application services are MongoDB, MySQL, PostgreSQL, RabbitMQ and Redis. There is also [Cloud Foundry Micro](http://www.cloudfoundry.com/micro), which is basically a Virtual machine image of Cloud Foundry, so that you can develop and test Cloud Foundry applications on your computer before you put them to the cloud. I haven't tested Cloud Foundry Micro yet, but I plan to do so, soon.

Recently I received an email from Cloud Foundry accepting my request to participate in beta testing of their service. They sent me credentials and basic instructions how to log in and start using Cloud Foundry, so I decided to build simple Spring application to try it out and share my experiences here. As a backing database I will use MongoDB, accessed through MongoTemplate from [Spring data](http://www.springsource.org/spring-data/mongodb) project.

### Getting our hands dirty

So, let's first create a new web application from maven archetype:

```
mvn archetype:generate -DgroupId=eu.semberal.example -DartifactId=cloudfoundry-mangodb-app
    -DarchetypeGroupId=org.codehaus.mojo.archetypes
    -DarchetypeArtifactId=webapp-jee5 -DarchetypeVersion=1.3
```


Now let's add a new repository containing Spring framework milestone artifacts to our pom.xml:

```xml
<repository>
    <id>spring-milestone-repo</id>
    <name>SpringSource milestone repository</name>
    <url>http://maven.springframework.org/milestone</url>
</repository>
```

and dependencies we will need:

```xml
<dependency>
    <groupId>org.mongodb</groupId>
    <artifactId>mongo-java-driver</artifactId>
    <version>2.7.1</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>3.1.0.RC1</version>
</dependency>
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-mongodb</artifactId>
    <version>1.0.0.M5</version>
</dependency>
<dependency> <!-- cloud foundry runtime (necessary for the cloud namespace to work correctly) -->
    <groupId>org.cloudfoundry</groupId>
    <artifactId>cloudfoundry-runtime</artifactId>
    <version>0.8.1</version>
</dependency>
```

In application context we will use component scan feature to discover our beans and, therefore, in our applicationContext.xml there will be just the definition of [MongoTemplate](http://static.springsource.org/spring-data/data-document/docs/current/api/org/springframework/data/mongodb/core/MongoTemplate.html), which will be autowired into our dao classes and used for communication with underlying MongoDB service. Spring application context configuration follows:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:context="http://www.springframework.org/schema/context"
        xmlns:cloud="http://schema.cloudfoundry.org/spring"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.1.xsd
        http://schema.cloudfoundry.org/spring http://schema.cloudfoundry.org/spring/cloudfoundry-spring-0.8.xsd">

    <context:component-scan base-package="eu.semberal.cloudfoundrysample.dao"/>
    <context:component-scan base-package="eu.semberal.cloudfoundrysample.service"/>
    <context:annotation-config/>

    <bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
        <constructor-arg name="mongoDbFactory" ref="mongoDbFactory" />
    </bean>

    <cloud:mongo-db-factory id="mongoDbFactory"/>
</beans>
```

You can see the `cloud` namespace here. By default, parameters for configuring MongoDB are passed in Cloud Foundry via environment variables. `Cloud` namespace simplifies the configuration by autodetecting MongoDB service and autoconfiguring [MongoDbFactory](http://static.springsource.org/spring-data/data-document/docs/current/api/org/springframework/data/mongodb/MongoDbFactory.html) which is passed to the constuctor of MongoTemplate.

Now it's time to create service and dao beans, controllers,  entities, configure Spring MVC and web.xml properly, create JSP views, etc. <strike>I won't put all the code here, but you can find everything in zipped sources attached to this post</strike>.

### Putting it up there

So, when our application is ready, we can deploy it to Cloud Foundry. If you are a SpringSource Tool Suite user, you can deploy applications right from your IDE. Those who are not STS users (including me) must settle for command-line interface called `vmc`. If you have ruby in your classpath, you can install `vmc` with command:

```
gem install vmc
```

After successful installation of vmc gem, we can connect to Cloud Foundry and authenticate with e-mail and password from the e-mail:

```
vmc target api.cloudfoundry.com
vmc login
```

After successful login we can start deploying the application. So, let's move to the directory with the project and type:

```
vmc push
```

During the deployment it will ask for couple of things, such as name of the application, URL, memory allocation, requested services etc. It is a pretty straightforward configuration process and you shouldn't run into any problems. At the end you should see "Starting application: OK". If the application didn't start correctly, you will have to check logs to find out what happened and why the deployment failed. To get access to application logs, use command (`vmc logs <appname>`).

When I first deployed my application to Cloud Foundry and tried to access it, I was getting an error: "VCAP ROUTER: 404 - DESTINATION NOT FOUND", even though `vmc` confirmed successful startup with: "Starting Application: OK". After checking crash logs (`vmc crashlogs <appname>`), I found out that it ran out of memory (since I was greedy and allocated only 128M out of 2GB quota). After raising the amount of memory to 256MB (`mem <appname> 256M`), it started working.

### Conclusion

<strike>You can see the application running at [http://mongodb-cloudfoundry-example.cloudfoundry.com/](http://mongodb-cloudfoundry-example.cloudfoundry.com/) where you might add yourself to the list.</strike> Don't get scared of the look of the application. You can guess I'm neither talented web designer nor was it important to show you how Cloud Foundry works.

I hope this article was helpful to everyone beginning with Cloud Foundry. If you couldn't get my sample application working or had any other questions or comments, don't hesitate to ask in the discussion.

Source code: [GitHub](https://github.com/semberal/blog-examples/tree/master/configuring-cloud-foundry-application-with-mongodb)
