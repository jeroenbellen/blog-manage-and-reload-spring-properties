# Manage and reload spring application properties on the fly

By now, everybody is aware that configurable application properties should reside outside your artifact (war, jar, ear). We don't want to release a new version of your application/service, just to change a config file. Furthermore, we want the production artifact, to be identical to the object of development, test, etc.
  

Luckily there are many options to externalise your configuration, Spring Boot even supports this logic out of the box. 
The majority of those solutions require a restart of your application/service, but today I'll demonstrate how to omit this restriction. 

First, let me show you a way how to externalise your application properties and manage them from git. Git makes sense because it keeps track of any changes. You can see who altered what at any given point in time!

To do this, we create a (micro) service. Let's call it the config-service.

Code is available on [github](https://github.com/jeroenbellen/blog-manage-and-reload-spring-properties).

Add the following dependencies to your config-service.

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
            <version>1.1.3.RELEASE</version>
        </dependency>
    </dependencies>
```

Add the @EnableConfigServer annotation to your Spring Boot entry point. 
This annotation will take care of all the configuration of the config server.
Clean and straightforward!

```java
package com.github.jeroenbellen;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
public class ConfigServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServiceApplication.class, args);
    }
}
```
Specify the path of your git repo which contains your properties within 'application.properties.'  

```
server.port=8888
spring.cloud.config.server.git.uri=/tmp/example-properties
```

Create a new git project, which contains your properties.


```bash
mkdir /tmp/example-properties
cd /tmp/example-properties
git init
echo foo.bar=Hi! > application.properties
git add application.properties
git commit -m 'Add foo.bar property'
```

That's it; your basic config service is ready!

Now let's create a sample service.

Add the following dependencies to your example service.

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-client</artifactId>
            <version>1.1.3.RELEASE</version>
        </dependency>
```

Define a sample controller which returns the value of our sample property.

```java
package com.github.jeroenbellen;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
public class ExampleServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(ExampleServiceApplication.class, args);
    }

    @RestController
    class ExampleController {

        @Value("${foo.bar}")
        private String value;

        @RequestMapping
        public String sayValue() {
            return value;
        }
    }
}

```

If you start the application, you should see the following messages withing your log file. Clearly, a sign that something cool is happening! ;)
```
2016-08-20 13:38:14.878  INFO 60864 --- [           main] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at: http://localhost:8888
2016-08-20 13:38:15.527  INFO 60864 --- [           main] c.c.c.ConfigServicePropertySourceLocator : Located environment: name=application, profiles=[default], label=master, version=1e4c80569fc05655903c12c6087ffdf797a0c71e
2016-08-20 13:38:15.527  INFO 60864 --- [           main] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource [name='configService', propertySources=[MapPropertySource [name='/tmp/example-properties/application.properties']]]
```

Time to call our service, and see the result.
```
curl -v http://localhost:8080
* Rebuilt URL to: http://localhost:8080/
*   Trying ::1...
* Connected to localhost (::1) port 8080 (#0)
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.43.0
> Accept: */*
> 
< HTTP/1.1 200 
< X-Application-Context: application
< Content-Type: text/plain;charset=UTF-8
< Content-Length: 3
< Date: Sat, 20 Aug 2016 11:41:12 GMT
< 
* Connection #0 to host localhost left intact
Hi!% 
```

Isn't that nice? 
Ok, but how do we change this property on the fly?
In theory, you could refresh the application context, but I wouldn't recommend this. Spring Cloud has provided an annotation to mark a bean as refreshable. By adding spring actuator, we can refresh those beans on the fly.

Add spring-boot-starter-actuator to your example service.
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

Add the @RefreshScope annotation to your bean which contains properties that should be reloadable.

```java
package com.github.jeroenbellen;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
public class ExampleServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(ExampleServiceApplication.class, args);
    }

    @RestController
    @RefreshScope
    class ExampleController {

        @Value("${foo.bar}")
        private String value;

        @RequestMapping
        public String sayValue() {
            return value;
        }
    }
}

```

Now let's test this!

```bash
#within your git folder

echo foo.bar=Change! > application.properties
git add application.properties
git commit -m 'A change'

```

The config service will automatically detect the commit and start serving the new value. However, we want to see the change within our example service. To do this, we have to call the refresh endpoint that spring actuator has added for us.
```bash
curl -X POST http://localhost:8080/refresh
["foo.bar"]%                                    
```

A call to the example service will now contain our new value.

```bash
curl -v http://localhost:8080             
* Rebuilt URL to: http://localhost:8080/
*   Trying ::1...
* Connected to localhost (::1) port 8080 (#0)
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.43.0
> Accept: */*
> 
< HTTP/1.1 200 
< X-Application-Context: application
< Content-Type: text/plain;charset=UTF-8
< Content-Length: 7
< Date: Sat, 20 Aug 2016 11:54:10 GMT
< 
* Connection #0 to host localhost left intact
Change!%  
```

### Conclusion
By adding some simple, yet powerful, annotations, you can easily manage your properties from git. The refresh mechanism makes it easy to apply your property change on the fly.
However, it 'd be good that the config service would push it's new values to its clients. That is possible, but I'll explain this in an upcoming blog post!

### References
[http://projects.spring.io/spring-cloud/spring-cloud.html](http://projects.spring.io/spring-cloud/spring-cloud.html)