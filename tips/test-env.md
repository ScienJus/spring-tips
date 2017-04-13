# 快速构造测试环境

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

