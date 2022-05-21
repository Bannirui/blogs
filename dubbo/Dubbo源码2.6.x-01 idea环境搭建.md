## git地址

https://github.com/Bannirui/dubbo.git

 

## 项目环境

```bash
" 代码clone
git clone https://github.com/Bannirui/dubbo.git

" 分支 study-2.6.x
git checkout -b study-2.6.x origin/2.6.x

" push
git push -u origin study-2.6.x
```



### demo

以dubbo api方式构建服务 整合spring的思路肯定是利用后置处理器封装api的构建方式 目前将关注集中在dubbo本身

```bash
.
├── dubbo-demo
│   ├── dubbo-demo-api      // 接口
│   ├── dubbo-demo-consumer // 消费者
│   ├── dubbo-demo-provider // 提供者
└── tree.txt
```

 

### 生产者

```java
/**
 * <p>以dubbo api方式构建 整合spring的思路肯定是利用后置处理器封装api的构建方式 目前将关注集中在dubbo本身</p>
 * @since 2022/5/18
 * @author dingrui
 */
public class ApiProvider {

    public static void main(String[] args) throws IOException {
        // 服务实现
        DemoService demoService = new DemoServiceImpl();

        // 当前应用配置
        ApplicationConfig application = new ApplicationConfig();
        application.setName("demo-provider");

        // 连接注册中心配置
        RegistryConfig registry = new RegistryConfig();
        registry.setAddress("multicast://224.5.6.7:1234");

        // 服务提供者协议配置
        ProtocolConfig protocol = new ProtocolConfig();
        protocol.setName("dubbo");
        protocol.setPort(20880);
        protocol.setThreads(200);

        // 服务提供者暴露服务配置 封装了与注册中心的连接
        ServiceConfig<DemoService> service = new ServiceConfig<DemoService>();
        service.setApplication(application);
        // 注册中心
        service.setRegistry(registry);
        // 协议
        service.setProtocol(protocol);
        service.setInterface(DemoService.class);
        service.setRef(demoService);

        // 暴露及注册服务
        service.export();

        System.in.read();
    }
}
```

 

### 消费者

```java
/**
 * <p>以dubbo api方式构建</p>
 * @since 2022/5/18
 * @author dingrui
 */
public class ApiConsumer {

    public static void main(String[] args) {
        // 当前应用配置
        ApplicationConfig application = new ApplicationConfig();
        application.setName("demo-consumer");

        // 连接注册中心配置
        RegistryConfig registry = new RegistryConfig();
        registry.setAddress("multicast://224.5.6.7:1234");

        // 引用远程服务
        ReferenceConfig<DemoService> reference = new ReferenceConfig<DemoService>();
        reference.setApplication(application);
        reference.setRegistry(registry); // 多个注册中心可以用setRegistries()
        reference.setInterface(DemoService.class);

        // 和本地bean一样使用service
        DemoService demoService = reference.get();
        String ret = demoService.sayHello("world");
        System.out.println("--------------");
        System.out.println(ret);
    }
}
```

 