原文出处： [张开涛](http://sishuok.com/forum/blogPost/list/0/2508.html)

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章将同步到我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《Spring和SpringMVC源码分析》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从spring基础入手，一步步地学习spring基础和springmvc的框架知识，并上手进行项目实战，spring框架是每一个Java工程师必须要学习和理解的知识点，进一步来说，你还需要掌握spring甚至是springmvc的源码以及实现原理，才能更完整地了解整个spring技术体系，形成自己的知识框架。

后续还会有springboot和springcloud的技术专题，陆续为大家带来，敬请期待。

为了更好地总结和检验你的学习成果，本系列文章也会提供部分知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->


在讲源码之前，先让我们回顾一下一下Spring的基本概念，当然，在看源码之前你需要使用过spring或者spirngmvc。

##  Spring是什么
Spring是一个开源的轻量级Java SE（Java 标准版本）/Java EE（Java 企业版本）开发应用框架，其目的是用于简化企业级应用程序开发。应用程序是由一组相互协作的对象组成。而在传统应用程序开发中，一个完整的应用是由一组相互协作的对象组成。所以开发一个应用除了要开发业务逻辑之外，最多的是关注如何使这些对象协作来完成所需功能，而且要低耦合、高内聚。

业务逻辑开发是不可避免的，那如果有个框架出来帮我们来创建对象及管理这些对象之间的依赖关系。可能有人说了，比如“抽象工厂、工厂方法设计模式”不也可以帮我们创建对象，“生成器模式”帮我们处理对象间的依赖关系，不也能完成这些功能吗？

可是这些又需要我们创建另一些工厂类、生成器类，我们又要而外管理这些类，增加了我们的负担，如果能有种通过配置方式来创建对象，管理对象之间依赖关系，我们不需要通过工厂和生成器来创建及管理对象之间的依赖关系，这样我们是不是减少了许多工作，加速了开发，能节省出很多时间来干其他事。Spring框架刚出来时主要就是来完成这个功能。

Spring框架除了帮我们管理对象及其依赖关系，还提供像通用日志记录、性能统计、安全控制、异常处理等面向切面的能力，还能帮我管理最头疼的数据库事务，本身提供了一套简单的JDBC访问实现，提供与第三方数据访问框架集成（如Hibernate、JPA），与各种Java EE技术整合（如Java Mail、任务调度等等），提供一套自己的web层框架Spring MVC、而且还能非常简单的与第三方web框架集成。

从这里我们可以认为Spring是一个超级粘合平台，除了自己提供功能外，还提供粘合其他技术和框架的能力，从而使我们可以更自由的选择到底使用什么技术进行开发。而且不管是JAVA SE（C/S架构）应用程序还是JAVA EE（B/S架构）应用程序都可以使用这个平台进行开发。让我们来深入看一下Spring到底能帮我们做些什么？

## Spring能帮我们做什么
Spring除了不能帮我们写业务逻辑，其余的几乎什么都能帮助我们简化开发：

 

一、传统程序开发，创建对象及组装对象间依赖关系由我们在程序内部进行控制，这样会加大各个对象间的耦合，如果我们要修改对象间的依赖关系就必须修改源代码，重新编译、部署；而如果采用Spring，则由Spring根据配置文件来进行创建及组装对象间依赖关系，只需要改配置文件即可，无需重新编译。所以，Spring能帮我们根据配置文件创建及组装对象之间的依赖关系。

 

二、当我们要进行一些日志记录、权限控制、性能统计等时，在传统应用程序当中我们可能在需要的对象或方法中进行，而且比如权限控制、性能统计大部分是重复的，这样代码中就存在大量重复代码，即使有人说我把通用部分提取出来，那必然存在调用还是存在重复，像性能统计我们可能只是在必要时才进行，在诊断完毕后要删除这些代码；还有日志记录，比如记录一些方法访问日志、数据访问日志等等，这些都会渗透到各个要访问方法中；

还有权限控制，必须在方法执行开始进行审核，想想这些是多么可怕而且是多么无聊的工作。如果采用Spring，这些日志记录、权限控制、性能统计从业务逻辑中分离出来，通过Spring支持的面向切面编程，在需要这些功能的地方动态添加这些功能，无需渗透到各个需要的方法或对象中；


有人可能说了，我们可以使用“代理设计模式”或“包装器设计模式”，你可以使用这些，但还是需要通过编程方式来创建代理对象，还是要耦合这些代理对象，而采用Spring 面向切面编程能提供一种更好的方式来完成上述功能，一般通过配置方式，而且不需要在现有代码中添加任何额外代码，现有代码专注业务逻辑。

所以，Spring 面向切面编程能帮助我们无耦合的实现日志记录，性能统计，安全控制。

 

三、在传统应用程序当中，我们如何来完成数据库事务管理？需要一系列“获取连接，执行SQL，提交或回滚事务，关闭连接”，而且还要保证在最后一定要关闭连接，多么可怕的事情，而且也很无聊；如果采用Spring，我们只需获取连接，执行SQL，其他的都交给Spring来管理了，简单吧。所以，Spring能非常简单的帮我们管理数据库事务。

 

四、Spring还提供了与第三方数据访问框架（如Hibernate、JPA）无缝集成，而且自己也提供了一套JDBC访问模板，来方便数据库访问。

 

五、Spring还提供与第三方Web（如Struts、JSF）框架无缝集成，而且自己也提供了一套Spring MVC框架，来方便web层搭建。

 

六、Spring能方便的与Java EE（如Java Mail、任务调度）整合，与更多技术整合（比如缓存框架）。

 

Spring能帮我们做这么多事情，提供这么多功能和与那么多主流技术整合，而且是帮我们做了开发中比较头疼和困难的事情，那可能有人会问，难道只有Spring这一个框架，没有其他选择？当然有，比如EJB需要依赖应用服务器、开发效率低、在开发中小型项目是宰鸡拿牛刀，虽然发展到现在EJB比较好用了，但还是比较笨重还需要依赖应用服务器等。那为何需要使用Spring，而不是其他框架呢？让我们接着往下看。

 
## 为何需要Spring
一 首先阐述几个概念

1、应用程序：是能完成我们所需要功能的成品，比如购物网站、OA系统。

2、框架：是能完成一定功能的半成品，比如我们可以使用框架进行购物网站开发；框架做一部分功能，我们自己做一部分功能，这样应用程序就创建出来了。而且框架规定了你在开发应用程序时的整体架构，提供了一些基础功能，还规定了类和对象的如何创建、如何协作等，从而简化我们开发，让我们专注于业务逻辑开发。

3、非侵入式设计：从框架角度可以这样理解，无需继承框架提供的类，这种设计就可以看作是非侵入式设计，如果继承了这些框架类，就是侵入设计，如果以后想更换框架之前写过的代码几乎无法重用，如果非侵入式设计则之前写过的代码仍然可以继续使用。

4、轻量级及重量级：轻量级是相对于重量级而言的，轻量级一般就是非入侵性的、所依赖的东西非常少、资源占用非常少、部署简单等等，其实就是比较容易使用，而重量级正好相反。

5、POJO：POJO（Plain Old Java Objects）简单的Java对象，它可以包含业务逻辑或持久化逻辑，但不担当任何特殊角色且不继承或不实现任何其它Java框架的类或接口。

6、容器：在日常生活中容器就是一种盛放东西的器具，从程序设计角度看就是装对象的的对象，因为存在放入、拿出等操作，所以容器还要管理对象的生命周期。

7、控制反转：即Inversion of Control，缩写为IoC，控制反转还有一个名字叫做依赖注入（Dependency Injection），就是由容器控制程序之间的关系，而非传统实现中，由程序代码直接操控。

8、Bean：一般指容器管理对象，在Spring中指Spring IoC容器管理对象。

 

## 为什么需要Spring及Spring的优点

●非常轻量级的容器：以集中的、自动化的方式进行应用程序对象创建和装配，负责对象创建和装配，管理对象生命周期，能组合成复杂的应用程序。Spring容器是非侵入式的（不需要依赖任何Spring特定类），而且完全采用POJOs进行开发，使应用程序更容易测试、更容易管理。而且核心JAR包非常小，Spring3.0.5不到1M，而且不需要依赖任何应用服务器，可以部署在任何环境（Java SE或Java EE）。

●AOP：AOP是Aspect Oriented Programming的缩写，意思是面向切面编程，提供从另一个角度来考虑程序结构以完善面向对象编程（相对于OOP），即可以通过在编译期间、装载期间或运行期间实现在不修改源代码的情况下给程序动态添加功能的一种技术。通俗点说就是把可重用的功能提取出来，然后将这些通用功能在合适的时候织入到应用程序中；比如安全，日记记录，这些都是通用的功能，我们可以把它们提取出来，然后在程序执行的合适地方织入这些代码并执行它们，从而完成需要的功能并复用了这些功能。

● 简单的数据库事务管理：在使用数据库的应用程序当中，自己管理数据库事务是一项很让人头疼的事，而且很容易出现错误，Spring支持可插入的事务管理支持，而且无需JEE环境支持，通过Spring管理事务可以把我们从事务管理中解放出来来专注业务逻辑。

●JDBC抽象及ORM框架支持：Spring使JDBC更加容易使用；提供DAO（数据访问对象）支持，非常方便集成第三方ORM框架，比如Hibernate等；并且完全支持Spring事务和使用Spring提供的一致的异常体系。

●灵活的Web层支持：Spring本身提供一套非常强大的MVC框架，而且可以非常容易的与第三方MVC框架集成，比如Struts等。

●简化各种技术集成：提供对Java Mail、任务调度、JMX、JMS、JNDI、EJB、动态语言、远程访问、Web Service等的集成。

Spring能帮助我们简化应用程序开发，帮助我们创建和组装对象，为我们管理事务，简单的MVC框架，可以把Spring看作是一个超级粘合平台，能把很多技术整合在一起，形成一个整体，使系统结构更优良、性能更出众，从而加速我们程序开发，有如上优点，我们没有理由不考虑使用它。

##  如何学好Spring
要学好Spring，首先要明确Spring是个什么东西，能帮我们做些什么事情，知道了这些然后做个简单的例子，这样就基本知道怎么使用Spring了。

Spring核心是IoC容器，所以一定要透彻理解什么是IoC容器，以及如何配置及使用容器，其他所有技术都是基于容器实现的；

理解好IoC后，接下来是面向切面编程，首先还是明确概念，基本配置，最后是实现原理，接下来就是数据库事务管理，其实Spring管理事务是通过面向切面编程实现的，所以基础很重要，IoC容器和面向切面编程搞定后，其余都是基于这俩东西的实现，学起来就更加轻松了。要学好Spring不能急，一定要把基础打牢，基础牢固了，这就是磨刀不误砍柴工。
