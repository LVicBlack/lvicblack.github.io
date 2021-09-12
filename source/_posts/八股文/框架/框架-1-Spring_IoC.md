---
title: 框架 - 1.Spring IoC
date: 2020-08-16 17:31:03
categories: 
- 八股文
- 框架
tags:
- 框架
- 面试
---

### **Spring IoC 11**

#### **Q1：IoC 是什么？**

IoC 即控制反转，简单来说就是把原来代码里需要实现的对象创建、依赖反转给容器来帮忙实现，需要创建一个容器并且需要一种描述让容器知道要创建的对象间的关系，在 Spring 中管理对象及其依赖关系是通过 Spring 的 IoC 容器实现的。

IoC 的实现方式有依赖注入和依赖查找，由于依赖查找使用的很少，因此 IoC 也叫做依赖注入。依赖注入指对象被动地接受依赖类而不用自己主动去找，对象不是从容器中查找它依赖的类，而是在容器实例化对象时主动将它依赖的类注入给它。假设一个 Car 类需要一个 Engine 的对象，那么一般需要需要手动 new 一个 Engine，利用 IoC 就只需要定义一个私有的 Engine 类型的成员变量，容器会在运行时自动创建一个 Engine 的实例对象并将引用自动注入给成员变量。

---

#### **Q2：IoC 容器初始化过程？**

**基于 XML 的容器初始化**

当创建一个 ClassPathXmlApplicationContext 时，构造方法做了两件事：

① 调用父容器的构造方法为容器设置好 Bean 资源加载器。

② 调用父类的 setConfigLocations 方法设置 Bean 配置信息的定位路径。

ClassPathXmlApplicationContext 通过调用父类 AbstractApplicationContext 的 refresh 方法启动整个 IoC 容器对 Bean 定义的载入过程，refresh 是一个模板方法，规定了 IoC 容器的启动流程。在创建 IoC 容器前如果已有容器存在，需要把已有的容器销毁，保证在 refresh 方法后使用的是新创建的 IoC 容器。

容器创建后通过 loadBeanDefinitions 方法加载 Bean 配置资源，该方法做两件事：

① 调用资源加载器的方法获取要加载的资源。

② 真正执行加载功能，由子类 XmlBeanDefinitionReader 实现。加载资源时首先解析配置文件路径，读取配置文件的内容，然后通过 XML 解析器将 Bean 配置信息转换成文档对象，之后按照 Spring Bean 的定义规则对文档对象进行解析。

Spring IoC 容器中注册解析的 Bean 信息存放在一个 HashMap 集合中，key 是字符串，值是 BeanDefinition，注册过程中需要使用 synchronized 保证线程安全。当配置信息中配置的 Bean 被解析且被注册到 IoC 容器中后，初始化就算真正完成了，Bean 定义信息已经可以使用且可被检索。Spring IoC 容器的作用就是对这些注册的 Bean 定义信息进行处理和维护，注册的 Bean 定义信息是控制反转和依赖注入的基础。

**基于注解的容器初始化**

分为两种：

① 直接将注解 Bean 注册到容器中，可以在初始化容器时注册，也可以在容器创建之后手动注册，然后刷新容器使其对注册的注解 Bean 进行处理。

② 通过扫描指定的包及其子包的所有类处理，在初始化注解容器时指定要自动扫描的路径。

---

#### **Q3：依赖注入的实现方法有哪些？**

**构造方法注入：** IoC Service Provider 会检查被注入对象的构造方法，取得它所需要的依赖对象列表，进而为其注入相应的对象。这种方法的优点是在对象构造完成后就处于就绪状态，可以马上使用。缺点是当依赖对象较多时，构造方法的参数列表会比较长，构造方法无法被继承，无法设置默认值。对于非必需的依赖处理可能需要引入多个构造方法，参数数量的变动可能会造成维护的困难。

**setter 方法注入：** 当前对象只需要为其依赖对象对应的属性添加 setter 方法，就可以通过 setter 方法将依赖对象注入到被依赖对象中。setter 方法注入在描述性上要比构造方法注入强，并且可以被继承，允许设置默认值。缺点是无法在对象构造完成后马上进入就绪状态。

**接口注入：** 必须实现某个接口，接口提供方法来为其注入依赖对象。使用少，因为它强制要求被注入对象实现不必要接口，侵入性强。

---

#### **Q4：依赖注入的相关注解？**

@Autowired：自动按类型注入，如果有多个匹配则按照指定 Bean 的 id 查找，查找不到会报错。

@Qualifier：在自动按照类型注入的基础上再按照 Bean 的 id 注入，给变量注入时必须搭配 @Autowired，给方法注入时可单独使用。

@Resource ：直接按照 Bean 的 id 注入，只能注入 Bean 类型。

@Value ：用于注入基本数据类型和 String 类型。

---

#### **Q5：依赖注入的过程？**

getBean 方法获取 Bean 实例，该方\*\*\*调用 doGetBean ，doGetBean 真正实现从 IoC 容器获取 Bean 的功能，也是触发依赖注入的地方。

具体创建 Bean 对象的过程由 ObjectFactory 的 createBean 完成，该方法主要通过 createBeanInstance 方法生成 Bean 包含的 Java 对象实例和 populateBean 方法对 Bean 属性的依赖注入进行处理。

在 populateBean方法中，注入过程主要分为两种情况：① 属性值类型不需要强制转换时，不需要解析属性值，直接进行依赖注入。② 属性值类型需要强制转换时，首先解析属性值，然后对解析后的属性值进行依赖注入。依赖注入的过程就是将 Bean 对象实例设置到它所依赖的 Bean 对象属性上，真正的依赖注入是通过 setPropertyValues 方法实现的，该方法使用了委派模式。

BeanWrapperImpl 类负责对完成初始化的 Bean 对象进行依赖注入，对于非集合类型属性，使用 JDK 反射，通过属性的 setter 方法为属性设置注入后的值。对于集合类型的属性，将属性值解析为目标类型的集合后直接赋值给属性。

当容器对 Bean 的定位、载入、解析和依赖注入全部完成后就不再需要手动创建对象，IoC 容器会自动为我们创建对象并且注入依赖。

---

#### **Q6：Bean 的生命周期？**

在 IoC 容器的初始化过程中会对 Bean 定义完成资源定位，加载读取配置并解析，最后将解析的 Bean 信息放在一个 HashMap 集合中。当 IoC 容器初始化完成后，会进行对 Bean 实例的创建和依赖注入过程，注入对象依赖的各种属性值，在初始化时可以指定自定义的初始化方法。经过这一系列初始化操作后 Bean 达到可用状态，接下来就可以使用 Bean 了，当使用完成后会调用 destroy 方法进行销毁，此时也可以指定自定义的销毁方法，最终 Bean 被销毁且从容器中移除。

XML 方式通过配置 bean 标签中的 init\-Method 和 destory\-Method 指定自定义初始化和销毁方法。

注解方式通过 @PreConstruct 和 @PostConstruct 注解指定自定义初始化和销毁方法。

---

#### **Q7：Bean 的作用范围？**

通过 scope 属性指定 bean 的作用范围，包括：

① singleton：单例模式，是默认作用域，不管收到多少 Bean 请求每个容器中只有一个唯一的 Bean 实例。

② prototype：原型模式，和 singleton 相反，每次 Bean 请求都会创建一个新的实例。

③ request：每次 HTTP 请求都会创建一个新的 Bean 并把它放到 request 域中，在请求完成后 Bean 会失效并被垃圾收集器回收。

④ session：和 request 类似，确保每个 session 中有一个 Bean 实例，session 过期后 bean 会随之失效。

⑤ global session：当应用部署在 Portlet 容器时，如果想让所有 Portlet 共用全局存储变量，那么该变量需要存储在 global session 中。

---

#### **Q8：如何通过 XML 方式创建 Bean？**

默认无参构造方法，只需要指明 bean 标签中的 id 和 class 属性，如果没有无参构造方\*\*\*报错。

静态工厂方法，通过 bean 标签中的 class 属性指明静态工厂，factory\-method 属性指明静态工厂方法。

实例工厂方法，通过 bean 标签中的 factory\-bean 属性指明实例工厂，factory\-method 属性指明实例工厂方法。

---

#### **Q9：如何通过注解创建 Bean？**

@Component 把当前类对象存入 Spring 容器中，相当于在 xml 中配置一个 bean 标签。value 属性指定 bean 的 id，默认使用当前类的首字母小写的类名。

@Controller，@Service，@Repository 三个注解都是 @Component 的衍生注解，作用及属性都是一模一样的。只是提供了更加明确语义，@Controller 用于表现层，@Service用于业务层，@Repository用于持久层。如果注解中有且只有一个 value 属性要赋值时可以省略 value。

如果想将第三方的类变成组件又没有源代码，也就没办法使用 @Component 进行自动配置，这种时候就要使用 @Bean 注解。被 @Bean 注解的方法返回值是一个对象，将会实例化，配置和初始化一个新对象并返回，这个对象由 Spring 的 IoC 容器管理。name 属性用于给当前 @Bean 注解方法创建的对象指定一个名称，即 bean 的 id。当使用注解配置方法时，如果方法有参数，Spring 会去容器查找是否有可用 bean对象，查找方式和 @Autowired 一样。

---

#### **Q10：如何通过注解配置文件？**

@Configuration 用于指定当前类是一个 spring 配置类，当创建容器时会从该类上加载注解，value 属性用于指定配置类的字节码。

@ComponentScan 用于指定 Spring 在初始化容器时要扫描的包。basePackages 属性用于指定要扫描的包。

@PropertySource 用于加载 .properties 文件中的配置。value 属性用于指定文件位置，如果是在类路径下需要加上 classpath。

@Import 用于导入其他配置类，在引入其他配置类时可以不用再写 @Configuration 注解。有 @Import 的是父配置类，引入的是子配置类。value 属性用于指定其他配置类的字节码。

---

#### **Q11：BeanFactory、FactoryBean 和 ApplicationContext 的区别？**

BeanFactory 是一个 Bean 工厂，使用简单工厂模式，是 Spring IoC 容器顶级接口，可以理解为含有 Bean 集合的工厂类，作用是管理 Bean，包括实例化、定位、配置对象及建立这些对象间的依赖。BeanFactory 实例化后并不会自动实例化 Bean，只有当 Bean 被使用时才实例化与装配依赖关系，属于延迟加载，适合多例模式。

FactoryBean 是一个工厂 Bean，使用了工厂方法模式，作用是生产其他 Bean 实例，可以通过实现该接口，提供一个工厂方法来自定义实例化 Bean 的逻辑。FactoryBean 接口由 BeanFactory 中配置的对象实现，这些对象本身就是用于创建对象的工厂，如果一个 Bean 实现了这个接口，那么它就是创建对象的工厂 Bean，而不是 Bean 实例本身。

ApplicationConext 是 BeanFactory 的子接口，扩展了 BeanFactory 的功能，提供了支持国际化的文本消息，统一的资源文件读取方式，事件传播以及应用层的特别配置等。容器会在初始化时对配置的 Bean 进行预实例化，Bean 的依赖注入在容器初始化时就已经完成，属于立即加载，适合单例模式，一般推荐使用。
