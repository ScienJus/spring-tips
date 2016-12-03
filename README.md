# Spring Tips

Spring Tips 是我在写 Spring / Spring Boot 时遇到的一些问题和解决方案，因为一般篇幅不大所以不想写成博客，就稍微整理一些发布到该项目中，欢迎交流。

## Conditional

在写 Spring Boot 的 Auto Configuration 时，经常会需要使用 Conditional 相关的类定义一个 Configuration 的加载条件，以保证这个类在初始化 Bean 时所需要的必要配置或者依赖都存在。

Spring Boot 默认提供了很多常用的 Conditional，像是 `@ConditionalOnProperty`、`@ConditionalOnClass`。但是在实际开发中依旧会遇到一些无法解决的情况，这时候就可以使用一些扩展类完成自定义的实现。

`@Conditional` 注解接收一个 `Condition` 类，使用该类的 `matches` 方法作为加载的前置条件，例如像是下面这样的一个配置：

```
dataSource:
  user:
    url: jdbc:mysql://127.0.0.1:3306/user
    username: xxx
    password: yyy
  product:
    url: jdbc:mysql://127.0.0.1:3306/product
    username: aaa
    password: bbb
```

如果想要判断 `dataSource` 下至少有一个数据源配置才运行 DataSourceAutoConfiguration，但是`@ConditionalOnProperty` 无法却校验一个这样格式的参数是否存在，所以就可以通过下面这种做法：

```
public class DataSorucePropertyCondition extends SpringBootCondition {
  
  @Override
  public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
    RelaxedPropertyResolver resolver = new RelaxedPropertyResolver(context.getEnvironment());
    Map<String, Object> properties = resolver.getSubProperties("dataSource.");
    return new ConditionOutcome(!properties.isEmpty(), "DataSource property is empty");
  }

}

@Configuration
@Conditional(DataSourcePropertyCondition.class)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

  // ... 通过配置注册一些 Bean
}
```

这种做法虽然可以满足要求，但是却并不具有通用性，因为在代码中写死了 `dataSource.` 作为前缀，如果可以将其抽出来，就可以作为一个通用的参数条件判断了。

`Conditional` 除了直接用在 Configuration 上，还可以用在一个自定义的注解上。当用在注解上时，就可以通过 `AnnotatedTypeMetadata` 参数得到注解中的值，所以上面的代码就可以改为：

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
@Documented
@Conditional(MapPropertyCondition.class)
public @interface ConditionalOnMapProperty {

  String prefix();

}

public class MapPropertyCondition extends SpringBootCondition {

  @Override
  public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
    String prefix = attribute(metadata, "prefix");
    RelaxedPropertyResolver resolver = new RelaxedPropertyResolver(context.getEnvironment());
    Map<String, Object> properties = resolver.getSubProperties(prefix);
    return new ConditionOutcome(!properties.isEmpty(), String.format("Map property [%s] is empty", prefix));
  }

  private static String attribute(AnnotatedTypeMetadata metadata, String name) {
    return (String) metadata.getAnnotationAttributes(ConditionalOnMapProperty.class.getName()).get(name);
  }

}

@Configuration
@MapPropertyCondition(prefix = "dataSource.")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

  // ... 通过配置注册一些 Bean
}
```

如此，就可以用于所有 Map 相关的参数判断了。

## 轻量级的组件测试

在实现自定义组件之后，提供全面且有效的单元也是必备的一环，这时候最重要的问题就是如何构建小而灵活的 Spring 容器，只加载必要的配置和组件。然后在不同的容器中运行组件，以验证组件的正确性。

想要了解如何写 Spring 组件的测试，不如直接去看看官方的开发者是如何写单元测试的。例如对于上面 `@ConditionalOnMapProperty` ，只需要少量的代码就可以完成一个覆盖率还不错的测试：

```
public class ConditionalOnMapPropertyTest {

  private final AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();

  @After
  public void closeContext() {
    this.context.close();
  }

  @Test
  public void bindValidProperties() {
    EnvironmentTestUtils.addEnvironment(
        this.context,
        buildProperty("foo.bar1", "bar1"),
        buildProperty("foo.bar2", "bar2"),
        buildProperty("foo.bar3", "bar3")
    );
    this.context.register(ConditionalOnMapPropertyConfiguration.class);
    this.context.refresh();
    Assert.assertTrue(this.context.containsBean("foo"));
    Assert.assertEquals(this.context.getBean("foo"), "foo");
  }

  @Test
  public void bindInvalidProperties() {
    EnvironmentTestUtils.addEnvironment(
        this.context,
        buildProperty("foo1.bar1", "bar1"),
        buildProperty("foo2.bar2", "bar1"),
        buildProperty("foo3.bar3", "bar1")
    );
    this.context.register(ConditionalOnMapPropertyConfiguration.class);
    this.context.refresh();
    Assert.assertFalse(this.context.containsBean("foo"));
  }


  private static String buildProperty(String key, String value) {
    return key + ":" + value;
  }

  @Configuration
  @ConditionalOnMapProperty(prefix = "foo.")
  public static class ConditionalOnMapPropertyConfiguration {

    @Bean
    public String foo() {
      return "foo";
    }

  }

}
```

在上面的测试中，我做了以下事情：

1. 创建了一个 `AnnotationConfigApplicationContext` 实例，它可以通过解析注解初始化 Spring 容器
2. 通过 `EnvironmentTestUtils.addEnvironment` 为容器添加配置
3. 通过 `register` 方法注册 `ConditionalOnMapPropertyConfiguration` 类
4. 调用 `refresh` 方法刷新容器，这时候容器会尝试读取已注册的 Configuration 并注册 Spring Bean
5. 如果配置符合 `@ConditionalOnMapProperty` 的要求，`ConditionalOnMapPropertyConfiguration` 就会被装载并注册一个名为 `foo` 的 Spring Bean，所以可以通过 `containsBean` 方法查看是否存在这个 Bean 判断该类是否被加载
6. 每个测试结束后调用 `close` 方法关闭容器

