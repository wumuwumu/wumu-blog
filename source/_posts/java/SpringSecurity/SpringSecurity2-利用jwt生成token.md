---
title: SpringSecurity-利用jwt生成token
abbrlink: '46051755'
date: 2021-01-31 20:00:00
---

## 开篇

实现Token的方式有很多，本篇介绍的是利用Json Web Token(JWT)生成的Token.JWT生成的Token有什么好处呢？

- 安全性比较高，加上密匙加密而且支持多种算法。
- 携带的信息是自定义的，而且可以做到验证token是否过期。
- 验证信息可以由前端保存，后端不需要为保存token消耗内存。

本篇分3部分进行讲解。

- 1. 什么是JWT
- 1. JWT的代码实现
     - 用HS256 对称算法加密
     - 用RS256 非对称算法加密
- 1. 总结

> 如果原理很难懂，没关系。可以直接看JWT的代码实现。代码已经上传[github](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FMyBaron%2FJAVA_JWT_Token)。已经对代码进行封装成工具类。可以直接使用。

## 什么是JWT

JSON Web Token 简称JWT。
 一个JWT实际上就是一个字符串，它由三部分组成，`头部`、`载荷`与`签名`。
 JWT生成的token是这样的



```css
eyJpc3MiOiJKb2huI.eyJpc3MiOiJ.Kb2huIFd1IEp
```

> 生成的token，是3段，用`.`连接。下面有解释。

### 头部

用于描述关于该JWT的最基本的信息，例如其类型以及签名所用的算法等。这也可以被表示成一个JSON对象。
 例如：



```json
{
   "typ": "JWT",
  "alg": "HS256"
}
```

### 载荷

其实就是自定义的数据，一般存储用户Id，过期时间等信息。也就是JWT的核心所在，因为这些数据就是使后端知道此token是哪个用户已经登录的凭证。而且这些数据是存在token里面的，由前端携带，所以后端几乎不需要保存任何数据。
 例如：



```json
{
  "uid": "xxxxidid",  //用户id
  "exp": "12121212"  //过期时间
}
```

### 签名

签名其实就是：
 1.头部和载荷`各自base64加密后用.连接起来`，然后就形成了xxx.xx的前两段token。
 2.最后一段token的形成是，前两段加入一个密匙用HS256算法或者其他算法加密形成。

1. 所以token3段的形成就是在签名处形成的。

> [JWT的原理参考文章](https://links.jianshu.com/go?to=http%3A%2F%2Fblog.leapoahead.com%2F2015%2F09%2F06%2Funderstanding-jwt%2F)

## 代码实现

1.看代码前一定要知道JWT是由`头部`、`载荷`与`签名`组成。
 2.[代码已上传github](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FMyBaron%2FJAVA_JWT_Token),希望点个赞

1. 代码将JWT封装成两个工具类，可以直接调用。

### 需要下载的jar包



```xml
<dependency>
            <groupId>com.nimbusds</groupId>
            <artifactId>nimbus-jose-jwt</artifactId>
            <version>6.0</version>
</dependency>
```

### HS256  对称加密

#### 生成token



```java
 /**
     * 创建秘钥
     */
    private static final byte[] SECRET = "6MNSobBRCHGIO0fS6MNSobBRCHGIO0fS".getBytes();

    /**
     * 过期时间5秒
     */
    private static final long EXPIRE_TIME = 1000 * 5;


    /**
     * 生成Token
     * @param account
     * @return
     */
    public static String buildJWT(String account) {
        try {
            /**
             * 1.创建一个32-byte的密匙
             */
            MACSigner macSigner = new MACSigner(SECRET);
            /**
             * 2. 建立payload 载体
             */
            JWTClaimsSet claimsSet = new JWTClaimsSet.Builder()
                    .subject("doi")
                    .issuer("http://www.doiduoyi.com")
                    .expirationTime(new Date(System.currentTimeMillis() + EXPIRE_TIME))
                    .claim("ACCOUNT",account)
                    .build();

            /**
             * 3. 建立签名
             */
            SignedJWT signedJWT = new SignedJWT(new JWSHeader(JWSAlgorithm.HS256), claimsSet);
            signedJWT.sign(macSigner);

            /**
             * 4. 生成token
             */
            String token = signedJWT.serialize();
            return token;
        } catch (KeyLengthException e) {
            e.printStackTrace();
        } catch (JOSEException e) {
            e.printStackTrace();
        }
        return null;
    }
```

#### 验证token



```java
 /**
     * 校验token
     * @param token
     * @return
     */
    public static String vaildToken(String token ) {
        try {
            SignedJWT jwt = SignedJWT.parse(token);
            JWSVerifier verifier = new MACVerifier(SECRET);
            //校验是否有效
            if (!jwt.verify(verifier)) {
                throw ResultException.of(-1, "Token 无效");
            }

            //校验超时
            Date expirationTime = jwt.getJWTClaimsSet().getExpirationTime();
            if (new Date().after(expirationTime)) {
                throw ResultException.of(-2, "Token 已过期");
            }

            //获取载体中的数据
            Object account = jwt.getJWTClaimsSet().getClaim("ACCOUNT");
            //是否有openUid
            if (Objects.isNull(account)){
                throw ResultException.of(-3, "账号为空");
            }
            return account.toString();
        } catch (ParseException e) {
            e.printStackTrace();
        } catch (JOSEException e) {
            e.printStackTrace();
        }
        return null;
    }
```

#### 调用的业务逻辑



```java
 public class TestHS256 {

    public static void main(String[] args) throws InterruptedException {
        TestHS256 t = new TestHS256();
        t.testHS256();
    }

    //测试HS256加密生成Token
    public void testHS256() throws InterruptedException {
        String token = JWTHS256.buildJWT("account123");
        //解密token
        String account = JWTHS256.vaildToken(token);
        System.out.println("校验token成功，token的账号："+account);

        //测试过期
        Thread.sleep(10 * 1000);
        account = JWTHS256.vaildToken(token);
        System.out.println(account);
    }
}
```

#### 结果



```java
校验token成功，token的账号：account123
测试token过期------
Exception in thread "main" token.ResultException: Token 已过期
    at token.ResultException.of(ResultException.java:59)
    at token.jwt.JWTHS256.vaildToken(JWTHS256.java:89)
    at token.jwt.TestHS256.testHS256(TestHS256.java:24)
    at token.jwt.TestHS256.main(TestHS256.java:11)
```

### RS256 非对称加密

#### 生成加密密钥



```java
/**
     * 创建加密key
     */
    public static RSAKey getKey() throws JOSEException {
        RSAKeyGenerator rsaKeyGenerator = new RSAKeyGenerator(2048);
        RSAKey rsaJWK = rsaKeyGenerator.generate();
        return rsaJWK;
    }
```

#### 生成token



```java
 /**
     * 过期时间5秒
     */
    private static final long EXPIRE_TIME = 1000 * 5;
    private static RSAKey rsaKey;
    private static RSAKey publicRsaKey;

    static {
        /**
         * 生成公钥，公钥是提供出去，让使用者校验token的签名
         */
        try {
            rsaKey = new RSAKeyGenerator(2048).generate();
            publicRsaKey = rsaKey.toPublicJWK();

        } catch (JOSEException e) {
            e.printStackTrace();
        }
    }


    public static String buildToken(String account) {
        try {
            /**
             * 1. 生成秘钥,秘钥是token的签名方持有，不可对外泄漏
             */
            RSASSASigner rsassaSigner = new RSASSASigner(rsaKey);

            /**
             * 2. 建立payload 载体
             */
            JWTClaimsSet claimsSet = new JWTClaimsSet.Builder()
                    .subject("doi")
                    .issuer("http://www.doiduoyi.com")
                    .expirationTime(new Date(System.currentTimeMillis() + EXPIRE_TIME))
                    .claim("ACCOUNT",account)
                    .build();

            /**
             * 3. 建立签名
             */
            SignedJWT signedJWT = new SignedJWT(new JWSHeader(JWSAlgorithm.RS256), claimsSet);
            signedJWT.sign(rsassaSigner);

            /**
             * 4. 生成token
             */
            String token = signedJWT.serialize();
            return token;

        } catch (JOSEException e) {
            e.printStackTrace();
        }
        return null;
    }
```

#### 验证token



```java
 public static String volidToken(String token) {
        try {
            SignedJWT jwt = SignedJWT.parse(token);
            //添加私密钥匙 进行解密
            RSASSAVerifier rsassaVerifier = new RSASSAVerifier(publicRsaKey);

            //校验是否有效
            if (!jwt.verify(rsassaVerifier)) {
                throw ResultException.of(-1, "Token 无效");
            }

            //校验超时
            if (new Date().after(jwt.getJWTClaimsSet().getExpirationTime())) {
                throw ResultException.of(-2, "Token 已过期");
            }

            //获取载体中的数据
            Object account = jwt.getJWTClaimsSet().getClaim("ACCOUNT");


            //是否有openUid
            if (Objects.isNull(account)){
                throw ResultException.of(-3, "账号为空");
            }
            return account.toString();
        } catch (ParseException e) {
            e.printStackTrace();
        } catch (JOSEException e) {
            e.printStackTrace();
        }
        return "";
    }
```

#### 业务逻辑调用



```java
public class TestRS256 {

    public static void main(String[] args) throws InterruptedException {
        TestRS256 t = new TestRS256();
        t.testRS256();
    }

    //测试RS256加密生成Token
    public void testRS256() throws InterruptedException {
        String token = JWTRSA256.buildToken("account123");
        //解密token
        String account = JWTRSA256.volidToken(token);
        System.out.println("校验token成功，token的账号："+account);

        //测试过期
        Thread.sleep(10 * 1000);
        account = JWTRSA256.volidToken(token);
        System.out.println(account);
    }
}
```

#### 结果



```java
校验token成功，token的账号：account123
测试token过期------
Exception in thread "main" token.ResultException: Token 已过期
    at token.ResultException.of(ResultException.java:59)
    at token.jwt.JWTRSA256.volidToken(JWTRSA256.java:96)
    at token.jwt.TestRS256.testRS256(TestRS256.java:24)
    at token.jwt.TestRS256.main(TestRS256.java:11)
```

## 总结

JWT 的实践其实还是挺简单。安全性也是得到了保证，后端只需要保存着密匙，其他数据可以保存在token，由前端携带，这样可以减低后端的内存消耗。
 虽然token是加密的，但是携带的验证数据还是不要是敏感数据.

