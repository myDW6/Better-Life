# @Resource和@Autowired的区别



相同点：
spring中都可以用来注入bean，同时都还可以作为注入属性的修饰。在接口仅有单一实现类时，两个注解的修饰效果是相同的，他们之间可以相互替换，不影响使用。



不同点：

- @Resource是Java的注解，@Resource有两个主要的属性，分别是name和type；spring对@Resource注解的name属性解析为bean的名字，而type属性则解析为bean的类型。所以如果使用name属性，则使用byName的自动注入策略。而使用type属性，则是使用byType的自动注入策略。如果既不指定name属性也不指定type属性，这时将通过反射机制使用byName的自动注入策略。
- @Autowired是spring的注解，是spring2.5版本引入的，@Autowired注解只根据type进行注入，不会去匹配name。如果涉及到根据type无法辨别的注入对象，将需要依赖@Qualifier或@Primary注解一起来修饰。



推荐使用：@Resource注解在字段上，这样就不用写setter方法了，并且这个注解是属于J2EE的，减少了与spring的耦合。这样代码看起就比较优雅

常用的是@Autowired注解，因为平时百分之九十以上都是一个接口仅有一个实现类，所以用@Autowired比较方便，当然这种情况用两个中的哪一个都一样，没什么太大的区别，纯属个人习惯。但是如果是生产环境的话，架构师对这方面要求有比较严格的情况下，会让大家使用@Resource(name="xxx")这种，因为在生产环境就会要求更高的效率问题，这样使用效率是最高的，类似于根据id查询数据库一样，效率高一些。





