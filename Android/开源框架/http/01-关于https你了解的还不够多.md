# 1. HTTPS 

Android9.0开始强制使用https请求，不仅如此，各大巨头公司都自己的操作系统、浏览器等产品中强制要求使用https。对于开发者来说，可能查看一些文档就能做适配，有很多人并没有真正了解https的原理，包括我自己在之前也没想那么多，只知道它是对数据加密后进行网络传输的，这篇文章使用通俗的语言以及实操示例来一起深入理解https的工作流程。

## 1.1 密码学

说到加密就想到有《密码学》这本书，笔者第一家公司是做加密锁的，对密码学还算比较熟悉，对于https中涉及的密码学，大家需要对下面几点知识有一个概念性的认识：

**对称加密**：对称加密的代表算法有AES、DES等，加密和解密使用同一个密钥（可看作是字符串），其加解密效率高，但是密钥在传递过程中可能有被窃取的风险（加解密双方需要得到同一个密钥）。

**非对称加密**：非对称加密的代表算法有RSA、ElGamal等，其密码分为公钥和私钥，使用公钥加密的数据只有用私钥才能解密，反过来使用私钥加密的数据也只有用公钥才能解密。公钥通常存放在客户端，私钥通常存放在服务器。非对称加密的优点是安全性更高，因为客户端发送给服务器的加密信息只有用服务器的私钥才能解密，因此不用担心被别人破解，但加解密的效率相比于对称加密要差很多。非对称加密还可以用于数字签名，所以通过工具生成的密钥文件中不仅包含公私钥，还包含证书信息。

**数字签名**：使用非对称加密算法，用私钥对数据加密的过程叫做签名，使用公钥对其解密的过程叫做验证签名，数字签名的目的是保证数据的一致性(中途不被篡改)。如果数据非常大，通常使用数字摘要技术(常见的hash算法、MD5等)得到数据的摘要，然后对摘要进行签名。

**数字证书**：数字证书从本质上是一种电子文档，一个文件，里面包含一系列的信息，比如证书颁发者、使用者、签名算法、哈希算法、公钥等，数字证书和签名是两个概念，证书就是一个文件，而签名是通过签名算法对证书加密得到签名信息。如果数字证书被签名，它里面就包含了签名链和签名信息。数字证书文件有多种格式，不同的格式存放的内容不太一样，不同平台可能支持不同格式的证书。

**证书格式（了解一下即可）**
PKCS 全称是 Public-Key Cryptography Standards ，是由 RSA 实验室与其它安全系统开发商为促进公钥密码的发展而制订的一系列标准，PKCS 目前共发布过 15 个标准。 常用的有：

- PKCS#7 Cryptographic Message Syntax Standard
- PKCS#10 Certification Request Standard
- PKCS#12 Personal Information Exchange Syntax Standard

X.509是常见通用的证书格式。所有的证书都符合为Public Key Infrastructure (PKI) 制定的 ITU-T X509 国际标准。

- PKCS#7常用的后缀是： .P7B .P7C .SPC
- PKCS#12常用的后缀有： .P12 .PFX
- X.509 DER编码(ASCII)的后缀是： .DER .CER .CRT
- X.509 PAM编码(Base64)的后缀是： .PEM .CER .CRT
- **.cer/.crt是用于存放证书，它是2进制形式存放的，不含私钥。**
- .pem跟crt/cer的区别是它以Ascii来表示。
- **pfx/p12用于存放个人证书/私钥，他通常包含保护密码，2进制方式**
- p10是证书请求
- p7r是CA对证书请求的回复，只用于导入
- p7b以树状展示证书链(certificate chain)，同时也支持单个证书，不含私钥。


## 1.2 HTTP安全问题

传统http协议传输数据是通过明文，存在着**数据被监听**和**数据被篡改**的风险，为了让数据安全，需要对数据进行加密后传输，下面一步步来看https协议加接密过程是怎样演进的。

**对称密钥安全问题**

网络传输数据追求高效率，如果要对数据进行加解密首先考虑到的就是使用对称加密算法。对称密钥是浏览器还是服务器生成并不重要，重要的是如何让这个密钥只让它们俩知晓，而不被任何监听者知晓呢。比如让浏览器随机生成一个对称加密密钥，用于后续网络请求对数据进行加密，为了让服务端收到加密后的数据对其进行解密，浏览器首先会将对称密钥（明文）发送给服务端。你会发现不管怎么商定，浏览器和网站的**首次通信过程必定是明文的**，这就意味着，按照上述的工作流程始终无法创建一个安全的对称加密密钥。

**使用非对称加密对对称密钥加密**

上述对称密钥传输的安全问题可以通过非对称加密解决。服务端生成非对称加密的公钥和私钥，并将公钥公布出来(发给浏览器)，私钥自己藏好。浏览器如果要访问服务器，时首先**对对称密钥使用公钥加密后传输给服务端**，服务端收到加密后的对称密钥密文使用私钥解密，从而安全的得到浏览器生成的对称密钥。通过这种方式，只有浏览器和服务器首次商定对称密钥时需要使用非对称加密，当服务器拿到对称密钥后双方就可以对通信数据使用对称加密了，避免每次通信使用非对称加解密造成效率低下的问题。

**非对称公钥的安全问题**

上面通过非对称加密保证了对称密钥的安全性，但是真的安全吗？公钥的公开可能导致公钥被篡改，如果浏览器使用了假的公钥对数据进行加密，篡改者就可以使用假私钥解密，然后对数据篡改后通过网站的公钥加密再发给网站，网站收到的就是被篡改后的数据。本来引进非对称加密是为了保证对称密钥的安全，结果又引来了新的密钥安全问题。
导致这个问题的根源就是浏览器并不知道拿到的公钥是否真的是网站的公钥。解决这个问题有一种办法是将世界上所有网站的公钥都预置在操作系统中，这显然是不现实的，这时候**CA机构**就出现了。

## 1.3 CA机构(Catificate Authority)

> CA机构的作用就是对网站的数字证书进行签名，从而让浏览器（客户端）验证数字证书合法且未被篡改，数字证书中的公钥确实是网站的公钥

CA机构专门用于给各个网站签发数字证书，从而保证浏览器可以安全地获得各个网站的公钥。网站的管理员需要向CA机构进行申请，将自己的公钥和网站信息提交给CA机构，CA机构则会使用网站的公钥和一些其他的信息(网站域名、有效时长等)来制作证书(certificate)，然后使用摘要算法得到证书的摘要(指纹字符串)，再使用自己的私钥对摘要进行签名(加密)，最后将签名信息打包进数字证书返回给网站，管理员只需要将获得的证书配置到网站服务器上即可。

每当有浏览器请求网站时，首先会将证书返回给浏览器，此时浏览器会用CA机构的公钥来对证书中的签名信息验签(解密)，如果能解密成功得到摘要和证书摘要一致，就说明这个证书中的**网站的公钥**确实来自这个网站，没有被篡改。我们可以在浏览器的地址栏上点击网址左侧的小锁图标来查看证书的详细信息。当浏览器安全的拿到网站的公钥，其他的步骤就如同上面描述的过程了。

如果使用CA公钥无法对网站返回的证书签名信息进行解密验签，则说明证书不是合法的CA机构签发的，也有可能被篡改了，浏览器上就会显示异常界面**您的连接不是私密连接**。

**CA机构公钥的安全问题**

上面说过公钥的公开可能导致公钥被篡改，CA机构的公钥不也一样吗？世界上的网站是无限的，但是CA机构总共也没多少家，任何正版操作系统都会将所有主流CA机构的证书（内包含公钥）内置到操作系统当中，我们不用额外获取，解密时只需遍历系统中所有内置的CA机构的公钥，只要有任何一个公钥能够正常解密出数据，就说明它是合法的。

**签名链**

windows10运行`certmgr.msc`查看操作系统内置的证书，这些证书分为很多种，比如根证书颁发CA机构的证书、受信任的发布者、三方根证书、个人等，为什么要这样分呢？说的通俗一点就是公认的CA机构就只有那么几家，如果全球所有网站都找它们签名，那就忙不过来了，所以出现一些三方CA机构，这些三方CA机构可以对网站签名，被签名的网站如果需要被浏览器信任，那就需要将三方CA机构的证书内置在操作系统，如果没有被内置，那就让根CA机构对三方CA机构的证书进行签名，这样一来，三方CA机构的证书就被浏览器信任了，而通过它签名的网站证书也被信任。通常情况下网站的数字证书中会包含它的签名链，只要签名链的末端证书是被系统信任的即可。

## 1.4 HTTPS工作流程

https采用的是对称加密和非对称加密结构的方式，使用对称加密对正常传输数据加解密，使用非对称加密协商对称密钥。

CA中心可以对网站的数字证书进行签名，使得浏览器可以分辨哪些网站的数字证书是可信任的，其实就是确保浏览器拿到的网站公钥确实来自这个网站

HTTPS在传输数据之前需要浏览器网站之间进行一次握手，以确立双方加密传输数据的对称密钥，过程如下：

1. 客户端将它所支持的算法列表和一个用作产生密钥的随机数发送给服务器
2. 服务器从算法列表中选择一种加密算法，并将它和一份包含服务器公用密钥的证书发送给客户端；该证书还包含了用于认证目的的服务器标识，服务器同时还提供了一个用作产生密钥的随机数
3. 客户端对服务器的证书进行验证（有关验证证书，可以参考数字签名），并抽取服务器的公用密钥；然后，再产生一个称作`pre_master_secret`的随机密码串作为通讯过程中对称加密的秘钥，并使用服务器的公用密钥对其进行加密，将加密后的信息发送给服务器
4. 网站接收浏览器发来的数据之后，通过私钥进行解密，然后HASH校验，如果一致，则使用浏览器发来的数字串使加密一段握手消息发给浏览器。
5. 浏览器解密，并HASH校验，没有问题，则握手结束。接下来的传输过程将由之前浏览器生成的随机密码并利用对称加密算法进行加密。

# 2. 网站支持https

下面通过SpringBoot实现一个简单的web工程，并一步步配置证书来体验一下这个过程

## 2.1 SpringBoot

**工程结构**

首先应该创建一个web项目，这里使用的是`SpringBoot`，通过`IntelliJ IDEA`创建一个maven工程，工程结构如下：

![](01-01-webproject.png)
```xml
HttpsTest
	|-src
	|	|-main
	|		|-java      //1.代码
	|			|-com.openxu.api
	|				|-controller
	|					|-UserController.java
	|                   |-module
	|                       |-Response.java
	|                       |-User.java
	|               |-MyApplication.java 
	|		|-resources  //2.配置资源
	|   		 |-application.yaml
	|-pom.xml   //3.maven配置
```

**maven配置**

```xml
//pom.xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>HttpsTest</groupId>
    <artifactId>HttpsTest</artifactId>
    <version>1.0-SNAPSHOT</version>
    <dependencies>
        <!--springBoot 启动jar-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>2.4.0</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.54</version>
        </dependency>
    </dependencies>
</project>
```

**项目配置**

```xml
# application.yaml
server:
  port: 8081
```

**源码**

```java
//MyApplication.java
package com.openxu.api;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class,args);
    }
}

//User.java 省略了getter\setter\toString
public class User {
    private String userId;
    private String userName;
}
//Response.java 省略了getter\setter\toString
public class Response<T> implements Serializable {
    private Integer code;
    private T data;
    private String msg;
}

//UserController.java
package com.openxu.api.controller;
import com.openxu.api.module.Response;
import com.openxu.api.module.User;
import org.springframework.web.bind.annotation.CrossOrigin;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.util.ArrayList;
import java.util.List;

@CrossOrigin(origins = "*")
@RestController
@RequestMapping(value="/user")
public class UserController {
    //http://192.168.1.129:8081/user/get   ipconfig得到本机ip
    @GetMapping(value = "/get")
    public Response<List<User>> getUser(){
        List<User> users = new ArrayList<User>();
        User user = new User();
        user.setUserId("1");
        user.setUserName("openXu");
        users.add(user);
        return new Response(1, users, "请求成功");
    }
}
```

然后运行`MyApplication`，成功后在浏览器访问`http://192.168.1.129:8081/user/get`（ipconfig得到本机ip为192.168.1.129）得到接口返回的数据:

```xml
{"code":1,"data":[{"userId":"1","userName":"openXu"}],"msg":"请求成功"}
```

## 2.2 开启SSL

这里首先介绍一下SSL(Secure Socket Layer安全套接层)，它位于HTTP和TCP层之间，简单的说它就是一个技术标准，用于规范网络传输中数据怎样加解密。SSL更新了很多版本，在3.0时互联网标准化组织对其进行了标准化，标准化之后更名为TLS1.0(Transport Layer Security安全传输层协议)，可以说TLS就是SSL的3.1版本，它们是同一个东西，而https = http + TLS/SSL。

网站需要支持https，首先需要有一个证书(其实就是一个密钥库文件)。怎样生成密钥库呢？可以通过一些工具实现，比如OpenSSL，它是一个开源的强大的安全套接字层密码库。还可以使用jdk自带的keytool(密钥和证书管理工具)。下面通过keytool生成的`openxu_serve.p12`，.p12是一种证书文件格式，可以看作是一个密钥库，可以存放公钥、私钥和证书信息，通过密钥库口令保护私钥。

```xml
//-alias设置别名
//-storetype 设置证书格式
//-keyalg设置加密算法
//-keysize设置证书大小
//-keystore设置证书文件地址
//-validity设置有效天数
//-storepass 设置密钥库口令
//-ext <value> X.509 扩展  可以配置ip，这个非常重要，浏览器可判断当前请求的ip是否被证书包含

D:\WorkSpace\IntelliJ\HttpsTest>keytool -genkeypair -alias openxu_server -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore src/main/resources/openxu_serve
r.p12 -validity 3650 -ext san=ip:192.168.1.129 -storepass 123456
您的名字与姓氏是什么?
  [Unknown]:  xu
您的组织单位名称是什么?
  [Unknown]:  open
您的组织名称是什么?
  [Unknown]:  openXu
您所在的城市或区域名称是什么?
  [Unknown]:  bj
您所在的省/市/自治区名称是什么?
  [Unknown]:  bj
该单位的双字母国家/地区代码是什么?
  [Unknown]:  cn
CN=xu, OU=open, O=openXu, L=bj, ST=bj, C=cn是否正确?
  [否]:  y
```

在`application.yaml`中配置ssl：

```xml
server:
  port: 8081
  ssl:
    #刚才生成的https证书地址
    key-store: classpath:openxu_server.p12  # 刚才生成的https证书地址
    key-store-password: 123456        # 密钥库口令
    key-password: 123456
    key-store-type: PKCS12             # 协议类型
    key-alias: openxu_server
    enabled: true
```

重新运行`MyApplication`，成功后在浏览器访问`http://192.168.1.129:8081/user/get`:

```xml
Bad Request
This combination of host and port requires TLS.
```

被告知请求需要错误的请求，当前网站需要TLS，也就是说需要使用https请求，所以我们换成`https://192.168.1.129:8081/user/get`。浏览器发出警告：您的连接不是私密连接，它说我们的数字证书是无效的不受信任的。点击高级->继续前往，就可以成功请求到数据。这里只是演示没有被CA机构签名的网站数字证书是不被浏览器信任的。

## 2.3 自签名证书

上面我们只是给网站生成了一个密钥库文件(包含数字证书)，然后在配置文件中开启了ssl，配置了密钥文件。当浏览器请求网站时，发现使用的是https，网站就会通过配置的密钥库中导出数字证书，这个证书中包含了公钥，将证书发给浏览器，浏览器拿到证书后使用系统内置CA公钥去验签，由于我们的数字证书并没有通过CA签名，所以浏览器认为这个数字证书是不合法的。

一般情况下，如果需要浏览器信任网站的证书，可以向CA机构申请对网站的证书进行签名(使用CA机构私钥加密)，这样浏览器访问网站拿到网站的证书和CA签名信息，就可以验证网站的证书有没有被篡改，如果验证签名没有出问题那么网站的证书就可信任。但是有很多网站可能因为各种原因并没有通过CA签名，比如之前(现在已经通过CA签名)的中国铁路，在浏览器访问`https://kyfw.12306.cn/otn/`被告知它的网站证书是无效的。没有通过CA机构签名的数字证书称之为自签名证书（网站充当自己的CA)，当然我们模拟CA生成一对公私钥，对网站的证书进行签名，但是浏览器并没有内置我们的签名证书，被签名的网站证书照样是不被信任的。

让浏览器信任一个网站的数字证书有两种方法，第一种就是网站的数字证书通过系统信任的CA机构签名，第二种是配置浏览器使其信任网站的数字证书或者对其签名的证书。

下面我们设置浏览器信任网站的证书，首先通过`keytool -export`命令从密钥库证书文件中导出一个数字证书文件`openxu_server.cer`，.cer同.p12一样也是一种证书格式，不同的是.cer只包含公钥：

```xml
//-export导出证书
//-alias 证书别名
//-file 证书输出文件
//-keystore 证书使用的密钥库（从里面拿到公钥）
//-storepass密钥库密码

D:\WorkSpace\IntelliJ\HttpsTest>keytool -export -alias openxu_server -file openxu_server.cer -keystore src\main\resources\openxu_server.p12 -storepass 123456
存储在文件 <openxu_server.cer> 中的证书
```

设置浏览器信任我们的网站证书：

`设置-->隐私设置和安全性-->安全性-->管理证书-->受信任的根证书颁发机构-->导入-->选择刚刚生成的openxu_server.cer文件-->下一步...-->完成-->重启浏览器`

通过上面的步骤将网站的数字证书(未签名)导入到“受信任的根证书颁发机构”目录下，这时再请求`https://192.168.1.129:8081/user/get`就不会发出警告了，而且浏览器会告诉我们这个连接是安全的。

## 2.4 获取网站的数字证书

不同的系统内置的CA证书可能不一样，下面仅仅只是举一个栗子，不一定是真的。比如`https://www.baidu.com/`百度的数字证书是通过某一个CA机构签名了的，而windows系统中内置了这个CA的证书，在windows上的浏览器则可以顺利的访问百度https。但是Android系统中并没有内置对百度签名的CA机构的证书，android上访问百度就会报错，这种情况下我们可以在windows浏览器中下载百度的数字证书文件，然后在android中进行一些设置(下面会讲)就可以访问`https://www.baidu.com/`了。怎样下载某个网站的数字证书呢？

点击浏览器地址栏左边的小锁-->证书-->证书信息-->复制到文件-->下一步...-->输出导出的文件名C:\Users\admin\Desktop\baidu.cer-->完成

# 3. Android 9.0强制使用https

Android P(9.0)网络安全策略不允许明文通信，强制使用https，会阻塞http请求，如果app使用的第三方sdk有http，将全部被阻塞。报如下错误：

```java
//网络安全策略不允许与本地主机进行明文通信
UnknownServiceException: CLEARTEXT communication to localhost not permitted by network security policy
```
或者
```java
//不允许向*发送明文HTTP请求
IOException java.io.IOException: Cleartext HTTP traffic to * not permitted
```

## 3.1 usesCleartextTraffic允许使用明文通信

最简单的兼容方式是在AndroidManifest文件的application设置`android:usesCleartextTraffic="true"`表示当前应用允许使用明文通信，也就是说可以使用http。

```xml
//允许使用明文通信
android:usesCleartextTraffic="true"
```

## 3.2 代码配置网站证书

上面的适配办法只能是一个过渡，不是长久之计，况且上面示例中的web工程只支持https，并不支持http(如果需要支持http还需要写一些代码，这个大家可以网上查一下Springboot同时支持http和https)，所以我们必须要使用https怎么办呢？其实我们的app就和上面的浏览器一样只需要配置app的网络请求框架信任网站的数字证书就可以了，如果不配置直接请求上面的接口会报错：

```xml
javax.net.ssl.SSLHandshakeException: java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.
```

首先需要将网站的数字证书拷贝到项目assets文件夹下，其实可以随便放，只要能读取到就行，然后对OkHttpClient进行设置信任我们网站的证书就可以完成请求了，接口ip是局域网，这里运行需要使用模拟器：

```java
private void getUser(){
    try{
        //CertificateFactory可以从资源文件或者InputStream流中加载数字证书
        CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
        InputStream inputStream = getResources().getAssets().open("openxu_server.cer");
        Certificate cer = certificateFactory.generateCertificate(inputStream);

        //KeyStore用于存储加密密钥和证书，可以保存多个（key value的形式保存）
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        keyStore.load(null);
        keyStore.setCertificateEntry("openxu_server", cer);

        //信任管理器工厂
        TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        //利用keyStore去初始化TrustManagerFactory
        trustManagerFactory.init(keyStore);
        TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
        if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
            throw new IllegalStateException("Unexpected default trust managers:" + Arrays.toString(trustManagers));
        }
        X509TrustManager trustManager = (X509TrustManager) trustManagers[0];

        //SSLContext表示一个安全套接字协议实现，TLS（Transport Layer Security，安全传输层)，TLS是建立在传输层TCP协议之上的协议，服务于应用层，它的前身是SSL（Secure Socket Layer，安全套接字层），它实现了将应用层的报文进行加密后再交由TCP进行传输的功能。
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, new TrustManager[] { trustManager }, new SecureRandom());

        //设置https
        OkHttpClient client = new OkHttpClient.Builder()
                //OkHttpClient设置SSL
                .sslSocketFactory(sslContext.getSocketFactory(), (X509TrustManager) trustManagers[0])
                .build();

        client.newCall(new Request.Builder()
                .url("https://192.168.1.129:8081/user/get")
                .get()
                .build())
                .enqueue(new Callback() {
                    @Override
                    public void onFailure(Call call, IOException e) {
                        e.printStackTrace();
                        Log.e("FragmentHome", "请求数据失败："+e.getMessage());
                    }
                    @Override
                    public void onResponse(Call call, Response response) throws IOException {
                        Log.w("FragmentHome", "请求数据："+response.body().string());
                    }
                });
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

**提取证书内容**

我们还可以将证书中的内容提取出来，作为字符串常量，这样就不需要将证书打包到app中了：

```xml
D:\WorkSpace\IntelliJ\HttpsTest>keytool -printcert -rfc -file openxu_server.cer
-----BEGIN CERTIFICATE-----
MIIDWDCCAkCgAwIBAgIEVeFUNTANBgkqhkiG9w0BAQsFADBUMQswCQYDVQQGEwJj
bjELMAkGA1UECBMCYmoxCzAJBgNVBAcTAmJqMQ8wDQYDVQQKEwZvcGVuWHUxDTAL
BgNVBAsTBG9wZW4xCzAJBgNVBAMTAnh1MB4XDTIwMTExODA3MTI1OVoXDTMwMTEx
NjA3MTI1OVowVDELMAkGA1UEBhMCY24xCzAJBgNVBAgTAmJqMQswCQYDVQQHEwJi
ajEPMA0GA1UEChMGb3Blblh1MQ0wCwYDVQQLEwRvcGVuMQswCQYDVQQDEwJ4dTCC
ASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAKrJTJnjggd4k8lP31+D34He
VJFQgPSL9f/6JCHAldoQ+XJJjoUQ0k6TNwJ6rqy9tg+iIogS6tk7UWPwbp1julha
U1jtuHAfRNWSvMniq2MZsznBJBJm0lwHZn2I+J1f5jlPDcvr9NPr7iincdMV1aJN
sr+zuMggiug9t2xRYydCUMrgJ+ckQFEHH97fPCOng9rXwT4VTs6ESgemokiHDbdo
duUdC5vdF455/NEiI+0AYaJ2u9N5TKypn2rREjPVCxpGlN8SHNOvdIUysAXaZSwD
qBpxWnuembbCqw5fMegf6+tuo1XODLf8TcJdNqpPfvsCI99BE+6FOSYHSsHp0U0C
AwEAAaMyMDAwDwYDVR0RBAgwBocEwKgBgTAdBgNVHQ4EFgQUZjsaBT2iG/zwAQOM
ONdSzz+M5TcwDQYJKoZIhvcNAQELBQADggEBAGXFdmr/fkmFZbDKq39ahmeKQw8+
Hi2pUCL2psXukCjd5jJ+hkuaeEGNgvrRpKHInpdJ3FID7Gtd89GJTEDzdZwksDE3
DQQdjfYe2OEqjPbC0GPblnYjdJLsWEzOJ9EYPgmPIpaf4wyC3M18yu6bdU8WZUGJ
V05r4LytGvXr0WugGywJh8NW0gjXPwGQUlly8JEflPrySq8e+3vdP2rbxIPt4VqL
+duA8UvIm2MVmdGN25uzV0GKuZDu11B4p7GAfb6B6yu8ETLIpATTw0bgXF4mgUfP
twhCtSIPKtOF4Ev+0jOnzpc12bAlyBGpP99vL2ZbjdaJU00VFNzeWysicts=
-----END CERTIFICATE-----

```

```java

private final String certStr = "-----BEGIN CERTIFICATE-----\n" +
        "MIIDWDCCAkCgAwIBAgIEVeFUNTANBgkqhkiG9w0BAQsFADBUMQswCQYDVQQGEwJj\n" +
        "bjELMAkGA1UECBMCYmoxCzAJBgNVBAcTAmJqMQ8wDQYDVQQKEwZvcGVuWHUxDTAL\n" +
        "BgNVBAsTBG9wZW4xCzAJBgNVBAMTAnh1MB4XDTIwMTExODA3MTI1OVoXDTMwMTEx\n" +
        "NjA3MTI1OVowVDELMAkGA1UEBhMCY24xCzAJBgNVBAgTAmJqMQswCQYDVQQHEwJi\n" +
        "ajEPMA0GA1UEChMGb3Blblh1MQ0wCwYDVQQLEwRvcGVuMQswCQYDVQQDEwJ4dTCC\n" +
        "ASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAKrJTJnjggd4k8lP31+D34He\n" +
        "VJFQgPSL9f/6JCHAldoQ+XJJjoUQ0k6TNwJ6rqy9tg+iIogS6tk7UWPwbp1julha\n" +
        "U1jtuHAfRNWSvMniq2MZsznBJBJm0lwHZn2I+J1f5jlPDcvr9NPr7iincdMV1aJN\n" +
        "sr+zuMggiug9t2xRYydCUMrgJ+ckQFEHH97fPCOng9rXwT4VTs6ESgemokiHDbdo\n" +
        "duUdC5vdF455/NEiI+0AYaJ2u9N5TKypn2rREjPVCxpGlN8SHNOvdIUysAXaZSwD\n" +
        "qBpxWnuembbCqw5fMegf6+tuo1XODLf8TcJdNqpPfvsCI99BE+6FOSYHSsHp0U0C\n" +
        "AwEAAaMyMDAwDwYDVR0RBAgwBocEwKgBgTAdBgNVHQ4EFgQUZjsaBT2iG/zwAQOM\n" +
        "ONdSzz+M5TcwDQYJKoZIhvcNAQELBQADggEBAGXFdmr/fkmFZbDKq39ahmeKQw8+\n" +
        "Hi2pUCL2psXukCjd5jJ+hkuaeEGNgvrRpKHInpdJ3FID7Gtd89GJTEDzdZwksDE3\n" +
        "DQQdjfYe2OEqjPbC0GPblnYjdJLsWEzOJ9EYPgmPIpaf4wyC3M18yu6bdU8WZUGJ\n" +
        "V05r4LytGvXr0WugGywJh8NW0gjXPwGQUlly8JEflPrySq8e+3vdP2rbxIPt4VqL\n" +
        "+duA8UvIm2MVmdGN25uzV0GKuZDu11B4p7GAfb6B6yu8ETLIpATTw0bgXF4mgUfP\n" +
        "twhCtSIPKtOF4Ev+0jOnzpc12bAlyBGpP99vL2ZbjdaJU00VFNzeWysicts=\n" +
        "-----END CERTIFICATE-----";

private void getUser(){
    try{
        ...
//          InputStream inputStream = getResources().getAssets().open("openxu_server.cer");
            InputStream inputStream = new Buffer().writeUtf8(certStr).inputStream();
        ...
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

## 3.3 android:networkSecurityConfig网络安全性配置

3.2中通过代码配置网站的数字证书看起来非常繁琐，android提供了xml配置的方式来实现网络安全配置。在AndroidManifest文件的application节点配置

```xml
android:networkSecurityConfig="@xml/network_security_config"
```
network_security_config.xml内容如下，具体配置参考[网络安全配置](https://developer.android.google.cn/training/articles/security-config.html):

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!--cleartextTrafficPermitted = true允许明文通信，支持http-->
    <!--Android 9（API 28）以下默认cleartextTrafficPermitted="true" ，9.0以上默认为false，这就是为什么9.0以上不允许http请求的原因-->
    <base-config cleartextTrafficPermitted="true" >
        <trust-anchors>
            <!--信任系统预装CA证书，所有使用这些内置CA机构签名的网站数字证书都将被信任-->
            <!--可在设置-安全和隐私-系统安全-加密与凭据-信任的凭据 中查看系统内置CA证书-->
            <certificates src="system" />
            <!--添加自签名数字证书-->
            <certificates src="@raw/baidu"/>  <!--添加百度的证书（通过浏览器导出的）-->
            <certificates src="@raw/openxu_server"/>
        </trust-anchors>
    </base-config>
    <!--添加自签名数字证书，网站有多个域名的情况-->
    <domain-config cleartextTrafficPermitted="false">
        <!--可配置多个ip-->
        <domain includeSubdomains="true">192.168.1.129</domain>
        <!--<domain includeSubdomains="true">xxx.xxx.xxx.xxx</domain>-->
        <trust-anchors>
            <!--将自签名证书放在raw目录下-->
            <certificates src="@raw/openxu_server"/>
        </trust-anchors>
    </domain-config>

    <!--配置用于调试的CA，上线时需要删除，应用商店不接受被标记为可调试的应用-->
    <!--<debug-overrides>
        <trust-anchors>
            <certificates src="@raw/openxu_server"/>
        </trust-anchors>
    </debug-overrides>-->
</network-security-config>
```

这样在项目中请求接口就不需要通过代码配置ssl了：

```java
private void getUser(){
    OkHttpClient client = new OkHttpClient.Builder()
            //.sslSocketFactory(sslContext.getSocketFactory(), (X509TrustManager) trustManagers[0])
            .build();

    client.newCall(new Request.Builder()
    		//访问自己的网站和百度的网站都可以了
            //.url("https://192.168.1.129:8081/user/get")
            .url("https://www.baidu.com/")
            .get()
            .build())
            .enqueue(new Callback() {
                @Override
                public void onFailure(Call call, IOException e) {
                    e.printStackTrace();
                    Log.e("FragmentHome", "请求数据失败："+e.getMessage());
                }
                @Override
                public void onResponse(Call call, Response response) throws IOException {
                    Log.w("FragmentHome", "请求数据："+response.body().string());
                }
            });
}
```

# 4. 双向证书验证

上面https都是单向证书验证，目的是为了验证服务器的身份，校验数字证书中的公钥是否是服务器真实的公钥。一般情况下，客户端访问网站，只要拿着正确的网站公钥对数据加密后传输，网站都会做出响应，而不管客户端是谁，单向认证已经保证了数据传输过程的安全。

但是有一些情况下仅仅保证数据传输安全是不够的，还需要验证客户端身份，不能不管谁发的请求服务端都处理。比如网银，之前我们去银行办卡，都会有一个U盾，这个U盾中实际上保存的是用户的私钥，当交易时，客户端会通过U盾中的私钥对数据加密，服务端收到数据后使用用户的公钥解密，从而实现对客户端的校验，只有持有U盾(私钥)的用户才能完成交易，这就是双向证书验证。当然具体的认证细节可能不是这样，但大概就是这么个意思。

上面的web工程中我们已经实现了SSL单向认证，同样如果服务端要校验客户端身份，也需要为客户端生成密钥库文件`openxu_client.p12`，并导出数字证书`openxu_client.cer`:

```xml
//生成openxu_client.p12
D:\WorkSpace\IntelliJ\HttpsTest> keytool -genkeypair -alias openxu_client -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore src/main/resources/openxu_client.p12 -validity 3650 -storepass 123456

//提取openxu_client.p12
D:\WorkSpace\IntelliJ\HttpsTest> keytool -export -alias openxu_client -file src\main\resources\openxu_client.cer -keystore src\main\resources\openxu_client.p12 -storepass 123456
```

## 4.1 SpringBoot开启双向认证

修改web工程中的`application.yaml`

```xml
server:
  port: 8081
  ssl:
    # 单向认证，客户端校验服务端
    key-store: classpath:openxu_server.p12  # 密钥库
    key-store-password: 123456        # 密钥库口令
    key-password: 123456
    key-store-type: PKCS12             # 协议类型
    key-alias: openxu_server
    enabled: true
    # 双向认证，服务端校验客户端
    client-auth: need             # 开启客户端验证 ClientAuth.need
    # 
    trust-store: classpath:openxu_client.cer   # 信任库：存放了服务端信任的客户端的证书
```

重启报错`Caused by: java.io.IOException: Invalid keystore format`，它说密钥库格式错误，上面我们配置`trust-store`它接受的是密钥库文件，而我们给的是.cer证书文件。密钥库文件可以包含证书，但是证书文件中不一定包含私钥，反正这里就是一个文件格式的问题，所以需要将.cer导入到一个新的密钥库`openxu_client_cer
s.p12`中，openxu_client_cer
s.p12就是一个容器，它可以存放很多个证书或者密钥数据：

```xml
D:\WorkSpace\IntelliJ\HttpsTest>keytool -import -alias openxu_client -file src\main\resources\openxu_client.cer -keystore src\main\resources\openxu_client_cer
s.p12
输入密钥库口令: 123456789 # 这里输入的口令是新的openxu_client_cer
s.p12密钥库的口令
再次输入新口令: 123456789
所有者: CN=client, OU=client, O=client, L=client, ST=cleitn, C=client
发布者: CN=client, OU=client, O=client, L=client, ST=cleitn, C=client
序列号: 4581056c
有效期为 Thu Nov 19 10:50:20 CST 2020 至 Sun Nov 17 10:50:20 CST 2030
证书指纹:
         MD5:  52:CA:44:1D:D8:70:F0:16:EB:3F:05:D5:79:D0:27:5F
         SHA1: 1A:EC:0F:F2:97:4C:FA:D1:1D:F6:4C:A5:B1:AC:4F:64:7D:F3:2D:91
         SHA256: 6F:58:45:C5:A0:A2:77:02:46:EC:30:0B:5F:0D:91:3F:48:20:ED:51:26:1F:BD:FD:F4:92:99:66:25:16:FC:05
签名算法名称: SHA256withRSA
主体公共密钥算法: 2048 位 RSA 密钥
版本: 3
...

是否信任此证书? [否]:  y
证书已添加到密钥库中
```

修改web工程中的`application.yaml`

```xml
server:
  ...
    # 双向认证，服务端校验客户端
    client-auth: need             # 开启客户端验证 ClientAuth.need
    trust-store: classpath:openxu_client_cers.p12   # 信任库：存放了服务端信任的客户端证书的公钥文件
    # 此处并不需要提供密钥库的口令密码，因为口令密码是保护密钥库中的私钥的，而上面配置的信任库中只有客户端的公钥证书
    # trust-store-password: 123456789
```

重启服务，然后浏览器访问`https://192.168.1.129:8081/user/get`报如下错误：

```xml
此网站无法提供安全连接192.168.1.129 不接受您的登录证书，或者您可能没有提供登录证书。
请尝试联系系统管理员。
ERR_BAD_SSL_CLIENT_AUTH_CERT
```

因为服务端开启了客户端校验，只有拿着服务端配置的信任库中证书公钥对应的私钥的客户端才能被服务端信任，否则不接受。怎样在客户端配置自己的证书呢？

## 4.2 windows安装客户证书

这个过程就相当于安装银行卡U盾中的证书，我们刚刚为客户端生成了一个密钥库文件`openxu_client.p12`，找到该文件后直接双击运行安装，下一步、下一步、输入密钥库口令123456、下一步、下一步、完成

![](01-02-installcert.png)

安装完证书后，再次访问`https://192.168.1.129:8081/user/get`，会出现下面对话框，让我们选择一个客户端证书，这个证书列表中就有我们刚刚安装的证书，选择后点击确定就能顺利访问服务器了:

![](01-03-clientcertrequest.png)

## 4.3 Android配置客户端证书

我们web工程开启了SSL双向认证，但是现在android工程只配置了单向认证（受信任的服务器），如果直接请求服务器，会报如下错误：

```xml
//SSL握手已终止
Caused by: javax.net.ssl.SSLProtocolException: SSL handshake terminated:
```

这是因为服务端校验客户端没通过，服务端不信任客户端所以不给他提供服务。怎样让服务器信任我们的android客户端请求呢？其实跟上面windows安装客户端证书是同一个概念，只是方式不同而已，在android端，我们需要使用代码配置https请求（network_security_config.xml中没有配置客户端证书的标签），还记得上面用代码配置单向ssl认证吗？调用`SSLContext.init()`的时候第一个参数就是用来配置客户端密钥库的，单向认证时我们传的是null，双向认证我们需要将客户端密钥库加载进来，然后传给init()方法：

```java
private void getUser(){
    try{
        /**配置受信任的服务器证书信息*/
        //CertificateFactory可以从资源文件或者InputStream流中加载数字证书
        CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
        InputStream inputStream = getResources().getAssets().open("openxu_server.cer");
        Certificate cer = certificateFactory.generateCertificate(inputStream);

        //KeyStore用于存储加密密钥和证书，可以保存多个（key value的形式保存）
        KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        keyStore.load(null);
        keyStore.setCertificateEntry("openxu_server", cer);

        //信任管理器工厂
        TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        //利用keyStore去初始化TrustManagerFactory
        trustManagerFactory.init(keyStore);
        TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
        if (trustManagers.length != 1 || !(trustManagers[0] instanceof X509TrustManager)) {
            throw new IllegalStateException("Unexpected default trust managers:" + Arrays.toString(trustManagers));
        }
        X509TrustManager trustManager = (X509TrustManager) trustManagers[0];

        /**★★★配置当前客户端的密钥库，以便服务端校验客户端*/
        //加载客户端密钥库
        KeyStore clientKeyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        clientKeyStore.load(getResources().getAssets().open("openxu_client.p12"), "123456".toCharArray());
        //密钥库管理器
        KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        keyManagerFactory.init(clientKeyStore, "123456".toCharArray());

        //SSLContext表示一个安全套接字协议实现，TLS（Transport Layer Security，安全传输层)，TLS是建立在传输层TCP协议之上的协议，服务于应用层，它的前身是SSL（Secure Socket Layer，安全套接字层），它实现了将应用层的报文进行加密后再交由TCP进行传输的功能。
        SSLContext sslContext = SSLContext.getInstance("TLS");
        /**
         * init()方法接受三个参数：
         * KeyManager[] km ： 客户端密码管理器数组，配置了客户端密钥库的请求才能被服务端校验信任
         * TrustManager[] tm ：服务端信任管理器数组，哪些服务器被信任就将它的证书加到这个数组中
         * SecureRandom random : 生成器的随机性源
         */
        sslContext.init(keyManagerFactory.getKeyManagers(),
                new TrustManager[] { trustManager },
                new SecureRandom());

        //设置https
        OkHttpClient client = new OkHttpClient.Builder()
                //OkHttpClient设置SSL
                .sslSocketFactory(sslContext.getSocketFactory(), (X509TrustManager) trustManagers[0])
                .build();
        client.newCall(new Request.Builder()
                .url("https://192.168.1.129:8081/user/get")
//                    .url("https://www.baidu.com/")
                .get()
                .build())
                .enqueue(new Callback() {
                    @Override
                    public void onFailure(Call call, IOException e) {
                        e.printStackTrace();
                        Log.e("FragmentHome", "请求数据失败："+e.getMessage());
                    }
                    @Override
                    public void onResponse(Call call, Response response) throws IOException {
                        Log.w("FragmentHome", "请求数据："+response.body().string());
                    }
                });
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

添加之后请求再次报错`java.io.IOException: Wrong version of key store.`，这是因为android平台不支持`.p12`格式的证书文件，它只支持bks格式，所以我们需要将p12转换成bks。这里需要下载一个工具jar包[portecle-1.11.zip](https://sourceforge.net/projects/portecle/files/)，下载后解压里面包含一个`portecle.jar`，cmd进入解压目录，执行`java -jar portecle.jar`即可运行portecle可视化程序:

![](01-04-portecle.png)

打开密钥库：菜单File-->Open Keystore File -->选择要转换的openxu_client.p12-->输入密钥口令123456 --> 

转换为bks:菜单Tools-->Change Keystore Type --> 输入密钥口令123456 -->提示Change Keystore Type Successful

保存bks:菜单File-->Save Keystore As -->输入文件名openxu_client.bks -->保存

将得到的`openxu_client.bks`拷贝到assets目录中，修改上面代码加载bks证书，然后运行项目，发现请求成功了。

```java
private void getUser(){
    try{
       ...

        /**★★★配置当前客户端的密钥库，以便服务端校验客户端*/
        //加载客户端密钥库
        KeyStore clientKeyStore = KeyStore.getInstance(KeyStore.getDefaultType());
        clientKeyStore.load(getResources().getAssets().open("openxu_client.bks"), "123456".toCharArray());
        ...
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

[web工程源码：https://github.com/openXu/HttpsTest](https://github.com/openXu/HttpsTest)