Springboot Http->Https

```bash
keytool 
-genkey  创建一个新的密钥。
-alias tomcat  表示 keystore 的别名。
-storetype PKCS12   指定密钥仓库类型
-keyalg RSA  证书的算法名称，RSA是一种非对称加密算法
-keysize 2048  证书大小
-keystore keystore.p12  生成的证书文件的存储路径
-validity 3650  证书的有效天数

keytool -genkey -alias tomcat  -storetype PKCS12 -keyalg RSA -keysize 2048  -keystore keystore.p12 -validity 3650
```

```java
将keystore.p12放到项目根目录下
配置文件:
server:
  ssl:
    key-store: keystore.p12
    key-store-password: 123456
    keyStoreType: PKCS12
    keyAlias: tomcat
  port: 10001
配置类:
@Configuration
public class HttpsConfig {

    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory(){
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory(){
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                securityConstraint.addCollection(collection);
                context.addConstraint(securityConstraint);
            }
        };
        tomcat.addAdditionalTomcatConnectors(initiateHttpConnector());
        return tomcat;
    }

    private Connector initiateHttpConnector (){
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setScheme("http");
        //Connector监听的http的端口号
        connector.setPort(10000);
        connector.setSecure(false); //为true可以同时支持http和https访问
        //监听到http的端口号后转向到的https的端口号
        connector.setRedirectPort(10001);
        return connector;
    }
}


```

​	

chrome : thisisunsafe 操作

option cmd u快速查看 

