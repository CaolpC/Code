# Cookie Session 和 Token

    认证：验证当前用户的身份，如用户名密码登录、邮箱登录链接、手机号验证码

    授权：手机 APP 被询问是否允许授予权限，实现授权方式：cookie、session、token、OAuth

    凭证：实现认证和授权的前提是需要一种媒介（证书）

## Cookie

    HTTP 是无状态协议，对事务处理没有记忆能力，客户端和服务端会话完成，服务端不保存任何会话信息

    各请求完全独立，服务端无法确认当前访问者的身份，无法识别当前和上一次请求的是否为同一人

    服务器和浏览器为进行会话追踪，需要主动维护一个状态，该状态需要通过 cookie 和 session 去实现

### Cookie 介绍

***cookie 保存在客户端：***

    服务器发送到浏览器并且保存在本地的一小块数据，浏览器下次向同一服务器再发起请求时被携带并发送到服务器上

***cookie 不可跨域***

    每个 cookie 会绑定单一域名。无法跨域使用，一级、二级共享使用 cookie 靠 domain 实现

## Session

    session 是另外一种记录服务器和客户端会话状态的机制

    session 是基于 cookie 实现的， session 存储在服务端， sessionId 会被存储到客户端的 cookie 中

    客户端没有 Session 一说，服务器和客户端建立连接时添加客户端连接标志

    然后再服务器软件 Apache、Tomcat、JBoss 转换成临时 Cookie 发送给客户端

    客户端第一次请求时检查是否携带 Session（临时 Cookie）， 没有则会添加 Session, 有就拿出来进行操作

### Session 的工作流程

    1. 浏览器请求房网，服务端生成 Session ID
    2. 生成的 Session ID 保存到服务器中
    3. 生成的 Session ID 响应（Response）给浏览器, 通过 Set-Cookie
    4. 浏览器收到 Session ID，下一次请求的时候会带上该 Session ID
    5. 服务器收到浏览器发来的 Session ID, 从 Session 存储中找到用户状态数据，会话建立
    6. 以后的请求都会交换这个 Session ID, 进行有状态的会话

## Token

    令牌， Uid + time + sign[+固定参数]

    token 认证方式类似于临时的证书签名，服务端无状态的认证方式，无状态指的是服务端不会保存身份认证相关的数据

    Uid: 用户唯一身份标识

    time: 当前时间时间戳

    sign: 签名， hash/encrypt 压缩的十六进制字符串，放置第三方恶意拼接

    固定参数：常用的固定参数加入 token, 避免重复查库

### 存放

    客户端: 存放于 localStorage、cookie、或者 sessionStorage 中

    服务器：一般存储于数据库或者 Redis 中（方便设置过期时间）

### token 认证流程

    与 cookie 相似：

    1. 用户登录，成功后服务器生成 token 返回给客户端
    2. 客户端保存在本地
    3. 客户端再次访问服务器，token 放入 headers 中
    4. 服务器采用 filter 过滤器校验（对 token 进行解密和签名认证），校验成功返回请求数据，失败返回错误码
   
    token 服务端无状态，服务端不用存放 token 数据，每次客户端请求的时候，携带 token，服务器进行解密和签名认证

    用 token 的计算时间换取 session 的存储空间，减轻服务器压力，减少频繁查询数据库

    token 由应用管理，避开同源策略

#### token 分类

    access_token:

        有效期比较短

    refresh_token:

        access_token 失效时，对 access_token 进行刷新，客户端直接用 refresh token 去更新 access_token

        refresh token 如果也失效，用户重新登录
        
        申请新的 access_token 才会验证 refresh_token 是否失效，refresh_token 存储在服务器数据库中

## Cookie 和 Session 区别

    安全性：Session 比 Cookie 安全，Session 存储在服务器，Cookie 存储在客户端

    存取值的类型不同： Cookie 只支持字符串，Session 可以存任意类型数据

    有效期：Cookie 长时间保持，Session 一般失效时间较短

    存储大小不同：单个 Cookie 保存的数据不能超过 4k, Session 可存储远高于 Cookie, 访问量过多，占用资源会很多

## Session 和 Token 区别

    服务端状态：
        Session 记录服务器和客户端会话状态机制，服务端有状态化，记录会话信息
        Token 是令牌，访问资源接口 API 所需要的资源凭证，使服务端无状态化，不会存储会话信息
    安全性：
        身份认证中，Token 比 Session 安全性好，每个请求都有签名，防止监听以及重放攻击
        Session: 依赖链路层保证通讯安全，
    其它：
        Session 认证：简单，只是将 User 信息存储到 Session 中，Session 不可预测

        Token 认证：针对用户
        Token 授权：针对 App, 让 App 有权访问某个用户信息， Token 唯一，不能转移到其它 App, 也不能转移到其它用户

    用户数据可能和第三方共享，允许第三方调用 API 接口，用 Token

## Cookie、Session、Token 相同点和区别

### 相同点

    都用于鉴权，都是服务器产生的

### 区别

    Cookie 存储在客户端，Session 存储在服务器

    Session 的安全性比 Cookie 高，所以一般情况下把重要的信息放在 Session, 不重要的信息放在 Cookie 中

    Session 存在服务器内存中，Token 存在服务器的文件或者数据库中

    Token 的好处是比 Session 更节省服务器资源，只需要在服务器端解密即可

## Sign 接口签名

    金融、银行，要求安全性更高

## JSON Web Token(JWT)
  
    JWT 是目前最流行的跨域认证解决方案， 一种认证授权机制

    JWT 声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，以便从资源服务器获取资源

### JWT 结构

![JWT示例](https://cdn.jsdelivr.net/gh/RamboCao/PicGo/images/20210630132409.png)

    JWT 被 `.` 分割成三部分，依次为：

    Header(头部)
    PayLoad(负载)
    Signature(签名)

Header

Header 是一个 Json 对象，描述 JWT 元数据

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

其中 alg 表示签名的算法， typ 表示该 token 的类型， JWT 令牌统一为 JWT，上面的 Json 对象使用 Base64URL 算法进行编码，转为字符串

Payload

Payload 也是 Json 对象，存放实际需要传递的数据

```json
{
    "iss" (issuer)：签发人
    "exp" (expiration time)：过期时间
    "sub" (subject)：主题
    "aud" (audience)：受众
    "nbf" (Not Before)：生效时间
    "iat" (Issued At)：签发时间
    "jti" (JWT ID)：编号
}
```

同样也可以自定义私有数据

```json
{
    "sub": "123456",
    "name": "caolp",
    "admin": true
}
```

Base64URL 编码

    为什么要使用 Base64URL 编码：因为 JWT 作为令牌，有可能会放在 URL 里边，而 "+", "/", "-" 在 URL 里有特殊的含义，
    所以需要进行替换。"=" 被省略，"+" 被替换成 "-", "/" 替换成 "_" 

Signature
Signature 是对前面两部分的签名，防止数据篡改，先指定一个密钥 (secret), 密钥只有服务器知道，使用 Header 里指定的签名算法，生成签名

```java
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload),secret)
```

最后将三部分的内容进行拼接，返回给用户

### JWT 使用

    服务端产生 JWT, 客户端接收 JWT, 可以存储在 Cookie 中，也可以存储在 localStorage

    客户端与服务端进行通信会携带 JWT, 如果存放在 Cookie 中，则不能跨越

    所以可以放在 Authorization 字段里面

    服务端接收到 JWT 后，使用密钥进行解密，可以得到客户端相关信息

### Token 和 JWT 的比较

    相同点：
    1. 都是访问资源的令牌
    2. 都可以记录用户信息
    3. 都是使服务端无状态化
    4. 都只有验证成功后、客户端才能访问服务端上受保护的资源

    不同点：
    1. Token: 服务端验证客户端发送过来的 Token 时，需要查询数据库获取用户信息，验证 Token 是否有效
    2. JWT: Token 和 Payload 加密后存储在客户端，服务端只需要密钥解密进行校验，无需查询或者减少查询数据库， JWT 包含了用户信息和加密的数据

### JWT 实现

***创建 SpringBootApplication, 引入的依赖：***

```java
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.11.2</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.11.2</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId> <!-- or jjwt-gson if Gson is preferred -->
        <version>0.11.2</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>2.1.5.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.58</version>
    </dependency>
```

***创建拦截器***

```java
/**
 * @author Caolp
 */
public class MyFilter implements Filter {

    @Override
    public void doFilter(ServletRequest servletRequest, 
                ServletResponse servletResponse, 
                FilterChain filterChain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        response.setCharacterEncoding("utf-8");

        String token = request.getHeader("Authorization");

        if (StringUtils.isEmpty(token)) {
            response.getWriter().write("请携带 token");
            return;
        }

        Claims claims = JwtService.parsePersonJwt(token);

        if (Objects.isNull(claims)) {
            response.getWriter().write("请携带token");
        } else {
            filterChain.doFilter(request, response);
        }
    }

}
```

***注册拦截器***

```java
@Configuration
public class BeanRegisterConfig {

    @Bean
    public FilterRegistrationBean<MyFilter> createFilterBean() {

        FilterRegistrationBean<MyFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(new MyFilter());
        registrationBean.addUrlPatterns("/user/hello");
        return registrationBean;

    }
}
```

***编写 JWTUtils 工具类***

```java
public class JwtUtils {

    public static Claims parseJwt(String jsonWebToken, Key signingKey) {
        try{
            return Jwts.parserBuilder().setSigningKey(signingKey).build().parseClaimsJws(jsonWebToken).getBody();
        }catch (JwtException e){
            return null;
        }
    }

    public static String createJwt(Map<String, Object> map, String audience, 
                                    String issuer, String jwtId, long TTLMillis,
                                    Key signingKey, SignatureAlgorithm signatureAlgorithm){

        long nowMillis = System.currentTimeMillis();
        Date now = new Date(nowMillis);

        JwtBuilder jwtBuilder = Jwts.builder()
                .setHeaderParam("typ", "JWT")
                .setIssuedAt(now)
                .setSubject(map.toString())
                .setIssuer(issuer)
                .setId(jwtId)
                .setAudience(audience)
                //设置签名使用的签名算法和签名使用的秘钥
                .signWith(signingKey, signatureAlgorithm);

        //添加Token过期时间
        if (TTLMillis >= 0) {
            // 过期时间
            long expMillis = nowMillis + TTLMillis;
            // 现在是什么时间
            Date exp = new Date(expMillis);
            // 系统时间之前的token都是不可以被承认的
            jwtBuilder.setExpiration(exp).setNotBefore(now);
        }
        //生成JWS（加密后的JWT）
        return jwtBuilder.compact();
    }
}
```

***对 JWTUtil 进行包装***

```java
/**
 * @author Caolp
 */
public class JwtService {


    /**
     * token 过期时间, 单位: 秒. 这个值表示 30 天
     */
    private static final long TOKEN_EXPIRED_TIME = 30 * 24 * 60 * 60;

    /**
     * 签名密钥算法
     */
    private static final SignatureAlgorithm SIGNATURE_ALGORITHM = SignatureAlgorithm.HS256;

    private static final Key SIGNING_KEY = Keys.secretKeyFor(SignatureAlgorithm.HS256);

    private static final String JWT_ISSUER = "CLP";

    /**
     * 描述:创建令牌
     *
     * @param map      主题，也差不多是个人的一些信息，为了好的移植，采用了map放个人信息，而没有采用JSON
     * @param audience 发送谁
     * @return java.lang.String
     */
    public static String createPersonToken(Map<String, Object> map, 
                                            String audience) {
        return JwtUtils.createJwt(map, audience, UUID.randomUUID().toString(), 
                        JWT_ISSUER, TOKEN_EXPIRED_TIME, SIGNING_KEY, SIGNATURE_ALGORITHM);
    }

    /**
     * 描述:解密JWT
     *
     * @param personToken JWT字符串,也就是token字符串
     * @return io.jsonwebtoken.Claims
     */
    public static Claims parsePersonJwt(String personToken) {
        return JwtUtils.parseJwt(personToken, SIGNING_KEY);
    }
}
```

***编写 Controller 进行测试***

```java
/**
 * @author Caolp
 */
@RestController
public class HelloController {


    @RequestMapping("user/hello")
    public String user(){
        return "hello";
    }


    @RequestMapping("user/token")
    public String token(){
        Map<String, Object> map = new HashMap<>();
        map.put("name", "caolp");
        map.put("age", 21);
        return JwtService.createPersonToken(map, "caolp");
    }

}
```

### 实验测试 JWT

```shell
C:\Code\JwtLab>curl http://localhost:8087/user/hello
请携带token

C:\Code\JwtLab>curl http://localhost:8087/user/token
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE2MjUxMDM3NDMsInN1YiI6IntuYW1lPWNhb2xwLCBhZ2U9MjF9IiwiaXNzIjoiOWYxNDA3YTMtMDg4Mi00ZTBlLTlkYTktMWI5NzRmMmM5MjViIiwianRpIjoiQ0xQIiwiYXVkIjoiY2FvbHAiLCJleHAiOjE2MjUxMDYzMzUsIm5iZiI6MTYyNTEwMzc0M30.FZksVMVIop6BF-Iv96mL8pNASgtWKE2affoK-p_vbNA

C:\Code\JwtLab>curl http://localhost:8087/user/hello -H "Authorization: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE2MjUxMDM3NDMsInN1YiI6IntuYW1lPWNhb2xwLCBhZ2U9MjF9IiwiaXNzIjoiOWYxNDA3YTMtMDg4Mi00ZTBlLTlkYTktMWI5NzR
mMmM5MjViIiwianRpIjoiQ0xQIiwiYXVkIjoiY2FvbHAiLCJleHAiOjE2MjUxMDYzMzUsIm5iZiI6MTYyNTEwMzc0M30.FZksVMVIop6BF-Iv96mL8pNASgtWKE2affoK-p_vbNA"
hello
```
