# 动态生成 Bean

Spring 提供的 `@Bean` 或 `@Component` 已经可以应用于 99% 的场景了，但是有些时候依旧需要通过其他的方法，例如通过一个 List 或 Map 格式的配置文件，动态的定义多个独立的 Bean，这时候 `@Bean` 就力不从心了。

一个勉强可以用的方法是通过实现 `BeanFactoryPostProcessor` 接口，在 `postProcessBeanFactory` 中通过 `ConfigurableListableBeanFactory` 动态的将对象注册进容器。

例如将之前例子中的配置，注册为多个 DataSource：

```
@ConditionalOnMapProperty(prefix = "dataSource.")
public class DynamicDataSourcesConfiguration implements BeanFactoryPostProcessor, EnvironmentAware {

  private ConfigurableEnvironment environment;

  @Override
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
      throws BeansException {
    MultiMysqlProperties multiMysqlProperties = resolverSetting("", MultiMysqlProperties.class);
    multiMysqlProperties.getMysql().forEach((name, properties) ->
      createBean(beanFactory, name, properties)
    );
  }

  private void createBean(ConfigurableListableBeanFactory beanFactory, String name, MysqlProperties mysqlProperties) {
    DataSource dataSource = new DruidDataSource(
        mysqlProperties.getUrl(),
        mysqlProperties.getUserName(),
        mysqlProperties.getPassword(),
    );

    beanFactory.registerSingleton(name + "DataSource", dataSource);
  }

  /**
   * 注入环境以便读取配置
   * @param environment
   */
  @Override
  public void setEnvironment(Environment environment) {
    this.environment = (ConfigurableEnvironment) environment;
  }

  // 读取配置并转换成对象
  private <T> T resolverSetting(String targetName, Class<T> clazz) {
    PropertiesConfigurationFactory<Object> factory = new PropertiesConfigurationFactory<>(clazz);
    factory.setTargetName(targetName);
    factory.setPropertySources(environment.getPropertySources());
    factory.setConversionService(environment.getConversionService());
    try {
      factory.bindPropertiesToTarget();
      return (T) factory.getObject();
    } catch (Exception e) {
      log.error("Could not bind DataSourceSettings properties: " + e.getMessage(), e);
      throw new FatalBeanException("Could not bind DataSourceSettings properties", e);
    }
  }

}
```

