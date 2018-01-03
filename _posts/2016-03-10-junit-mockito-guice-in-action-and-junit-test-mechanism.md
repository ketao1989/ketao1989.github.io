---
layout: post
title: "基于JUnit Mockito Guice测试实践及JUnit运行机制浅析"
date: 2016-03-10 21:38:06 +0800
comments: true
categories: 
 - JUnit
 - Guice
 - Java
---

* content
{:toc}

## <a id="Introduction">前言</a>

对于一个合格的项目来说,单元测试是必不可少的一部分。尤其是，如果对于TDD思想来说，单元测试则是整个项目开发的基石。对于Javaer来说,Junit 是一个基础的单元测试框架.对于Spring框架来说, 其实现了JUnit的接口来直接支持单元测试, 但是对于Guice来说, 其定位为一个轻量级的依赖注入框架, 所以这些就需要自己来实现. 此外, 对于依赖外部的接口服务的应用, 我们在测试的时候, 是不希望其服务的不稳定导致我们单测失败; 此外, 对外部接口进行屏蔽, 也可以达到对每一个外部服务返回逻辑分支的覆盖.

因此，本文基于`JUnit + Mockito + Guice`框架说，然后基于简单地实例来说明JUnit的运行机制。

## <a id="Action"> 单元测试实践</a>

Guice 作为一个注入框架，和Spring相比，并没有什么特别的。使用Guice介绍单元测试，其一是项目开发中使用Guice，其二由于我们需要去自己实现JUnit接口来支持Guice，能够更深入地了解JUnit结构。Mockito 是java中我比较喜欢的mock工具，当然，我也没有用过其他的。O(∩_∩)O~

首先，需要引入两个实现：

``` java

/**
 * @author tao.ke Date: 16/3/9 Time: 下午4:24
 */
public class GuiceJUnitRunner extends BlockJUnit4ClassRunner {

    private Injector injector;

    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Inherited
    public @interface GuiceModules {
        Class<?>[] value();
    }

    @Override
    public Object createTest() throws Exception {
        Object obj = super.createTest();
        injector.injectMembers(obj);
        return obj;
    }

    public GuiceJUnitRunner(Class<?> klass) throws InitializationError {
        super(klass);
        Class<?>[] classes = getModulesFor(klass);
        injector = createInjectorFor(classes);
    }

    private Injector createInjectorFor(Class<?>[] classes) throws InitializationError {
        Module[] modules = new Module[classes.length];
        for (int i = 0; i < classes.length; i++) {
            try {
                modules[i] = (Module) (classes[i]).newInstance();
            } catch (InstantiationException | IllegalAccessException e) {
                throw new InitializationError(e);
            }
        }
        return Guice.createInjector(modules);
    }

    private Class<?>[] getModulesFor(Class<?> klass) throws InitializationError {
        GuiceModules annotation = klass.getAnnotation(GuiceModules.class);
        if (annotation == null) {
            return new Class[0];
        }
        return annotation.value();
    }
}

```

GuiceJUnitRunner 实现了JUnit框架`BlockJUnit4ClassRunner`接口，Runner是JUnit的核心，所有测试的运行，最后都是Runner来衔接运作起来的。在`JuiceJUnitRunner`中，我们根据annotation来初始化Guice环境需要的一些初始化配置，拦截器等；此外，熟悉Guice的同学知道，其有一个非常不好的体验，就是需要在最外层手动注入实例调用服务，这里我们希望可以直接注入service实例，因此在Runner初始化时完成这一动作。

<!-- more -->

``` java

/**
 * @author tao.ke Date: 16/3/9 Time: 下午4:30
 */
@RunWith(GuiceJUnitRunner.class)
@GuiceJUnitRunner.GuiceModules({ GuiceModule.class})
public class BaseTest {

    @Before
    public void init() {
        MockitoAnnotations.initMocks(this);
    }
}

// 全局初始化，加载数据库配置，注入拦截器等
public class GuiceModule implements Module {

    public void configure(Binder binder) {

       //binder.bind(GuiceBiz.class);

    }
}
```

`BaseTest`是一个基础的测试类，完成所有测试共同的工作。各个测试类可以继承该基类，然后就可以最简单的关注业务逻辑的测试。比如，下面这个测试类：

``` java

public class GuiceUnitSampleTest extends BaseTest {

    @Inject
    private GuiceBiz guiceBiz;

    @Test
    public void testConcatString(){
        //就是一个简单地测试而已
        System.out.println(guiceBiz.concatString("ktcoder","hello,"));
        Assert.assertEquals("hello,ktcoder",guiceBiz.concatString("ktcoder","hello,"));
    }
}

```

接下来，来一个复杂的测试case，这里，我们调用的外部的服务，比如数据库查询，或者其他RPC服务等等，但是我们只想测试自己的业务分支是否运行正常。

``` java

public class GuiceUnitMockSampleTest extends BaseTest{

    @InjectMocks
    @Inject
    private GuiceBiz guiceBiz;

    @Mock
    private GuiceServiceImpl guiceServiceImpl;


    @Test
    public void testGetRoleAdmin(){

        Mockito.when(guiceServiceImpl.getRole(Matchers.anyString())).thenReturn("admin");

        Assert.assertEquals("hello Admin",guiceBiz.helloRole("test"));

    }

    @Test
    public void testGetRoleGuest(){

        Mockito.when(guiceServiceImpl.getRole(Matchers.anyString())).thenReturn("");

        Assert.assertEquals("hello Guest",guiceBiz.helloRole("test"));

    }
}

@Singleton
public class GuiceBiz {

    @Inject
    private GuiceServiceImpl guiceServiceImpl;

    public String helloRole(String name){

        String res = guiceServiceImpl.getRole(name);

        if (StringUtils.isBlank(res)){
            return "hello " + "Guest";
        }

        return "hello " + "Admin";
    }
}

```

从上面的case可以看到，不管外部的GuiceServiceImpl是如何处理的，是否服务可靠都无所谓。我们只需要关注自己的业务逻辑，然后mock掉外部的接口，这样就可以更好地覆盖各个逻辑分支了。

## <a id="Flow"> JUnit 运行浅析</a>

一般在开发的时候，我们都使用idea 或者 eclipse 来运行测试用例，那么对于可运行的java代码，必然有一个main方法入口。

###  idea JUnit 主方法入口

在 idea 中存在一个`JUnitStarter`类，该类就有一个main方法，也就是说，当我们在idea ide工具中点击执行test方法时，实际上执行的就是这个类的入口方法。

main 方法很简单，就是准备输入输出流，一些参数准备，然后调用`prepareStreamsAndStart` 方法：

``` java
private static int prepareStreamsAndStart(String[] args, final boolean isJUnit4, ArrayList listeners, SegmentedOutputStream out, SegmentedOutputStream err) {

    PrintStream oldOut = System.out;
    PrintStream oldErr = System.err;
    int result;
    try {
      System.setOut(new PrintStream(out));
      System.setErr(new PrintStream(err));
      IdeaTestRunner testRunner = (IdeaTestRunner)getAgentClass(isJUnit4).newInstance();
      testRunner.setStreams(out, err);
      result = testRunner.startRunnerWithArgs(args, listeners);
    }
    // ......
    return result;
  }

```

因为我们使用的是JUnit4，所以会接着调用 `JUnit4IdeaTestRunner`类，在这个类中将直接和JUnit的相关接口交互，调用相关方法，进入到具体的JUnit测试中。该类首先构造JUnit工具中需要的Request类，通过Request可以获取Runner实例，然后开始运行测试case。
比如我们这里需要运行的时Test类中得一个方法，所以这里列出对应的处理逻辑代码：

``` java
// suiteClassNames[0]="io.github.ketao1989.guice.GuiceUnitSampleTest,testConcatString"
public static Request buildRequest(String[] suiteClassNames, final String name, boolean notForked) {
    if (suiteClassNames.length == 0) {
      return null;
    }
    Vector result = new Vector();
    for (int i = 0; i < suiteClassNames.length; i++) {
        String suiteClassName = suiteClassNames[i];
        int index = suiteClassName.indexOf(',');
        if (index != -1) {
            final Class clazz = loadTestClass(suiteClassName.substring(0, index));
            final String methodName = suiteClassName.substring(index + 1);
            final RunWith clazzAnnotation = (RunWith)clazz.getAnnotation(RunWith.class);
            final Description testMethodDescription = Description.createTestDescription(clazz, methodName);
            final Request request = getParameterizedRequest(name, methodName, clazz, clazzAnnotation);/反射构造runner及request
            if (request != null) {
                return request;
            }
        }
    }
}
```

构造完了Request，然后就可以通过new 一个 JUnitCore对象来运行request中得runner了。

``` java

private Result startRunnerWithArgs(String[] args, ArrayList listeners, String name, boolean sendTree){

    final Request request = JUnit4TestRunnerUtil.buildRequest(args, name, sendTree);
    final Runner testRunner = request.getRunner();
    Description description = getDescription(request, testRunner);
    //.......
    final JUnitCore runner = new JUnitCore();
    return runner.run(testRunner);

}

```

在上面的`startRunnerWithArgs`方法中会设置一些监听器负责收集运行成功失败的结果，开始结束运行等，为了了解整个运行流程，就不分析了。

###  Request 构建 Runner

从上面的代码中看到，idea 是通过buildRequest来获取 Runner的。实际上，Request 就是一个RunnerBuilder构建工具，默认有三种实现：`FilterRequest`、`ClassRequest`和`SortingRequest`，顾名思义了，核心自然是 ClassRequest。

``` java

public class ClassRequest extends Request {
    private final Object runnerLock = new Object();
    private final Class<?> fTestClass;
    private final boolean canUseSuiteMethod;
    private volatile Runner runner;
    // ......
    @Override
    public Runner getRunner() {
        if (runner == null) {
            synchronized (runnerLock) {
                if (runner == null) {
                    runner = new AllDefaultPossibilitiesBuilder(canUseSuiteMethod).safeRunnerForClass(fTestClass);
                }
            }
        }
        return runner;
    }
}

```

从`getRunner`方法中可以知道这里根据需要测试的类 fTestClass 来构造runner. 

  >> Notes:这里使用fTestClass明明变量，竟然是因为在idea的调用代码中使用`field = ClassRequest.class.getDeclaredField("fTestClass");`,所以无法更改成正常的testClass.

  AllDefaultPossibilitiesBuilder类是一个builder的总入口，其会根据配置选择正确的RunnerBuilder构建Runner对象：

``` java

public class AllDefaultPossibilitiesBuilder extends RunnerBuilder {
    private final boolean canUseSuiteMethod;

    public AllDefaultPossibilitiesBuilder(boolean canUseSuiteMethod) {
        this.canUseSuiteMethod = canUseSuiteMethod;
    }

    // 从这个方法可以看出构造器的优先级，构造器会一个一个尝试，直到找到正确地为止，一般使用注解方式指定Runner
    // 所以这里介绍annotatedBuilder.
    @Override
    public Runner runnerForClass(Class<?> testClass) throws Throwable {
        List<RunnerBuilder> builders = Arrays.asList(
                ignoredBuilder(),
                annotatedBuilder(),
                suiteMethodBuilder(),
                junit3Builder(),
                junit4Builder());

        for (RunnerBuilder each : builders) {
            Runner runner = each.safeRunnerForClass(testClass);
            if (runner != null) {
                return runner;
            }
        }
        return null;
    }

    // 根据RunWith 注解获取Runner类，然后反射实例化一个对象出来，即：
    // runnerClass.getConstructor(Class.class).newInstance(testClass);
    protected AnnotatedBuilder annotatedBuilder() {
        return new AnnotatedBuilder(this);
    }
    // ......
}

```

### JUnitCore 运行 Runner

JUnitCore 是对 JUnit的Runner类的一个Fecade，兼容3.x和4.x的不同runner的运行，同时，可以在command中执行命令`java org.junit.runner.JUnitCore TestClass1 TestClass2` 运行测试用例。

此外，作为Facade来说，其在调用runner的run方法时，会再方法开始和结束的时候，触发相应地监听器，发布相关通知事件。如下：

``` java
    //通过命令运行，这就是入口
    public static void main(String... args) {
        Result result = new JUnitCore().runMain(new RealSystem(), args);
        System.exit(result.wasSuccessful() ? 0 : 1);
    }

    // 测试case真正的运行方法
    public Result run(Runner runner) {
        Result result = new Result();
        RunListener listener = result.createListener();
        notifier.addFirstListener(listener);
        try {
            notifier.fireTestRunStarted(runner.getDescription());
            runner.run(notifier);
            notifier.fireTestRunFinished(result);
        } finally {
            removeListener(listener);
        }
        return result;
    }
```

  >> Notes: 这里特别说明下通知notify的实现，代码中创建一个`SafeNotifier`类，来发布一个通知，但是我们知道监听器不可能只有一个，因此，对于外部的调用来说，必须调用`SafeNotifier.run`方法，这个方法会依次遍历listeneres监听器列表，然后调用对应的方法触发监听事件。
  
在实现中，我们自己自定义了Runner方法为`GuiceJUnitRunner`继承了`ParentRunner`,所以，我们直接看其run方法的实现：

``` java

    @Override
    public void run(final RunNotifier notifier) {
        Statement statement = classBlock(notifier);
        statement.evaluate();
    }

    // 这里只分析最简单的方法调用，before和after是单独实现了Statement子类，执行时根据
    // 先后属性来决定先执行test方法还是先运行before class工作
    protected Statement classBlock(final RunNotifier notifier) {
        Statement statement = childrenInvoker(notifier);
        if (!areAllChildrenIgnored()) {
            statement = withBeforeClasses(statement);
            statement = withAfterClasses(statement);
            statement = withClassRules(statement);
        }
        return statement;
    }    

    // statement其实就是对需要执行动作的一个封装，其真正的动作就是调用evaluate方法
    protected Statement childrenInvoker(final RunNotifier notifier) {
        return new Statement() {
            @Override
            public void evaluate() {
                runChildren(notifier);
            }
        };
    }


    // 下面两个是实现很有意思。对于RunnerScheduler来说，字面意思就是调度，因此，其是针对不同的调用策略，执行动作的接口
    // 当然这里，我们直接调用运行，然后执行完所有测试的case方法之后，调用finished动作。
    private void runChildren(final RunNotifier notifier) {
        final RunnerScheduler currentScheduler = scheduler;
        try {
            for (final T each : getFilteredChildren()) {
                currentScheduler.schedule(new Runnable() {
                    public void run() {
                        ParentRunner.this.runChild(each, notifier);
                    }
                });
            }
        } finally {
            currentScheduler.finished();
        }
    }

    private volatile RunnerScheduler scheduler = new RunnerScheduler() {
        public void schedule(Runnable childStatement) {
            childStatement.run();
        }

        public void finished() {
            // do nothing
        }
    };

```

其实，根据 `ParentRunner`类的代码，基本上已经能够很清晰的明白了运行测试case的流程。其流程就是：首先，我们拿到测试类中所有需要执行的test方法；然后，把测试类中需要执行的test方法封装为一个执行动作，这个动作就是分别运行各个子方法，这些方法的执行则是根据RunnerScheduler内部的调度执行策略来的。

所以，根据上面的代码，发现其实有两个方法，是需要子类去实现的，其一是怎么获取测试类中需要test的方法，其二就是如果执行这些方法，之所以会如此，是因为`ParentRunner`这个类除了针对测试类里地方法，还针对测试包下面各种测试类，所以不同的情形，其处理逻辑是不一样的。针对包Suite，其会继续遍历解析，知道拆分成测试类，对应下面的测试方法FrameworkMethod，然后调用针对测试类的处理逻辑执行。

### GuiceJUnitRunner 继承 BlockJUnit4ClassRunner

如上所述，`ParentRunner` 关注的两个方法，需要在子类去实现。

获取需要test的方法，这个显然 BlockJUnit4ClassRunner 就可以给出具体实现，即根据`@Test`注解来获取测试类中需要执行test的方法，如下：

``` java

    // getTestClass 获取的就是需要执行的测试类
    protected List<FrameworkMethod> computeTestMethods() {
        return getTestClass().getAnnotatedMethods(Test.class);
    }
```

而，另外一个问题就是如何执行测试方法。其实现也很简单，就是首先构造测试类实例，然后根据具体的测试方法，依次通过menthod.invoke调用执行就可以了。而针对构造类实例，由于Guice获取方式是通过注入，所以是需要覆盖重写的。因此，我们看父类BlockJUnit4ClassRunner代码：

``` java

    @Override
    protected void runChild(final FrameworkMethod method, RunNotifier notifier) {
        Description description = describeChild(method);
        if (isIgnored(method)) {
            notifier.fireTestIgnored(description);
        } else {
            // runLeaf 核心动作就是 statement.evaluate()
            runLeaf(methodBlock(method), description, notifier);
        }
    }

    // 下面createTest就是需要实例化一个测试类对象了，然后再构造一个 methodInvoker 对象，methodInvoker
    // 从名字就可看出来其作用就是为了反射执行method方法。
    protected Statement methodBlock(FrameworkMethod method) {
        Object test = new ReflectiveCallable() {
                @Override
                protected Object runReflectiveCall() throws Throwable {
                    return createTest();
                }
        }.run();
        // 实际上就是方法执行 method.invoke(test, params)
        Statement statement = methodInvoker(method, test);
        // ......
        return statement;
    }

```

ok，整个流程基本上就已经分析清楚了。此外，需要注意的是，对于继承实现Runner来说，其构造函数是带有一个testClass的参数的。例如：

``` java

    public GuiceJUnitRunner(Class<?> klass) throws InitializationError {
        super(klass);
        Class<?>[] classes = getModulesFor(klass);
        injector = createInjectorFor(classes);//创建一个injector，在createTest中，使用该injector来注入类中得各种依赖
    }

```

## <a id="Problem_and_Summary">遗留问题和总结</a>

本文主要是基于Guice+JUnit+Mockito来构建一个单元测试实践，然后简要分析了JUnit的源码运行逻辑，但是并没深入到源码的细节。但是不得不说，JUnit不仅仅是一个非常优秀的测试框架，而且其代码结构和实现也非常的优美和可扩展，非常值得学习和借鉴。

后面有机会，会更深入分析其内部的实现细节，以及一些非常值得借鉴的代码设计思想。
