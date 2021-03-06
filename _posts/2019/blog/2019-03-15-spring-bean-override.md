---
title:  "Spring Boot: Bean definition overriding"
excerpt: In this article I am going to talk about the tricky spring boot bean definition overriding mechanism.
date:   2019-08-01 00:00:00 +0100
categories: blog
toc: true
toc_label: On this pages
toc_sticky: true

header:
  teaser: assets/images/2019/blog/spring-bean-override/spring-bean-override.jpg
  overlay_image: assets/images/2019/blog/spring-bean-override/spring-bean-override.jpg

---

## Intro

Just to give a bit more clarity about the topic, let's start with a small quiz.
Please take a look at the next simple example.

So we have 2 configurations, which instantiating bean with the name **beanName** and in the main application we just print the value of this bean (very important that both of them have same name).

What do you think is going to be printed?

## Example 1
```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        ApplicationContext applicationContext = SpringApplication.run(Application.class, args);
        System.out.println(applicationContext.getBean("beanName"));
    }
}

@Configuration
class config1 {

    @Primary
    @Order(Ordered.HIGHEST_PRECEDENCE)
    @Bean
    String beanName() {
        return "BEAN1";
    }
}

@Configuration
class config2 {

    @Bean
    String beanName() {
        return "BEAN2";
    }
}
```

### Possible answers:
1. '_BEAN1_' is printed. Probably because it has @Primary annotation and even @Order
2. '_BEAN2_' is printed.
3. _Exception_ is thrown, because it's not allowed to have several beans with same name.
4. Any other version?



### Correct answer
The curious thing here, is that the correct answer will be different for the `spring boot 1.*` and `spring boot 2.*` versions.

If you run this code using `spring boot 1` - **"BEAN2"** will be printed in the console.
With `spring boot 2` - `exception` will be thrown.
Did you know correct the answer? If yes - probably you are working in Pivotal :)

Let's go one by one:
For `spring boot 1`. If we check the logs, we will find the next line there:

```properties
INFO --- [main] o.s.b.f.s.DefaultListableBeanFactory:
Overriding bean definition for bean 'beanName' with a different definition:
replacing [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=true; factoryBeanName=config1; factoryMethodName=beanName; initMethodName=null; destroyMethodName=(inferred);
defined in class path resource [com/example/test/config1.class]] with [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=config2; factoryMethodName=beanName; initMethodName=null; destroyMethodName=(inferred);
defined in class path resource [com/example/test/config2.class]]
```
So `config1` bean was overridden by `config2`  and **"BEAN2"** is printed.

For `spring boot 2`. If we check the logs, we will find the next line there:

```properties
***************************
APPLICATION FAILED TO START
***************************
Description:
The bean 'beanName', defined in class path resource [com/example/test/config2.class],
could not be registered. A bean with that name has
already been defined in class path resource [com/example/test/config1.class]
and overriding is disabled.

Action:
Consider renaming one of the beans or enabling overriding
by setting spring.main.allow-bean-definition-overriding=true
```

So, in `spring boot 2` the default behaviour was changed and bean overriding is not a valid case anymore. And if we want to fix this and make it similar to `spring boot 1` we should add the next configuration:
`spring.main.allow-bean-definition-overriding=true`

From now they are working in the same way.

But this is not the end. Let's check Example 2:


## Example 2
```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        ApplicationContext applicationContext = SpringApplication.run(Application.class, args);
        System.out.println(applicationContext.getBean("beanName"));
    }
}

@Configuration
class config1 {

    @Bean
    String beanName() {
        return "BEAN1";
    }
}

@Configuration
class a_config2 {

    @Bean
    String beanName() {
        return "BEAN2";
    }
}
```

So it's totally the same, **BUT** the name of the second configuration class is different: now it's `a_config2`, but could be let's say `config0` as well.

And now if we run this code the result will be **BEAN1**


## So how is it possible? Answers.

1. Spring is completely ignoring any additional annotations for beans with the same names, like `@Primary` and `@Order`. They don't change anything in this case.
2. Spring is processing @Configurations in an unpredictable way. In our **Example 2** it's ordering config classes **BY NAME**, so based on that one can override another, which we saw in **Example 1** and **Example 2**.
3. In a more complicated application it's possible to have additional configurations in `xml loaded with @Import(Configuration.class)/groovy/whatever`. And in this case the behaviour will be different again. I have no clue as to which one will be loaded latest and override the previous one. And I didn't find any strong  explanations in Spring docs about this.

What I found, that is `@Import` is always loaded first and `XML` configuration always last, so it will override everything else. And names don't matter in this case.

Please check the following example:

```java
@SpringBootApplication
@ImportResource("classpath:config.xml")
@Import(Config0.class)
public class Application {
    public static void main(String[] args) {
        ApplicationContext applicationContext = SpringApplication.run(Application.class, args);
        System.out.println(applicationContext.getBean("beanName"));
    }
}

@Configuration
class config1 {
    @Bean
    String beanName() {
        return "BEAN1";
    }
}

@Configuration
class config2 {
    @Bean
    String beanName() {
        return "BEAN2";
    }
}

//separate java config which is loaded by @Import
@Configuration
class Config0 {
    @Bean
    String beanName() {
        return "BEAN0";
    }
}

//separate xml config which is loaded by @ImportResource
<?xml version = "1.0" encoding = "UTF-8"?>
<beans xmlns = "http://www.springframework.org/schema/beans"
       xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation = "http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">

    <bean id = "beanName"  class = "java.lang.String">
        <constructor-arg value="XML_BEAN"></constructor-arg>
    </bean>

</beans>
```

The output here will be: **"XML_BEAN"**

So, it almost impossible to predict which bean will override another, especially if you have a complex context with a lot of different configurations inside and it's really confusing.


## Summary
As far as you can see from this example this behaviour is totally unpredictable and it's super easy to make a mistake here.
And there is only one rule here that I can see:

> **Bean with the same name as another one, which is processed later, overrides the older one, but it's not clear at all which one will be processed later.**


The mechanism which caused us so much confusion is called **bean overriding**. It is used when Spring encounters a declaration of a bean with the same name as another bean already existing in the context.


Here is a real-life example of this problem. We had a custom configuration for spring RestTemplate. And the name was just **restTemplate**.
A bit later we got one additional **restTemplate** with exactly the same name from the configuration from the external dependency.
And of course, it happened that external **restTemplate** override our own with our custom "tuning".

After the investigation I found out how spring manages such situations.

## Solution
1. First of all, I strongly suggest enabling this configuration:
`spring.main.allow-bean-definition-overriding=false`
It will immediately give you the information that you have beans with the same names and conflicts between them.

2. If this code is yours and in any way it's possible to change the names of the beans - just do it and Inject that one you need. And you will never face such an issue.

3. If for some reason point 2 is not the case for you - I suggest trying to exclude the wrong bean.
As you can see, it's very difficult to predict which bean will be overridden, so it would be much better to simply remove it from the context.

Here is an example:

```java
@SpringBootApplication
@ComponentScan(excludeFilters = @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, classes = config2.class))
public class Application {

    public static void main(String[] args) {
        ApplicationContext applicationContext = SpringApplication.run(Application.class, args);
        System.out.println(applicationContext.getBean("beanName"));
    }
}

@Configuration
class config1 {

    @Bean
    String beanName() {
        return "BEAN1";
    }
}

@Configuration
class config2 {

    @Bean
    String beanName() {
        return "BEAN2";
    }
}
```
In this case config2.class will not be scanned, so we will have only one **beanName** and the result will be **"BEAN1"** here.



P.S. If you have found any gaps or have anything to add or discuss - please feel free to put your thoughts in the comments.
