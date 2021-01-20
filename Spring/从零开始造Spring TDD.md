# 从零开始造Spring TDD

版本号:Spring3.2.18(IOC和AOP很完整,支持注解)

## IOC

![image-20201122220733976](https://gitee.com/shao_dw/pic/raw/master/upload/image-20201122220733976.png)

非常朴素的思想:一个java类,稍微声明下,A依赖B,B依赖C,C依赖D,容器就会帮我们把依赖设置好,这个容器就是Spring容器,Spring容器会实例化业务类,装配.

![image-20201122221056015](https://gitee.com/shao_dw/pic/raw/master/upload/image-20201122221056015.png)

xml 注解 和 JavaConfig

## AOP

![image-20201122221159710](https://gitee.com/shao_dw/pic/raw/master/upload/image-20201122221159710.png)

左边是功能性需求,右边是非功能性需求,我们会把非功能性需求也包装成模块让其被调用,非功能性需求会在业务模块多处被调用,不利于拓展.  假设我要修改日志的部分功能, 那么但是依赖了日志的用户管理,这些都需要修改.  那么如果我们把从用户管理,订单管理找日志,安全这个过程的代码变成让日志安全这些代码去找我们的业务模块,就简单多了 而这也正是aop所做的事.

![image-20201122221628454](https://gitee.com/shao_dw/pic/raw/master/upload/image-20201122221628454.png)

这是正交的威力. 业务代码和非业务代码正交,非业务代码之间也正交.

我们利用容器,将日志 安全这些代码织入到业务代码之中. Cglib 和 JDK动态代理  , 运行时进行代码织入, AspectJ在编译器织入.



## 使用TDD

程序员基本技能

1. 写测试用例
2. 重构(没有安全网做保障,就有可能破坏现有功能, 安全网其实就是我们的测试用例)  13.5

