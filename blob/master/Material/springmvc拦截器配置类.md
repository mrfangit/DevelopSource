springmvc拦截器配置类

```java

@EnableWebMvc
@ComponentScan(basePackages = "com.mrfangit", useDefaultFilters = false, includeFilters = {
    @ComponentScan.Filter(type = FilterType.ANNOTATION, value = {Controller.class,
        ControllerAdvice.class, RestController.class})})
public class MvcConfiguration extends WebMvcConfigurerAdapter {

    /**
     * 视图前缀
     */
    private static final String VIEW_PREFIX = "classpath:/templates/";

    /**
     * 视图后缀
     */
    private static final String VIEW_SUFFIX = ".jsp";

    /**
     * 视图的内容类型。
     */
    private static final String VIEW_CONTENT_TYPE = "text/html;charset=UTF-8";
    
    @Autowired
    private ObjectMapper objectMapper;

    /**
     * 注册试图处理器
     * 
     * @return
     */
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setCache(true);
        viewResolver.setPrefix(VIEW_PREFIX);
        viewResolver.setSuffix(VIEW_SUFFIX);
        viewResolver.setExposeContextBeansAsAttributes(true);
        viewResolver.setContentType(VIEW_CONTENT_TYPE);
        return viewResolver;
    }

    /**
     * 国际化配置
     * 
     * @return LocaleResolver
     */
    @Bean
    public LocaleResolver localeResolver() {
        CookieLocaleResolver localeResolver = new CookieLocaleResolver();
        localeResolver.setCookieName("clientLanguage");
        localeResolver.setCookieMaxAge(94608000);
        // 默认语言
        localeResolver.setDefaultLocale(Locale.SIMPLIFIED_CHINESE);
        return localeResolver;
    }

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
        // 参数名
        lci.setParamName("language");
        return lci;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor());
        registry.addInterceptor(new LogInterceptor());
        registry.addInterceptor(new SessionInterceptor());
    }

    @Override
    public Validator getValidator() {
        LocalValidatorFactoryBean validator = new LocalValidatorFactoryBean();
        validator.setProviderClass(org.hibernate.validator.HibernateValidator.class);
        validator.setValidationMessageSource(getMessageSource());
        return validator;
    }

    public ResourceBundleMessageSource getMessageSource() {
        ResourceBundleMessageSource rbms = new ResourceBundleMessageSource();
        rbms.setDefaultEncoding("UTF-8");
        rbms.setBasenames("classpath:message");
        return rbms;
    }

    /**
     * 静态资源配置
     */
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // weblogic 12c中resources/common的目录作为静态资源访问有问题，改为public
        registry.addResourceHandler("/public/**").addResourceLocations(
            "classpath:/META-INF/resources/");
        registry.addResourceHandler("/**").addResourceLocations("classpath:/static/");
    }
    
    /**
     * 跨域请求支持
     */
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**").allowedOrigins("*")
        .allowedMethods("GET", "HEAD", "POST","PUT", "DELETE", "OPTIONS")
        .allowCredentials(false).maxAge(3600);
    }

    /**
     * 扩展Spring消息处理器
     */
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        for(HttpMessageConverter<?> converter : converters) {
            if(converter instanceof StringHttpMessageConverter) {
                ((StringHttpMessageConverter)converter).setDefaultCharset(StandardCharsets.UTF_8);
            }
            else if(converter instanceof MappingJackson2HttpMessageConverter) {
                MappingJackson2HttpMessageConverter JacksonConverter = (MappingJackson2HttpMessageConverter)converter;
                JacksonConverter.setObjectMapper(objectMapper);
                JacksonConverter.setDefaultCharset(StandardCharsets.UTF_8);
            }
        }
    }
}
```