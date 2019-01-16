---
layout: post
title: Springboot+Security实现鉴权功能
date: 2019-01-08
categories: blog
tags: [JAVA]
description: Springboot+Security
---

<h1>使用Springboot+Security实现无状态鉴权功能体系</h1>

### 简介

无状态鉴权：不依赖会话（session不会存储用户身份信息）实现用户身份鉴别，提高系统安全性。

如图所示：无状态鉴权主要分为三种情况：注册、登录、访问
![鉴权流程图]({{ site.url }}/img/picture/springbootsecurity.png)

系统设计泳道图如下：
![系统设计]({{ site.url }}/img/picture/SpringBoot+Security实现无状态鉴权体系.png)

## 环境配置
### 1.依赖配置

	 <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.0</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.13</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.7.0</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.39</version>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.6</version>
        </dependency>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.5.1</version>
        </dependency>
    </dependencies>

### 2.application.properties配置

	server.port=8080
	#DB
	mybatis.type-aliases-package=edu.charles.tf.entity
	spring.datasource.driverClassName=com.mysql.jdbc.Driver
	spring.datasource.url=jdbc:mysql://localhost:3306/edu_charles?useUnicode=true&characterEncoding=UTF-8&serverTimezone=UTC
	spring.datasource.username=root
	spring.datasource.password=898218
	#security
	#security.basic.enabled=false
	#security.enable-csrf=false
	security.key=tfsk
	#token头名称
	jwt.header=Authorization
	#token固定头
	jwt.customer.customToken.head=xxcbearer:


## 第一步：实现UserDetails/UserDetailsService 接口

### UserDetails 用户核心接口
该接口定义了用户信息的基本信息，包括用户名、密码、账号锁定/启用状态等

Customer[用户类]

	public class Customer extends CustomerEntity implements UserDetails {
	    public Customer() {
	    }
	
	    public Customer(CustomerEntity entity) {
	        BeanUtils.copyProperties(entity, this);
	    }
	
	    @Override
	    public Collection<? extends GrantedAuthority> getAuthorities() {
	        List<GrantedAuthority> authorities = new ArrayList<>();
	        SimpleGrantedAuthority authority = new SimpleGrantedAuthority(AuthorityEnum.CUSTOMER.name());
	        authorities.add(authority);
	        return authorities;
	    }
	
	    @Override
	    public String getUsername() {
	        return super.getAccount();
	    }
	
	    @Override
	    public boolean isAccountNonExpired() {
	        if (getPhase() == PhaseEnum.ENABLED)
	            return true;
	        else
	            return false;
	    }
	
	    @Override
	    public boolean isAccountNonLocked() {
	        if (getPhase() == PhaseEnum.ENABLED)
	            return true;
	        else
	            return false;
	    }
	
	    @Override
	    public boolean isCredentialsNonExpired() {
	        return false;
	    }
	
	    @Override
	    public boolean isEnabled() {
	        return false;
	    }
	}

AuthorityEnum[用户类型，用于区分用户权限]

	public enum AuthorityEnum implements ValueEnum {
	    CUSTOMER,//普通用户
	    ADMIN;//管理员
	
	    public String getValue() {
	        return this.name();
	    }
	}

CustomerUserDetailService[校验用户信息时使用，加载用户数据]

	@Service
	@Qualifier("customerUserDetailService")
	public class CustomerUserDetailService implements UserDetailsService {
	    @Autowired
	    private CustomerMapper customerMapper;
	
	    @Override
	    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
	        List<Customer> list = customerMapper.selectByAccount(username);
	        if (list != null && !list.isEmpty()) {
	            return list.get(0);
	        } else
	            throw new UsernameNotFoundException("account : " + username + " is not found!");
	    }
	}

## 第二步：重写AbstractUserDetailsAuthenticationProvider类的additionalAuthenticationChecks、retrieveUser 方法
retrieveUser方法：根据第一步实现的userDetailService，加载用户数据
additionalAuthenticationChecks：校验用户的账号、密码

	@Component
	public abstract class CommonAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
	    @Autowired
	    private PasswordEncoder passwordEncoder;
	
	    @Override
	    protected void additionalAuthenticationChecks(UserDetails userDetails,
	            UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
	        if (authentication.getCredentials() == null) {
	            logger.debug("Authentication failed: 权限错误！");
	
	            throw new BadCredentialsException(messages.getMessage(
	                    "AbstractUserDetailsAuthenticationProvider.badCredentials",
	                    "密码错误！"));
	        }
	        if (authentication.getPrincipal() == null) {
	            logger.debug("Authentication failed: 账号错误！");
	
	            throw new BadCredentialsException(messages.getMessage(
	                    "AbstractUserDetailsAuthenticationProvider.badCredentials",
	                    "账号错误！"));
	        }
	
	        String presentedUsername = authentication.getPrincipal().toString();
	        String presentedPassword = authentication.getCredentials().toString();
	        String username = userDetails.getUsername();
	        String password = userDetails.getPassword();
	        if (username.equals(presentedUsername) && isPasswordValid(presentedPassword, password)) {
	            return;
	        } else {
	            throw new BadCredentialsException("用户名或密码错误！");
	        }
	    }
	
	    protected boolean isPasswordValid(String rawPass, String encPass) {
	        return passwordEncoder.matches(rawPass, encPass);
	    }
	}

	@Component("customerAuthenticationProvider")
	public class CustomerAuthenticationProvider extends CommonAuthenticationProvider {
	    @Autowired
	    @Qualifier("customerUserDetailService")
	    private UserDetailsService customerUserDetailService;
	
	    @Override
	    protected UserDetails retrieveUser(String username,
	            UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
	        UserDetails loadedUser;
	
	        try {
	            loadedUser = customerUserDetailService.loadUserByUsername(username);
	        } catch (UsernameNotFoundException notFound) {
	
	            throw notFound;
	        } catch (Exception repositoryProblem) {
	            throw new InternalAuthenticationServiceException(
	                    repositoryProblem.getMessage(), repositoryProblem);
	        }
	
	        if (loadedUser == null) {
	            throw new InternalAuthenticationServiceException(
	                    "UserDetailsService returned null, which is an interface contract violation");
	        }
	        return loadedUser;
	    }
	}


## 第三步：定义TokenFilter类
JwtAuthenticationTokenFilter:用于拦截request请求，解析校验用户传递的token，正确放行，错误拦截

	public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {
	    @Autowired
	    @Qualifier("customerUserDetailService")
	    private UserDetailsService userDetailsService;
	
	    @Autowired
	    private JwtService jwtService;
	
	    @Value("${jwt.header}")
	    private String tokenHeader;
	
	    private String tokenHead;
	
	    @Value("${security.key}")
	    private String securityKey;
	
	    @Override
	    protected void doFilterInternal(
	            HttpServletRequest request,
	            HttpServletResponse response,
	            FilterChain chain) throws ServletException, IOException {
	        String authToken = request.getHeader(tokenHeader);
	        if (StringUtils.isNoneBlank(authToken)) {
	            String account = null;
	            try {
	                account = jwtService.getCustomerAccount(authToken);
	            } catch (Exception e) {
	                logger.error("jwt auToken: " + authToken + " is error!", e);
	            }
	
	            if (account != null && SecurityContextHolder.getContext().getAuthentication() == null) {
	
	                UserDetails userDetails = this.userDetailsService.loadUserByUsername(account);
	
	                if (null != userDetails && jwtService.validateToken(authToken, userDetails)) {
	                    UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(
	                            userDetails, null, userDetails.getAuthorities());
	                    authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(
	                            request));
	                    logger.info("authenticated user:" + account + ", setting security context");
	                    SecurityContextHolder.getContext().setAuthentication(authentication);
	                }
	            }
	        }
	        chain.doFilter(request, response);
	    }
	
	    public UserDetailsService getUserDetailsService() {
	        return userDetailsService;
	    }
	
	    public void setUserDetailsService(UserDetailsService userDetailsService) {
	        this.userDetailsService = userDetailsService;
	    }
	
	    public String getTokenHeader() {
	        return tokenHeader;
	    }
	
	    public void setTokenHeader(String tokenHeader) {
	        this.tokenHeader = tokenHeader;
	    }
	
	    public String getSecurityKey() {
	        return securityKey;
	    }
	
	    public void setSecurityKey(String securityKey) {
	        this.securityKey = securityKey;
	    }
	
	    public String getTokenHead() {
	        return tokenHead;
	    }
	
	    public void setTokenHead(String tokenHead) {
	        this.tokenHead = tokenHead;
	    }
	}


## 第四步：实现TokenService
JwtService是用于生成、存储、校验token的service

	@Service
	public class JwtService {
	    protected final Logger log = LoggerFactory.getLogger(getClass());
	    @Value("${security.key}")
	    private String securityKey;
	
	    @Autowired
	    private CacheService cacheService;
	    @Value("${jwt.customer.customToken.head}")
	    private String customerTokenHead;
	    //用户token失效时间
	    private static final int customerTokenExpiredDays = 1;
	    private static final String  ACCOUNT = "account";
	    /**
	     * 生成用户token
	     *
	     * @auther: CharlesZheng
	     * @Date 15:11 2019/1/8
	     */
	    public String generateCustomerToken(UserDetails userDetails, String account) {
	        return generateToken(userDetails, ACCOUNT, customerTokenHead, customerTokenExpiredDays);
	    }
	
	    protected String generateToken(UserDetails userDetails, String name, String tokenHead, int expiredDays) {
	        Map<String, Object> claims = new HashMap<>();
	        String username=userDetails.getUsername();
	        claims.put(name, username);
	        String token = Jwts.builder()
	                .setClaims(claims)
	                .setExpiration(DateUtils.addDays(new Date(), expiredDays))
	                .signWith(SignatureAlgorithm.HS512, securityKey) //采用什么算法是可以自己选择的，不一定非要采用HS512
	                .compact();
	        token = null == tokenHead ? token : tokenHead + token;
	        //增加缓存
	        try {
	            cacheService.put(username, token);
	        } catch (Exception e) {
	            log.error(e.getMessage(), e);
	        }
	        return token;
	    }
	
	    public String getCustomerAccount(String token) {
	        token = cutCustomPrefix(token);
	        return Jwts.parser().setSigningKey(securityKey).parseClaimsJws(token).getBody().get(ACCOUNT).toString();
	    }
	
	    /**
	     * 校验token
	     *
	     * @auther: CharlesZheng
	     * @Date 16:08 2019/1/8
	     */
	    public boolean validateToken(String token, UserDetails userDetails) {
	        //TODO:这里需要完善
	        String tk = cacheService.get(userDetails.getUsername());
	        return token.equals(tk);
	    }
	
	    public void removeToken(UserDetails userDetails) {
	        String key = userDetails.getUsername();
	        cacheService.del(key);
	    }
	
	    protected String cutCustomPrefix(String token) {
	        return token.substring(null == customerTokenHead ? 0 : customerTokenHead.length());
	    }
	}

## 第五步：配置程序入口

将前三步定义的customerUserDetailService、customerTokenHead、customerAuthenticationProvider、JwtAuthenticationTokenFilter配置在WebSecurityConfig中，装载BCrypt密码编码器用于加密用户密码（用户密码在数据库是密文存储的，需要这个bean来加密和校验用户密码）

	@SpringBootApplication
	@EnableTransactionManagement
	@EnableScheduling
	@MapperScan({ "edu.charles.tf.mapper" })
	public class TFApplication {
	
	    public static void main(String[] args) {
	        SpringApplication.run(TFApplication.class, args);
	    }
	
	    @Configuration
	    @EnableWebSecurity
	    @EnableGlobalMethodSecurity(prePostEnabled = true)
	    public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
	
	        // Spring会自动寻找同样类型的具体类注入，这里就是JwtUserDetailsServiceImpl了
	        @Autowired
	        @Qualifier("customerUserDetailService")
	        private UserDetailsService customerUserDetailService;
	
	        @Value("${jwt.customer.customToken.head}")
	        private String customerTokenHead;
	
	        @Autowired
	        @Qualifier("customerAuthenticationProvider")
	        private AuthenticationProvider customerProvider;
	
	        @Autowired
	        public void configureAuthentication(AuthenticationManagerBuilder authenticationManagerBuilder) {
	            authenticationManagerBuilder
	                    .authenticationProvider(customerProvider);
	        }
	
	        @Bean
	        public JwtAuthenticationTokenFilter authenticationTokenFilterBean() {
	            JwtAuthenticationTokenFilter filter = new JwtAuthenticationTokenFilter();
	            filter.setUserDetailsService(customerUserDetailService);
	            filter.setTokenHead(customerTokenHead);
	            return filter;
	        }
	
	        @Override
	        protected void configure(HttpSecurity httpSecurity) throws Exception {
	            httpSecurity
	                    // 由于使用的是JWT，我们这里不需要csrf
	                    .csrf().disable()
	
	                    // 基于token，所以不需要session
	                    .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
	                    .authorizeRequests()
	
	                    .antMatchers("/auth/**").permitAll()
	                    // 除上面外的所有请求全部需要鉴权认证
	                    .anyRequest().authenticated().and()
	                    .addFilterBefore(authenticationTokenFilterBean(), UsernamePasswordAuthenticationFilter.class);
	
	            // 禁用缓存
	            httpSecurity.headers().cacheControl();
	        }
	    }
	
	    // 装载BCrypt密码编码器
	    @Bean
	    public PasswordEncoder passwordEncoder() {
	        return new BCryptPasswordEncoder();
	    }
	}


## 第六步：定义用户注册、登录、访问的api

登录注册contoller入口：注意注册、登录url，一定要在第四步配置的放行规则[/auth/**]内，不然无法正常请求服务，完成注册登录

	@RestController
	@RequestMapping("/auth")
	public class LoginController extends AbstractBaseController {
	    @Autowired
	    private AuthService authService;
	
	    @PostMapping("/login")
	    public WResult login(@RequestParam String account, @RequestParam String password) {
	        WResult wResult = authService.login(account, password);
	        return wResult;
	    }
	
	
	    @PostMapping("/register")
	    public WResult register(@ModelAttribute Customer customer) throws Exception {
	        return authService.register(customer);
	    }
	}


service逻辑：
	
	@Service
	public class AuthService extends BaseService {
	    @Autowired
	    private CustomTokenMapper customTokenMapper;
	    @Autowired
	    private CustomerMapper customerMapper;
	    @Autowired
	    @Qualifier("customerUserDetailService")
	    private UserDetailsService customerUserDetailService;
	
	    @Autowired
	    private JwtService jwtService;
	
	    @Autowired
	    private PasswordEncoder passwordEncoder;
	
	    public WResult login(String account, String password) {
	        List<Customer> list = customerMapper.selectByAccount(account);
	        if (null == list || list.isEmpty()) {
	            return WResultTools.getWResult(false, null, "账号不存在");
	        }
	        Customer customer = list.get(0);
	        String encPass = customer.getPassword();
	        if (validatePassword(password, encPass)) {
	            String customerToken = jwtService.generateCustomerToken(customer, account);
	            return WResultTools.getWResult(true, customerToken, null);
	        } else {
	            return WResultTools.getWResult(false, "密码错误", ErrCodeConstants.PASSWORD_ERROR);
	        }
	    }
	
	
	
	    @Transactional
	    public WResult register(Customer customer) {
	        String account = customer.getAccount();
	        List<Customer> list = customerMapper.selectByAccount(account);
	        if (null == list || list.isEmpty()) {
	            String password = customer.getPassword();
	            if (null == password || password.equals("")) {
	                return WResultTools.getWResult(false, "密码错误", ErrCodeConstants.PASSWORD_ERROR);
	            }
	            String encPass = passwordEncoder.encode(password);
	            CustomerEntity entity = new CustomerEntity();
	            BeanUtils.copyProperties(customer, entity);
	            entity.setUpdateTime(new Date());
	            entity.setCreateTime(new Date());
	            entity.setPhase(PhaseEnum.ENABLED);
	            entity.setType(TypeEnum.NORMAL);
	            entity.setPassword(encPass);
	            customerMapper.insert(entity);
	            //获取用户信息
	            final UserDetails userDetails = customerUserDetailService.loadUserByUsername(account);
	            String token = jwtService.generateCustomerToken(userDetails, account);
	            return WResultTools.getWResult(true, token, null, ErrCodeConstants.SUCCESS);
	        } else {
	            return WResultTools.getWResult(false, "账号重复", ErrCodeConstants.ACCOUNT_ERROR);
	        }
	    }
	
	    /**
	     * 更新token信息
	     *
	     * @auther: CharlesZheng
	     * @Date 15:23 2019/1/8
	     */
	    private void updateToken(Long customerId, String token) {
	
	    }
	
	    protected boolean validatePassword(String rawPass, String encPass) {
	        return passwordEncoder.matches(rawPass, encPass);
	    }
	}

本文限于时间等原因，token的存储直接放在mysql数据库表中，这很显然不合理，但是也足够说明问题了，后期只需要继续改造，引入redis等缓存功能，就是一个完整的基于token的无状态后台校验体系。

[本文源码Github地址：](https://github.com/zbw898218/springboot-20181230-charleszheng)















