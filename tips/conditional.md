# 自定义加载条件

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

