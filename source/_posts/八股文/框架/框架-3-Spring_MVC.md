---
title: 框架 - 3.Spring MVC
date: 2020-08-16 17:33:03
categories: 
- 八股文
- 框架
tags:
- 框架
- 面试
---

### Spring MVC 3

#### **Q1：Spring MVC 的处理流程？**

Web 容器启动时会通知 Spring 初始化容器，加载 Bean 的定义信息并初始化所有单例 Bean，然后遍历容器中的 Bean，获取每一个 Controller 中的所有方法访问的 URL，将 URL 和对应的 Controller 保存到一个 Map 集合中。

所有的请求会转发给 DispatcherServlet 前端处理器处理，DispatcherServlet 会请求 HandlerMapping 找出容器中被 @Controler 注解修饰的 Bean 以及被 @RequestMapping 修饰的方法和类，生成 Handler 和 HandlerInterceptor 并以一个 HandlerExcutionChain 处理器执行链的形式返回。

之后 DispatcherServlet 使用 Handler 找到对应的 HandlerApapter，通过 HandlerApapter 调用 Handler 的方法，将请求参数绑定到方法的形参上，执行方法处理请求并得到 ModelAndView。

最后 DispatcherServlet 根据使用 ViewResolver 试图解析器对得到的 ModelAndView 逻辑视图进行解析得到 View 物理视图，然后对视图渲染，将数据填充到视图中并返回给客户端。

---

#### **Q2：Spring MVC 有哪些组件？**

DispatcherServlet：SpringMVC 中的前端控制器，是整个流程控制的核心，负责接收请求并转发给对应的处理组件。

Handler：处理器，完成具体业务逻辑，相当于 Servlet 或 Action。

HandlerMapping：完成 URL 到 Controller 映射，DispatcherServlet 通过 HandlerMapping 将不同请求映射到不同 Handler。

HandlerInterceptor：处理器拦截器，是一个接口，如果需要完成一些拦截处理，可以实现该接口。

HandlerExecutionChain：处理器执行链，包括两部分内容：Handler 和 HandlerInterceptor。

HandlerAdapter：处理器适配器，Handler执行业务方法前需要进行一系列操作，包括表单数据验证、数据类型转换、将表单数据封装到JavaBean等，这些操作都由 HandlerAdapter 完成。DispatcherServlet 通过 HandlerAdapter 来执行不同的 Handler。

ModelAndView：装载模型数据和视图信息，作为 Handler 处理结果返回给 DispatcherServlet。

ViewResolver：视图解析器，DispatcherServlet 通过它将逻辑视图解析为物理视图，最终将渲染的结果响应给客户端。

---

#### **Q3：Spring MVC 的相关注解？**

@Controller：在类定义处添加，将类交给IoC容器管理。

@RequtestMapping：将URL请求和业务方法映射起来，在类和方法定义上都可以添加该注解。value 属性指定URL请求的实际地址，是默认值。method 属性限制请求的方法类型，包括GET、POST、PUT、DELETE等。如果没有使用指定的请求方法请求URL，会报405 Method Not Allowed 错误。params 属性限制必须提供的参数，如果没有会报错。

@RequestParam：如果 Controller 方法的形参和 URL 参数名一致可以不添加注解，如果不一致可以使用该注解绑定。value 属性表示HTTP请求中的参数名。required 属性设置参数是否必要，默认false。defaultValue 属性指定没有给参数赋值时的默认值。

@PathVariable：Spring MVC 支持 RESTful 风格 URL，通过 @PathVariable 完成请求参数与形参的绑定。
