---
title: Java-Chassis 2.0 新版本重大变更预览
tags: [ ServiceComb, Java-Chassis, microservice ]
date: 2020-01-03 21:15:56
categories:
- 软件技术
index_img: /img/blog/java-chassis-2-0-major-feature-preview/index.png
banner_img: /img/blog/java-chassis-2-0-major-feature-preview/banner.jpg
---
Java-Chassis 2.0 版本发布前夕，解读新版本核心机制的变更。
<!-- more -->

## Java-Chassis与契约

如果让我给出一个[ServiceComb-Java-Chassis][]与其他微服务框架最本质的区别，私以为不是handler、filter等扩展机制，也不是能够充分发挥Vert.x性能优势的Reactive模式，而是它以契约为中心的设计思想。

### 基于契约的抽象模型

Java-Chassis对OpenAPI契约的原生支持使其对于用户的服务接口的约束和掌握能力达到了一个比较高的水准，大家可以去看看Java-Chassis对于其[框架架构][]的描述，它可以将SpringMVC、JAX-RS、透明RPC三种服务端编程模型和RPC、RestTemplate两种客户端变成模型都归一化到OpenAPI契约上，在此基础上可以提供统一的运行模型（服务治理、负载均衡等）和通信模型（HTTP协议、highway协议），让治理和传输变成了与业务代码形态无关的公共能力。这一切都是Java-Chassis原生自带的，不需要用户再去手动集成任何其他的框架了。

![](/img/blog/java-chassis-2-0-major-feature-preview/java-chassis-architecture.png)

### 强类型内核——运行时参数与契约声明完全一致

我们还可以更进一步地设想一下，因为契约的存在，Java-Chassis是可以在运行模型中知道每个微服务有多少个接口，各个接口的url和参数，以及对象类型参数的内部详细字段信息的。因此如果用户想要基于Java-Chassis开发一些比较高阶的，需要依赖这些接口细节信息的治理逻辑，契约信息是可以让事情变得更简单的。  
但要想做到比较理想化的状态，Java-Chassis需要确保运行时在框架里传输的参数类型确实是契约声明的参数类型。大家可以用Java-Chassis 1.3版本写个[EdgeService][]网关服务，再写个后台demo服务做个实验。通常我们是不会在网关服务引用后端服务的接口jar包的，这也意味着网关服务的classpath里找不到后端服务的参数类型。但Java-Chassis会根据契约里面记录的信息（`x-java-class`字段）去搜索网关服务的classpath。**如果在EdgeService网关服务里找不到这个参数类型（类名和包名得完全匹配上），框架会根据后端服务契约的记载去动态生成这样一个Java类型，以确保当请求经过EdgeService的运行模型时，里面的参数类型就是契约里面记录的类型。**可以在EdgeService的handler链里面打个断点，比如打在`org.apache.servicecomb.core.Invocation#next`方法里，这里是handler链遍历执行的必经之路。关注一下`Invocation`的`getArgs`方法返回的对象数组，这里面按照后端服务契约的声明记录了请求参数。你会发现里面的参数顺序和类型都和契约里面记载的一模一样。***这就是Java-Chassis所说的“强类型内核”——保证Java-Chassis框架里传输的参数数据是强类型的，与契约声明完全一致***。

### 促进系统接口规范性

此外，这种框架上的约束可以促使用户设计出更符合REST风格的微服务接口。虽然完成一份课程设计作业可能完全不需要你去关注服务的接口设计的好坏，但这在大型生产系统中却是一个要命的课题——大公司的IT系统通常都比较庞大，而且可能被划分为几个不同的系统交给不同的部门（或承包商）开发，这些系统之间有着各种交互逻辑。如果接口设计得太烂的话，要理解这些接口的行为都是个难题。随之而来的问题还有部门间沟通成本的提高：如果你想确定这么一个接口应该如何调用，最方便的方式当然是去找它的开发或者维护者当面询问了。但其实对于程序开发而言，语音交流是很低效的，而且信息得不到沉淀，有多少人要对接你的系统，你就会被询问多少次。至于接口文档？老实说一个团队都把代码维护到这个份上了，你也很难指望他们写的文档有多好，很大可能是文档已经完全过时，或者根本不存在。即使上头有整改的命令下来，让部门梳理接口文档，花大力气整出来的资料也很有可能在一段时间后再次陷入过时的状态……文档不存在你还知道找人问，文档要是过时了，会埋下多大坑呢？这个只能看你的运气了。  
笔者曾经见识过不同公司、部门的业务系统，它们都免不了有一些放飞自我的接口……举个简单的例子，接口单一性原则很容易随着系统的膨胀而被打破。你以为只是往里面不停地加新参数吗？实际情况可能更糟糕！有些开发者使用的微服务开发框架具备多态序列化能力，也就是说他们可以把微服务接口参数定义为一个抽象类、接口或者Object，然后让业务代码根据实际传入参数的类型来判断执行哪种业务逻辑。这种代码一旦出现问题你很难分析，找开发都没用，搞不好开发已经换过几批了，这个时候程序的行为只有在运行起来时才能知道，你已经无法通过走读代码分析业务系统的逻辑了。BTW，一些使用Servlet开发风格的旧系统通常也会有类似的问题。  
等到你发现这个系统已经没救了的时候……你也只能捏着鼻子忍下来，否则一堆人会找你反馈接口兼容性问题。在很多业务场景下，维持对外接口的兼容性是更加不能触犯的原则。
**要想解决这个问题，成本最低的方式当然是从源头下手**，一开始就别写烂接口，就不会有那么多的幺蛾子了。Java-Chassis要原生支持OpenAPI规范的契约，就要求用户写出形式上足够规范的接口，否则服务启动就会在[框架自动生成契约][隐式契约]的时候报错。接口形式上没什么大问题，写出烂接口的可能性自然也会降低了。天生自带一份Swagger契约文档的框架也可以让开发人员在跟其他部门的人打交道的时候少费些口舌，与代码同源的接口文档在可信度上也比手写的文档要高得多。

## 硬币的反面

然而成也萧何败萧何，Java-Chassis这种基于契约的设计虽然给内部各组件的归一化设计带来了诸多便利，也让它的治理扩展能力有更多的遐想空间，但是它确实对用户的接口设计多了一些限制。据我观察，这也是大部分用户上手Java-Chassis的最大难点。此外还有两点比较容易被人诟病。

### 网关转发接口行为变化

Java-Chassis的这一强类型内核也让框架的行为在某些场景下变得有些令人迷惑。举个例子，部分用户可能在后端服务接口参数里面做一些特别的设置，像是在属性声明的时候给一个默认值，如下：
```java
public class Person {
  // 给name属性赋一个默认值"Bob"
  private String name = "Bob";

  // getter/setter & toString omitted
}
```
微服务接口逻辑如下：
```java
  @PostMapping(path = "/greeting")
  public String greeting(@RequestBody Person person) {
    return "Hello, " + person.getName();
  }
```
在直接调用后端服务接口时一切看上去都是符合预期的：
- 带name调用接口：

  ```
  > $ curl -XPOST -H 'Content-Type: application/json' \
  > 'http://localhost:8080/provider/v0/greeting' \
  > -d '{"name":"Alice"}'
  "Hello, Alice"
  ```

- 不带name调用接口：

  ```
  > $ curl -XPOST -H 'Content-Type: application/json' \
  > 'http://localhost:8080/provider/v0/greeting' \
  > -d '{}'
  "Hello, Bob"
  ```

但如果通过EdgeService网关服务调用后端服务，效果就不是这样了：
```
> $ curl -XPOST -H 'Content-Type: application/json' \
> 'http://localhost:8000/rest/provider/v0/greeting' \
> -d '{}'
"Hello, null"
```
也许你已经意识到问题出在哪里了：*EdgeService网关是没有依赖后端服务的接口jar包的，在classpath中找不到`Person`类的情况下，Java-Chassis会根据后端服务契约的描述动态生成一个`Person`类。框架生成的`Person`类只会有契约所声明的属性及其getter/setter方法（也就是标准的Java Bean），至于用户在后端服务写的属性初始化赋值代码，框架当然是不知道的。*所以在EdgeService网关里，`Person`类大概长这样：
```java
public class Person {
  private String name;

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }
}
```
EdgeService转发给后端的body也就变了，它接到的请求body是`{}`，但在EdgeService内部的运行模型经历一轮反序列化和序列化后，转发给后端服务的body是`{"name":null}`这样的。

类似的场景还有在后端服务接口参数上打上Jackson的相关注解来修改参数的序列化行为等。

> 这个现象也不好说是Java-Chassis的问题，因为定义微服务接口的时候最好是不要让自己的接口依赖于“空”值或者`null`值这种概念，否则很容易出问题。不过现象确实有点反直觉，不少人以为框架里面传的是像`String`或者`byte[]`这种类型的原始json串，但实际上却不是这样。

### 潜在的性能问题

在一些旧系统改造为微服务的项目里，第一步通常是把一个单体系统切换到使用Java-Chassis开发，此时的一个“微”服务通常很大，单个服务有着大量的接口和参数定义。对于后端服务之间的调用场景可能还好，因为客户端服务一般会引入服务端的接口定义jar包，不需要客户端在加载服务端契约的时候去动态生成Java类型了。而对于EdgeService网关而言，每加载一个版本的后端服务契约，就需要生成大量的动态Java类（Java-Chassis 1.x版本是一个微服务版本对应一个`ClassLoader`实例的）。由于客户端，无论是普通的后端服务还是EdgeService，都需要将一个微服务版本的契约一次性完全加载完成才能做调用，服务接口太多的话加载会很消耗时间。  
某些用户在使用EdgeService时还会长时间运行不升级不重启，`ClassLoader`实例太多了占用的内存也会变多，最终导致EdgeService网关的堆内存消耗殆尽，系统响应变慢甚至OOM，要解决这种问题只能重启微服务实例。

## Java-Chassis 2.0：弱类型内核

为了改进强类型内核方案下Java-Chassis的使用不便，ServiceComb-Java-Chassis开发团队对Java-Chassis的核心机制进行了一番重构，为了区别于旧版本，新版本的Java-Chassis被称为“弱类型内核”版本。

> 强类型内核版本对应于Java-Chassis的[1.3.x分支][]，版本是 1.3.x。弱类型内核版本对应于Java-Chassis的[master分支][]，版本是 2.0。
> *注意，这里讨论的场景是REST传输方式。*

### 核心原理变更

在2.0版本中，consumer端服务在加载provider端微服务契约时仍然会根据`x-java-class`字段去classpath里查找是否有现成的类型，如果找得到则仍然沿用找到的参数类型；**找不到声明的参数类型时，框架不会根据契约描述去动态生成参数Java类型，而是依赖Jackson的反序列化能力尝试将其转换为`Object`类型数据。实际得到的参数值，依照原始json串的格式，可能是`LinkedHashMap`/`String`/primitive type等。**使用过Jackson的朋友可以把这看成是`ObjectMapper`的原生能力，指定将json串反序列化为`Object`类型。也就是说，Java-Chassis减弱了契约对于运行时实际参数类型的约束力，以此得到了更好的契约加载性能、更小的内存占用量，以及某些使用场景下更符合“直觉”的使用体验。

对于后端微服务之间的调用场景，如果consumer端加载了provider端的接口jar包，那么从1.3版本升级到2.0版本可能对性能和内存占用没有多大的影响。而对于EdgeService调用后端服务，或者consumer端使用的参数类型与provider端声明的不一致（总之就是consumer端的classpath里找不到provider声明的参数类型）的场景，由于consumer端不再根据契约动态生成参数和接口类型，加载服务端契约的耗时和内存消耗量都会减少——契约对于consumer端的框架而言更像是一份“参考文档”了。

除此之外，EdgeService网关的请求转发行为看上去也更像是原样转发了。还是以上文所举的场景为例，如果发给网关的请求body是`{}`，那么EdgeService会将其反序列化为一个空的`LinkedHashMap`，经过其内部的handler链再转发给后端服务，后端服务接收到的请求body仍然是`{}`。

对于一些特别的场景，例如基于Java-Chassis二次开发业务调用系统，请求转发和调用操作很可能是使用通用逻辑统一执行的，代码形式与普通业务代码不同。由于强类型内核要求传参类型与契约声明一致，在实现的复杂度上会高一些。而对于弱类型内核而言，使用LinkedHashMap等类型传参也是可以的，不强制要求加载契约声明的参数类型，实现起来会更容易。

### 升级兼容性

目前看来，Java-Chassis从1.3升级到2.0版本的主要变更在于传输方面的内部逻辑，从用户体验上来看区别不大。  
如果系统使用Java-Chassis的方式比较常规，那么升级时碰到不兼容问题的概率就比较小，可能需要注意的地方包括，在EdgeService的handler、filter里面拿到的`Invocation`携带的REST请求参数和返回值的类型会有变化。  
如果业务系统以深度定制的方式使用了Java-Chassis，那么在升级过程中可能会发现一些底层的类、方法的变动。

在新旧版本混用的问题上，对于REST传输方式，因为其底层传输实现是HTTP协议+json格式body体，新旧版本的Java-Chassis在这方面的实现都是符合业界标准的，可以将基于不同版本的Java-Chassis开发的微服务组合在一起相互调用。但对于highway传输方式，目前来看highway会进行重新的设计和实现，因此2.0版本和1.x版本的Java-Chassis无法以highway的方式互相调用。

[ServiceComb-Java-Chassis]: https://github.com/apache/servicecomb-java-chassis
[框架架构]: https://docs.servicecomb.io/java-chassis/zh_CN/start/architecture.html
[隐式契约]: https://docs.servicecomb.io/java-chassis/zh_CN/build-provider/code-first.html "使用隐式契约"
[EdgeService]: https://docs.servicecomb.io/java-chassis/zh_CN/edge/by-servicecomb-sdk.html
[1.3.x分支]: https://github.com/apache/servicecomb-java-chassis/tree/1.3.x
[master分支]: https://github.com/apache/servicecomb-java-chassis/tree/master
