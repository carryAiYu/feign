
## 核心类

[`Feign`](./core/src/main/java/feign/Feign.java) 动态代理构建实现类的抽象基类
[`ReflectiveFeign`](./core/src/main/java/feign/ReflectiveFeign.java) 创建代理的实现类

```
  @SuppressWarnings("unchecked")
  @Override
  public <T> T newInstance(Target<T> target) {
    // 解析方法调用参数生成MethodHandler
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    // 持有Method和MethodHandler方便调用
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    // Object的默认方法 equals()
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if (Util.isDefault(method)) {
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    // 创建jdk代理handler 默认实现是FeignInvocationHandler
    InvocationHandler handler = factory.create(target, methodToHandler);
    // 生成代理
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
        new Class<?>[] {target.type()}, handler);

    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }

```

```
  static class FeignInvocationHandler implements InvocationHandler {

    private final Target target;
    private final Map<Method, MethodHandler> dispatch;

    FeignInvocationHandler(Target target, Map<Method, MethodHandler> dispatch) {
      this.target = checkNotNull(target, "target");
      this.dispatch = checkNotNull(dispatch, "dispatch for %s", target);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      if ("equals".equals(method.getName())) {
        try {
          Object otherHandler =
              args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
          return equals(otherHandler);
        } catch (IllegalArgumentException e) {
          return false;
        }
      } else if ("hashCode".equals(method.getName())) {
        return hashCode();
      } else if ("toString".equals(method.getName())) {
        return toString();
      }
      // 通过dispatch分派给对应的MethodHandler执行
      return dispatch.get(method).invoke(args);
    }
```