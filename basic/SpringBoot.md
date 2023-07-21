#### @SpringBootApplication

核心注解：@EnableAutoConfiguration、@SpringBootConfiguration、@ComponentScan

## Starter原理

作用：整合依赖，防止冲突，方便使用

SprintBoot之前创建Bean：在xml中声明Bean，通过反射创建

![image-20220603210701838](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/image-20220603210701838.png)



#### 属性配置类

定义需要的配置信息和默认配置项，在application.yml中以"zcs.hello"开头的配置都会被注入在这里

```java
@ConfigurationProperties(prefix = "zcs.hello")
@Data
public class HelloProperties {
    private String name = "default";
}
```

#### 业务执行类

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class HelloService {
 
    private HelloProperties helloProperties;
 
    public String hello() {
        return "welcome " + helloProperties.getName() + "！";
    }
 
}
```



#### 自动装配类

```java
@ConditionalOnClass(HelloService.class)
@EnableConfigurationProperties(HelloProperties.class)
@Configuration
public class HelloAutoConfiguration {
 
    @Bean
    @ConditionalOnMissingBean(HelloService.class)
    public HelloService helloStarter(HelloProperties helloProperties) {
        return new HelloService(helloProperties);
    }
 
}
```



在 src/main/resources/META-INF目录下新建 spring.factories 文件，输入以下内容：

```xml
#声明配置类的全路径
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.zcs.HelloAutoConfiguration
```



