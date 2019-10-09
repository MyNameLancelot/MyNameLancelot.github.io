---
layout: post
title: "Spring Security使用指南"
date: 2019-10-09 10:06:30
categories: spring
---

# 一、基于游览器的安全

## 表单登陆

**设置表单登陆【继承WebSecurityConfigurerAdapter进行适配】**

```java
/**
 * 游览器端安全配置，当访问非登陆请求时，会在FilterSecurityInterceptor进行判断是否有权限
 */
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
    
    // 密码的加密器，高版本必须使用
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    private AuthenticationSuccessHandler successHandler = new LoginSuccessHandler();
    private AuthenticationFailureHandler failureHandler = new LoginFailureHandler();
    
 	/**
     * 登陆、安全拦截配置
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin()                       			// 设置为基于表带验证
            // 当没有进行身份认证且访问需要权限的Api时跳入到这个请求
            .loginPage("/authentication/require")
            // 提交表单的地址，默认/login【查看UsernamePasswordAuthenticationFilter】
            .loginProcessingUrl("/authentication/commit")
            // 登陆成功的处理
            .successHandler(successHandler)
            // 登陆失败的处理
            .failureHandler(failureHandler)
            .and()
            .authorizeRequests()            						//认证的请求
            .antMatchers(HttpMethod.GET,
                         "/authentication/require",					//先访问的请求
                         "/authentication/commit",					//提交表单的地址
                         "/login/user.html")						//登录页
            .permitAll()                    						//允许所有
            .anyRequest()                   						//任何请求
            .authenticated()                						//需要身份认证
            .and().csrf().disable()         						//屏蔽csrf
    }
}

@RestController
public class BrowserSecurityController {

    private Logger log = LoggerFactory.getLogger(BrowserSecurityController.class);

    private RedirectStrategy redirectStrategy = new DefaultRedirectStrategy();

    /**
     * 当没有进行身份认证且访问需要权限的Api时跳入到这个请求
     * 如果没权限根据请求的方式【Ajax或者HTML】进行返回指定信息
     */
    @GetMapping("/authentication/require")
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public AuthenticationResponse requireAuthentication(HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
        String requestedWith = request.getHeader("X-Requested-With");
        if (!"XMLHttpRequest".equals(requestedWith)) {
            log.debug("未进行身份认证，游览器请求将跳转至登陆页面");
            redirectStrategy.sendRedirect(request, response, "/login/user.html");
            return null;
        }else {
            log.debug("未进行身份认证，AJAX请求将返回状态码401");
            return new AuthenticationResponse(request.getRequestURI(), "未登陆", HttpStatus.UNAUTHORIZED.value());
        }
    }
}

/**
 * 用户登陆成功
 */
public class LoginSuccessHandler implements AuthenticationSuccessHandler {

    private ObjectMapper objectMapper = new ObjectMapper();

    private Logger log = LoggerFactory.getLogger(LoginSuccessHandler.class);

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        log.debug("用户{}登陆成功", authentication.getPrincipal());
        SimpleResponse simpleResponse = new SimpleResponse(String.valueOf(HttpStatus.OK.value()), "登陆成功");
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write(objectMapper.writeValueAsString(simpleResponse));
        response.flushBuffer();
    }
}

/**
 * 用户登陆失败
 */
public class LoginFailureHandler implements AuthenticationFailureHandler {

    private Logger log = LoggerFactory.getLogger(LoginFailureHandler.class);

    private ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        log.info("登录失败");
        response.setStatus(HttpStatus.OK.value());
        response.setContentType("application/json;charset=UTF-8");
        SimpleResponse simpleResponse = new SimpleResponse(String.valueOf(HttpStatus.INTERNAL_SERVER_ERROR.value()), exception.getMessage());
        response.getWriter().write(objectMapper.writeValueAsString(simpleResponse));
    }
}

/**
 * 实现 UserDetailsService 接口，用户获取用户登陆信息
 */
@Service
public class SecurityUserDetailsServiceImpl implements UserDetailsService {
    
    private Logger log = LoggerFactory.getLogger(SecurityUserDetailsServiceImpl.class);

    @Autowired
    private UserService userService;
    
    // 过呢根据用户名进行登陆
	@Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        log.debug("根据用户名{}查找用户", username);
        // 查询user
        User user = userService.getUserByUserName(username);
        // 用户不存在
        if (user== null) {
            throw new UsernameNotFoundException( "用户名" + username + "账户不存在");
        }
        return user;
    }
}

/*
 * 实现UserDetails满足UserDetailsService需要返回的类型
 */
public class User implements UserDetails {
   private Integer id;
   private String  username;
   private String  password;
    
   // 权限信息
   @Override
   public Collection<? extends GrantedAuthority> getAuthorities() {
      return AuthorityUtils.createAuthorityList("admin");
   }

   @Override
   public String getPassword() {
      return this.password;
   }

   @Override
   public String getUsername() {
      return this.username;
   }

   // 账户没有过期
   @Override
   public boolean isAccountNonExpired() {
      return true;
   }

   // 账户没有锁定
   @Override
   public boolean isAccountNonLocked() {
      return true;
   }

   // 密码没有过期
   @Override
   public boolean isCredentialsNonExpired() {
      return true;
   }

   // 账户是否可用【注销？】
   @Override
   public boolean isEnabled() {
      return true;
   }
   //=======================Getter/Setter=======================
}
```

**请求经过过滤器的流程图**

![form-request-filter](/img/springsecurity/form-request-filter.png)



## 图片验证码

```java
/**
 * 验证码生成工具类
 */
@Configuration
@ConditionalOnProperty(prefix = "login.verify.image", name = "enable")
@EnableConfigurationProperties(ImageCodeProperties.class)
public class ImageCodeUtil {

    private Logger log = LoggerFactory.getLogger(ImageCodeUtil.class);

    private ImageCodeProperties imageCodeProperties;

    public ImageCodeUtil(ImageCodeProperties imageCodeProperties) {
        this.imageCodeProperties = imageCodeProperties;
    }

    /**
     * code为生成的验证码
     * codePic为生成的验证码BufferedImage对象
     */
    public ImageValidateCode generateCodeAndPic() {
        Long expireDate     = imageCodeProperties.getExpireDate();      // 过期时间
        int  imageWidth     = imageCodeProperties.getImageWidth();      // 定义图片的width
        int  imageHeight    = imageCodeProperties.getFontHeight();      // 定义图片的height
        int  codeCount      = imageCodeProperties.getCodeCount();       // 定义图片上显示验证码的个数
        int  fontHeight     = imageCodeProperties.getFontHeight();      // 字体高度
        int  offsetX        = imageCodeProperties.getOffsetX();         // 验证码生成X轴间隔
        int  offsetY        = imageCodeProperties.getOffsetY();         // 验证码生成Y轴间距
        String fontName     = imageCodeProperties.getFontName();        // 字体
        char[] codeSequence = imageCodeProperties.getCodeSequence();    // 生成序列

        // 定义图像buffer
        BufferedImage buffImg = new BufferedImage(imageWidth, imageHeight, BufferedImage.TYPE_INT_RGB);
        Graphics graphics = buffImg.getGraphics();
        // 创建一个随机数生成器类
        Random random = new Random();
        // 将图像填充为白色
        graphics.setColor(Color.WHITE);
        graphics.fillRect(0, 0, imageWidth, imageHeight);
        // 创建字体，字体的大小应该根据图片的高度来定。
        Font font = new Font(fontName, Font.BOLD, fontHeight);
        // 设置字体。
        graphics.setFont(font);
        // 画边框。
        graphics.setColor(Color.BLACK);
        graphics.drawRect(0, 0, imageWidth - 1, imageHeight - 1);
        // 随机产生30条干扰线，使图象中的认证码不易被其它程序探测到。
        graphics.setColor(Color.BLACK);
        for (int i = 0; i < 0; i++) {
            int x1 = random.nextInt(imageWidth);
            int y1 = random.nextInt(imageHeight);
            int x2 = random.nextInt(imageWidth);
            int y2 = random.nextInt(imageHeight);
            graphics.drawLine(x1, y1, x1 + x2, y1 + y2);
        }
        // randomCode用于保存随机产生的验证码，以便用户登录后进行验证。
        StringBuffer randomCode = new StringBuffer();
        int red   = 0;
        int green = 0;
        int blue  = 0;
        // 随机产生codeCount数字的验证码。
        for (int i = 0; i < codeCount; i++) {
            // 得到随机产生的验证码数字。
            String code = String.valueOf(codeSequence[random.nextInt(codeSequence.length)]);
            // 产生随机的颜色分量来构造颜色值，这样输出的每位数字的颜色值都将不同。设置为230避免使用纯白色
            red = random.nextInt(230);
            green = random.nextInt(230);
            blue = random.nextInt(230);
            // 用随机产生的颜色将验证码绘制到图像中。
            graphics.setColor(new Color(red, green, blue));
            graphics.drawString(code, (i + 1) * offsetX, offsetY);
            // 将产生的四个随机数组合在一起。
            randomCode.append(code);
        }
        ImageValidateCode imageCode = new ImageValidateCode(randomCode.toString(), buffImg);
        if (expireDate != null){
            imageCode.setExpireDate(Instant.now().getEpochSecond() + expireDate);
        }
        log.debug("生成图片验证码{}", imageCode.getCode());
        return imageCode ;
    }
}

@ConfigurationProperties(prefix = "login.verify.image", ignoreUnknownFields = true)
public class ImageCodeProperties {
    private static final int DEFAULT_IMAGE_WIDTH  = 90;
    private static final int DEFAULT_IMAGE_HEIGHT = 90;
    private static final int DEFAULT_CODE_COUNT   = 4;
    private static final int DEFAULT_FONT_HEIGHT  = 18;
    private static final int DEFAULT_IMAGE_OFFSET_X   = 15;
    private static final int DEFAULT_IMAGE_OFFSET_Y   = 16;
    private static final String DEFAULT_FONT_NAME     = "微软雅黑";
    private static final char[] DEFAULT_CODE_SEQUENCE = { 'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'J', 'K', 'L', 'M',
            'N', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', '2', '3', '4', '5', '6', '7', '8', '9' };

    private Long expireDate  = 60L;                      // 过期时间
    private int  imageWidth  = DEFAULT_IMAGE_WIDTH;      // 定义图片的width
    private int  imageHeight = DEFAULT_IMAGE_HEIGHT;     // 定义图片的height
    private int  codeCount   = DEFAULT_CODE_COUNT;       // 定义图片上显示验证码的个数
    private int  fontHeight  = DEFAULT_FONT_HEIGHT;      // 字体高度
    private int  offsetX     = DEFAULT_IMAGE_OFFSET_X;   // 验证码生成X轴间隔
    private int  offsetY     = DEFAULT_IMAGE_OFFSET_Y;   // 验证码生成Y轴间距
    private String  fontName = DEFAULT_FONT_NAME;        // 字体
    private char[] codeSequence = DEFAULT_CODE_SEQUENCE; // 生成序列

   //=======================Getter/Setter=======================
}

/*
 * 获得验证码的Controller
 */
@ConditionalOnProperty(prefix = "login.verify.image", value = "enable", havingValue = "true")
@RestController
public class ImageCodeController {

    // spring工具类用于操作session
    private SessionStrategy sessionStrategy = new HttpSessionSessionStrategy();

    @Autowired
    private ImageCodeUtil imageCodeUtil;

    public static final String SESSION_IMAGE_VERIFY_CODE = "SESSION_IMAGE_VERIFY_CODE";

    @GetMapping("/verify/image/code")
    public void getVerifyCode(HttpServletRequest request, HttpServletResponse response) throws IOException {
        ImageValidateCode imageCode = imageCodeUtil.generateCodeAndPic();

        // 设置响应的类型格式为图片格式
        response.setContentType("image/jpeg");
        // 禁止图像缓存。
        response.setHeader("Pragma", "no-cache");
        response.setHeader("Cache-Control", "no-cache");
        response.setDateHeader("Expires", 0);
        // 写入图片
        ImageIO.write(imageCode.getCodePic(), "jpg", response.getOutputStream());
        response.getOutputStream().flush();

        // code放入session
        imageCode.setCodePic(null);
        sessionStrategy.setAttribute(new ServletWebRequest(request),SESSION_IMAGE_VERIFY_CODE,imageCode);
    }
}

/**
 * 验证码校验Filter，需要在BrowserSecurityConfig中在UsernamePasswordAuthenticationFilter
 * 前配置http.addFilterBefore(imageValidateCodeFilter, UsernamePasswordAuthenticationFilter.class)
 */ 
public class ImageValidateCodeFilter extends OncePerRequestFilter {

    @Autowired
    private AuthenticationFailureHandler authenticationFailureHandler;

    private SessionStrategy sessionStrategy = new HttpSessionSessionStrategy();

    public ImageValidateCodeFilter(AuthenticationFailureHandler authenticationFailureHandler) {
        this.authenticationFailureHandler = authenticationFailureHandler;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 是登陆请求而且为POST
        if(StringUtils.equals("/authentication/commit", request.getRequestURI()) && StringUtils.equalsIgnoreCase(request.getMethod(), "POST")) {
            try {
                validate(new ServletWebRequest(request));
                filterChain.doFilter(request, response);
            } catch (ValidateCodeException e) {
                authenticationFailureHandler.onAuthenticationFailure(request, response, e);
            } 
        }else {
            filterChain.doFilter(request, response);
        }
    }

    private void validate(ServletWebRequest servletWebRequest) throws ServletRequestBindingException{
        ImageValidateCode codeInSession = (ImageValidateCode) sessionStrategy.getAttribute(servletWebRequest, ImageCodeController.SESSION_IMAGE_VERIFY_CODE);
        String codeInRequest = ServletRequestUtils.getStringParameter(servletWebRequest.getRequest(),"imageCode");

        if(StringUtils.isEmpty(codeInRequest)) {
            throw new ValidateCodeException("提交的验证码不能为空");
        }

        if(codeInSession == null) {
            throw new ValidateCodeException("验证码不存在");
        }
        if(codeInSession.compareExpired()) {
            sessionStrategy.removeAttribute(servletWebRequest, ImageCodeController.SESSION_IMAGE_VERIFY_CODE);
            throw new ValidateCodeException("验证码已过期");
        }


        if(!StringUtils.equalsIgnoreCase(codeInRequest, codeInSession.getCode())) {
            throw new ValidateCodeException("验证码不匹配");
        }

        // 匹配成功
        sessionStrategy.removeAttribute(servletWebRequest, ImageCodeController.SESSION_IMAGE_VERIFY_CODE);
    }
}
```

## 记住我

```java
/**
 * Security配置，部分信息省略
 */
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {
    
    // 创建表语句在JdbcTokenRepositoryImpl包含
    @Bean
    public PersistentTokenRepository persistentTokenRepository(DataSource dataSource) {
        JdbcTokenRepositoryImpl jdbcTokenRepository = new JdbcTokenRepositoryImpl();
        jdbcTokenRepository.setDataSource(dataSource);
        return jdbcTokenRepository;
    }
   /**
     * 登陆、安全拦截配置
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {   
          http.addFilterBefore(imageValidateCodeFilter, UsernamePasswordAuthenticationFilter.class)
              //基于表带验证
              .formLogin()                            
              //当没有进行身份认证且访问需要权限的Api时跳入到这个请求
              .loginPage("/authentication/require")   
              .loginProcessingUrl("/authentication/commit")
              .successHandler(successHandler)
              .failureHandler(failureHandler)
              .and()
              	  // 会加入RememberMeAuthenticationFilter
                  .rememberMe()									 //启用rememberMe
                  .tokenRepository(persistentTokenRepository())
                  .userDetailsService(userDetailsService)
                  .tokenValiditySeconds(60 * 60 * 24 * 7)	    //记住我的有效时间
                  .rememberMeParameter("rememberMe")			//rememberMe参数名称
              .and()
              .authorizeRequests()            						//认证的请求
              .antMatchers(HttpMethod.GET,
                         "/authentication/require",					//先访问的请求
                         "/authentication/commit",					//提交表单的地址
                         "/login/user.html")						//登录页
              .permitAll()                    						//允许所有
              .anyRequest()                   						//任何请求
              .authenticated()                						//需要身份认证
              .and().csrf().disable()         						//屏蔽csrf    
    }
}
```

## 手机验证码

```java
// 伪短信验证码生成
@Configuration
@ConditionalOnProperty(prefix = "login.verify.sms", name = "enable")
@EnableConfigurationProperties(SmsCodeProperties.class)
public class SmsCodeUtil {

    private Logger log = LoggerFactory.getLogger(SmsCodeUtil.class);

    private SmsCodeProperties smsCodeProperties;

    public SmsCodeUtil(SmsCodeProperties smsCodeProperties) {
        this.smsCodeProperties = smsCodeProperties;
    }

    /**
     * code为生成的验证码,此过程要发送短信
     */
    public SmsValidateCode generateCode(String mobile) {
        Long expireDate = smsCodeProperties.getExpireDate();     // 过期时间
        int  codeCount  = smsCodeProperties.getCodeCount();      // 定义短信验证码的个数

        SmsValidateCode smsCode = new SmsValidateCode(RandomStringUtils.randomNumeric(codeCount), mobile);
        if (expireDate != null){
            smsCode.setExpireDate(Instant.now().getEpochSecond() + expireDate);
        }
        log.debug("生成短信验证码{}", smsCode.getCode());
        return smsCode ;
    }
}

@ConfigurationProperties(prefix = "login.verify.sms", ignoreUnknownFields = true)
public class SmsCodeProperties {

    private static final int DEFAULT_CODE_COUNT   = 4;
    private static final Long DEFAULT_EXPIRE_DATE = 300L;

    private Long    expireDate = DEFAULT_EXPIRE_DATE;  // 过期时间
    private Integer codeCount  = DEFAULT_CODE_COUNT;   // 长度
    //=======================Getter/Setter=======================
}

//短信验证码请求的URL
@RestController
@ConditionalOnProperty(prefix="login.verify.sms", name = "enable")
public class SmsCodeController {

    private Logger log = LoggerFactory.getLogger(SmsCodeController.class);

    public static final String SMS_CODE_MOBILE_PREFIX = "SMS_CODE_MOBILE_";

    @Autowired
    private SmsCodeUtil smsCodeUtil;

    // spring工具类用于操作session
    private SessionStrategy sessionStrategy = new HttpSessionSessionStrategy();

    @GetMapping("/verify/sms/code/{mobile}", produces = MimeTypeUtils.APPLICATION_JSON_VALUE)
    public SimpleResponse getVerifyCode(@PathVariable("mobile") String mobile, HttpServletRequest request, HttpServletResponse response) throws IOException {
        SmsValidateCode smsValidateCode = smsCodeUtil.generateCode(mobile);
        log.info("给{}手机发送验证短信，验证码{}", smsValidateCode.getMobile(), smsValidateCode.getCode());

        sessionStrategy.setAttribute(new ServletWebRequest(request), SMS_CODE_MOBILE_PREFIX + mobile, smsValidateCode);
        
        SimpleResponse simpleResponse = new SimpleResponse();
        simpleResponse.setContent("短信发送成功");
        simpleResponse.setCode("200");
        return SimpleResponse;
    }
}

// 短信验证码校验
public class SmsValidateCodeFilter extends OncePerRequestFilter {

    @Autowired
    private AuthenticationFailureHandler authenticationFailureHandler;

    private SessionStrategy sessionStrategy = new HttpSessionSessionStrategy();

    public SmsValidateCodeFilter(AuthenticationFailureHandler authenticationFailureHandler) {
        this.authenticationFailureHandler = authenticationFailureHandler;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 是登陆请求而且为POST
        if(StringUtils.equals("/authentication/sms/commit", request.getRequestURI()) && StringUtils.equalsIgnoreCase(request.getMethod(), "POST")) {
            try {
                validate(new ServletWebRequest(request));
                filterChain.doFilter(request, response);
            } catch (ValidateCodeException e) {
                authenticationFailureHandler.onAuthenticationFailure(request, response, e);
            }
        }else {
            filterChain.doFilter(request, response);
        }
    }

    private void validate(ServletWebRequest servletWebRequest) {

        String questInMobile =  null;
        String questInCode =  null;

        try {
            questInMobile = ServletRequestUtils.getRequiredStringParameter(servletWebRequest.getRequest(), "mobile");
            questInCode = ServletRequestUtils.getRequiredStringParameter(servletWebRequest.getRequest(), "code");
        }catch (ServletRequestBindingException e) {
            throw new ValidateCodeException("提交的短信验证码不能为空");
        }

        SmsValidateCode smsValidateCode = (SmsValidateCode)sessionStrategy.getAttribute(servletWebRequest, SmsCodeController.SMS_CODE_MOBILE_PREFIX + questInMobile);

        if(StringUtils.isEmpty(questInMobile)) {
            throw new ValidateCodeException("提交的手机号不能为空");
        }

        if(StringUtils.isEmpty(questInCode)) {
            throw new ValidateCodeException("提交的短信验证码不能为空");
        }

        if(smsValidateCode == null) {
            throw new ValidateCodeException("手机号" + questInMobile + "短信验证码不存在");
        }
        if(smsValidateCode.compareExpired()) {
            throw new ValidateCodeException("短信验证码已过期");
        }

        if(!StringUtils.equalsIgnoreCase(questInCode, smsValidateCode.getCode())) {
            throw new ValidateCodeException("验证码不匹配");
        }
    }
}

// 校验通过时，ProviderManager挑选对应Provider加载用户信息【权限】
public class SmsCodeAuthenticationProvider implements AuthenticationProvider {

    private Logger log = LoggerFactory.getLogger(SmsCodeAuthenticationProvider.class);

    private UserService userService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        log.debug("根据手机号{}查找用户", authentication.getPrincipal());

        SmsCodeAuthenticationToken smsCodeAuthenticationToken =  (SmsCodeAuthenticationToken) authentication;
        User user = userService.getUserByMobile((String) smsCodeAuthenticationToken.getPrincipal());

        return new SmsCodeAuthenticationToken(user.getUsername(),user.getPassword(),user.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return  SmsCodeAuthenticationToken.class.isAssignableFrom(authentication);
    }

    public void setUserService(UserService userService) {
        this.userService = userService;
    }
}

/**
 * 短信认证的Token
 */
public class SmsCodeAuthenticationToken extends AbstractAuthenticationToken {

    private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;

    private  Object principal;
    private Object credentials;

    public SmsCodeAuthenticationToken(Object principal, Object credentials) {
        super(null);
        this.principal = principal;
        this.credentials = credentials;
        setAuthenticated(false);
    }

    public SmsCodeAuthenticationToken(Object principal, Object credentials,
                                      Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        this.credentials = credentials;
        super.setAuthenticated(true); // must use super, as we override
    }

    public Object getCredentials() {
        return this.credentials;
    }

    public Object getPrincipal() {
        return this.principal;
    }

    public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
        if (isAuthenticated) {
            throw new IllegalArgumentException(
                    "Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
        }

        super.setAuthenticated(false);
    }

    @Override
    public void eraseCredentials() {
        super.eraseCredentials();
    }
}

/**
 * Security配置，部分信息省略
 */
@Configuration
public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {

    /**
     * 登陆、安全拦截配置
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {   
          http.addFilterBefore(imageValidateCodeFilter, UsernamePasswordAuthenticationFilter.class)
              .addFilterBefore(smsValidateCodeFilter, UsernamePasswordAuthenticationFilter.class)
              //基于表带验证
              .formLogin()                            
              //当没有进行身份认证且访问需要权限的Api时跳入到这个请求
              .loginPage("/authentication/require")   
              .loginProcessingUrl("/authentication/commit")
              .successHandler(successHandler)
              .failureHandler(failureHandler)
              .and()
              	  // 会加入RememberMeAuthenticationFilter
                  .rememberMe()									 //启用rememberMe
                  .tokenRepository(persistentTokenRepository())
                  .userDetailsService(userDetailsService)
                  .tokenValiditySeconds(60 * 60 * 24 * 7)	    //记住我的有效时间
                  .rememberMeParameter("rememberMe")			//rememberMe参数名称
              .and()
              .authorizeRequests()            						//认证的请求
              .antMatchers(HttpMethod.GET,
                         "/authentication/require",					//先访问的请求
                         "/authentication/commit",					//提交表单的地址
                         "/login/user.html")						//登录页
              .permitAll()                    						//允许所有
              .anyRequest()                   						//任何请求
              .authenticated()                						//需要身份认证
              .and().csrf().disable()         						//屏蔽csrf    
    }
}
```

## 第三方登陆【以QQ为例】

![spring-oauth2-detail](/img/springsecurity/spring-oauth2-detail.png)

**QQ登陆**

```java
public class QQUserInfo {
    /**
     * 	返回码
     */
    private String ret;
    /**
     * 如果ret<0，会有相应的错误信息提示，返回数据全部用UTF-8编码。
     */
    private String msg;
    /**
     *
     */
    private String openId;
    /**
     * 不知道什么东西，文档上没写，但是实际api返回里有。
     */
    private String is_lost;
    /**
     * 省(直辖市)
     */
    private String province;
    /**
     * 市(直辖市区)
     */
    private String city;
    /**
     * 出生年月
     */
    private String year;
    /**
     * 	用户在QQ空间的昵称。
     */
    private String nickname;
    /**
     *  星座
     */
    private String constellation;
    /**
     * 	大小为30×30像素的QQ空间头像URL。
     */
    private String figureurl;
    /**
     * 	大小为50×50像素的QQ空间头像URL。
     */
    private String figureurl_1;
    /**
     * 	大小为100×100像素的QQ空间头像URL。
     */
    private String figureurl_2;
    /**
     * 	大小为40×40像素的QQ头像URL。
     */
    private String figureurl_qq_1;
    /**
     * 	大小为100×100像素的QQ头像URL。需要注意，不是所有的用户都拥有QQ的100×100的头像，但40×40像素则是一定会有。
     */
    private String figureurl_qq_2;
    /**
     * 	性别。 如果获取不到则默认返回”男”
     */
    private String gender;
    /**
     * 	标识用户是否为黄钻用户（0：不是；1：是）。
     */
    private String is_yellow_vip;
    /**
     * 	标识用户是否为黄钻用户（0：不是；1：是）
     */
    private String vip;
    /**
     * 	黄钻等级
     */
    private String yellow_vip_level;
    /**
     * 	黄钻等级
     */
    private String level;
    /**
     * 标识是否为年费黄钻用户（0：不是； 1：是）
     */
    private String is_yellow_year_vip;
    //=======================Getter/Setter=======================
}

/**
 * 获取QQ用户信息
 */
public interface QQ {
    QQUserInfo getUserInfo();
}

/**
 * 获取QQ用户信息实现,继承AbstractOAuth2ApiBinding完成用户信息获取
 */
public class QQImpl extends AbstractOAuth2ApiBinding implements QQ{

    // 获取openid【用户的唯一标识】的URL
    private static final String GET_OPENID_URL = "https://graph.qq.com/oauth2.0/me?access_token={0}";

    // 获取用户信息的URL，accessToken会因为TokenStrategy.ACCESS_TOKEN_PARAMETER自动带上
    private static final String GET_USER_INFO_URL = "https://graph.qq.com/user/get_user_info?oauth_consumer_key={0}&openid={1}";

    // 申请QQ互联的APP ID
    private String oauthConsumerKey;

    // 用户openid【用户的唯一标识】
    private String openid;

    public QQImpl(String accessToken, String oauthConsumerKey) {
        super(accessToken, TokenStrategy.ACCESS_TOKEN_PARAMETER);
        this.oauthConsumerKey = oauthConsumerKey;
        // 获取openid
        String openidResult = getRestTemplate().getForObject(MessageFormat.format(GET_OPENID_URL, accessToken), String.class);
        if(StringUtils.isEmpty(openidResult)) {
            throw new RuntimeException("openid获取失败");
        }
        this.openid = StringUtils.substringBetween(openidResult, "\"openid\":\"", "\"}");
    }

    @Override
    public QQUserInfo getUserInfo() {
        // 填充数据
        String url = MessageFormat.format(GET_USER_INFO_URL, oauthConsumerKey, openid);
        String userInfoJson = getRestTemplate().getForObject(url, String.class);
        QQUserInfo qqUserInfo = null;
        try {
            // 解析用户信息
            ObjectMapper objectMapper = new ObjectMapper();
            objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
            qqUserInfo = objectMapper.readValue(userInfoJson, QQUserInfo.class);
            qqUserInfo.setOpenId(openid);
        } catch (IOException e) {
            throw new RuntimeException("json转换失败");
        }
        return qqUserInfo;
    }
}

/**
 * 将返回的QQ用户信息转换为标准的用户信息用于存储
 */
public class QQApiAdapter implements ApiAdapter<QQ> {

    /**
     * 不对网络进行检测
     */
    @Override
    public boolean test(QQ api) {
        return true;
    }

    /**
     * 获取用户信息
     */
    @Override
    public void setConnectionValues(QQ api, ConnectionValues values) {
        QQUserInfo qqUserInfo = api.getUserInfo();
        // 显示的用户名（QQ为昵称）
        values.setDisplayName(qqUserInfo.getNickname());
        // 头像（QQ头像）
        values.setImageUrl(qqUserInfo.getFigureurl_qq_1());
        // 个人主页（QQ没有）
        values.setProfileUrl(null);
        // 用户的唯一标识ID
        values.setProviderUserId(qqUserInfo.getOpenId());
    }

    /**
     * 解绑
     */
    @Override
    public UserProfile fetchUserProfile(QQ api) {
        return null;
    }

    @Override
    public void updateStatus(QQ api, String message) {
        //发送消息等，QQ没有个人主页，时间线不能更新
    }
}

/**
 * 和服务提供商交换信息的类
 */
public class QQServiceProvider extends AbstractOAuth2ServiceProvider<QQ> {

    // QQ登录页URL
    private static final String AUTHORIZE_URL =    "https://graph.qq.com/oauth2.0/authorize";

    // QQ获取Token的URL
    private static final String ACCESS_TOKEN_URL = "https://graph.qq.com/oauth2.0/token";

    // QQ互联申请的App ID
    private String oauthConsumerKey;

    /**
     * clientId : QQ互联申请的App ID
     * clientSecret : QQ互联申请的APP Key
     */
    public QQServiceProvider(String clientId, String clientSecret) {
        super(new QQOAuthTemplate(clientId, clientSecret, AUTHORIZE_URL, ACCESS_TOKEN_URL));
        this.oauthConsumerKey = clientId;
    }

    @Override
    public QQ getApi(String accessToken) {
        return new QQImpl(accessToken, oauthConsumerKey);
    }
}

/**
 * 发出请求的类
 */
public class QQOAuthTemplate extends OAuth2Template {

    public QQOAuthTemplate(String clientId, String clientSecret, String authorizeUrl, String accessTokenUrl) {
        super(clientId, clientSecret, authorizeUrl, accessTokenUrl);
        // setUseParametersForClientAuthentication设置为true才能将clientId、clientSecret以请求参数的方式带出去
        setUseParametersForClientAuthentication(true);
    }


    /**
     * QQ返回的用户信息是text/html，所以需要处理返回类型
     */
    @Override
    protected RestTemplate createRestTemplate() {
        RestTemplate restTemplate = super.createRestTemplate();
        restTemplate.getMessageConverters().add(new StringHttpMessageConverter());
        return restTemplate;
    }

    @Override
    protected AccessGrant postForAccessGrant(String accessTokenUrl, MultiValueMap<String, String> parameters) {
        String response = getRestTemplate().postForObject(accessTokenUrl, parameters, String.class);
        String[] items = response.split("&");
        String accessToken = StringUtils.substringAfterLast(items[0], "=");
        Long expiresIn = Long.valueOf(StringUtils.substringAfterLast(items[1], "="));
        String refreshToken = StringUtils.substringAfterLast(items[2], "=");
        return new AccessGrant(accessToken, null, refreshToken, expiresIn);
    }
}

/**
 * 获取连接信息的类
 */
public class QQConnectFactory extends OAuth2ConnectionFactory<QQ> {

    /**
     * providerId：要与发出请求的路径最后结尾相同
     * clientId：APP ID
     * clientSecret：APP Key
     */
    public QQConnectFactory(String providerId, String clientId, String clientSecret ) {
        super(providerId, new QQServiceProvider(clientId, clientSecret), new QQApiAdapter());
    }
}

@ConfigurationProperties(prefix = "qq.social", ignoreUnknownFields = true)
public class QQSocialProperties {

    private String appId;
    private String appKey;
    private String callBackUri;
    private String providerId;
    
    //=======================Getter/Setter=======================
}

@Configuration
@ConditionalOnProperty(prefix = "qq.social", name = "enable")
@EnableConfigurationProperties(QQSocialProperties.class)
@EnableSocial
public class QQSocialConfig extends SocialConfigurerAdapter {

    private DataSource dataSource;

    private QQSocialProperties qqSocialProperties;

    @Autowired
    private UserService userService;

    public QQSocialConfig(QQSocialProperties qqSocialProperties, DataSource dataSource) {
        this.dataSource = dataSource;
        this.qqSocialProperties = qqSocialProperties;
    }

    @Override
    public UserIdSource getUserIdSource() {
        return new AuthenticationNameUserIdSource();
    }

    @Override
    public UsersConnectionRepository getUsersConnectionRepository(ConnectionFactoryLocator connectionFactoryLocator) {
        JdbcUsersConnectionRepository jdbcUsersConnectionRepository = new JdbcUsersConnectionRepository(dataSource, connectionFactoryLocator, Encryptors.noOpText());
        // 自注册，注意权限也要设置
        jdbcUsersConnectionRepository.setConnectionSignUp(new ConnectionSignUp() {
            @Override
            public String execute(Connection<?> connection) {
                User user = new User();
                user.setUsername(UUID.randomUUID().toString());
                userService.save(user);
                return user.getUserId();
            }
        });
        return jdbcUsersConnectionRepository;
    }

    @Override
    public void addConnectionFactories(ConnectionFactoryConfigurer connectionFactoryConfigurer, Environment environment) {
        connectionFactoryConfigurer.addConnectionFactory(
                new QQConnectFactory(qqSocialProperties.getProviderId(), qqSocialProperties.getAppId(), qqSocialProperties.getAppKey()));
    }
}

@Configuration
@ConditionalOnProperty(prefix = "qq.social", name = "enable")
@EnableConfigurationProperties(QQSocialProperties.class)
public class QQSpringSocialConfigurer extends SpringSocialConfigurer {

    private QQSocialProperties qqSocialProperties;

    // 如果是没设置setConnectionSignUp则需要设置注册页
    public QQSpringSocialConfigurer(QQSocialProperties qqSocialProperties) {
        this.qqSocialProperties = qqSocialProperties;
        super.signupUrl("/register.html");
    }

    @Override
    protected <T> T postProcess(T object) {
        SocialAuthenticationFilter filter = (SocialAuthenticationFilter)super.postProcess(object);
        filter.setFilterProcessesUrl(qqSocialProperties.getCallBackUri());
        return (T) filter;
    }
}

@RestController
public class UserController {

    private BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
    @Autowired
    private UserService userService;

    @Autowired
    private ProviderSignInUtils providerSignInUtils;

    @PostMapping("/user/register")
    public String register(User user, HttpServletRequest request) {
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        userService.save(user);
        // 调用再次尝试使用第三方账户登陆
        providerSignInUtils.doPostSignUp(user.getUserId(), new ServletRequestAttributes(request));
        // 前台应该请求index页面
        return "ok";
    }
}

public class BrowserSecurityConfig extends WebSecurityConfigurerAdapter {

    /**
     * 获取社交账户信息
     */
    @Bean
    public ProviderSignInUtils providerSignInUtils(ConnectionFactoryLocator connectionFactoryLocator, UsersConnectionRepository connectionRepository){
        return new ProviderSignInUtils(connectionFactoryLocator, connectionRepository);
    }
}
```

**查看绑定状态&绑定&解绑**

前置需要手动`@Import(ConnectController.class)`并多配置一个回调接口为`/connect/{providerId}`

- 查看绑定状态
  - 拿到已绑定的列表[/connect method=get]
  - 配置视图

  ```java
  @Component("connect/status")
  public class ConnectStatusView extends AbstractView {
  
      private ObjectMapper objectMapper = new ObjectMapper();
  
      @Override
      protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
          Map<String, List<Connection<?>>> connectionMap = (Map<String, List<Connection<?>>>)model.get("connectionMap");
          Map<String, Boolean> connections = new HashMap<>();
          for (Map.Entry<String, List<Connection<?>>> entry : connectionMap.entrySet()) {
              connections.put(entry.getKey(), !CollectionUtils.isEmpty(entry.getValue()));
          }
          response.setContentType(MimeTypeUtils.APPLICATION_JSON_VALUE);
          response.setCharacterEncoding(CharEncoding.UTF_8);
          response.getWriter().write(objectMapper.writeValueAsString(connections));
      }
  }
  ```

- 绑定&解绑

  - 绑定[/connect/qq 			method=post]
  - 解绑[/connect/wechat 	method=delete]
  - 绑定视图&解绑视图

  ```java
  @Component({"connect/qqConnected", "connect/qqConnect"})
  public class ConnectView extends AbstractView {
      @Override
      protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
          response.setCharacterEncoding(CharEncoding.UTF_8);
          response.setContentType(MimeTypeUtils.TEXT_HTML_VALUE);
          if(model.get("connections") == null) {
              response.getWriter().write("<h3>解绑成功</h3>");
          }else {
              response.getWriter().write("<h3>绑定成功</h3>");
          }
      }
  }
  ```

## Session管理

**session超时设置**

```properties
server.servlet.session.timeout=PT30M
```

**session并发控制**

```java
 /**
  * 登陆、安全拦截配置
  */
@Override
protected void configure(HttpSecurity http) throws Exception {
    
    http //省略各项配置
        .and()
        	.sessionManagement()
        	.invalidSessionUrl("/session/invalid")	//session失效的处理URL
        	.maximumSessions(1)			        //只允许一个用户登陆,后登陆的把前登陆顶掉
        	//.maxSessionsPreventsLogin(true) 	//只允许一个用户登陆,不允许在在别的地方登陆
        	.expiredSessionStrategy(new BrowserSessionInformationExpiredStrategy())   //重复登陆提示
        .and()  //省略各项配置
}

public class BrowserSessionInformationExpiredStrategy implements SessionInformationExpiredStrategy {

    @Override
    public void onExpiredSessionDetected(SessionInformationExpiredEvent event) throws IOException, ServletException {
        event.getResponse().setContentType(MimeTypeUtils.APPLICATION_JSON_VALUE);
        event.getResponse().setCharacterEncoding(CharEncoding.UTF_8);
        event.getResponse().getWriter().write("账号在别处登陆");
    }
}

@RestController
public class BrowserSecurityController {
   
    @GetMapping("/session/invalid")
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public AuthenticationResponse invalidSession(HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException {
        String requestedWith = request.getHeader("X-Requested-With");
        if (!"XMLHttpRequest".equals(requestedWith)) {
            log.debug("会话过期，游览器请求将跳转至登陆页面");
            redirectStrategy.sendRedirect(request, response, "/login/user.html");
        }
        log.debug("会话过期，AJAX请求将返回状态码401");
        return new AuthenticationResponse(request.getRequestURI(), "会话超时", HttpStatus.UNAUTHORIZED.value());
    }
}
```

**集群session管理**

```properties
#配置Spring-Session之后将redis存储到redis
spring.session.store-type=redis
spring.redis.host=192.168.1.158
spring.redis.port=6379
```

```java
// 配置redisTemplate序列化存储为json，否则前文验证码模块等需要实现java序列化接口才可使用
@Bean
public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
    Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<Object>(Object.class);
    RedisTemplate<Object, Object> template = new RedisTemplate<Object, Object>();
    template.setConnectionFactory(redisConnectionFactory);
    template.setKeySerializer(jackson2JsonRedisSerializer);
    template.setValueSerializer(jackson2JsonRedisSerializer);
    template.setHashKeySerializer(jackson2JsonRedisSerializer);
    template.setHashValueSerializer(jackson2JsonRedisSerializer);
    return template;
}
```

## 退出登陆

```java
 /**
  * 登陆、安全拦截配置
  */
@Override
protected void configure(HttpSecurity http) throws Exception {
    
    http //省略各项配置
        .and()
        .logout()						  // 会使session失效
       //.logoutSuccessHandler()		  // 配置退出成功的处理器，进行扫尾工作，或者返回JSON
        .logoutUrl("/authentication/out") // 退出请求的url
        .logoutSuccessUrl("/logout.html") // 退出成功请求的url，默认跳转到登录页，可不配置
		//.deleteCookies()				  // 删除cookie等操作
        .and()  //省略各项配置
}
```

# 二、基于App的安全

![Spring-OAuth2-Authentication](/img/springsecurity/Spring-OAuth2-Authentication.png)

​	

## 表单登陆

![app-form-login](/img/springsecurity/app-form-login.png)

需要继承配置`ResourceServerConfigurerAdapter`重载`public void configure(HttpSecurity http) `方法

```java
@Configuration
public class AppResourceServerConfigurer extends ResourceServerConfigurerAdapter {
    @Override
    public void configure(HttpSecurity http) throws Exception {
        // ....
        
        http.userDetailsService(userDetailsService);
        //表单登陆功能
        FormLoginConfigurer<HttpSecurity> formLoginConfigurer = http.formLogin();
        formLoginConfigurer.loginPage(coreSecurityProperties.getLoginPageUrl());
        formLoginConfigurer.loginProcessingUrl(coreSecurityProperties.getAuthenticationUrl());
        formLoginConfigurer.successHandler(successHandler);
        formLoginConfigurer.failureHandler(failureHandler);
        
        // ....
}
    
/**
 * APP默认的登陆失败处理器
 */
public class DefaultLoginFailureHandler implements AuthenticationFailureHandler {

    private static Logger log = LoggerFactory.getLogger(DefaultLoginFailureHandler.class);

    private ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception) throws IOException, ServletException {
        log.info("登录失败");
        response.setStatus(HttpStatus.OK.value());
        response.setContentType(MimeTypeUtils.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding(CharEncoding.UTF_8);
        SimpleResponse simpleResponse = new SimpleResponse(String.valueOf(HttpStatus.INTERNAL_SERVER_ERROR.value()), exception.getMessage());
        response.getWriter().write(objectMapper.writeValueAsString(simpleResponse));
    }
}
    
    
/**
 * APP登陆成功处理器,需要在登陆成功时发放OAuth2令牌
 */
public class DefaultLoginSuccessHandler implements AuthenticationSuccessHandler {

    private ObjectMapper objectMapper = new ObjectMapper();

    private static Logger log = LoggerFactory.getLogger(DefaultLoginSuccessHandler.class);

    @Autowired
    private ClientDetailsService clientDetailsService;

    @Autowired
    private AuthorizationServerTokenServices authorizationServerTokenServices;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        response.setCharacterEncoding(CharEncoding.UTF_8);
        response.setContentType(MimeTypeUtils.APPLICATION_JSON_VALUE);
        //用Basic 认证传入clientId和secret
        String header = request.getHeader("Authorization");
        if (header == null || !header.toLowerCase().startsWith("basic ")) {
            throw new AuthenticationException("没有OAuth2需要的的认证信息");
        }
        String[] tokens = extractAndDecodeHeader(header, request);
        String clientId = tokens[0];
        String secret   = tokens[1];

        ClientDetails clientDetails = clientDetailsService.loadClientByClientId(clientId);
        if(clientDetails == null) {
            response.getWriter().write(objectMapper.writeValueAsString(new AuthenticationException("clientId不正确")));
            return;
        }

        if(!passwordEncoder.matches(secret, clientDetails.getClientSecret())) {
            response.getWriter().write(objectMapper.writeValueAsString(new AuthenticationException("secret不正确")));
            return;
        }

        TokenRequest tokenRequest = new TokenRequest(Collections.EMPTY_MAP, clientId, clientDetails.getScope(), "custom_login");

        OAuth2Request oauth2Request = tokenRequest.createOAuth2Request(clientDetails);

        OAuth2Authentication oAuth2Authentication = new OAuth2Authentication(oauth2Request, authentication);

        OAuth2AccessToken accessToken = authorizationServerTokenServices.createAccessToken(oAuth2Authentication);

        log.debug("用户{}登陆成功", authentication.getPrincipal());

        response.getWriter().write(objectMapper.writeValueAsString(accessToken));
    }


    private String[] extractAndDecodeHeader(String header, HttpServletRequest request)
            throws IOException {

        byte[] base64Token = header.substring(6).getBytes("UTF-8");
        byte[] decoded;
        try {
            decoded = Base64.getDecoder().decode(base64Token);
        }
        catch (IllegalArgumentException e) {
            throw new BadCredentialsException(
                    "Failed to decode basic authentication token");
        }

        String token = new String(decoded, CharEncoding.UTF_8);

        int delimiter = token.indexOf(":");

        if (delimiter == -1) {
            throw new BadCredentialsException("Invalid basic authentication token");
        }
        return new String[] { token.substring(0, delimiter), token.substring(delimiter + 1) };
    }
}
```

## 图片验证码

注意：不能使用Session存储方式，需要将验证码存放如Redis中

```java
@ConditionalOnProperty(prefix = "login.verify.image", value = "enable", havingValue = "true")
@Controller
public class ImageCodeController {

    @Autowired
    private ImageCodeUtil imageCodeUtil;

    public static final String REDIS_IMAGE_VERIFY_CODE = "VERIFY_CODE:IMAGE_CODE:";

    @Autowired
    private RedisTemplate redisTemplate;

    @GetMapping(SecurityBrowserConstants.VALIDATE_IMAGE_URL)
    @ResponseBody
    public void getVerifyCode( @RequestParam("device") String device, HttpServletResponse response) throws IOException {
        ImageValidateCode imageCode = imageCodeUtil.generateCodeAndPic();
        ImageValidateCode redisInCode = new ImageValidateCode();
        redisInCode.setCode(imageCode.getCode());
        redisInCode.setExpireDate(imageCode.getExpireDate());
        // 设置响应的类型格式为图片格式
        response.setContentType(MimeTypeUtils.IMAGE_JPEG_VALUE);
        // 禁止图像缓存。
        response.setHeader("Pragma", "no-cache");
        response.setHeader("Cache-Control", "no-cache");
        response.setDateHeader("Expires", 0);
        // 写入图片
        ImageIO.write(imageCode.getCodePic(), "jpeg", response.getOutputStream());
        response.getOutputStream().flush();
        if (imageCode.getExpireDate() == null) {
            redisTemplate.boundValueOps(REDIS_IMAGE_VERIFY_CODE + device).set(redisInCode);
        }else {
            redisTemplate.boundValueOps(REDIS_IMAGE_VERIFY_CODE + device).set(redisInCode, imageCode.getExpireDate(), TimeUnit.MILLISECONDS);
        }
    }
}

/**
 * 图片验证码拦截匹配器
 */
public class ImageValidateCodeFilter extends OncePerRequestFilter {

    @Autowired
    private CoreSecurityProperties coreSecurityProperties;

    private AuthenticationFailureHandler authenticationFailureHandler;

    @Autowired
    private RedisTemplate redisTemplate;

    public ImageValidateCodeFilter(AuthenticationFailureHandler authenticationFailureHandler) {
        this.authenticationFailureHandler = authenticationFailureHandler;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 是登陆请求而且为POST
        if(StringUtils.equals(coreSecurityProperties.getAuthenticationUrl(), request.getRequestURI()) && StringUtils.equalsIgnoreCase(request.getMethod(), "POST")) {
            try {
                validate(new ServletWebRequest(request));
                filterChain.doFilter(request, response);
            } catch (ValidateCodeException e) {
                authenticationFailureHandler.onAuthenticationFailure(request, response, e);
            }
        }else {
            filterChain.doFilter(request, response);
        }
    }

    private void validate(ServletWebRequest servletWebRequest) throws ServletRequestBindingException{
        String questInDevice = null;
        String questInCode   = null;
        try {
            questInDevice = ServletRequestUtils.getRequiredStringParameter(servletWebRequest.getRequest(), "device");
            questInCode   = ServletRequestUtils.getStringParameter(servletWebRequest.getRequest(),"imageCode");
        }catch (ServletRequestBindingException e) {
            throw new ValidateCodeException("提交的图片验证码参数缺失");
        }

        String keyName = ImageCodeController.REDIS_IMAGE_VERIFY_CODE + questInDevice;
        ImageValidateCode codeInRedis = (ImageValidateCode)redisTemplate.boundValueOps(keyName).get();

        if(codeInRedis == null) {
            // 过期判断由Redis控制,Redis设置了过期时间，程序不判断过期行为
            throw new ValidateCodeException("验证码不存在，或已过期");
        }
        if(!StringUtils.equalsIgnoreCase(questInCode, codeInRedis.getCode())) {
            throw new ValidateCodeException("验证码不匹配");
        }
        // 匹配成功
        redisTemplate.delete(keyName);
    }
}
```

## 手机验证码

注意：不能使用Session存储方式，需要将验证码存放如Redis中

```java
@ConditionalOnProperty(prefix = "login.verify.sms", value = "enable", havingValue = "true")
@Controller
public class SmsCodeController {

    private static Logger log = LoggerFactory.getLogger(SmsCodeController.class);

    public static final String SMS_CODE_MOBILE_PREFIX = "VERIFY_CODE:SMS_CODE_MOBILE:";

    @Autowired
    private SmsCodeUtil smsCodeUtil;

    @Autowired
    private RedisTemplate redisTemplate;

    @Autowired
    private LoginVerifySmsSender loginVerifySmsSender;

    @GetMapping(value = "/verify/sms/code/{mobile}", produces = MimeTypeUtils.APPLICATION_JSON_VALUE)
    @ResponseBody
    public SimpleResponse getVerifyCode(@PathVariable("mobile") String mobile, @RequestParam("device") String device, HttpServletRequest request, HttpServletResponse response) throws IOException {
        SmsValidateCode smsValidateCode = smsCodeUtil.generateCode(mobile);
        if(loginVerifySmsSender.sendVerifyCode(smsValidateCode)) {
            //发送短信
            log.debug("给{}手机发送验证短信，验证码{}成功", smsValidateCode.getMobile(), smsValidateCode.getCode());
        }else {
            log.debug("给{}手机发送验证短信，验证码{}失败", smsValidateCode.getMobile(), smsValidateCode.getCode());
            return new SimpleResponse(String.valueOf(HttpStatus.OK.value()), "短信发送失败");
        }
        redisTemplate.boundValueOps(SMS_CODE_MOBILE_PREFIX + device + ":" + mobile).set(smsValidateCode, smsValidateCode.getExpireDate(), TimeUnit.MILLISECONDS);
        SimpleResponse simpleResponse = new SimpleResponse(String.valueOf(HttpStatus.OK.value()), "短信发送成功");
        return simpleResponse;
    }
}

/**
 * 短信认证的Token
 */
public class SmsCodeAuthenticationToken extends AbstractAuthenticationToken {

    private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;

    private  Object principal;
    private Object credentials;

    public SmsCodeAuthenticationToken(Object principal, Object credentials) {
        super(null);
        this.principal = principal;
        this.credentials = credentials;
        setAuthenticated(false);
    }

    public SmsCodeAuthenticationToken(Object principal, Object credentials,
                                      Collection<? extends GrantedAuthority> authorities) {
        super(authorities);
        this.principal = principal;
        this.credentials = credentials;
        super.setAuthenticated(true); // must use super, as we override
    }

    public Object getCredentials() {
        return this.credentials;
    }

    public Object getPrincipal() {
        return this.principal;
    }

    public void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException {
        if (isAuthenticated) {
            throw new IllegalArgumentException(
                    "Cannot set this token to trusted - use constructor which takes a GrantedAuthority list instead");
        }

        super.setAuthenticated(false);
    }

    @Override
    public void eraseCredentials() {
        super.eraseCredentials();
    }
}

/**
 * 短信认证登陆处理
 */
public class SmsAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    public static final String SECURITY_FORM_MOBILE_KEY = "mobile";

    private String mobileParameter = SECURITY_FORM_MOBILE_KEY;
    private boolean postOnly = true;

    public SmsAuthenticationFilter() {
        super(new AntPathRequestMatcher("/authentication/sms/commit", "POST"));
    }

    public Authentication attemptAuthentication(HttpServletRequest request,
                                                HttpServletResponse response) throws AuthenticationException {
        if (postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException(
                    "Authentication method not supported: " + request.getMethod());
        }

        String mobile = obtainMobile(request);

        if (mobile == null) {
            mobile = "";
        }

        mobile = mobile.trim();

        SmsCodeAuthenticationToken authRequest = new SmsCodeAuthenticationToken(mobile, null);

        setDetails(request, authRequest);

        return this.getAuthenticationManager().authenticate(authRequest);
    }

    protected String obtainMobile(HttpServletRequest request) {
        return request.getParameter(mobileParameter);
    }

    protected void setDetails(HttpServletRequest request,
                              SmsCodeAuthenticationToken authRequest) {
        authRequest.setDetails(authenticationDetailsSource.buildDetails(request));
    }

    public void setMobileParameter(String mobileParameter) {
        Assert.hasText(mobileParameter, "mobileParameter parameter must not be empty or null");
        this.mobileParameter = mobileParameter;
    }
}

/**
 * 短信验证码拦截匹配器
 */
public class SmsValidateCodeFilter extends OncePerRequestFilter {

    private AuthenticationFailureHandler authenticationFailureHandler;

    @Autowired
    private CoreSecurityProperties coreSecurityProperties;

    @Autowired
    private RedisTemplate redisTemplate;

    public SmsValidateCodeFilter(AuthenticationFailureHandler authenticationFailureHandler) {
        this.authenticationFailureHandler = authenticationFailureHandler;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 是登陆请求而且为POST
        if(StringUtils.equals(coreSecurityProperties.getSmsAuthenticationUrl(), request.getRequestURI()) && StringUtils.equalsIgnoreCase(request.getMethod(), "POST")) {
            try {
                validate(new ServletWebRequest(request));
                filterChain.doFilter(request, response);
            } catch (ValidateCodeException e) {
                authenticationFailureHandler.onAuthenticationFailure(request, response, e);
            }
        }else {
            filterChain.doFilter(request, response);
        }
    }

    private void validate(ServletWebRequest servletWebRequest) {

        String questInMobile = null;
        String questInCode   = null;
        String questInDevice = null;

        try {
            questInMobile = ServletRequestUtils.getRequiredStringParameter(servletWebRequest.getRequest(), "mobile");
            questInCode = ServletRequestUtils.getRequiredStringParameter(servletWebRequest.getRequest(), "code");
            questInDevice = ServletRequestUtils.getRequiredStringParameter(servletWebRequest.getRequest(), "device");
        }catch (ServletRequestBindingException e) {
            throw new ValidateCodeException("提交的短信验证码参数缺失");
        }

        String keyName = SmsCodeController.SMS_CODE_MOBILE_PREFIX + questInDevice + ":" +questInMobile;
        SmsValidateCode smsValidateCode = (SmsValidateCode) redisTemplate.boundValueOps(keyName).get();

        if(StringUtils.isEmpty(questInMobile)) {
            throw new ValidateCodeException("提交的手机号不能为空");
        }

        if(StringUtils.isEmpty(questInCode)) {
            throw new ValidateCodeException("提交的短信验证码不能为空");
        }

        if(smsValidateCode == null) {
            // 过期判断由Redis控制,Redis设置了过期时间，程序不判断过期行为
            throw new ValidateCodeException("手机号" + questInMobile + "短信验证码不存在或已过期");
        }

        if(!StringUtils.equalsIgnoreCase(questInCode, smsValidateCode.getCode())) {
            throw new ValidateCodeException("验证码不匹配");
        }

        // 匹配成功
        redisTemplate.delete(keyName);
    }
}
```

## 第三方登陆

- 简化验证模式需要将OpenId服务器
- 授权码模式直接将请求转给服务器即可

# 三、权限设置

## 概览

**请求经过的过滤器**

![permission-filter](/img/springsecurity/permission-filter.png)

**判断是否允许访问的过程**

![permission-struct](/img/springsecurity/permission-struct.png)

## 简单的只区分是否登陆

```java
http. //省略其它配置
    .formLogin()//基于表带验证
    .loginPage("/authentication/require")   //设置先访问的请求
    .loginProcessingUrl("/authentication/commit")
    .authorizeRequests()            //认证的请求
    .antMatchers(HttpMethod.GET,"/authentication/require", "/me")//匹配请求
    .permitAll()                    //允许所有
    .anyRequest()                   //认证的请求
    .authenticated()                //需要进行登陆身份认证
    //省略其它配置
```

## 区分简单、有限角色【硬编码】

```java
http.
    .formLogin()//基于表带验证
    .loginPage("/authentication/require")   //设置先访问的请求
    .loginProcessingUrl("/authentication/commit")
    .and()
    .authorizeRequests()            					  //认证的请求
    .antMatchers(HttpMethod.GET,"/authentication/require")//匹配请求
    .permitAll()                    //允许所有
    .antMatchers("/user/**")		//匹配请求
    .hasRole("ADMIN")				//需要管理员角色，匹配的是ROLE_ADMIN,会加前缀ROLE
    .antMatchers("/me")				//匹配请求
    .hasRole("ANONYMOUS")			//匿名角色可访问【访问此请求如果未登陆会创建一个匿名授权】
    //.antMatchers("/me").anonymous()  也可使用这种方式，与上等价
    .anyRequest()                   //认证的请求
    .authenticated()                //需要身份认证
```

如果访问的URL不是指定的允许所有可访问，也不是指定角色为匿名用户可访问，那么访问时会抛出`Access is denied (user is anonymous)`的异常

## 复杂角色【一般用于后台管理】

**Security权限表达式**

| 表达式                                 | 说明                                              |
| -------------------------------------- | ------------------------------------------------- |
| permitAll                              | 永远返回true                                      |
| denyAll                                | 永远返回false                                     |
| anonymous                              | 当前用户是anonymous时返回true                     |
| authenticated                          | 当前用户不是anonymous时返回true                   |
| rememberMe                             | 当前用户是rememberMe时返回true                    |
| fullAuthenticated                      | 当前用户即不是rememberMe也不是anonymous时返回true |
| hasRole(role)                          | 当前用户拥有指定角色时返回true                    |
| hasAnyRole(role1,role2)                | 当前用户拥有任意指定角色时返回true                |
| hasAuthority(authority)                | 当前用户拥有指定权限时返回true                    |
| hasAnyAuthority(authority1,authority2) | 当前用户拥有任意指定权限时返回true                |
| hasIpAddress(Ip)                       | 请求匹配IP时返回true                              |

**使用方法**

```java
//方法一：使用代码的方式指定条件，但是不能指定连续条件
expressionInterceptUrlRegistry.antMatchers("/user/add").hasRole("USER_MANAGER");

//方法二：使用表达式，可以指定连续条件
expressionInterceptUrlRegistry.antMatchers("/user/add")
    .access("hasRole('USER_MANAGER') and hasIpAddress('192.168.1.100')");
    
//方法三：使用@PreAuthorize(express)注解
@GetMapping("/user/add")
@PreAuthorize("hasRole('ADMIN')")
public String userAdd() {
    // doSomething
}
```

# 四、代码

[相关代码地址](https://github.com/MyNameLancelot/security)