# 从零开始造Spring笔记(单元测试-Junit)

##  单元测试

开发人员编写的小段代码.用于检测被测代码的一个明确功能的小模块是否正确

+ 判断某个函数和类的行为
+ 白盒测试
+ 开发人员受益

![image-20201121160718037](https://gitee.com/shao_dw/pic/raw/master/upload/image-20201121160718037.png)

Junit4 提供的注解@Test 想想Spring测试里面runwith()

按照规定写代码,剩下的交给junit框架,框架找到测试用例,然后执行测试用例,通过Assert来比较实际值和期待值是否一样.



## 当测试用例很多,成百上千个怎么办?



现在有一个AllTest类,这个测试类一执行,会执行V1AllTests类, V2AllTest类...通过这种方式来组织所有的测试用例.提供了一种粒度,单个包下的测试用例,单个测试用例.

```java
**
 * Junit提供Suite套件 可以一层层组织测试用例,组织成一棵树
 */
@RunWith(Suite.class)
@Suite.SuiteClasses({Test1.class, Test2.class})
public class AllTests {

    @Test
    public void m1(){
        System.out.println("主测试");
    }
}

@RunWith(Suite.class)
@Suite.SuiteClasses(Test2.class)
public class Test1 {

    @Test
    public void  m(){
        System.out.println("去运行类2下的测试");
    }
}

public class Test2 {
    @Test
    public void  m(){
        System.out.println("类2下的测试");
    }
}
//组织结果如下图
AllTests以套件的形式运行Test1.class 和 Test2.class 所以Test1和Test2平级  Test1中运行Test2 
```

![image-20201121180750722](https://gitee.com/shao_dw/pic/raw/master/upload/image-20201121180750722.png)

这种suite组织方式其实就是设计模式中的组合模式.

## Junit常用的几种断言

```java
Assert.assertEquals();
Assert.assertTrue();
Assert.assertNotNull();
Assert.assertArrayEquals();
```

数值,对象,变量是否一样,是否为true,是否为null, 数组相等.

## 对Excetion进行测试

```java
//方法1
@Test
public void testEx(){
    Cal cal = new Cal();
    try{
        int result = cal.evalute("10/0");
    }catch (ArithmeticException e){
        //代码应该进这个分支
        return;
    }
    //若走到这里说明Cal代码有误
    Assert.fail();
}

//方法2  不需要捕获异常 期待抛出异常
    @Test(expected = ArithmeticException.class)
    public void testEx(){
        Cal cal = new Cal();
        int result = cal.evalute("10/0");
    }
```



## 两个特殊方法

```java
@Test
public void test1(){
    System.out.println("test1");
}

@Test
public void test2(){
    System.out.println("test2");
}

@Before
public void before(){
    System.out.println("before");
}

@After
public void after(){
    System.out.println("after");
}
//执行结果:
before
test1
after
before
test2
after
```

## 两个更特殊的方法

@BeforeClass @AfterClass只会在开始和结束时执行一次

## 单元测试的优点

+ 验证行为
  + 保证代码正确性
  + 回归测试,到了项目后期,仍可以增加新功能,修改程序结构,不担心破坏重要功能
  + 给重构带来保障
+ 设计行为
  + 测试驱动迫使从调用者去思考 迫使代码设计成可测试和松耦合的
+ 文档行为
  + 单元测试是文档,精准描述代码行为,如何使用函数和类.



## 单元测试原则

+ 测试代码和被测试代码同等重要,需要同时维护
  + 测试代码不是附属品
  + 不但要重构代码,还要重构单元测试
+ 单元测试一定是隔离的
  + 一个测试用例的结果不能影响其他的测试用例(运行@Before和@After)
  + 测试用例不能相互依赖,应可以以任意顺序执行
+ 单元测试一定是可以重复执行的
  + 不依赖于环境的变化(不依赖时间,时间会流逝, 不依赖机器)
+ 保证单元测试简单性和可读性
+ 尽量对接口进行测试(一个类10个方法,对外暴露3个,只测试对外暴露的三个接口方法)
+ 可以迅速执行
  + 及时的反馈
  + 使用mock对象对db,network进行解耦(假数据假网络 确保代码在内存中执行)
+ 自动化单元测试
  + 集成到build过程



## 使用Mock对象

+ 真实的对象不易构造(难于构造,依赖容器)
  + 比如HttpServlet(只是个接口)必须在Servlet容器(Tomcat, Jetty)中才可以被创建出来
+ 真实的对象很复杂
  + JDBC的Connection 和 ResultSet
+ 真实的对象的行为具有不确定性,难于控制他们的输出或者返回结果
+ 真实的对象有些行为难以触发,比如硬盘已满,网络连接断开(异常和边界条件难以触发)
+ 真实的对象还不存在,依赖的另一个模块还在开发(前后端分离)
+ 使用mock对象替代或冒充真实模块和被测试对象进行交互
  + 开发可以精确的定制期待的行为(返回1就返回1,不会返回2)
+ 对TDD的支持
  + 帮助发现对象的职责和角色
  + 对接口编程而不是对实现编程

```java
//要对这个parse方法进行测试 如何进行
public class UrlParser {
    public void parse(HttpServletRequest httpServletRequest){
        String param = httpServletRequest.getParameter("");
        //do something
    }
}
//既然是接口 我就给个实现类 但是这个接口有几十个无用的方法要实现  但是只用到了getParam方法

//使用录制-回放原理 如下
```

![image-20201121183723159](../Library/Application%20Support/typora-user-images/image-20201121183723159.png)

## 对遗留代码的测试

+ 遗留代码不可避免
  + 工作不从第一行代码开始
+ 遗留代码不是坏代码
  + 可以工作的软件/组件
  + 设计和开发时没考虑可测试性
+ 遗留代码难于测试
  + 长久失修,导致业务逻辑难以理解
  + 依赖的资源太多,导致测试无法下手
  + 不敢修改,害怕牵一发动全身

## 处理遗留代码的策略

+ 重构代码,提高可测试性
+ 使用Mock Object解除依赖
+ 测试分解
  + 先写粗粒度的测试代码,再写细粒度的测试代码
  + Package  -> Class -> Method

## 处理遗留代码的步骤

1. 确认要测试的类和函数
2. 解除依赖(important)
3. 编写测试用例
4. 重构代码



```java
public class Account {
    public void transfer(){
        TransactionManger transactionManger = new TransacrionManagerImpl();
        //如果业务逻辑依赖于其他服务,通常会用new或者单元模式来创建,但这会对测试带来困难
        transactionManger.begin();
        //do busioness logic
        transactionManger.commit()
    }
}

//重构
public class Account {
    public void transfer(){
        TransactionManger transactionManger = getTransactionManger();
        transactionManger.begin();
        //do busioness logic
        transactionManger.commit()
    }

    protected TransacrionManagerImpl getTransactionManger() {
        return new TransacrionManagerImpl();
    }
}
//测试之前,创建一个Account的子类,重写getTransactionManger方法,来提供一个mock的实现,mock实现在调用begin和commit时什么都不做
Account acc = new Account(){
	protected TransacrionManagerImpl getTransactionManger() {
        return new MockTransactionManager();
    }
}
//测试业务逻辑 Mock实现会被调用
account.transfer().
```

## 借助IDE的Code Coverage工具

 -- 修改代码的艺术 --