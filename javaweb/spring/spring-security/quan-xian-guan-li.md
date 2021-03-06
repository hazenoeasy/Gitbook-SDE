# 权限管理

## 1.基本概念

### 1.1.什么是认证

系统为什么要认证？认证是为了保护系统的隐私数据与资源，用户的身份合法方可访问该系统的资源。

认证 ：用户认证就是判断一个用户的身份是否合法的过程，用户去访问系统资源时系统要求验证用户的身份信 息，身份合法方可继续访问，不合法则拒绝访问。常见的用户身份认证方式有：用户名密码登录，二维码登录，手 机短信登录，指纹认证等方式。

### 1.2 什么是会话

用户认证通过后，为了避免用户的每次操作都进行认证可将用户的信息保证在会话中。会话就是系统为了保持当前 用户的登录状态所提供的机制，常见的有基于session方式、基于token方式等。

基于session的认证方式如下图：

![1595922976684](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200801190032-508624.png)

它的交互流程是，用户认证成功后，在服务端生成用户相关的数据保存在session(当前会话)中，发给客户端的 该session对应的sesssion\_id ，客户端将该seession\_id 存放到 cookie 中，这样用户客户端请求时带上 session\_id 就可以验证服务器端是否存在改session\_id对应的 session 数据，以此完成用户的合法校验，当用户退出系统或session过期销毁时，客户端的session\_id也就无效了。

基于token方式如下图：

![1595923126894](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200728155847-506720.png)

它的交互流程是，用户认证成功后，服务端生成一个token发给客户端，客户端可以放到 cookie 或 localStorage 等存储中，每次请求时带上 token，服务端收到token通过验证后即可确认用户身份

基于session的认证方式由Servlet规范定制，服务端要存储session信息需要占用内存资源，客户端需要支持 cookie；基于token的方式则一般不需要服务端存储token，并且不限制客户端的存储方式。如今移动互联网时代 更多类型的客户端需要接入系统，系统多是采用前后端分离的架构进行实现，所以基于token的方式更适合。

### 1.2 什么是授权

认证是为了保证用户身份的合法性，授权则是为了更细粒度的对隐私数据进行划分，授权是在认证通过后发生的， 控制不同的用户能够访问不同的资源。 授权： 授权是用户认证通过根据用户的权限来控制用户访问资源的过程，拥有资源的访问权限则正常访问，没有 权限则拒绝访问。

### 1.3 授权的数据模型

如何进行授权即如何对用户访问资源进行控制，首先需要学习授权相关的数据模型。 授权可简单理解为Who对What(which)进行How操作，包括如下：

1. Who，即主体（Subject），主体一般是指用户，也可以是程序如爬虫，需要访问系统中的资源。
2. What，即资源（Resource），如系统菜单、页面、按钮、代码方法、系统商品信息、系统订单信息等。
   1. 系统菜单、页面、按钮、代码方法都属于系统功能资源，对于web系统每个功能资源通常对应一个URL；
   2. 系统商品信息、系统订单信息都属于实体资源（数据资源），实体资源由资源类型和资源实例组成，比如商品信息为资源类型，商品编号 为001的商品为资源实例。
3. How，权限/许可（Permission），规定了用户对资源的操作许可，权限离开资源没有意义，如用户对某资源的查询权限、用户对某资源的添加权限、用户对某的某个代码方法的调用权限、用户对编号为001商品修改权限等，通过权限可知用户对哪些资源都有哪些操作许可。

主体、资源、权限关系如下图：

![1595924525927](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200801190042-71507.png)

主体、资源、权限相关的数据模型如下：

1. 主体（用户id、账号、密码、...）
2. 资源（资源id、资源名称、访问地址、...）
3. 权限（权限id、权限标识、权限名称、资源id、...）
4. 角色（角色id、角色名称、...）
5. 角色和权限关系（角色id、权限id、...）
6. 主体（用户）和角色关系（用户id、角色id、...）

![1595924879313](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200801190046-652027.png)

通常企业开发中将资源和权限表合并为一张权限表，如下：

资源（资源id、资源名称、访问地址、...） 权限（权限id、权限标识、权限名称、资源id、...） 合并为： 权限（权限id、权限标识、权限名称、资源名称、资源访问地址、...） 修改后数据模型之间的关系如下图：

![1595925002583](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200728163003-152987.png)

### 1.4 RBAC

如何实现授权？业界通常基于RBAC实现授权。

#### 1.4.1 基于角色的访问控制

RBAC基于角色的访问控制（Role-Based Access Control）是按角色进行授权，比如：主体的角色为总经理可以查 询企业运营报表，查询员工工资信息等，访问控制流程如下：

![1595926667491](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200728165748-42159.png)

根据上图中的判断逻辑，授权代码可表示如下：

```java
if(主体.hasRole("总经理角色id")){
	查询工资
}
```

如果上图中查询工资所需要的角色变化为总经理和部门经理，此时就需要修改判断逻辑为“判断用户的角色是否是 总经理或部门经理”，修改代码如下：

```java
if(主体.hasRole("总经理角色id") || 主体.hasRole("部门经理角色id")){
	查询工资
}
```

根据上边的例子发现，当需要修改角色的权限时就需要修改授权的相关代码，系统可扩展性差。

#### 1.4.2 基于资源的访问控制

RBAC基于资源的访问控制（Resource-Based Access Control）是按资源（或权限）进行授权，比如：用户必须 具有查询工资权限才可以查询员工工资信息等，访问控制流程如下：

![1595926726521](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200801190052-313019.png)

根据上图中的判断，授权代码可以表示为：

```java
if(主体.hasPermission("查询工资权限标识")){
	查询工资
}
```

优点：系统设计时定义好查询工资的权限标识，即使查询工资所需要的角色变化为总经理和部门经理也不需要修改 授权代码，系统可扩展性强。

## 2.基于Session的认证方式

基于Session的认证机制由Servlet规范定制，Servlet容器已实现，用户通过HttpSession的操作方法即可实现，如 下是HttpSession相关的操作API。

![1595928066758](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200728172106-385948.png)

代码实例：参考物品管理项目；

## 3.Spring Security快速上手

### 3.3 认证

#### 3.3.2.安全配置

spring security提供了用户名密码登录、退出、会话管理等认证功能，只需要配置即可使用。

1. 在config包下定义WebSecurityConfig，安全配置的内容包括：用户信息、密码编码器、安全拦截机制。

```java
@EnableWebSecurit
 public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
        //配置用户信息服务，用来告诉spring security怎么查询用户的信息
        @Bean
        public UserDetailsService userDetailsService() {
            InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
            manager.createUser(User.withUsername("zhangsan").password("123").authorities("p1").build());
            manager.createUser(User.withUsername("lisi").password("456").authorities("p2").build());
            return manager;
        }
     // 密码编码器
        @Bean
        public PasswordEncoder passwordEncoder() {
            return NoOpPasswordEncoder.getInstance();
        }
        //配置安全拦截机制
        @Override
        protected void configure(HttpSecurity http) throws Exception {
            http
                .authorizeRequests()
                .antMatchers("/r/**").authenticated() （1）
                .anyRequest().permitAll() （2）
                .and()
                .formLogin().successForwardUrl("/login‐success"); （3）
        }
    }
```

在userDetailsService()方法中，我们返回了一个UserDetailsService给spring容器，Spring Security会使用它来 获取用户信息。我们暂时使用InMemoryUserDetailsManager实现类，并在其中分别创建了zhangsan、lisi两个用 户，并设置密码和权限。

而在configure()中，我们通过HttpSecurity设置了安全拦截规则，其中包含了以下内容： （1）url匹配/r/\*\*的资源，经过认证后才能访问。 （2）其他url完全开放。 （3）支持form表单认证，认证成功后转向/login-success。

代码实例看这里：[地址](https://gitee.com/gu\_chun\_bo/java-construct/tree/master/%E4%BB%A3%E7%A0%81%E5%B0%8F%E5%AE%9E%E4%BE%8B/security-spring-boot)

## 4.SpringSecurity应用详解

### 4.1集成SpringBoot

代码实例看这里：[地址](https://gitee.com/gu\_chun\_bo/java-construct/tree/master/%E4%BB%A3%E7%A0%81%E5%B0%8F%E5%AE%9E%E4%BE%8B/security-spring-boot)

### 4.2工作原理

#### 4.2.1 结构总览

Spring Security所解决的问题就是安全访问控制，而安全访问控制功能其实就是对所有进入系统的请求进行拦截， 校验每个请求是否能够访问它所期望的资源。根据前边知识的学习，可以通过Filter或AOP等技术来实现，Spring Security对Web资源的保护是靠Filter实现的，所以从这个Filter来入手，逐步深入Spring Security原理。

当初始化Spring Security时，会创建一个名为 SpringSecurityFilterChain 的Servlet过滤器，类型为 org.springframework.security.web.FilterChainProxy，它实现了javax.servlet.Filter，因此外部的请求会经过此 类，下图是Spring Security过虑器链结构图：

![1595940586386](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200801190055-422443.png)

FilterChainProxy是一个代理，真正起作用的是FilterChainProxy中SecurityFilterChain所包含的各个Filter，同时 这些Filter作为Bean被Spring管理，它们是Spring Security核心，各有各的职责，但**他们并不直接处理用户的认** **证，也不直接处理用户的授权，而是把它们交给了认证管理器（AuthenticationManager）和决策管理器** **（AccessDecisionManager）进行处理**，下图是FilterChainProxy相关类的UML图示。

![1595940625586](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200728205027-959035.png)

spring Security功能的实现主要是由一系列过滤器链相互配合完成。

![1595940683432](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200801190058-440832.png)

下面介绍过滤器链中主要的几个过滤器及其作用：

1. SecurityContextPersistenceFilter 这个Filter是整个拦截过程的入口和出口（也就是第一个和最后一个拦截 器），会在请求开始时从配置好的 SecurityContextRepository 中获取 SecurityContext，然后把它设置给 SecurityContextHolder。在请求完成后将 SecurityContextHolder 持有的 SecurityContext 再保存到配置好 的 SecurityContextRepository，同时清除 securityContextHolder 所持有的 SecurityContext；
2. UsernamePasswordAuthenticationFilter 用于处理来自表单提交的**认证**。该表单必须提供对应的用户名和密 码，其内部还有登录成功或失败后进行处理的 AuthenticationSuccessHandler 和 AuthenticationFailureHandler，这些都可以根据需求做相关改变；
3.  FilterSecurityInterceptor 是用于保护web资源的，使用AccessDecisionManager对当前用户进行**授权访问**，前 面已经详细介绍过了；

    > 2，3的认证和授权都是交给了认证管理器（AuthenticationManager）和决策管理器 （AccessDecisionManager）进行处理
4. ExceptionTranslationFilter 能够捕获来自 FilterChain 所有的异常，并进行处理。但是它只会处理两类异常： AuthenticationException 和 AccessDeniedException，其它的异常它会继续抛出。

#### 4.2.2.认证流程

**4.2.2.1 认证流程**

![1595941718976](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200801190101-898185.png)

让我们仔细分析认证过程：

1. 用户提交用户名、密码被SecurityFilterChain中的 UsernamePasswordAuthenticationFilter 过滤器获取到， 封装为请求Authentication，通常情况下是UsernamePasswordAuthenticationToken这个实现类。
2. 然后过滤器将Authentication提交至认证管理器（AuthenticationManager）进行认证
3. 认证成功后， AuthenticationManager 身份管理器返回一个被填充满了信息的（包括上面提到的权限信息， 身份信息，细节信息，但密码通常会被移除） Authentication 实例。
4. SecurityContextHolder 安全上下文容器将第3步填充了信息的 Authentication ，通过 SecurityContextHolder.getContext().setAuthentication(…)方法，设置到其中。
5. 可以看出AuthenticationManager接口（认证管理器）是认证相关的核心接口，也是发起认证的出发点，它 的实现类为ProviderManager。而Spring Security支持多种认证方式，因此ProviderManager维护着一个 List 列表，存放多种认证方式，最终实际的认证工作是由 AuthenticationProvider完成的。咱们知道web表单的对应的AuthenticationProvider实现类为AbstractUserDetailsAuthenticationProvider，AbstractUserDetailsAuthenticationProvider的实现类为 DaoAuthenticationProvider，它的内部又维护着一个UserDetailsService负责UserDetails的获取。最终 AuthenticationProvider将UserDetails填充至Authentication。

认证核心组件的大体关系如下：

![1595941761120](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200801190107-785607.png)

**4.2.2.2.AuthenticationProvider**

通过前面的Spring Security认证流程我们得知，认证管理器（AuthenticationManager）委托 AuthenticationProvider完成认证工作

AuthenticationProvider是一个接口，定义如下：

```java
public interface AuthenticationProvider {
Authentication authenticate(Authentication authentication) throws AuthenticationException;
	boolean supports(Class<?> var1);
}
```

authenticate()方法定义了认证的实现过程，**它的参数是一个Authentication，里面包含了登录用户所提交的用 户、密码等。而返回值也是一个Authentication，这个Authentication则是在认证成功后，将用户的权限及其他信 息重新组装后生成。**

Spring Security的ProviderManager中维护着一个 List 列表，存放多种认证方式，不同的认证方式使用不 同的AuthenticationProvider。如使用用户名密码登录时，使用AuthenticationProvider1，短信登录时使用 AuthenticationProvider2等等这样的例子很多。

每个AuthenticationProvider需要实现supports（）方法来表明自己支持的认证方式，如我们使用表单方式认证， 在提交请求时Spring Security会生成UsernamePasswordAuthenticationToken，它是一个Authentication，里面 封装着用户提交的用户名、密码信息。而对应的，哪个AuthenticationProvider来处理它？

我们在DaoAuthenticationProvider的基类AbstractUserDetailsAuthenticationProvider发现以下代码：

```java
public boolean supports(Class<?> authentication) {
	return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
}
```

也就是说当web表单提交用户名密码时，Spring Security由DaoAuthenticationProvider处理。

最后，我们来看一下Authentication(认证信息)的结构，它是一个接口，我们之前提到的 UsernamePasswordAuthenticationToken就是它的实现之一：

```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    Object getCredentials();
    Object getDetails();
    Object getPrincipal();
    boolean isAuthenticated();
    void setAuthenticated(boolean var1) throws IllegalArgumentException;
}
```

（1）Authentication是spring security包中的接口，直接继承自Principal类，而Principal是位于 java.security 包中的。它是表示着一个抽象主体身份，任何主体都有一个名称，因此包含一个getName()方法。 （2）getAuthorities()，权限信息列表，默认是GrantedAuthority接口的一些实现类，通常是代表权限信息的一系 列字符串。 （3）getCredentials()，凭证信息，用户输入的密码字符串，在认证过后通常会被移除，用于保障安全。 （4）getDetails()，细节信息，web应用中的实现接口通常为 WebAuthenticationDetails，它记录了访问者的ip地 址和sessionId的值。 （5）getPrincipal()，身份信息，大部分情况下返回的是UserDetails接口的实现类，UserDetails代表用户的详细 信息，那从Authentication中取出来的UserDetails就是当前登录用户信息，它也是框架中的常用接口之一。

**4.2.2.3.UserDetailsService**

1）认识UserDetailsService 现在咱们现在知道DaoAuthenticationProvider处理了web表单的认证逻辑，认证成功后既得到一个 Authentication(UsernamePasswordAuthenticationToken实现)，里面包含了身份信息（Principal）。这个身份 信息就是一个 Object ，大多数情况下它可以被强转为UserDetails对象。 DaoAuthenticationProvider中包含了一个UserDetailsService实例，它负责根据用户名提取用户信息 UserDetails(包含密码)，而后DaoAuthenticationProvider会去对比UserDetailsService提取的用户密码与用户提交 的密码是否匹配作为认证成功的关键依据，因此可以通过将自定义的 UserDetailsService 公开为spring bean来定 义自定义身份验证。

```java
public interface UserDetailsService {
	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

很多人把DaoAuthenticationProvider和UserDetailsService的职责搞混淆，其实UserDetailsService只负责从特定 的地方（通常是数据库）加载用户信息，仅此而已。而DaoAuthenticationProvider的职责更大，它完成完整的认 证流程，同时会把UserDetails填充至Authentication。 上面一直提到UserDetails是用户信息，咱们看一下它的真面目：

```java
public interface UserDetails extends Serializable {
    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

它和Authentication接口很类似，比如它们都拥有username，authorities。Authentication的getCredentials()与 UserDetails中的getPassword()需要被区分对待，前者是用户提交的密码凭证，后者是用户实际存储的密码，认证 其实就是对这两者的比对。（在AuthenticationProvider实现类AbstractUserDetailsAuthenticationProvider中）**Authentication中的getAuthorities()实际是由UserDetails的getAuthorities()传递而形** **成的。还记得Authentication接口中的getDetails()方法吗？Authentication中的UserDetails用户详细信息便是UserDetails经过了AuthenticationProvider认证之后被填充的。** 通过实现UserDetailsService和UserDetails，我们可以完成对用户信息获取方式以及用户信息字段的扩展。 Spring Security提供的InMemoryUserDetailsManager(内存认证)，JdbcUserDetailsManager(jdbc认证)就是 UserDetailsService的实现类，主要区别无非就是从内存还是从数据库加载用户。

**4.3.2.4.PasswordEncoder**

1）认识PasswordEncoder DaoAuthenticationProvider认证处理器通过UserDetailsService获取到UserDetails后，它是如何与请求 Authentication中的密码做对比呢？ 在这里Spring Security为了适应多种多样的加密类型，又做了抽象，DaoAuthenticationProvider通过 PasswordEncoder接口的matches方法进行密码的对比，而具体的密码对比细节取决于实现：

```java
public interface PasswordEncoder {
    String encode(CharSequence var1);
    boolean matches(CharSequence var1, String var2);
    default boolean upgradeEncoding(String encodedPassword) {
        return false;
    }
}
```

而Spring Security提供很多内置的PasswordEncoder，能够开箱即用，使用某种PasswordEncoder只需要进行如 下声明即可，如下：

```java
@Bean
public PasswordEncoder passwordEncoder() {
	return NoOpPasswordEncoder.getInstance();
}
```

NoOpPasswordEncoder采用字符串匹配方法，不对密码进行加密比较处理，密码比较流程如下： 1、用户输入密码（明文 ） 2、DaoAuthenticationProvider获取UserDetails（其中存储了用户的正确密码） 3、DaoAuthenticationProvider使用PasswordEncoder对输入的密码和正确的密码进行校验，密码一致则校验通 过，否则校验失败。

NoOpPasswordEncoder的校验规则拿 输入的密码和UserDetails中的正确密码进行字符串比较，字符串内容一致 则校验通过，否则 校验失败。

实际项目中推荐使用BCryptPasswordEncoder, Pbkdf2PasswordEncoder, SCryptPasswordEncoder等，感兴趣 的大家可以看看这些PasswordEncoder的具体实现。 2）使用BCryptPasswordEncoder 1、配置BCryptPasswordEncoder 在安全配置类中定义：

```java
@Bean
public PasswordEncoder passwordEncoder() {
	return new BCryptPasswordEncoder();
}
```

#### 4.2.3.授权流程

**4.2.3.1 授权流程**

通过快速上手我们知道，Spring Security可以通过 http.authorizeRequests() 对web请求进行授权保护。Spring Security使用标准Filter建立了对web请求的拦截，最终实现对资源的授权访问。Spring Security的授权流程如下：

![1595991975664](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200729110616-776834.png)

分析授权流程：

1. 拦截请求，已认证用户访问受保护的web资源将被SecurityFilterChain中的 FilterSecurityInterceptor 的子 类拦截。
2. 获取资源访问策略，FilterSecurityInterceptor的父类AbstractSecurityInterceptor的beforeInvocation方法中会从 SecurityMetadataSource 的子类 DefaultFilterInvocationSecurityMetadataSource 获取要访问当前资源所需要的权限Collection 。 SecurityMetadataSource其实就是读取访问策略的抽象，而读取的内容，其实就是我们配置的访问规则， 读 取访问策略如：

```java
http
.authorizeRequests()
.antMatchers("/r/r1").hasAuthority("p1")
.antMatchers("/r/r2").hasAuthority("p2")
...
```

1. 最后，FilterSecurityInterceptor的父类AbstractSecurityInterceptor的beforeInvocation方法中会调用 AccessDecisionManager 进行授权决策，若决策通过，则允许访问资 源，否则将禁止访问。

AccessDecisionManager（访问决策管理器）的核心接口如下:

```java
public interface AccessDecisionManager {
    /**
    * 通过传递的参数来决定用户是否有访问对应受保护资源的权限
    */
    void decide(Authentication authentication , Object object, Collection<ConfigAttribute>
    configAttributes ) throws AccessDeniedException, InsufficientAuthenticationException;
    //略..
}
```

这里着重说明一下decide的参数：

1. authentication：要访问资源的访问者的身份
2. object：要访问的受保护资源，web请求对应FilterInvocation
3. configAttributes：是受保护资源的访问策略，通过SecurityMetadataSource获取。
4. decide接口就是用来鉴定当前用户是否有访问对应受保护资源的权限。

**4.2.3.2 授权决策**

AccessDecisionManager采用投票的方式来确定是否能够访问受保护资源。

![1595992059093](https://gitee.com/gu\_chun\_bo/picture/raw/master/image/20200729110739-162380.png)

通过上图可以看出，AccessDecisionManager中包含的一系列AccessDecisionVoter将会被用来对Authentication 是否有权访问受保护对象进行投票，AccessDecisionManager根据投票结果，做出最终决策。 AccessDecisionVoter是一个接口，其中定义有三个方法，具体结构如下所示。

```java
public interface AccessDecisionVoter<S> {
    int ACCESS_GRANTED = 1;
    int ACCESS_ABSTAIN = 0;
    int ACCESS_DENIED = -1;
    boolean supports(ConfigAttribute var1);
    boolean supports(Class<?> var1);
    int vote(Authentication var1, S var2, Collection<ConfigAttribute> var3);
}
```

vote()方法的返回结果会是AccessDecisionVoter中定义的三个常量之一。ACCESS\_GRANTED表示同意， ACCESS\_DENIED表示拒绝，ACCESS\_ABSTAIN表示弃权。如果一个AccessDecisionVoter不能判定当前 Authentication是否拥有访问对应受保护对象的权限，则其vote()方法的返回值应当为弃权ACCESS\_ABSTAIN。 Spring Security内置了三个基于投票的AccessDecisionManager实现类如下，它们分别是 AffirmativeBased、ConsensusBased和UnanimousBased。

1. AffirmativeBased的逻辑是： （1）只要有AccessDecisionVoter的投票为ACCESS\_GRANTED则同意用户进行访问； （2）如果全部弃权也表示通过； （3）如果没有一个人投赞成票，但是有人投反对票，则将抛出AccessDeniedException。 Spring security默认使用的是AffirmativeBased。
2. ConsensusBased的逻辑是： （1）如果赞成票多于反对票则表示通过。 （2）反过来，如果反对票多于赞成票则将抛出AccessDeniedException。 （3）如果赞成票与反对票相同且不等于0，并且属性allowIfEqualGrantedDeniedDecisions的值为true，则表 示通过，否则将抛出异常AccessDeniedException。参数allowIfEqualGrantedDeniedDecisions的值默认为true。 （4）如果所有的AccessDecisionVoter都弃权了，则将视参数allowIfAllAbstainDecisions的值而定，如果该值 为true则表示通过，否则将抛出异常AccessDeniedException。参数allowIfAllAbstainDecisions的值默认为false。 UnanimousBased的逻辑与另外两种实现有点不一样，另外两种会一次性把受保护对象的配置属性全部传递 给AccessDecisionVoter进行投票，而UnanimousBased会一次只传递一个ConfigAttribute给 AccessDecisionVoter进行投票。这也就意味着如果我们的AccessDecisionVoter的逻辑是只要传递进来的 ConfigAttribute中有一个能够匹配则投赞成票，但是放到UnanimousBased中其投票结果就不一定是赞成了。
3. UnanimousBased的逻辑具体来说是这样的： （1）如果受保护对象配置的某一个ConfigAttribute被任意的AccessDecisionVoter反对了，则将抛出 AccessDeniedException。 （2）如果没有反对票，但是有赞成票，则表示通过。 （3）如果全部弃权了，则将视参数allowIfAllAbstainDecisions的值而定，true则通过，false则抛出 AccessDeniedException。

Spring Security也内置一些投票者实现类如RoleVoter、AuthenticatedVoter和WebExpressionVoter等，可以 自行查阅资料进行学习。
