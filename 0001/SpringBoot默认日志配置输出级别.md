Spring Boot默认日志配置输出级别

### Spring Boot默认配置

我们在测试类中来进行演示

```
package com.staticzz.springboot_logging;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbootLoggingApplicationTests {


    /**
     * SLF4J日志记录器
     */
    Logger logger = LoggerFactory.getLogger(getClass());

    @Test
    public void contextLoads() {

        /**
         * 日志级别,由低到高
         * 我们可以调整日志级别,日志就只会在这个级别以后生效
         */
        logger.trace("这是trace日志");
        logger.debug("这是debug信息");
        logger.info("这是info信息");
        /**
         * 运行后,从info信息开始输出,说明Springboot默认是info级别的日志信息
         */
        logger.warn("这是warning信息");
        logger.error("这是error信息");
    }
}
```

动态演示:

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbueWCyeEYVKDOgIjXUKO5YryUn1LLJJOicwxPdSJI8B95JQ0AlfuiavyhdH127mdicNpqQHoEU6leBZBA/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

### 那我们要是需要修改默认的日志级别怎样修改?

可以在SpringBoot的配置文件中写入

```
logging.level.com.staticzz=trace
```

这句代码的意思是我将`com.staticzz`包下的所有类的日志级别都设置为trace级别

动态演示:

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbueWCyeEYVKDOgIjXUKO5Yry1OZB9aibKiaCqggdopqriaKbfJpyJMuxOvTicmItJS89yIk8TLW34HdImQ/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

那我们可以在`application.properties`配置文件中指定`logging.level`,那能不能指定其他参数呢?

答案是可以的

```
#指定某个包下的日志级别
logging.level.com.staticzz=trace
#指定log文件名
#logging.file=SpringBoot.log
#指定log输出路径,如果不指定,默认输出到控制台
logging.path=C:/Users/socra
#指定控制台输出格式
#logging.pattern.console=
#指定文件输出格式
logging.pattern.file=%d{-yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-51level %logger{50} - %msg%n
```

这里要注意一个问题,`logging.file`与`logging.path`不能同时使用,会出现冲突,我们只需要二选一配置就行!

演示:

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbueWCyeEYVKDOgIjXUKO5Yry6vPejyAKMJDOye3G9y3FZbJd7lbDmTW6dsmHtC9YQzt3T42MgsLCAA/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

**那为什么Spring Boot日志默认级别是info级别,这个级别Spring Boot到底在哪里指定的呢?它的日志的默认配置到底是怎么样的呢?到底在默认配置中都配置了哪些东西呢?**

我们都知道Spring Boot使用的是`SLF4J+logback`来记录日志的,接下来我们在依赖的Spring Boot这个包找到这个`logback`这个jar包,来查看一下到底默认配置都配了什么东西?

![图片](https://mmbiz.qpic.cn/mmbiz_gif/eQPyBffYbueWCyeEYVKDOgIjXUKO5Yry5syQsR6u0IXPHQelddneCRndW3Sd8ts68cQLuxUPMzT0Utbl6Qguww/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

刚刚讲到`Spring Boot`中的日志级别默认是INFO级别,我们通过以上演示就可以看到，`Spring Boot`中的日志配置主要使用到了以下几个文件,这里我一一列举下:

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueWCyeEYVKDOgIjXUKO5YryATNBwQVynSN5xic5cr5UZ4ASUzeK7ABkD92BVjUQzc7uN98EPlqw8icA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- base.xml
- defaults.xml
- console-appender.xml
- file-appender.xml
- LoggingSystemProperties

这里我把这些文件内容全部列举出来,并说明它们的作用!

base.xml

```
<?xml version="1.0" encoding="UTF-8"?>

<!--
Base logback configuration provided for compatibility with Spring Boot 1.1
-->

<included>
    <!--引入了默认配置的xml文件-->
   <include resource="org/springframework/boot/logging/logback/defaults.xml" />
   <property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}}/spring.log}"/>
    
    <!--引入了控制台输出格式的xml文件-->
    <include resource="org/springframework/boot/logging/logback/console-appender.xml" />
    
    <!--引入了文件输出格式的xml文件-->
    <include resource="org/springframework/boot/logging/logback/file-appender.xml" />
   
    <!-- 这里就是Springboot日志默认级别的配置-->
    <root level="INFO"> 
      <appender-ref ref="CONSOLE" />
      <appender-ref ref="FILE" />
   </root>
</included>
```

`defaults.xml`

```
<?xml version="1.0" encoding="UTF-8"?>

<!--
Default logback configuration provided for import, equivalent to the programmatic
initialization performed by Boot
-->

<included>
   <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
   <conversionRule conversionWord="wex" converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter" />
   <conversionRule conversionWord="wEx" converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter" />
    
    <!--控制默认输出的格式->
   <property name="CONSOLE_LOG_PATTERN" value="${CONSOLE_LOG_PATTERN:-%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
    
    <!--文件默认输出的格式->
  <property name="FILE_LOG_PATTERN" value="${FILE_LOG_PATTERN:-%d{yyyy-MM-dd HH:mm:ss.SSS} ${LOG_LEVEL_PATTERN:-%5p} ${PID:- } --- [%t] %-40.40logger{39} : %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>

   <appender name="DEBUG_LEVEL_REMAPPER" class="org.springframework.boot.logging.logback.LevelRemappingAppender">
      <destinationLogger>org.springframework.boot</destinationLogger>
   </appender>
   <!--默认规定好的级别->
   <logger name="org.apache.catalina.startup.DigesterFactory" level="ERROR"/>
   <logger name="org.apache.catalina.util.LifecycleBase" level="ERROR"/>
   <logger name="org.apache.coyote.http11.Http11NioProtocol" level="WARN"/>
   <logger name="org.apache.sshd.common.util.SecurityUtils" level="WARN"/>
   <logger name="org.apache.tomcat.util.net.NioSelectorPool" level="WARN"/>
   <logger name="org.crsh.plugin" level="WARN"/>
   <logger name="org.crsh.ssh" level="WARN"/>
   <logger name="org.eclipse.jetty.util.component.AbstractLifeCycle" level="ERROR"/>
   <logger name="org.hibernate.validator.internal.util.Version" level="WARN"/>
   <logger name="org.springframework.boot.actuate.autoconfigure.CrshAutoConfiguration" level="WARN"/>
   <logger name="org.springframework.boot.actuate.endpoint.jmx" additivity="false">
      <appender-ref ref="DEBUG_LEVEL_REMAPPER"/>
   </logger>
   <logger name="org.thymeleaf" additivity="false">
      <appender-ref ref="DEBUG_LEVEL_REMAPPER"/>
   </logger>
</included>
```

那这里有一个问题?

`defaults.xml`文件中默认控制台,默认文件输出的格式都在代码中引用了相关的环境变量,那这些环境变量在哪呢?

我们在`Spring Boot`中也修改过这些配置,

```
${CONSOLE_LOG_PATTERN:-%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>
```

答案：在以下这个类中

`LoggingSystemProperties.java`

```
/*
 * Copyright 2012-2016 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

package org.springframework.boot.logging;

import org.springframework.boot.ApplicationPid;
import org.springframework.boot.bind.RelaxedPropertyResolver;
import org.springframework.core.env.Environment;

/**
 * Utility to set system properties that can later be used by log configuration files.
 *
 * @author Andy Wilkinson
 * @author Phillip Webb
 */
class LoggingSystemProperties {

   static final String PID_KEY = LoggingApplicationListener.PID_KEY;

   static final String EXCEPTION_CONVERSION_WORD =          LoggingApplicationListener.EXCEPTION_CONVERSION_WORD;

   static final String CONSOLE_LOG_PATTERN = LoggingApplicationListener.CONSOLE_LOG_PATTERN;

   static final String FILE_LOG_PATTERN = LoggingApplicationListener.FILE_LOG_PATTERN;

   static final String LOG_LEVEL_PATTERN = LoggingApplicationListener.LOG_LEVEL_PATTERN;

   private final Environment environment;

   LoggingSystemProperties(Environment environment) {
      this.environment = environment;
   }

   public void apply() {
      apply(null);
   }

   public void apply(LogFile logFile) {
      RelaxedPropertyResolver propertyResolver = RelaxedPropertyResolver
            .ignoringUnresolvableNestedPlaceholders(this.environment, "logging.");
      setSystemProperty(propertyResolver, EXCEPTION_CONVERSION_WORD,
            "exception-conversion-word");
      setSystemProperty(PID_KEY, new ApplicationPid().toString());
      setSystemProperty(propertyResolver, CONSOLE_LOG_PATTERN, "pattern.console");
      setSystemProperty(propertyResolver, FILE_LOG_PATTERN, "pattern.file");
      setSystemProperty(propertyResolver, LOG_LEVEL_PATTERN, "pattern.level");
      if (logFile != null) {
         logFile.applyToSystemProperties();
      }
   }

   private void setSystemProperty(RelaxedPropertyResolver propertyResolver,
         String systemPropertyName, String propertyName) {
      setSystemProperty(systemPropertyName, propertyResolver.getProperty(propertyName));
   }

   private void setSystemProperty(String name, String value) {
      if (System.getProperty(name) == null && value != null) {
         System.setProperty(name, value);
      }
   }

}
```

关于控制台输出的格式 `console-appender.xml`

```
<?xml version="1.0" encoding="UTF-8"?>

<!--
Console appender logback configuration provided for import, equivalent to the programmatic
initialization performed by Boot
-->

<included>
   <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
      <encoder>
         <pattern>${CONSOLE_LOG_PATTERN}</pattern>
         <charset>utf8</charset>
      </encoder>
   </appender>
</included>
```

关于日志文件输出的格式` file-appender.xml`

```
<?xml version="1.0" encoding="UTF-8"?>

<!--
File appender logback configuration provided for import, equivalent to the programmatic
initialization performed by Boot
-->

<included>
   <appender name="FILE"
      class="ch.qos.logback.core.rolling.RollingFileAppender">
      <encoder>
         <pattern>${FILE_LOG_PATTERN}</pattern>
      </encoder>
      <file>${LOG_FILE}</file>
      <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
         <fileNamePattern>${LOG_FILE}.%i</fileNamePattern>
      </rollingPolicy>
      <triggeringPolicy
         class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
         <MaxFileSize>10MB</MaxFileSize>
      </triggeringPolicy>
   </appender>
</included>
```

**那我们已经看见了SpringBoot日志的默认配置文件,如果去自定义日志的配置文件呢?**

方法是:

给类路径下，放上每个日志框架的配置文件即可

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueWCyeEYVKDOgIjXUKO5Yry3ja7IntdFE02X9cd4vQphFbgU9dwdQC3RHwYUTZnDg6j4XuoxGrCzA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

官方推荐使用`logback-spring.xml`这种格式：

![图片](https://mmbiz.qpic.cn/mmbiz_png/eQPyBffYbueWCyeEYVKDOgIjXUKO5YryKHhBADoJ4PUgEm0ibyp4xh3M86hNpZKXbq94NiaU8tXtuTxRUHOCkxDA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当我们使用

`logback.xml`时,日志框架直接加载该配置文件,就不能使用高级功能

`logback-spring.xml`时,由`Spring Boot`加载该配置文件,就可以使用高级功能。

比如`spring.profile `,这个属性咱们之前用过,为Spring底层的多环境支持,那我们就可以指定某个日志的配置文件只在某个环境下生效

```
<springProfile name="profile属性">
 <!-- configuration to be enabled when the "staging" profile is active -->
    指定日志的某个配置只在设置的环境下生效
</springProfile>
```

asdasd