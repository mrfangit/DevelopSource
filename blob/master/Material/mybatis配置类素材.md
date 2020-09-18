mybatis配置类素材

```java
@Configuration
@MapperScan("com.mrfangit.**.dao")
@AutoConfigureAfter(DataSourceConfiguration.class)
public class MybatisConfiguration {

    /**
     * 日志
     */
    private final Logger logger = LoggerFactory.getLogger(MybatisConfiguration.class);

    /**
     * Mapper.xml文件位置
     */
    @Value("${mybatis.mapperLocations}")
    private String mapperLocation = "classpath*:com/mrfangit/**/dao/sqlmap/**/*.xml";

    /**
     * mybatis配置文件路径
     */
//    private String CONFIG_LOCATION = "classpath*:config/mybatis-config.xml";

    /**
     * @return MapperScannerConfigurer
     */
    @Bean
    public MapperScannerConfigurer configMapperSacnner() {
        MapperScannerConfigurer scanner = new MapperScannerConfigurer();
        scanner.setAnnotationClass(Repository.class);
        scanner.setBasePackage("com.mrfangit");
        scanner.setSqlSessionFactoryBeanName("sqlSessionFactory");
        return scanner;
    }

    /**
     * @param dynamicDataSource 动态数据源
     * @return SqlSessionFactory
     * @throws Exception 异常
     */
    @Bean("sqlSessionFactory")
    public SqlSessionFactoryBean sqlSession(@Qualifier("dynamicDataSource") DataSource dynamicDataSource) {
        ResourcePatternResolver patternResolver = ResourcePatternUtils.getResourcePatternResolver(
            new DefaultResourceLoader());
        SqlSessionFactoryBean sqlSession = new SqlSessionFactoryBean();
        sqlSession.setDataSource(dynamicDataSource);
        sqlSession.setVfs(SpringBootVFS.class);
        org.apache.ibatis.session.Configuration conf = new org.apache.ibatis.session.Configuration();
        // 查询结果为map时不忽略空值
        conf.setCallSettersOnNulls(true);
        // 开启驼峰命名转换   seckill_id====>seckillId
        conf.setMapUnderscoreToCamelCase(true);
        sqlSession.setConfiguration(conf);
        try {
            sqlSession.setMapperLocations(patternResolver.getResources(mapperLocation));
        }
        catch (Exception e) {
            logger.error(e.getMessage());
        }
        return sqlSession;
    }
    
    /**
     * 自定注入分页插件
     *
     * @author liuzh
     */
    @Configuration
    @EnableConfigurationProperties(PageHelperProperties.class)
    @AutoConfigureAfter(MybatisAutoConfiguration.class)
    public class PageHelperAutoConfiguration {

        @Autowired
        private PageHelperProperties properties;
        
        @Autowired
        private SqlSessionFactory sqlSessionFactory;
        
        @Autowired
        private List<MybatisInterceptor> interceptors;
        
        @Autowired(required = false)
        private DataAuthorityService dataAuthorityService;
        
        @Autowired(required = false)
        private SecurityService securityService;
        
        @Autowired(required = false)
        private IDatabaseLocker dbLocker;

        /**
         * 接受分页插件额外的属性
         *
         * @return
         */
        @Bean
        @ConfigurationProperties(prefix = PageHelperProperties.PAGEHELPER_PREFIX)
        public Properties pageHelperProperties() {
            return new Properties();
        }

        @PostConstruct
        public void addinterceptors() {
            // 分页插件
            PageInterceptor interceptor = new PageInterceptor();
            Properties properties = new Properties();
            // 先把一般方式配置的属性放进去
            properties.putAll(pageHelperProperties());
            // 在把特殊配置放进去，由于close-conn 利用上面方式时，属性名就是 close-conn 而不是 closeConn，所以需要额外的一步
            properties.putAll(this.properties.getProperties());
            interceptor.setProperties(properties);
            sqlSessionFactory.getConfiguration().addInterceptor(interceptor);

            // 数据权限过滤器
            DataAuthorityInterceptor dataAuthorityInterceptor = new DataAuthorityInterceptor();
            dataAuthorityInterceptor.setDataAuthorityService(dataAuthorityService);
            dataAuthorityInterceptor.setSecurityService(securityService);
            interceptors.add(dataAuthorityInterceptor);

            // 数据库锁定插件
            DBLockInterceptor dbLockInterceptor = new DBLockInterceptor();
            dbLockInterceptor.setDbLocker(dbLocker);
            interceptors.add(dbLockInterceptor);

            OrderComparator.sort(interceptors);

            interceptors.forEach(item -> {
                sqlSessionFactory.getConfiguration().addInterceptor(item);
            });
        }
    }
    
    /**
     * Configuration properties for PageHelper.
     *
     * @author liuzh
     */
    @ConfigurationProperties(prefix = PageHelperProperties.PAGEHELPER_PREFIX)
    public class PageHelperProperties {

        public static final String PAGEHELPER_PREFIX = "pagehelper";

        private Properties properties = new Properties();

        public Properties getProperties() {
            return properties;
        }

        public String getOffsetAsPageNum() {
            return properties.getProperty("offsetAsPageNum");
        }

        public void setOffsetAsPageNum(String offsetAsPageNum) {
            properties.setProperty("offsetAsPageNum", offsetAsPageNum);
        }

        public String getRowBoundsWithCount() {
            return properties.getProperty("rowBoundsWithCount");
        }

        public void setRowBoundsWithCount(String rowBoundsWithCount) {
            properties.setProperty("rowBoundsWithCount", rowBoundsWithCount);
        }

        public String getPageSizeZero() {
            return properties.getProperty("pageSizeZero");
        }

        public void setPageSizeZero(String pageSizeZero) {
            properties.setProperty("pageSizeZero", pageSizeZero);
        }

        public String getReasonable() {
            return properties.getProperty("reasonable");
        }

        public void setReasonable(String reasonable) {
            properties.setProperty("reasonable", reasonable);
        }

        public String getSupportMethodsArguments() {
            return properties.getProperty("supportMethodsArguments");
        }

        public void setSupportMethodsArguments(String supportMethodsArguments) {
            properties.setProperty("supportMethodsArguments", supportMethodsArguments);
        }

        public String getDialect() {
            return properties.getProperty("dialect");
        }

        public void setDialect(String dialect) {
            properties.setProperty("dialect", dialect);
        }

        public String getHelperDialect() {
            return properties.getProperty("helperDialect");
        }

        public void setHelperDialect(String helperDialect) {
            properties.setProperty("helperDialect", helperDialect);
        }

        public String getAutoRuntimeDialect() {
            return properties.getProperty("autoRuntimeDialect");
        }

        public void setAutoRuntimeDialect(String autoRuntimeDialect) {
            properties.setProperty("autoRuntimeDialect", autoRuntimeDialect);
        }

        public String getAutoDialect() {
            return properties.getProperty("autoDialect");
        }

        public void setAutoDialect(String autoDialect) {
            properties.setProperty("autoDialect", autoDialect);
        }

        public String getCloseConn() {
            return properties.getProperty("closeConn");
        }

        public void setCloseConn(String closeConn) {
            properties.setProperty("closeConn", closeConn);
        }

        public String getParams() {
            return properties.getProperty("params");
        }

        public void setParams(String params) {
            properties.setProperty("params", params);
        }
    }
}
```

