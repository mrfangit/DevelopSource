跟着源码看lcn分布式事务

lcn分布式事务具体实现思路是服务器A创建事务，构建事务信息并将事务信息发送到事务处理器，处理过程中可能用到服务器B、C，会将事务组Id传给B、C,B、C业务处理完成后将事务信息加入到对应的事务组，并且创建一个线程对事务组状态进行检测A事务处理完成后向服务其发送的事务状态，B、C会根据监督的状态判断对事务进行具体的提交或者回滚操作。

分布式事务流程图

 ![分布式事务图示](C:\Users\Administrator\Desktop\分布式事务图示.jpg)

lcn分布式事务是通过切面进行实现，lcn内部共有两个aop切面，分别是DataSourceAspect和TransactionAspect。DataSourceAspect拦截对应的数据库连接（connection）,TransactionAspect进行具体的分布式事务的实现，这里事务的开始是通过TransactionAspect进行开始，这里仅截取lcn部分代码去介绍。

```java
private final DTXLogicWeaver dtxLogicWeaver;

@Pointcut("@annotation(com.codingapi.txlcn.tc.annotation.LcnTransaction)")
public void lcnTransactionPointcut() {
}    

@Around("lcnTransactionPointcut() && !txcTransactionPointcut()" +
        "&& !tccTransactionPointcut() && !txTransactionPointcut()")
public Object runWithLcnTransaction(ProceedingJoinPoint point) throws Throwable {
    DTXInfo dtxInfo = DTXInfo.getFromCache(point);
    LcnTransaction lcnTransaction = dtxInfo.getBusinessMethod().getAnnotation(LcnTransaction.class);
    dtxInfo.setTransactionType(Transactions.LCN);
    dtxInfo.setTransactionPropagation(lcnTransaction.propagation());
    return dtxLogicWeaver.runTransaction(dtxInfo, point::proceed);
}
```

事务拦截器通过拦截带有LcnTransaction注解的方法后进行拦截处理，然后启动事务runTransaction方法启动具体的分布式事务实现类。

```java
public Object runTransaction(DTXInfo dtxInfo, BusinessCallback business) throws Throwable {

    log.debug("<---- TxLcn start ---->");
  	//这里获取dtxLocalContext，创建事务时新建。每个事务之只能有一个存在
    DTXLocalContext dtxLocalContext = DTXLocalContext.getOrNew();
    TxContext txContext;
    // ---------- 保证每个模块在一个DTX下只会有一个TxContext ---------- //
  	//判断是否存在事务信息,没有启动事务，有加入到事务
    if (globalContext.hasTxContext()) {
        // 有事务上下文的获取事务上下文
        txContext = globalContext.txContext();
        dtxLocalContext.setInGroup(true);
        log.debug("Unit[{}] used parent's TxContext[{}].", dtxInfo.getUnitId(), txContext.getGroupId());
        // 本地事务调用
        if (Objects.nonNull(dtxLocalContext.getGroupId())) {
            dtxLocalContext.setDestroy(false);
        }
    } else {
        // 没有的开启本地事务上下文
        txContext = globalContext.startTx();
        dtxLocalContext.setInGroup(false);
    }

    dtxLocalContext.setUnitId(dtxInfo.getUnitId());
    dtxLocalContext.setGroupId(txContext.getGroupId());
    dtxLocalContext.setTransactionType(dtxInfo.getTransactionType());

    // 事务参数
    TxTransactionInfo info = new TxTransactionInfo();
    info.setBusinessCallback(business);
    info.setGroupId(txContext.getGroupId());
    info.setUnitId(dtxInfo.getUnitId());
    info.setPointMethod(dtxInfo.getBusinessMethod());
    info.setPropagation(dtxInfo.getTransactionPropagation());
    info.setTransactionInfo(dtxInfo.getTransactionInfo());
    info.setTransactionType(dtxInfo.getTransactionType());
    info.setTransactionStart(txContext.isDtxStart());

    //LCN事务处理器
    try {
        return transactionServiceExecutor.transactionRunning(info);
    } finally {
        // 线程执行业务完毕清理本地数据
        if (dtxLocalContext.isDestroy()) {
            // 通知事务执行完毕
            synchronized (txContext.getLock()) {
                txContext.getLock().notifyAll();
            }

            // TxContext生命周期是？ 和事务组一样（不与具体模块相关的）
            if (!dtxLocalContext.isInGroup()) {
                globalContext.destroyTx();
            }
        }
        if(info.isTransactionStart() && info.getPropagation().equals(DTXPropagation.REQUIRES_NEW)) {
            DTXLocalContext.makeNeverAppeared();
            TracingContext.tracing().destroy();
        }
        log.debug("<---- TxLcn end ---->");
    }
}
```

这里会存在一个unitId代表每个服务的唯一性代码，如果存在直接返回，如果不存在则新建，是从DTXInfo中获取，通过Transactions的unitId方式生成（具体代码放在了业务代码里，不方便贴出来）。

```java
//根据方法名生成unitId然后去dtxInfoCache中获取，存在直接返回，不存在存入缓存中然后返回
public static DTXInfo getFromCache(MethodInvocation methodInvocation) {
    String signature = methodInvocation.getMethod().toString();
    String unitId = Transactions.unitId(signature);
    DTXInfo dtxInfo = dtxInfoCache.get(unitId);
    if (Objects.isNull(dtxInfo)) {
        dtxInfo = new DTXInfo(methodInvocation.getMethod(),
                methodInvocation.getArguments(), methodInvocation.getThis().getClass());
        dtxInfoCache.put(unitId, dtxInfo);
    }
    dtxInfo.reanalyseMethodArgs(methodInvocation.getArguments());
    return dtxInfo;
}
//md5根据方法的signature进行计算得到
public static String unitId(String methodSignature) {
    return DigestUtils.md5DigestAsHex((APPLICATION_ID_WHEN_RUNNING + methodSignature).getBytes());
}
```

DTXLocalContext.getOrNew()，会获取一个本地事务上下文信息，由于同一个事务可能多次用到，这里使用的时getOrNew当第一次使用使用直接新建，并保存在ThreadLocal变量currentLocal里，否则返回已经存在的，这里是为了保证每个请求对应获取到的是同一个类，避免并发问题引起分布式事务问题。

```java
private final static ThreadLocal<Map<String, DTXLocalContext>> currentLocal = new InheritableThreadLocal<>();

public static DTXLocalContext getOrNew() {
    //这里是dataSource的名称，可能是动态数据源，系统不同获取就不同。这里把本系统的代码去掉了
    String dataSourceName = "dataSource";
    if (currentLocal.get() == null) {
      	//存在上下文信息，当时不存在对应的dataSource的名称，针对多数据源
        currentLocal.set(new ConcurrentHashMap<String, DTXLocalContext>());
    }
  	//首先获取上下文信息
    DTXLocalContext dtxLocalContext = currentLocal.get().get(dataSourceName);
    if(null == dtxLocalContext) {
      	// 不存在直接新建事务上下文并保存到本地集合
        dtxLocalContext = new DTXLocalContext();
        currentLocal.get().put(dataSourceName, dtxLocalContext);
    }
    //返回上下文信息
    return dtxLocalContext;
}
```

globalContext.startTx()，启动事务生成一个groupId然后将其存入ThreadLocal中。

```java
public TxContext startTx() {
    TxContext txContext = new TxContext();
    // 事务发起方判断，这里判断是否有事务根据事务组id（groupId）进行判断
    txContext.setDtxStart(!TracingContext.tracing().hasGroup());
    if (txContext.isDtxStart()) {
      	//没有启动事务则启动事务
        TracingContext.tracing().beginTransactionGroup();
    }
    txContext.setGroupId(TracingContext.tracing().groupId());
    String txContextKey = txContext.getGroupId() + ".dtx";
    attachmentCache.attach(txContextKey, txContext);
    log.debug("Start TxContext[{}]", txContext.getGroupId());
    return txContext;
}
```

新建事务时会存在一个groupId代表整个事务唯一id，当事务不存在的情况下（根据groupId进行判断，tracingContextThreadLocal中不存在groupId则不存在，否则就是存在，不存在的初始化一个，会随机生成一个groupId，在tracingContextThreadLocal中加入初始化的事务信息）

```java
private static ThreadLocal<TracingContext> tracingContextThreadLocal = new ThreadLocal<>();

public static TracingContext tracing() {
    if (tracingContextThreadLocal.get() == null) {
        tracingContextThreadLocal.set(new TracingContext());
    }
    return tracingContextThreadLocal.get();
}
    

public void beginTransactionGroup() {
    if (hasGroup()) {
        return;
    }
  	//不存在事务组的情况下，初始化一个。gourpId是一个随机数
    init(Maps.newHashMap(TracingConstants.GROUP_ID, RandomUtils.randomKey(), TracingConstants.APP_MAP, "{}"));
}

public static void init(Map<String, String> initFields) {
    // 将生成的TracingContext信息加入到tracingContextThreadLocal
    if (Objects.isNull(initFields)) {
        log.warn("init tracingContext fail. null init fields.");
        return;
    }
    TracingContext tracingContext = tracing();
    if (Objects.isNull(tracingContext.fields)) {
        tracingContext.fields = new HashMap<>();
    }
    tracingContext.fields.putAll(initFields);
}
```

具体处理逻辑在这里，这里会有事务的传播行为处理propagationResolver.resolvePropagationState(info)，事务处理前操作（向服务端发送请求，将事务加入到事务组）dtxLocalControl.preBusinessCode(info)、具体的业务处理dtxLocalControl.doBusinessCode(info)、业务成功后的操作dtxLocalControl.onBusinessCodeSuccess(info, result)、业务执行失败操作dtxLocalControl.onBusinessCodeError(info, e)、业务结束操作dtxLocalControl.postBusinessCode(info)。

```java
public Object transactionRunning(TxTransactionInfo info) throws Throwable {

        // 1. 获取事务类型
        String transactionType = info.getTransactionType();

        // 2. 获取事务传播状态
        DTXPropagationState propagationState = propagationResolver.resolvePropagationState(info);

        // 2.1 如果不参与分布式事务立即终止
        if (propagationState.isIgnored()) {
            return info.getBusinessCallback().call();
        }

        // 3. 获取本地分布式事务控制器
        DTXLocalControl dtxLocalControl = txLcnBeanHelper.loadDTXLocalControl(transactionType, propagationState);

        // 4. 织入事务操作
        try {
            // 4.1 记录事务类型到事务上下文
            Set<String> transactionTypeSet = globalContext.txContext(info.getGroupId()).getTransactionTypes();
            transactionTypeSet.add(transactionType);

            dtxLocalControl.preBusinessCode(info);

            // 4.2 业务执行前
            txLogger.txTrace(
                    info.getGroupId(), info.getUnitId(), "pre business code, unit type: {}", transactionType);

            // 4.3 执行业务
            Object result = dtxLocalControl.doBusinessCode(info);

            // 4.4 业务执行成功
            txLogger.txTrace(info.getGroupId(), info.getUnitId(), "business success");
            dtxLocalControl.onBusinessCodeSuccess(info, result);
            return result;
        } catch (TransactionException e) {
            txLogger.error(info.getGroupId(), info.getUnitId(), "before business code error");
            throw e;
        } catch (Throwable e) {
            // 4.5 业务执行失败
            txLogger.error(info.getGroupId(), info.getUnitId(), Transactions.TAG_TRANSACTION,
                    "business code error");
            dtxLocalControl.onBusinessCodeError(info, e);
            throw e;
        } finally {
            // 4.6 业务执行完毕
            dtxLocalControl.postBusinessCode(info);
        }
    }
```

传播行为处理操作，具体处理是根据传递的属性判断、将事务加入到事务组、新建事务或直接不加入事务，这里不介绍具体的传播行为，不理解的可以执行查找了解下。

```java
public DTXPropagationState resolvePropagationState(TxTransactionInfo txTransactionInfo) throws TransactionException {

        // 本地已在DTX，根据事务传播，静默加入
        if (DTXLocalContext.cur().isInGroup()) {
            log.debug("SILENT_JOIN group!");
            return DTXPropagationState.SILENT_JOIN;
        }

        // 发起方之前没有事务
        if (txTransactionInfo.isTransactionStart()) {
            // 根据事务传播，对于 SUPPORTS 不参与事务
            if (DTXPropagation.SUPPORTS.equals(txTransactionInfo.getPropagation())) {
                return DTXPropagationState.NON;
            }
            // 根据事务传播，创建事务
            return DTXPropagationState.CREATE;
        }

        // 已经存在DTX，根据事务传播，加入
        return DTXPropagationState.JOIN;
    }
```

业务处理前操作dtxLocalControl.preBusinessCode(info)，这里dtxLocalControl有两个实现类，分别对应创建事务LcnStartingTransaction和加入事务LcnRunningTransaction，这里使用的是创建事务。

```java
public class LcnStartingTransaction implements DTXLocalControl {

    private final TransactionControlTemplate transactionControlTemplate;

    private final TCGlobalContext globalContext;


@Autowired
public LcnStartingTransaction(TransactionControlTemplate transactionControlTemplate, TCGlobalContext globalContext) {
 	this.transactionControlTemplate = transactionControlTemplate;
    this.globalContext = globalContext;
}

    @Override
    public void preBusinessCode(TxTransactionInfo info) throws TransactionException {
        // 创建事务操作
        transactionControlTemplate.createGroup(
                info.getGroupId(), info.getUnitId(), info.getTransactionInfo(), info.getTransactionType());
        DTXLocalContext.makeProxy();
    }

    @Override
    public void onBusinessCodeError(TxTransactionInfo info, Throwable throwable) {
        DTXLocalContext.cur().setSysTransactionState(0);
    }

    @Override
    public void onBusinessCodeSuccess(TxTransactionInfo info, Object result) {
        DTXLocalContext.cur().setSysTransactionState(1);
    }

    @Override
    public void postBusinessCode(TxTransactionInfo info) {
        // 这里是通知事务完成
        transactionControlTemplate.notifyGroup(
                info.getGroupId(), info.getUnitId(), info.getTransactionType(),
                DTXLocalContext.transactionState(globalContext.dtxState(info.getGroupId())));
		DTXLocalContext.makeNeverAppeared();
    }
}
```

创建事务组，向事务服务器发送事务。TransactionControlTemplate通过createGroup方法想事务服务其发送事务。

```java
public void createGroup(String groupId, String unitId, TransactionInfo transactionInfo, String transactionType)
            throws TransactionException {
        //创建事务组
        try {
            // 日志
            txLogger.txTrace(groupId, unitId,
                    "create group > transaction type: {}", transactionType);
            // 创建事务组消息，想服务端发送服务请求
            reliableMessenger.createGroup(groupId);
            // 缓存发起方切面信息
            aspectLogger.trace(groupId, unitId, transactionInfo);
        } catch (RpcException e) {
            // 通讯异常
            dtxExceptionHandler.handleCreateGroupMessageException(groupId, e);
        } catch (LcnBusinessException e) {
            // 创建事务组业务失败
            dtxExceptionHandler.handleCreateGroupBusinessException(groupId, e.getCause());
        }
        txLogger.txTrace(groupId, unitId, "create group over");
    }
	
	//向服务器端发送请求创建事务
    @Override
    public void createGroup(String groupId) throws RpcException, LcnBusinessException {
        // TxManager创建事务组
        MessageDto messageDto = request(MessageCreator.createGroup(groupId));
        if (!MessageUtils.statusOk(messageDto)) {
            throw new LcnBusinessException(messageDto.loadBean(Throwable.class));
        }
    }
```

通知事务完成，向服务器发送请求通知事务完成，这里是由于非事务创建者会有一个线程来获取该状态进行回滚使用。

```java
@Override
public int notifyGroup(String groupId, int transactionState) throws RpcException, LcnBusinessException {
    NotifyGroupParams notifyGroupParams = new NotifyGroupParams();
    notifyGroupParams.setGroupId(groupId);
    notifyGroupParams.setState(transactionState);
    //具体的发送请求信息，这里不详细介绍
    MessageDto messageDto = request0(MessageCreator.notifyGroup(notifyGroupParams),
            clientConfig.getTmRpcTimeout() * clientConfig.getChainLevel());
    // 成功清理发起方事务
    if (!MessageUtils.statusOk(messageDto)) {
        throw new LcnBusinessException(messageDto.loadBean(Throwable.class));
    }
    return messageDto.loadBean(Integer.class);
}
```

这里是通知事务完成，会向服务器发送一个信息，来告诉事务服务器事务处理完成，成功了或者失败了。

```java
@Override
public int notifyGroup(String groupId, int transactionState) throws RpcException, LcnBusinessException {
    NotifyGroupParams notifyGroupParams = new NotifyGroupParams();
    notifyGroupParams.setGroupId(groupId);
    notifyGroupParams.setState(transactionState);
    MessageDto messageDto = request0(MessageCreator.notifyGroup(notifyGroupParams),
            clientConfig.getTmRpcTimeout() * clientConfig.getChainLevel());
    // 成功清理发起方事务
    if (!MessageUtils.statusOk(messageDto)) {
        throw new LcnBusinessException(messageDto.loadBean(Throwable.class));
    }
    return messageDto.loadBean(Integer.class);
}
```

具体的事务处理是直接调用业务代码，这里就不介绍了。下connect切面DataSourceAspect，业务中每次获取数据库连接都通过切面进行处理。

```java
@Around("execution(* javax.sql.DataSource.getConnection(..))")
    public Object around(ProceedingJoinPoint point) throws Throwable {
        return dtxResourceWeaver.getConnection(() -> (Connection) point.proceed());
    }
```

切面的具体处理是将connection通过TransactionResourceProxy代理类进行处理，然后统一返回代理类处理逻辑如下。

```java
public Object getConnection(ConnectionCallback connectionCallback) throws Throwable {
   DTXLocalContext dtxLocalContext = DTXLocalContext.cur();
   if (Objects.nonNull(dtxLocalContext) && dtxLocalContext.isProxy()) {
      String transactionType = dtxLocalContext.getTransactionType();
      TransactionResourceProxy resourceProxy = txLcnBeanHelper.loadTransactionResourceProxy(transactionType);
     //这里是创建连接或获取代理类，并将其存入的集合中（key使用groupId）
      Connection connection = resourceProxy.proxyConnection(connectionCallback);
      if(!connection.isClosed()) {
         if(connection instanceof LcnConnectionProxy) {
            log.debug("proxy a LcnConnectionProxy connection: {}.", ((LcnConnectionProxy)connection).getTarget());
         }
         else {
            log.debug("proxy a sql connection: {}.", connection);
         }
         return connection;
      }
   }
   return connectionCallback.call();
}
```

这里是数据库连接类的处理会从globalContext获取一个连接，如果没有将新建一个连接然后加入到对应的ConcurrentHashMap中（这里有groupId做key不用担心存在多线程问题），这里是为后面提交和回滚操作做铺垫。

```java
@Override
public Connection proxyConnection(ConnectionCallback connectionCallback) throws Throwable {
    String groupId = DTXLocalContext.cur().getGroupId();
    try {
      	//通过上下文获取数据库连接
        return globalContext.getLcnConnection(groupId, DynamicDataSourceHolder.getDataSource());
    } catch (TCGlobalContextException e) {
      	//如果不存在连接，直接新建一个连接，将自动提交设置成否，然后加入到ConcurrentHashMap中，这里是为后续操作做铺垫
        LcnConnectionProxy lcnConnectionProxy = new LcnConnectionProxy(connectionCallback.call());
        globalContext.setLcnConnection(groupId, DynamicDataSourceHolder.getDataSource(), lcnConnectionProxy);
        lcnConnectionProxy.setAutoCommit(false);
        return lcnConnectionProxy;
    }
}
```
加入事务，这里通过类LcnRunningTransaction进行处理，这里是将preBusinessCode方法中创建事务去掉，然后将onBusinessCodeSuccess的发送事务成功的方法方法改成事务向事务服务器发送加入事务组请求，其内同步添加异步检测方法。

```java
public class LcnRunningTransaction implements DTXLocalControl {
    
    private final TransactionCleanTemplate transactionCleanTemplate;
    
    private final TransactionControlTemplate transactionControlTemplate;
    
    @Autowired
    public LcnRunningTransaction(TransactionCleanTemplate transactionCleanTemplate,
                                 TransactionControlTemplate transactionControlTemplate) {
        this.transactionCleanTemplate = transactionCleanTemplate;
        this.transactionControlTemplate = transactionControlTemplate;
    }
    
    
    @Override
    public void preBusinessCode(TxTransactionInfo info) {
        DTXLocalContext.makeProxy();
    }
    
    
    @Override
    public void onBusinessCodeError(TxTransactionInfo info, Throwable throwable) {
        try {
          	//调用事务的清理方法
            transactionCleanTemplate.clean(info.getGroupId(), info.getUnitId(), info.getTransactionType(), 0);
        } catch (TransactionClearException e) {
            log.error("{} > clean transaction error." , Transactions.LCN);
        }
    }
    
    
    @Override
    public void onBusinessCodeSuccess(TxTransactionInfo info, Object result) throws TransactionException {
        log.debug("join group: [GroupId: {},Method: {}]" , info.getGroupId(),
                info.getTransactionInfo().getMethodStr());
        
        //加入事务组
        transactionControlTemplate.joinGroup(info.getGroupId(), info.getUnitId(), info.getTransactionType(),
                info.getTransactionInfo());
    }
    
}

```

非事务服务器创建成功后，将其向事务服务器发请求，告诉事务服务器该服务器事务处理完成，将其加入事务组，加入成功后会有一个发起一个线程进行监督事务事务是否处理完成，对应的是事务创建服务器中的发送状态。

```java
public void joinGroup(String groupId, String unitId, String transactionType, TransactionInfo transactionInfo)
            throws TransactionException {
        try {
            txLogger.txTrace(groupId, unitId, "join group > transaction type: {}", transactionType);
			//这里将事务加入到事务组
            reliableMessenger.joinGroup(groupId, unitId, transactionType, DTXLocalContext.transactionState(globalContext.dtxState(groupId)));

            txLogger.txTrace(groupId, unitId, "join group message over.");

            // 异步检测，在这里检测事务是否处理完成
            dtxChecking.startDelayCheckingAsync(groupId, unitId, transactionType);

            // 缓存参与方切面信息
            aspectLogger.trace(groupId, unitId, transactionInfo);
        } catch (RpcException e) {
            dtxExceptionHandler.handleJoinGroupMessageException(Arrays.asList(groupId, unitId, transactionType), e);
        } catch (LcnBusinessException e) {
            dtxExceptionHandler.handleJoinGroupBusinessException(Arrays.asList(groupId, unitId, transactionType), e);
        }
        txLogger.txTrace(groupId, unitId, "join group logic over");
    }
```

这里是具体的检测方法，这里是通过线程池进行定时向服务器发送请求，获取事物的状态，即事物的创建者是否完成事物的创建。

```java
public void startDelayCheckingAsync(String groupId, String unitId, String transactionType) {
        txLogger.taskTrace(groupId, unitId, "start delay checking task");
        ScheduledFuture scheduledFuture = scheduledExecutorService.schedule(() -> {
            try {
                TxContext txContext = globalContext.txContext(groupId);
                if (Objects.nonNull(txContext)) {
                    synchronized (txContext.getLock()) {
                        txLogger.taskTrace(groupId, unitId, "checking waiting for business code finish.");
                        txContext.getLock().wait();
                    }
                }
              	//这里是发送事务请求，查看是否发送成功
                int state = reliableMessenger.askTransactionState(groupId, unitId);
                txLogger.taskTrace(groupId, unitId, "ask transaction state {}", state);
                if (state == -1) {
                    txLogger.error(this.getClass().getSimpleName(), "delay clean transaction error.");
                  	//这里是发送请求后，事务失败后的操作
                    onAskTransactionStateException(groupId, unitId, transactionType);
                } else {
                  	//这里是事务处理成功的操作
                    transactionCleanTemplate.clean(groupId, unitId, transactionType, state);
                    aspectLogger.clearLog(groupId, unitId);
                }

            } catch (RpcException e) {
                onAskTransactionStateException(groupId, unitId, transactionType);
            } catch (TransactionClearException | InterruptedException e) {
                txLogger.error(this.getClass().getSimpleName(), "{} clean transaction error.", transactionType);
            }
        }, clientConfig.getDtxTime(), TimeUnit.MILLISECONDS);
        delayTasks.put(groupId + unitId, scheduledFuture);
    }
```

事务成功后操作具体的接口为TransactionCleanService，这里对应的是三个实现类，对应三种事务处理方法。我们的是lcn，通过groupId取到数据源切面的连接代理类。

```java


@Override
    public void clear(String groupId, int state, String unitId, String unitType) throws TransactionClearException {
        try {
        	//这里是获取数据库连接操作，跟上面的事务切面相对应。同时groupId取得事务组连接
            Set<LcnConnectionProxy> connectionProxy = globalContext.getLcnConnection(groupId);
            //取得的连接进行提交或者回滚操作
            connectionProxy.forEach(con -> con.notify(state));
            // todo notify exception
        } catch (TCGlobalContextException e) {
            log.warn("Non lcn connection when clear transaction.");
        } finally {
			DTXLocalContext.makeNeverAppeared();
		}
    }
```

这里是具体的事务成功或者失败，进行提交和回滚的操作。这里具体的类为LcnConnectionProxy，对应事务的成功操作后的提交和回滚操作。这里的connection适合DataSourceAspect生成的连接是同一个，之前是通过groupId存入ConcurrentHashMap，这里再通过groupId取出对应的数据库连接。

```java
public RpcResponseState notify(int state) {
        try {
        	if(!connection.isClosed()) {
        		if (state == 1) {
        			log.debug("commit transaction type[lcn] proxy connection:{}.", connection);
        			//事务成功进行提交操作
        			connection.commit();
        		} else {
        			log.debug("rollback transaction type[lcn] proxy connection:{}.", connection);
        			//事务失败进行回滚操作
        			connection.rollback();
        		}
        		connection.close();
        	}
            log.debug("transaction type[lcn] proxy connection:{} closed.", connection);
            return RpcResponseState.success;
        } catch (Exception e) {
            log.error(e.getLocalizedMessage(), e);
            return RpcResponseState.fail;
        }
    }
```

这里是事务处理失败的情况，是将事务发送状态设置为失败，然后调用上面的clean命令进行回滚。失败和成功最终都会调用clear方法进行事物的处理，失败释放在异常中进行处理，但是失败会分两种情况，是否创建过connection，创建了就进行回滚，未创建则不需要。这里可能会存在多种情况（由于发生异常的情况不同导致，有些是可以提交的，有些不能）这里不详细介绍，具体思路即能提交的就调将状态设置成1，然后clear方法进行提交，不能提交的则将状态设置成非1。

```java
@Override
    public void handleNotifyGroupBusinessException(Object params, Throwable ex) {
        List paramList = (List) params;
        String groupId = (String) paramList.get(0);
        int state = (int) paramList.get(1);
        String unitId = (String) paramList.get(2);
        String transactionType = (String) paramList.get(3);

        if(null != ex) {
            //用户强制回滚.
            if (ex instanceof UserRollbackException) {
                state = 0;
            }
            if ((ex.getCause() != null && ex.getCause() instanceof UserRollbackException)) {
                state = 0;
            }
        }
        else {
            // LCN异常
            state = 0;
        }

        // 结束事务
        try {
            transactionCleanTemplate.clean(groupId, unitId, transactionType, state);
        } catch (TransactionClearException e) {
            txLogger.error(groupId, unitId, "notify group", "{} > clean transaction error.", transactionType);
        }
    }
```

事物的groupId传递，Tracings这里通过将请求头进行处理，这里是通过restTemplate拦截器进行处理（其它方式如dubbo、feign可自行查看源码），具体思路是事务创建者将groupId存入请求头，事务的加入者会根据请求头来获取groupId然后根据groupId初始化TracingContext。

```java
public class RestTemplateTracingTransmitter implements ClientHttpRequestInterceptor {

    @Autowired
    public RestTemplateTracingTransmitter(@Autowired(required = false) List<RestTemplate> restTemplates) {
        if (Objects.nonNull(restTemplates)) {
            restTemplates.forEach(restTemplate -> {
                List<ClientHttpRequestInterceptor> interceptors = restTemplate.getInterceptors();
                interceptors.add(interceptors.size(), RestTemplateTracingTransmitter.this);
            });
        }
    }

    @Override
    @NonNull
    public ClientHttpResponse intercept(
            @NonNull HttpRequest httpRequest, @NonNull byte[] bytes,
            @NonNull ClientHttpRequestExecution clientHttpRequestExecution) throws IOException {
      	//这里是Lambda表达式写法httpRequest.getHeaders()::add是对TracingSetter的set方法的实现
        Tracings.transmit(httpRequest.getHeaders()::add);
        return clientHttpRequestExecution.execute(httpRequest, bytes);
    }
}
//这里springMvc的配置类，可以进行具体业务处理前的一些准备工作如解码
public class SpringTracingApplier implements com.codingapi.txlcn.tracing.http.spring.HandlerInterceptor, WebMvcConfigurer {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
      	//这里是Lambda表达式写法httpRequest.getHeaders()::add是对TracingGetter的get方法的实现
        Tracings.apply(request::getHeader);
        return true;
    }
	//将拦截器加入到容器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(this);
    }
}


public class Tracings {
  	//私有构造器，保证了该方法不会被外部直接构建，确保gourpId能够被正确传递
    private Tracings() {
    }

    public static void transmit(TracingSetter tracingSetter) {
      	//请求发起前，将groupId存入报文头
        if (TracingContext.tracing().hasGroup()) {
            log.debug("tracing transmit group:{}", TracingContext.tracing().groupId());
            tracingSetter.set(TracingConstants.HEADER_KEY_GROUP_ID, TracingContext.tracing().groupId());
            tracingSetter.set(TracingConstants.HEADER_KEY_APP_MAP,									     									Base64Utils.encodeToString(TracingContext.tracing().                                                                           					appMapString().getBytes(StandardCharsets.UTF_8)));
        }
    }

    public static void apply(TracingGetter tracingGetter) {
        String groupId = Optional.ofNullable(tracingGetter.get(TracingConstants.HEADER_KEY_GROUP_ID)).orElse("");
        String appList = Optional.ofNullable(tracingGetter.get(TracingConstants.HEADER_KEY_APP_MAP)).orElse("");
      	//这里和之前的init方法一致，这里是通过获取报文头的获取groupId进行初始化，保证事务创建者的groupId能传过来
        TracingContext.init(Maps.newHashMap(TracingConstants.GROUP_ID, groupId, TracingConstants.APP_MAP,
                StringUtils.isEmpty(appList) ? appList : new String(Base64Utils.decodeFromString(appList), 					   StandardCharsets.UTF_8)));
        if (TracingContext.tracing().hasGroup()) {
            log.debug("tracing apply group:{}, app map:{}", groupId, appList);
        }
    }

    public interface TracingSetter {
        void set(String key, String value);
    }


    public interface TracingGetter {
        String get(String key);
    }
}

```

这是lcn分布式事务的处理的具体思路，这里介绍的还是有点缺陷的，事物的补偿并未考虑（概率很小）。即当事务全部处理完成后，事务服务器收到事务创建者的信息后，事务服务器或者后面用到的服务器夯机如何处理。也就是创建者服务器的事务提交了，其他服务器的事务提交不了。我这里想法是通过redis对应事务组信息去进行补偿处理，各位小伙伴也可以考虑下如何保证进行 补偿或者如何保证事务的强一致性。