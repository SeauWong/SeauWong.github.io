---
layout: post
title: "Shutdown a Spring Boot Application"
subtitle: 'Shutdown a Spring Boot Application'
author: "WongCU"
header-style: text
tags:
  - SpringBoot
---

> by [baeldung](https://www.baeldung.com/spring-boot-shutdown)

**we'll have a look at different ways to shut down a Spring Boot Application.**

## 1. Shutdown Endpoint

By default, all the endpoints are enabled in Spring Boot Application except */shutdown*; this is, naturally, part of the *Actuator* endpoints.

Here's the Maven dependency to set up these up:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Lastly, we enable the shutdown endpoint in *application.properties* file:

```yml
management:
  endpoints:
    web.exposure.include: *
    shutdown.enabled: true
endpoints.shutdown.enabled: true
```

Note that we also have to expose any actuator endpoints that we want to use. In the example above, we've exposed all the actuator endpoints which will include the */shutdown* endpoint.

**To shut down the Spring Boot application, we simply call a POST method like this**:

```http
curl -X POST localhost:port/actuator/shutdown
```

In this call, the *port* represents the actuator port.



## 2. Close Application Context

We can also call the *close()* method directly using the application context.

Let's start with an example of creating a context and closing it:

```java
ConfigurableApplicationContext ctx = new
  SpringApplicationBuilder(Application.class).web(WebApplicationType.NONE).run();
System.out.println("Spring Boot application started");
ctx.getBean(TerminateBean.class);
ctx.close();
```

**This destroys all the beans, releases the locks, then closes the bean factory**. To verify the application shutdown, we use the Spring's standard lifecycle callback with *@PreDestroy* annotation:

```java
public class TerminateBean {
 
    @PreDestroy
    public void onDestroy() throws Exception {
        System.out.println("Spring Container is destroyed!");
    }
}
```

We also have to add a bean of this type:



```java
@Configuration
public class ShutdownConfig {
 
    @Bean
    public TerminateBean getTerminateBean() {
        return new TerminateBean();
    }
}
```

Here's the output after running this example:

```text
Spring Boot application started
Closing AnnotationConfigApplicationContext@39b43d60
DefaultLifecycleProcessor - Stopping beans in phase 0
Unregistering JMX-exposed beans on shutdown
Spring Container is destroyed!
```

The important thing here to keep in mind: **while closing the application context, the parent context isn't affected due to separate lifecycles**.



### 2.1. Close the Current Application Context

In the example above, we created a child application context, then used the *close()* method to destroy it.

If we want to close the current context, one solution is to simply call the actuator */shutdown* endpoint.

However, we can also create our own custom endpoint:

```java
@RestController
public class ShutdownController implements ApplicationContextAware {
     
    private ApplicationContext context;
     
    @PostMapping("/shutdownContext")
    public void shutdownContext() {
        ((ConfigurableApplicationContext) context).close();
    }
 
    @Override
    public void setApplicationContext(ApplicationContext ctx) throws BeansException {
        this.context = ctx;
         
    }
}
```

Here, we've added a controller that implements the *ApplicationContextAware* interface and overrides the setter method to obtain the current application context. Then, in a mapping method, we're simply calling the *close()*method.

We can then call our new endpoint to shut down the current context:

```http
curl -X POST localhost:port /shutdownContext
```

**Of course, if you add an endpoint like this in a real-life application, you'll want to secure it as well.**



## 3. Exit SpringApplication

*SpringApplication* registers a *shutdown* hook with the JVM to make sure the application exits appropriately.

Beans may implement the *ExitCodeGenerator* interface to return a specific error code:

```java
ConfigurableApplicationContext ctx = new SpringApplicationBuilder(Application.class)
  .web(WebApplicationType.NONE).run();
 
int exitCode = SpringApplication.exit(ctx, new ExitCodeGenerator() {
@Override
public int getExitCode() {
        // return the error code
        return 0;
    }
});
 
System.exit(exitCode);
```

The same code with the application of Java 8 lambdas:

```java
SpringApplication.exit(ctx, () -> 0);
```

**After calling the System.exit(exitCode), the program terminates with a 0 return code**:

```
Process finished with exit code 0
```



## 4. Kill the App Process

Finally, we can also shut down a Spring Boot Application from outside the application by using a bash script. Our first step for this option is to have the application context write it's PID into a file:

```java
SpringApplicationBuilder app = new SpringApplicationBuilder(Application.class)
  .web(WebApplicationType.NONE);
app.build().addListeners(new ApplicationPidFileWriter("./bin/shutdown.pid"));
app.run();
```

Next, create a *shutdown.bat* file with the following content:

```bash
kill $(cat ./bin/shutdown.pid)
```

The execution of *shutdown.bat* extracts the Process ID from the *shutdown.pid* file and uses the *kill* command to terminate the Boot application.



## **5. Conclusion**

In this quick write-up, we've covered few simple methods that can be used to shut down a running Spring Boot Application.

While it's up to the developer to choose an appropriate a method; all of these methods should be used by design and on purpose.

For example, *.exit()* is preferred when we need to pass an error code to another environment, say JVM for further actions. Using **Application PID gives more flexibility, as we can also start or restart the application** with the use of bash script.

Finally, **/shutdown** is here to make it possible to **terminate the applications externally via HTTP**. For all the other cases *.close()* will work perfectly.

As usual, the complete code for this article is available over on the [GitHub project](https://github.com/eugenp/tutorials/tree/master/spring-boot-ops).