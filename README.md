This is the demo app for the techniques described in [these](http://blog.palominolabs.com/2011/08/15/a-simple-java-web-stack-with-guice-jetty-jersey-and-jackson/) [two](http://blog.palominolabs.com/2012/03/31/extending-the-simple-javajettyguicejerseyjackson-web-stack-with-automatic-jersey-resource-method-metrics/) blog posts on a Java web stack with Jetty, Guice, Jersey, and Jackson. While the code does still work, I encourage you to check out the following production ready versions rather than this demo-grade app.

- [jersey-metrics-filter](https://github.com/palominolabs/jersey-metrics-filter), automatic [metrics](http://metrics.codahale.com/) for all [Jersey](https://jersey.java.net/) resource methods
- [jersey-new-relic](https://github.com/palominolabs/jersey-new-relic), Jersey integration into [New Relic](http://newrelic.com/) monitoring
- [jetty-http-server-wrapper](https://github.com/palominolabs/jetty-http-server-wrapper), a simple wrapper around the [Jetty](http://www.eclipse.org/jetty/) http server
- [jersey-cors-filter](https://github.com/palominolabs/jersey-cors-filter) to apply CORS headers to Jersey HTTP responses
- [url-builder](https://github.com/palominolabs/url-builder) to safely build URLs

There's a [demo app](https://github.com/palominolabs/new-relic-sample-app) that shows how to use the above libraries.

在Java领域有很多Web相关的技术，比如Struts, Stripes, Tapestry, Wicket, GWT, Spring MVC, Vaadin, Play, plain old servlets and JSP, Dropwizard等等，他们都有其优点及缺点。

本篇文章并不讨论他们的优缺点，而是介绍一种让人感觉清爽的处理HTTP请求的方法：用Jetty处理底层HTTP请求，用Jersey控制请求的路由，用Jackson序列化对象，用Guice将他们集成在一起。它不是传统的monolithic架构，仅仅使用了J2EE中的HttpServletRequests。

提请注意：如果您想利用本篇文章介绍的技术快速构建一个实际的项目话，建议您直接采用Dropwizard。如果您想学到更多有关构建类似技术栈的技能，可以参考本系列的第二篇文章。

如果您打算基于服务器技术构建Web UI的话（无论是使用GWT转化为JavaScript，还是基于JSP转换为HTML），我可以告诉你，这种方式可能已经没有什么吸引力了。本篇文章介绍的技术栈不会带给你一个预置各种前端功能（展现页面、登录页面、电话号码校验功能等等）的Web框架。如果你正打算让你的服务器端更专注于数据存储，而将展现逻辑拆分到客户端（不管是浏览器还是移动客户端）的话，本文介绍的这种轻量级框架构建方法对你或许有所帮助。当服务器端仅仅通过HTTP协议推送JSON、XML或者其他格式数据到客户端时，服务器端也就不再需要portlets或者任何JavaScript组件。

Jetty
Jetty是一个非常容易“嵌入使用”的HTTP服务器和Servlet容器。Jetty可以简单地通过main方法启动，非常适用于开发人员重复执行“编辑-重启-测试”工作（在我使用的集成了Servlet容器的IDE上，简单地执行main方法是最快、最稳定的）。如果你更喜欢在一个独立的Servlet容器中运行、调试你的应用，或者你需要容器提供的安全、事务、分布式事务等功能，你最好还是将你的应用打包成war部署到J2EE服务器上，而不是使用Jetty。下面是示例代码描述如何启动Jetty服务器，并监听localhost:8080端口。

Server server = new Server(8080);
// handlers go here
server.start();

就是这么简单！但是在我们告诉它该如何处理请求之前，它尚不能处理任何请求。所以在我们调用 start 方法之前，我们需要提供给server一个Handler，在Jetty中Handler可以在响应HTTP请求的时候做任何事，你可以提供一个Handle仅仅用于收集性能指标或者仅仅用于打印日志，在这种情况下，我们需要Handler能够调用servlet。

ServletContextHandler handler = new ServletContextHandler();
handler.setContextPath("/");
// set up handler
server.setHandler(handler);

Jetty中的ServletContextHandler有个地方比较奇怪，就是它总需要持有一个servlet，即使像我们的例子中一样没有任何业务逻辑。所以我们为这个Handler提供了一个servlet，它总是返回404。

@Override
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    resp.setStatus(HttpServletResponse.SC_NOT_FOUND);
    resp.setContentType("text/plain");
    resp.setContentType("UTF-8");
    resp.getWriter().append("404");
}

如果你用上述代码创建一个main方法启动并运行Jetty，你就可以在浏览器中通过http://localhost:8080去查看InvalidRequestServlet的执行结果。在我们加入实际的逻辑代码之前，我需要先解释一下Guice及servlet层的关系。

Guice

Guice是一个依赖注入的框架，尽管它实现了和Spring几乎相同的工作，但还是有些不同的观点。Spring倾向于用bean id的方式将代码集成在一起，而Guice倾向于使用bean type。他们都是不错的工具，强项有所不同，我现在仅展示如何使用Guice（不然这篇文章永远写不完了），下面我将快速介绍一下Guice及Guice Servlet，然后回来构建我们的技术栈。


MAKING SOME PB&J SANDWICHES 制作花生酱三明治
像Guice和Spring这样的DI/IoC框架都都尝试解决这样一个难题（他们还能解决其他难题）：通过多级构造函数传递类实例，我相信你一定看过这样的反模式设计：Class1构造函数需要Class2实例，Class2构造函数需要Class3实例，Class3需要在其构造函数里传入Foo实例，因此你需要在高层（Class1甚至更高层次）创建一个正确的Foo实例将其最终传递给Class3，Class3通过构造函数传入Foo的实例而不是在构造函数内部直接创建一个Foo对象实例，这种看似复杂的方式却很有价值，主要是便于测试，将关注点分离出来。

Guice使这种问题变得简单，举个例子，假设你有一个SandwichMaker类，在它的构造函数里需要传入PeanutButter的实现类实例，为了测试你的SandwichMaker，你希望使用mock的PeanutButter实现类，但是在实际部署的时候你还可以使用OrganicCrunchyValenciaPeanutButter（简写为OCVPB）

class SandwichMaker {
// ...
    SandwichMaker(PeanutButter peanutButter) {
        this.peanutButter = peanutButter;
    }
// ...
}
interface PeanutButter {
    void applyToSandwich(Sandwich sandwich, int grams);
}

现在我们的问题是实际应用中在哪里实例化你的OCVPB？在创建SandwichMaker的对象中创建OCVPB实例并不理想，因为此时必须创建实际应用中的OCVPB，这种硬编码的方式导致很难做单元测试。在更高层次的main方法中创建OCVPB实例就必须通过多级构造函数传递给SandwichMaker，这种方式同样不理想。下面解释如何通过Guice来解决这种问题，我们先来解释几个相关概念。

Guice相关概念
无需创建一个实例对象，Guice允许你定义类之间的依赖关系及如何将他们组装在一起，在我们的例子中，我们打算让SandwichMaker持有一个PeanutButter的实现类，在Guice中我们仅需使用@Inject这个注解就可以搞定这一切。（在Guice和JSR 330定定义中都有Inject这个注解，他们的作用几乎相同）

class SandwichMaker {
// ...
    @Inject
    SandwichMaker(PeanutButter peanutButter) {
        this.peanutButter = peanutButter;
    }
// ...
}

我们已经知道了在Guice中如何描述SandwichMaker需要一个PeanutButter，我们还需要指定使用哪个PeanutButter的实现类（PeanutButter是一个接口，有多个实现类）。Guice中通过在module中binding来描述这种关系。Guice中的module旨在划分应用的功能边界，划分module和划分package非常类似，你可以简单尝试一下以获得一个感官感受。现在就我们创建一个module来做我们的sandwiches

class SandwichModule extends AbstractModule {
    @Override
    protected void configure() {
        bind(PeanutButter.class).to(OrganicCrunchyValenciaPeanutButter.class);
    }
}

这样Guice就能在我们需要注入PeanutButter的时候为我们创建一个OCVPB的实例，看上去PeanutButter应该是单例的，下面我们就来将它变成单例。

bind(PeanutButter.class).to(OrganicCrunchyValenciaPeanutButter.class).in(Scopes.SINGLETON);

为了实现单例模式还可以在OCVPB类上使用@Singleton注解。现在无论创建多少个SandwichMakers，他们都使用同一个PeanutButter。所有的module都需要创建一个Injector来启动，同时通过它来获得SandwichMaker的一个实例。这样Guice就可以通过注入对象和bingding关系创建一个OCVPB实例并传递给SandwichMaker的构造函数。实际上我们并不需要一个单独的SandwichMaker实例，但是如果我们想获得它，就可以通过下面的方式

Injector injector = Guice.createInjector(new SandwichModule());
 
SandwichMaker maker = injector.createInstance(SandwichMaker.class);
// use the maker

以上就是Guice的简单用法，你可以查看Guice相关文档来了解更多内容

GUICE SERVLET

我们已经知道了如何绑定实例对象及如何注入他们，现在我们再来看看 Guice Servlet，这是个Guice的扩展，完全不依赖于web.xml，非常简单。假设你有一个叫FooServlet的sevlet，按照以前使用web.xml的方式，你需要配置servlet标签及servlet-mapping标签，而Guice Servlet可以用下面的方式

class FooServletModule extends ServletModule {
    @Override
    protected void configureServlets() {
        bind(FooServlet.class);
        serve("/foo").with(FooServlet.class);
 
        // other servlets
    }
}

Since Guice is instantiating your FooServlet class intead of relying on the servlet container to invoke the 0-args ctor, this also means that you can use @Inject on the ctor and get the objects you need that way instead of pulling them out of init params or servlet context. To feed incoming requests into the servlets you’ve laid out with Guice Servlet, you need to set up GuiceFilter as a filter for all requests. You can do this in web.xml if you’re using a war (it’s the only thing you’ll actually need in web.xml) or just configure Jetty directly.


