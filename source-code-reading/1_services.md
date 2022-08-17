* [一 子服务概览](#一-子服务概览)
  * [1 Config](#1-config)
    * [初始化](#初始化)
    * [关键模块](#关键模块)
      * [Config](#config)
      * [QueryConfig](#queryconfig)
      * [LogConfig](#logconfig)
      * [MetaConfig](#metaconfig)
      * [StorageConfig](#storageconfig)
      * [HiveCatalogConfig](#hivecatalogconfig)
  * [2 Tracing](#2-tracing)
  * [3 SessionManager](#3-sessionmanager)
    * [初始化](#初始化)
    * [关键模块](#关键模块)
      * [SessionManager](#sessionmanager)
      * [Config](#config)
      * [ClusterDiscovery](#clusterdiscovery)
      * [CatalogManager](#catalogmanager)
      * [HttpQueryManager](#httpquerymanager)
      * [CacheManager](#cachemanager)
  * [4 MySQL Handler](#4-mysql-handler)
    * [启动](#启动)
    * [关键模块](#关键模块)
      * [MySQLHandler](#mysqlhandler)
      * [MySQLConnection](#mysqlconnection)
      * [InteractiveWorker/InteractiveWorkerBase](#interactiveworkerinteractiveworkerbase)
      * [Session](#session)
      * [QueryContextShared](#querycontextshared)
      * [QueryContext](#querycontext)
  * [5 ClickHouse Handler](#5-clickhouse-handler)
  * [6 HTTP Handler](#6-http-handler)
  * [7 Metrics API Service](#7-metrics-api-service)
    * [启动](#启动)
    * [关键模块](#关键模块)
      * [MetricService](#metricservice)
      * [PROMETHEUS_HANDLE](#prometheus_handle)
  * [8 HTTP API Service](#8-http-api-service)
    * [启动](#启动)
    * [关键模块](#关键模块)
      * [HttpService](#httpservice)
  * [9 RPC API Service](#9-rpc-api-service)
    * [Service启动](#service启动)
    * [关键模块](#关键模块)
      * [RpcService](#rpcservice)
      * [DatabendQueryFlightService](#databendqueryflightservice)
      * [DatabendQueryFlightDispatcher ⭐️](#databendqueryflightdispatcher-)
      * [FlightScatter](#flightscatter)
      * [HashFligthScatter](#hashfligthscatter)
      * [BroadcastFlightScatter](#broadcastflightscatter)
  * [10 Cluster Register](#10-cluster-register)
    * [Woker节点注册](#woker节点注册)
    * [关键模块](#关键模块)
      * [SessionManager](#sessionmanager)
      * [ClusterDiscovery/ClusterHeartbeat/Cluster/ClusterMgr](#clusterdiscoveryclusterheartbeatclusterclustermgr)

## 一 子服务概览


| 模块                | 功能                        |
| --------------------- | ----------------------------- |
| Config              | 负责加载和管理配置文件      |
| Tracing             | 负责日志管理                |
| SessionManager      | 负责session管理             |
| MySQL Handler       | 负责对外提供MySQL服务       |
| ClickHouse Handler  | 负责对外提供ClickHouse服务  |
| HTTP Handler        | 负责对外提供HTTP接口服务    |
| Metrics API Service | 负责指标统计（Prometheus）  |
| RPC API Service     | 负责RPC接口服务和节点间通信 |
| Cluster Register    | 负责节点的注册              |

### 1 Config

#### 初始化

```rust
// file: query/bin/databend-query.rs
let conf: Config = Config::load()?;
```

#### 关键模块

##### Config

```rust
// file: query/src/config/inner.rs
pub struct Config {
    pub cmd: String,
    pub config_file: String,

    // Query engine config.
    pub query: QueryConfig,

    pub log: LogConfig,

    // Meta Service config.
    pub meta: MetaConfig,

    // Storage backend config.
    pub storage: StorageConfig,

    // external catalog config.
    // - Later, catalog information SHOULD be kept in KV Service
    // - currently only supports HIVE (via hive meta store)
    pub catalog: HiveCatalogConfig,
}
// func: load() -> Result<Self>
// 加载
    pub fn load() -> Result<Self> {
        let cfg = OuterV0Config::load()?.try_into()?; // 通过try_into()转换到Config

        Ok(cfg)
    }

// file: query/src/config/outer_v0.rs
pub struct Config {
    /// Run a command and quit
    #[clap(long, default_value_t)]
    pub cmd: String,

    #[clap(long, short = 'c', default_value_t)]
    pub config_file: String,

    // Query engine config.
    #[clap(flatten)]
    pub query: QueryConfig,

    #[clap(flatten)]
    pub log: LogConfig,

    // Meta Service config.
    #[clap(flatten)]
    pub meta: MetaConfig,

    // Storage backend config.
    #[clap(flatten)]
    pub storage: StorageConfig,

    // external catalog config.
    // - Later, catalog information SHOULD be kept in KV Service
    // - currently only supports HIVE (via hive meta store)
    #[clap(flatten)]
    pub catalog: HiveCatalogConfig,
}
// func: load() -> Self
// 从文件中加载
    /// Load will load config from file, env and args.
    ///
    /// - Load from file as default.
    /// - Load from env, will override config from file.
    /// - Load from args as finally override
    pub fn load() -> Result<Self> {
        let arg_conf = Self::parse();

        let mut builder: serfig::Builder<Self> = serfig::Builder::default(); // serfig::Builder是基于serde的分层配置系统

        // Load from config file first.
        {
            let config_file = if !arg_conf.config_file.is_empty() {
                arg_conf.config_file.clone()
            } else if let Ok(path) = env::var("CONFIG_FILE") {
                path
            } else {
                "".to_string()
            };

            builder = builder.collect(from_file(Toml, &config_file));
        }

        // Then, load from env.
        builder = builder.collect(from_env());

        // Finally, load from args.
        builder = builder.collect(from_self(arg_conf));

        Ok(builder.build()?)
    }
```

##### QueryConfig

```rust
// file: query/src/config/inner.rs
pub struct QueryConfig {
    /// Tenant id for get the information from the MetaSrv.
    pub tenant_id: String,
    /// ID for construct the cluster.
    pub cluster_id: String,
    pub num_cpus: u64,
    pub mysql_handler_host: String,
    pub mysql_handler_port: u16,
    pub max_active_sessions: u64,
    pub clickhouse_handler_host: String,
    pub clickhouse_handler_port: u16,
    pub http_handler_host: String,
    pub http_handler_port: u16,
    pub http_handler_result_timeout_millis: u64,
    pub flight_api_address: String,
    pub admin_api_address: String,
    pub metric_api_address: String,
    pub http_handler_tls_server_cert: String,
    pub http_handler_tls_server_key: String,
    pub http_handler_tls_server_root_ca_cert: String,
    pub api_tls_server_cert: String,
    pub api_tls_server_key: String,
    pub api_tls_server_root_ca_cert: String,
    /// rpc server cert
    pub rpc_tls_server_cert: String,
    /// key for rpc server cert
    pub rpc_tls_server_key: String,
    /// Certificate for client to identify query rpc server
    pub rpc_tls_query_server_root_ca_cert: String,
    pub rpc_tls_query_service_domain_name: String,
    /// Table engine memory enabled
    pub table_engine_memory_enabled: bool,
    /// Database engine github enabled
    pub database_engine_github_enabled: bool,
    pub wait_timeout_mills: u64,
    pub max_query_log_size: usize,
    /// Table Cached enabled
    pub table_cache_enabled: bool,
    /// Max number of cached table snapshot
    pub table_cache_snapshot_count: u64,
    /// Max number of cached table segment
    pub table_cache_segment_count: u64,
    /// Max number of cached table block meta
    pub table_cache_block_meta_count: u64,
    /// Table memory cache size (mb)
    pub table_memory_cache_mb_size: u64,
    /// Table disk cache folder root
    pub table_disk_cache_root: String,
    /// Table disk cache size (mb)
    pub table_disk_cache_mb_size: u64,
    /// If in management mode, only can do some meta level operations(database/table/user/stage etc.) with metasrv.
    pub management_mode: bool,
    pub jwt_key_file: String,
}
```

##### LogConfig

```rust
// file: common/tracing/src/config.rs
use common_tracing::Config as LogConfig;
pub struct Config {
    pub level: String,
    pub dir: String,
    pub query_enabled: bool,
}
```

##### MetaConfig

```rust
// file: query/src/config/inner.rs
pub struct MetaConfig {
    /// The dir to store persisted meta state for a embedded meta store
    pub embedded_dir: String,
    /// MetaStore backend address
    pub address: String,
    pub endpoints: Vec<String>,
    /// MetaStore backend user name
    pub username: String,
    /// MetaStore backend user password
    pub password: String,
    /// Timeout for each client request, in seconds
    pub client_timeout_in_second: u64,
    /// Certificate for client to identify meta rpc serve
    pub rpc_tls_meta_server_root_ca_cert: String,
    pub rpc_tls_meta_service_domain_name: String,
}
```

##### StorageConfig

```rust
// file: common/io/src/configs.rs
pub struct StorageConfig {
    pub num_cpus: u64,

    pub params: StorageParams,
}

pub enum StorageParams {
    Azblob(StorageAzblobConfig),
    Fs(StorageFsConfig),
    #[cfg(feature = "storage-hdfs")]
    Hdfs(StorageHdfsConfig),
    Memory,
    S3(StorageS3Config),
}

pub struct StorageFsConfig {
    pub root: String,
}

pub struct StorageS3Config {
    pub endpoint_url: String,
    pub region: String,
    pub bucket: String,
    pub access_key_id: String,
    pub secret_access_key: String,
    pub master_key: String,
    pub root: String,
}
```

##### HiveCatalogConfig

```rust
// file: query/src/config/inner.rs
pub struct HiveCatalogConfig {
    pub meta_store_address: String,
    pub protocol: ThriftProtocol,
}
```

### 2 Tracing

```rust
    let _guards = init_global_tracing(
        app_name.as_str(),
        conf.log.dir.as_str(),
        conf.log.level.as_str(),
    );
```

### 3 SessionManager

#### 初始化

```rust
// file: query/bin/databend-query.rs
    let session_manager = SessionManager::from_conf(conf.clone()).await?;
    let mut shutdown_handle = ShutdownHandle::create(session_manager.clone());
```

#### 关键模块

##### SessionManager

```rust
// file: query/src/sessions/session_mgr.rs
pub struct SessionManager {
    pub(in crate::sessions) conf: RwLock<Config>,
    pub(in crate::sessions) discovery: RwLock<Arc<ClusterDiscovery>>, // 集群发现模块
    pub(in crate::sessions) catalogs: RwLock<Arc<CatalogManager>>,
    pub(in crate::sessions) http_query_manager: Arc<HttpQueryManager>,

    pub(in crate::sessions) max_sessions: usize,
    pub(in crate::sessions) active_sessions: Arc<RwLock<HashMap<String, Arc<Session>>>>,
    pub(in crate::sessions) storage_cache_manager: RwLock<Arc<CacheManager>>,
    pub(in crate::sessions) query_logger:
        RwLock<Option<Arc<dyn tracing::Subscriber + Send + Sync>>>,
    pub status: Arc<RwLock<SessionManagerStatus>>,
    storage_operator: RwLock<Operator>,
    storage_runtime: Arc<Runtime>,
    _guards: Vec<WorkerGuard>,

    user_api_provider: RwLock<Arc<UserApiProvider>>,
    role_cache_manager: RwLock<Arc<RoleCacheMgr>>,
    // When typ is MySQL, insert into this map, key is id, val is MySQL connection id.
    pub(crate) mysql_conn_map: Arc<RwLock<HashMap<Option<u32>, String>>>,
    pub(in crate::sessions) mysql_basic_conn_id: AtomicU32,
}
```

##### Config

##### ClusterDiscovery

##### CatalogManager

##### HttpQueryManager

##### CacheManager

### 4 MySQL Handler

#### 启动

```rust
// file:  query/bin/databend-query.rs
// func: main()
   // MySQL handler.
    {
        let hostname = conf.query.mysql_handler_host.clone();
        let listening = format!("{}:{}", hostname, conf.query.mysql_handler_port); // 生成服务地址
        let mut handler = MySQLHandler::create(session_manager.clone()); // 创建MySQLHandler
        let listening = handler.start(listening.parse()?).await?; // 启动MySQLHandler
        shutdown_handle.add_service(handler); // 填加到shutdown_handle
    }
```

#### 关键模块

##### MySQLHandler

```rust
// file: query/src/servers/mysql/mysql_handler.rs
/// 实现了Server trait，并提供了创建MySQL server的方法，包括listener_tcp(), listen_loop(), accept_socket(), reject_session()等
pub struct MySQLHandler {
    sessions: Arc<SessionManager>,
    abort_handle: AbortHandle,
    abort_registration: Option<AbortRegistration>,
    join_handle: Option<JoinHandle<()>>,
}
// func: start(...)
// 启动MySQL server
    async fn start(&mut self, listening: SocketAddr) -> Result<SocketAddr> {
        match self.abort_registration.take() {
            None => Err(ErrorCode::LogicalError("MySQLHandler already running.")),
            Some(registration) => {
                let rejected_rt = Arc::new(Runtime::with_worker_threads(
                    1,
                    Some("mysql-handler".to_string()),
                )?); // 创建rejected runtime
                let (stream, listener) = Self::listener_tcp(listening).await?; // 创建TcpListenerStream
                let stream = Abortable::new(stream, registration);
                self.join_handle = Some(tokio::spawn(self.listen_loop(stream, rejected_rt))); // 监听客户端的连接请求，然后调用accept_socket()函数
                Ok(listener)
            }
        }
    }
// func: accept_socket(...)
// 接受或者拒绝session
    fn accept_socket(sessions: Arc<SessionManager>, executor: Arc<Runtime>, socket: TcpStream) {
        executor.spawn(async move {
            match sessions.create_session(SessionType::MySQL).await { // 创建session
                Err(error) => Self::reject_session(socket, error).await, // 拒绝
                Ok(session) => {
                    tracing::info!("MySQL connection coming: {:?}", socket.peer_addr());
                    if let Err(error) = MySQLConnection::run_on_stream(session, socket) { // 接受
                        tracing::error!("Unexpected error occurred during query: {:?}", error);
                    };
                }
            }
        });
    }
```

##### MySQLConnection

```rust
// query/src/servers/mysql/mysql_session.rs
// func: run_on_stream(...)
    pub fn run_on_stream(session: SessionRef, stream: TcpStream) -> Result<()> {
        let blocking_stream = Self::convert_stream(stream)?; // 转化为标准的TcpStream
        MySQLConnection::attach_session(&session, &blocking_stream)?; // attach session

        let non_blocking_stream = TcpStream::from_std(blocking_stream)?;
        let query_executor =
            Runtime::with_worker_threads(1, Some("mysql-query-executor".to_string()))?;
        Thread::spawn(move || {
            let join_handle = query_executor.spawn(async move {
                let client_addr = non_blocking_stream.peer_addr().unwrap().to_string();
                let interactive_worker = InteractiveWorker::create(session, client_addr); // 创建InteractiveWorker
                let opts = IntermediaryOptions {
                    process_use_statement_on_query: true,
                };
                AsyncMysqlIntermediary::run_with_options( // 创建AsyncMysqlIntermediary，这个是一个第三方库，具体实现在InteractiveWorker中
                    interactive_worker,
                    non_blocking_stream,
                    &opts,
                )
                .await
            });
            let _ = futures::executor::block_on(join_handle);
        });
        Ok(())
    }
```

##### InteractiveWorker/InteractiveWorkerBase

```rust
// query/src/servers/mysql/mysql_interactive_worker.rs
/// 实现了AsyncMysqlShim trait，封装了InteractiveWorkerBase
pub struct InteractiveWorker<W: std::io::Write> {
    session: SessionRef,
    base: InteractiveWorkerBase<W>, // 实际的操作在InteractiveWorkerBase中
    version: String,
    salt: [u8; 20],
    client_addr: String,
}
// func: on_query(...)
    async fn on_query<'a>(
        &'a mut self,
        query: &'a str,
        writer: QueryResultWriter<'a, W>,
    ) -> Result<()> {
        if self.session.is_aborting() {
            writer.error(
                ErrorKind::ER_ABORTING_CONNECTION,
                "Aborting this connection. because we are try aborting server.".as_bytes(),
            )?;

            return Err(ErrorCode::AbortedSession(
                "Aborting this connection. because we are try aborting server.",
            ));
        }

        let mut writer = DFQueryResultWriter::create(writer); // 创建Response Wirter

        let instant = Instant::now();
        let blocks = self.base.do_query(query).await; // 调用InteractiveWorkerBase的do_query()，获得查询结果blocks

        let format = self
            .session
            .get_shared_query_context()
            .await?
            .get_format_settings()?;
        let mut write_result = writer.write(blocks, &format); // 将查询结果写到writer

        if let Err(cause) = write_result {
            let suffix = format!("(while in query {})", query);
            write_result = Err(cause.add_message_back(suffix));
        }

        histogram!(
            super::mysql_metrics::METRIC_MYSQL_PROCESSOR_REQUEST_DURATION,
            instant.elapsed()
        );

        write_result
    }

/// 实现MySQL server的处理细节
struct InteractiveWorkerBase<W: std::io::Write> {
    session: SessionRef, // 是在MySQLHandler::accept_socket()中，由SessionManager::create_session()创建的
    generic_hold: PhantomData<W>,
}
// func: do_query(...)
// 查询入口
    async fn do_query(&mut self, query: &str) -> Result<(Vec<DataBlock>, String)> {
        match self.federated_server_command_check(query) { // 判断是不是Federated query
            Some(data_block) => { // 1 Federated查询
                tracing::info!("Federated query: {}", query);
                if data_block.num_rows() > 0 {
                    tracing::info!("Federated response: {:?}", data_block);
                }
                Ok((vec![data_block], String::from("")))
            }
            None => { // 2 普通查询
                tracing::info!("Normal query: {}", query);
                let context = self.session.create_query_context().await?; // 创建QueryContext，这里会获取Shared信息，具体查询Session的详解
                context.attach_query_str(query); // 记录query

                let settings = context.get_settings();

                let (stmts, hints) =
                    DfParser::parse_sql(query, context.get_current_session().get_type())?; // 对SQL进行词法语法分析，hits是什么？？？

                // 创建Interpreter，比如SelectInterpreter或者SelectInterpreterV2等
                let interpreter: Result<Arc<dyn Interpreter>> =
                    if settings.get_enable_new_processor_framework()? != 0
                        && context.get_cluster().is_empty()
                        && settings.get_enable_planner_v2()? != 0
                        && stmts.get(0).map_or(false, InterpreterFactoryV2::check)
                    {
                        let mut planner = Planner::new(context.clone()); // 新的Planner
                        planner
                            .plan_sql(query)
                            .await
                            .and_then(|v| InterpreterFactoryV2::get(context.clone(), &v.0))
                    } else {
                        let (plan, _) = PlanParser::parse_with_hint(query, context.clone()).await; // 旧的Planner
                        plan.and_then(|v| InterpreterFactory::get(context.clone(), v))
                    };

                let hint = hints
                    .iter()
                    .find(|v| v.error_code.is_some())
                    .and_then(|x| x.error_code);

                match (hint, interpreter) {
                    (None, Ok(interpreter)) => Self::exec_query(interpreter, &context).await, // 执行查询
                    (Some(code), Ok(interpreter)) => { // Error 1
                        let res = Self::exec_query(interpreter, &context).await;
                        match res {
                            Ok(_) => Err(ErrorCode::UnexpectedError(format!(
                                "Expected server error code: {} but got: Ok.",
                                code
                            ))),
                            Err(e) => {
                                if code != e.code() {
                                    return Err(ErrorCode::UnexpectedError(format!(
                                        "Expected server error code: {} but got: Ok.",
                                        code
                                    )));
                                }
                                Ok((vec![DataBlock::empty()], String::from("")))
                            }
                        }
                    }
                    (None, Err(e)) => { // Error 2 
                        InterpreterQueryLog::fail_to_start(context, e.clone()).await;
                        Err(e)
                    }
                    (Some(code), Err(e)) => { // Error 3
                        if code != e.code() {
                            InterpreterQueryLog::fail_to_start(context, e.clone()).await;
                            return Err(ErrorCode::UnexpectedError(format!(
                                "Expected server error code: {} but got: Ok.",
                                code
                            )));
                        }
                        Ok((vec![DataBlock::empty()], String::from("")))
                    }
                }
            }
        }
    }
```

##### Session

```rust
// file: query/src/sessions/session.rs
pub struct Session {
    pub(in crate::sessions) id: String,
    #[ignore_malloc_size_of = "insignificant"]
    pub(in crate::sessions) typ: RwLock<SessionType>,
    #[ignore_malloc_size_of = "insignificant"]
    pub(in crate::sessions) session_mgr: Arc<SessionManager>,
    pub(in crate::sessions) ref_count: Arc<AtomicUsize>,
    pub(in crate::sessions) session_ctx: Arc<SessionContext>,
    #[ignore_malloc_size_of = "insignificant"]
    session_settings: Settings,
    #[ignore_malloc_size_of = "insignificant"]
    status: Arc<RwLock<SessionStatus>>,
    pub(in crate::sessions) mysql_connection_id: Option<u32>,
}
// func: create_query_context(...) -> Result<Arc<QueryContext>>
// 创建QueryContext
    /// Create a query context for query.
    /// For a query, execution environment(e.g cluster) should be immutable.
    /// We can bind the environment to the context in create_context method.
    pub async fn create_query_context(self: &Arc<Self>) -> Result<Arc<QueryContext>> {
        let shared = self.get_shared_query_context().await?;

        Ok(QueryContext::create_from_shared(shared))
    }
// func: get_shared_query_context(...) -> Result<Arc<QueryContextShared>>
// 创建QueryContextShared
    pub async fn get_shared_query_context(self: &Arc<Self>) -> Result<Arc<QueryContextShared>> {
        let discovery = self.session_mgr.get_cluster_discovery();

        let session = self.clone();
        let cluster = discovery.discover().await?; // 获取集群信息
        let shared = QueryContextShared::try_create(session, cluster).await?; // 创建QueryContextShared
        self.session_ctx
            .set_query_context_shared(Some(shared.clone()));
        Ok(shared)
    }
```

##### QueryContextShared

```rust
// file: query/src/sessions/query_ctx_shared.rs
pub struct QueryContextShared {
    ...
}
```

##### QueryContext

```rust
// file: query/src/sessions/query_ctx.rs
pub struct QueryContext {
    version: String,
    statistics: Arc<RwLock<Statistics>>,
    partition_queue: Arc<RwLock<VecDeque<PartInfoPtr>>>,
    shared: Arc<QueryContextShared>,
    precommit_blocks: Arc<RwLock<Vec<DataBlock>>>,
}
```

### 5 ClickHouse Handler

### 6 HTTP Handler

### 7 Metrics API Service

#### 启动

```rust
// file: query/bin/databend-query.rs
    init_default_metrics_recorder(); // 初始化metrics

    // Metric API service.
    {
        let address = conf.query.metric_api_address.clone();
        let mut srv = MetricService::create(session_manager.clone());
        let listening = srv.start(address.parse()?).await?;
        shutdown_handle.add_service(srv);
        tracing::info!("Metric API server listening on {}/metrics", listening);
    }
```

#### 关键模块

##### MetricService

```rust
// file: query/src/metrics/metric_service.rs
pub struct MetricService {
    shutdown_handler: HttpShutdownHandler,
}
// func: start()
    async fn start(&mut self, listening: SocketAddr) -> Result<SocketAddr> {
        self.start_without_tls(listening).await
    }
// func: start_without_tls() 
    async fn start_without_tls(&mut self, listening: SocketAddr) -> Result<SocketAddr> {
        let prometheus_handle = common_metrics::try_handle().ok_or_else(|| {
            ErrorCode::InitPrometheusFailure("Prometheus recorder has not been initialized yet.")
        })?; // 创建PrometheusHandle
        let app = poem::Route::new()
            .at("/metrics", poem::get(metric_handler))
            .data(prometheus_handle); // 注册路由
        let addr = self
            .shutdown_handler
            .start_service(listening, None, app) // 启动服务
            .await?;
        Ok(addr)
    }
// func: metric_handler() -> impl IntoResponse
#[poem::handler]
pub async fn metric_handler(prom_extension: Data<&PrometheusHandle>) -> impl IntoResponse {
    prom_extension.0.render()
}
```

##### PROMETHEUS_HANDLE

```rust
// file: common/metrics/src/recorder.rs
static PROMETHEUS_HANDLE: Lazy<Arc<RwLock<Option<PrometheusHandle>>>> =
    Lazy::new(|| Arc::new(RwLock::new(None)));

pub const LABEL_KEY_TENANT: &str = "tenant";
pub const LABEL_KEY_CLUSTER: &str = "cluster_name";

pub fn init_default_metrics_recorder() {
    static START: Once = Once::new();
    START.call_once(init_prometheus_recorder)
}

/// Init prometheus recorder.
fn init_prometheus_recorder() {
    let recorder = PrometheusBuilder::new().build_recorder();
    let mut h = PROMETHEUS_HANDLE.as_ref().write();
    *h = Some(recorder.handle());
    metrics::clear_recorder();
    match metrics::set_boxed_recorder(Box::new(recorder)) {
        Ok(_) => (),
        Err(err) => tracing::warn!("Install prometheus recorder failed, cause: {}", err),
    };
}

pub fn try_handle() -> Option<PrometheusHandle> {
    PROMETHEUS_HANDLE.as_ref().read().clone()
}
```

### 8 HTTP API Service

#### 启动

```rust
// file: query/bin/databend-query.rs
    // HTTP API service.
    {
        let address = conf.query.admin_api_address.clone();
        let mut srv = HttpService::create(session_manager.clone());
        let listening = srv.start(address.parse()?).await?;
        shutdown_handle.add_service(srv);
        tracing::info!("HTTP API server listening on {}", listening);
    }
```

#### 关键模块

##### HttpService

```rust
// file: query/src/api/http_service.rs
pub struct HttpService {
    sessions: Arc<SessionManager>,
    shutdown_handler: HttpShutdownHandler,
}
// func: build_router() -> impl Endpoint
    fn build_router(&self) -> impl Endpoint {
        #[cfg_attr(not(feature = "memory-profiling"), allow(unused_mut))]
        let mut route = Route::new()
            .at("/v1/health", get(super::http::v1::health::health_handler))
            .at("/v1/config", get(super::http::v1::config::config_handler))
            .at("/v1/logs", get(super::http::v1::logs::logs_handler))
            .at("/v1/status", get(super::http::v1::status::status_handler))
            .at(
                "/v1/cluster/list",
                get(super::http::v1::cluster::cluster_list_handler),
            )
            .at(
                "/debug/home",
                get(super::http::debug::home::debug_home_handler),
            )
            .at(
                "/debug/pprof/profile",
                get(super::http::debug::pprof::debug_pprof_handler),
            );

        #[cfg(feature = "memory-profiling")]
        {
            route = route.at(
                // to follow the conversions of jepref, we arrange the path in
                // this way, so that jeprof could be invoked like:
                //   `jeprof ./target/debug/databend-query http://localhost:8080/debug/mem`
                // and jeprof will translate the above url into sth like:
                //    "http://localhost:8080/debug/mem/pprof/profile?seconds=30"
                "/debug/mem/pprof/profile",
                get(super::http::debug::jeprof::debug_jeprof_dump_handler),
            );
        };
        route.data(self.sessions.clone())
    }
```

### 9 RPC API Service

RpcClient会通过此服务启动Stage和获取Stage的输出数据流。

#### Service启动

```rust
// file: query/bin/databend-query.rs
    // RPC API service.
    {
        let address = conf.query.flight_api_address.clone();
        let mut srv = RpcService::create(session_manager.clone());
        let listening = srv.start(address.parse()?).await?;
        shutdown_handle.add_service(srv);
        tracing::info!("RPC API server listening on {}", listening);
    }
```

#### 关键模块

##### RpcService

```rust
// file: query/src/api/rpc_service.rs
pub struct RpcService {
    pub sessions: Arc<SessionManager>, // 会话管理器
    pub abort_notify: Arc<Notify>,
    pub dispatcher: Arc<DatabendQueryFlightDispatcher>, // 数据流调度器，RpcClinet从这里获取查询结果
}
// func: create(...) -> Box<dyn DatabendQueryServer>
// 创建RpcService
    pub fn create(sessions: Arc<SessionManager>) -> Box<dyn DatabendQueryServer> {
        Box::new(Self {
            sessions,
            abort_notify: Arc::new(Notify::new()),
            dispatcher: Arc::new(DatabendQueryFlightDispatcher::create()),
        })
    }
// func: start(...)
    async fn start(&mut self, listening: SocketAddr) -> Result<SocketAddr> {
        let (listener_stream, listener_addr) = Self::listener_tcp(listening).await?;
        self.start_with_incoming(listener_stream).await?;
        Ok(listener_addr)
    }
// func: start_with_incoming(...)
    pub async fn start_with_incoming(&mut self, listener_stream: TcpListenerStream) -> Result<()> {
        let sessions = self.sessions.clone();
        let flight_dispatcher = self.dispatcher.clone();
        let flight_api_service = DatabendQueryFlightService::create(flight_dispatcher, sessions); // 创建DatabendQueryFlightService
        let conf = self.sessions.get_conf();
        let builder = Server::builder();
        let mut builder = if conf.tls_rpc_server_enabled() {
            tracing::info!("databend query tls rpc enabled");
            builder
                .tls_config(Self::server_tls_config(&conf).await.map_err(|e| {
                    ErrorCode::TLSConfigurationFailure(format!(
                        "failed to load server tls config: {e}",
                    ))
                })?)
                .map_err(|e| {
                    ErrorCode::TLSConfigurationFailure(format!("failed to invoke tls_config: {e}",))
                })?
        } else {
            builder
        };

        let server = builder
            .add_service(FlightServiceServer::new(flight_api_service)) // 创建FlightServiceServer
            .serve_with_incoming_shutdown(listener_stream, self.shutdown_notify());

        common_base::base::tokio::spawn(server); // 启动FlightServiceServer
        Ok(())
    }
```

##### DatabendQueryFlightService

```rust
// file: query/src/api/rpc/flight_service.rs
pub struct DatabendQueryFlightService {
    sessions: Arc<SessionManager>,
    dispatcher: Arc<DatabendQueryFlightDispatcher>,
}
// func: do_action() 
// 启动查询计划
    async fn do_action(&self, request: Request<Action>) -> Response<Self::DoActionStream> {
        common_tracing::extract_remote_span_as_parent(&request);

        let action = request.into_inner();
        let flight_action: FlightAction = action.try_into()?;

        let action_result = match &flight_action {
            FlightAction::CancelAction(action) => {
                // We only destroy when session is exist
                let session_id = action.query_id.clone();
                if let Some(session) = self.sessions.get_session_by_id(&session_id).await {
                    // TODO: remove streams
                    session.force_kill_session();
                }

                FlightResult { body: vec![] }
            }
            FlightAction::BroadcastAction(action) => {
                let session_id = action.query_id.clone();
                let is_aborted = self.dispatcher.is_aborted();
                let session = self
                    .sessions
                    .create_rpc_session(session_id, is_aborted)
                    .await?;

                self.dispatcher
                    .broadcast_action(session, flight_action)
                    .await?;
                FlightResult { body: vec![] }
            }
            FlightAction::PrepareShuffleAction(action) => {
                let session_id = action.query_id.clone();
                let is_aborted = self.dispatcher.is_aborted();
                let session = self
                    .sessions
                    .create_rpc_session(session_id, is_aborted)
                    .await?;

                self.dispatcher
                    .shuffle_action(session, flight_action)
                    .await?;
                FlightResult { body: vec![] }
            }
        };

        // let action_result = do_flight_action.await?;
        Ok(RawResponse::new(
            Box::pin(tokio_stream::once(Ok(action_result))) as FlightStream<FlightResult>,
        ))
    }
// func: do_get()
// 获取查询结果数据流
    async fn do_get(&self, request: Request<Ticket>) -> Response<Self::DoGetStream> {
        common_tracing::extract_remote_span_as_parent(&request);
        let ticket: FlightTicket = request.into_inner().try_into()?;

        match ticket {
            FlightTicket::StreamTicket(steam_ticket) => {
                let (receiver, data_schema) = self.dispatcher.get_stream(&steam_ticket)?;
                let arrow_schema = data_schema.to_arrow();
                let ipc_fields = default_ipc_fields(&arrow_schema.fields);

                serialize_schema(&arrow_schema, Some(&ipc_fields));

                Ok(RawResponse::new(
                    Box::pin(FlightDataStream::create(receiver, ipc_fields))
                        as FlightStream<FlightData>,
                ))
            }
        }
    }
```

##### DatabendQueryFlightDispatcher ⭐️

```rust
// file: query/src/api/rpc/flight_dispatcher.rs
struct StreamInfo {
    #[allow(unused)]
    schema: DataSchemaRef,
    tx: mpsc::Sender<Result<DataBlock>>,
    rx: mpsc::Receiver<Result<DataBlock>>,
}

pub struct DatabendQueryFlightDispatcher {
    streams: Arc<RwLock<HashMap<String, StreamInfo>>>, // key = "query_id/stage_id/sink"
    stages_notify: Arc<RwLock<HashMap<String, Arc<Notify>>>>,
    abort: Arc<AtomicBool>,
}
// func: shuffle_action()
// 执行FlightAction::PrepareShuffleAction
    pub async fn shuffle_action(&self, session: SessionRef, action: FlightAction) -> Result<()> {
        let query_id = action.get_query_id();
        let stage_id = action.get_stage_id();
        let action_sinks = action.get_sinks();
        let data_schema = action.get_plan().schema();
        self.create_stage_streams(&query_id, &stage_id, &data_schema, &action_sinks);

        match action.get_sinks().len() {
            0 => Err(ErrorCode::LogicalError("")),
            1 => self.one_sink_action(session, &action).await, // 只有一个sink
            _ => {
                self.action_with_scatter::<HashFlightScatter>(session, &action) // 有多个sinks
                    .await
            }
        }
    }
// func: create_stage_streams()
// 记录Stream
    fn create_stage_streams(
        &self,
        query_id: &str,
        stage_id: &str,
        schema: &DataSchemaRef,
        streams_name: &[String],
    ) {
        let stage_name = format!("{}/{}", query_id, stage_id);
        self.stages_notify
            .write()
            .insert(stage_name.clone(), Arc::new(Notify::new()));

        let mut streams = self.streams.write();

        for stream_name in streams_name {
            let (tx, rx) = mpsc::channel(5); // 创建mpsc
            let stream_name = format!("{}/{}", stage_name, stream_name);

            streams.insert(stream_name, StreamInfo {
                schema: schema.clone(),
                tx,
                rx,
            });
        }
    }
// func: one_sink_action()
// 处理只有一个sink的情况
    async fn one_sink_action(&self, session: SessionRef, action: &FlightAction) -> Result<()> {
        let query_context = session.create_query_context().await?;
        let action_context = QueryContext::create_from(query_context.clone());
        let pipeline_builder = PipelineBuilder::create(action_context.clone());

        let query_plan = action.get_plan();
        action_context.attach_query_plan(&query_plan);
        let mut pipeline = pipeline_builder.build(&query_plan)?;

        let action_sinks = action.get_sinks();
        let action_query_id = action.get_query_id();
        let action_stage_id = action.get_stage_id();

        assert_eq!(action_sinks.len(), 1);
        let stage_name = format!("{}/{}", action_query_id, action_stage_id);
        let stages_notify = self.stages_notify.clone();

        let stream_name = format!("{}/{}", stage_name, action_sinks[0]); // stream_name = "query_id/stage_id/sink"
        let tx_ref = self.streams.read().get(&stream_name).map(|x| x.tx.clone()); // 获取tx，查询结果数据从这里发出
        let tx = tx_ref.ok_or_else(|| ErrorCode::NotFoundStream("Not found stream"))?;

        query_context.try_spawn(
            async move {
                let _session = session;
                wait_start(stage_name, stages_notify).await;

                match pipeline.execute().await { // 执行pipeline
                    Err(error) => {
                        tx.send(Err(error)).await.ok();
                    }
                    Ok(mut abortable_stream) => {
                        while let Some(item) = abortable_stream.next().await {
                            if let Err(error) = tx.send(item).await { // 发送结果到mpsc
                                tracing::error!(
                                    "Cannot push data when run_action_without_scatters. {}",
                                    error
                                );
                                break;
                            }
                        }
                    }
                };
            }
            .instrument(Span::current()),
        )?;
        Ok(())
    }
// func: action_with_scatter()
// 处理多个Sinks到情况
    async fn action_with_scatter<T>(
        &self,
        session: SessionRef,
        action: &FlightAction,
    ) -> Result<()>
    where
        T: FlightScatter + Send + 'static,
    {
        let query_context = session.create_query_context().await?;
        let action_context = QueryContext::create_from(query_context.clone());
        let pipeline_builder = PipelineBuilder::create(action_context.clone());

        let query_plan = action.get_plan();
        action_context.attach_query_plan(&query_plan);
        let mut pipeline = pipeline_builder.build(&query_plan)?; // 创建pipeline

        let action_query_id = action.get_query_id();
        let action_stage_id = action.get_stage_id();

        let sinks_tx = {
            let action_sinks = action.get_sinks();

            assert!(action_sinks.len() > 1);
            let mut sinks_tx = Vec::with_capacity(action_sinks.len());

            for sink in &action_sinks { // 遍历sinks
                let stream_name = format!("{}/{}/{}", action_query_id, action_stage_id, sink);
                match self.streams.read().get(&stream_name) {
                    Some(stream) => sinks_tx.push(stream.tx.clone()),
                    None => {
                        return Err(ErrorCode::NotFoundStream(format!(
                            "Not found stream {}",
                            stream_name
                        )))
                    }
                }
            }

            Result::Ok(sinks_tx)
        }?;

        let stage_name = format!("{}/{}", action_query_id, action_stage_id);
        let stages_notify = self.stages_notify.clone();

        let flight_scatter = T::try_create(
            query_context.clone(),
            action.get_plan().schema(),
            action.get_scatter_expression(),
            action.get_sinks().len(),
        )?;

        query_context.try_spawn(
            async move {
                let _session = session;
                wait_start(stage_name, stages_notify).await;

                let sinks_tx_ref = &sinks_tx;
                let forward_blocks = async move {
                    let mut abortable_stream = pipeline.execute().await?; // 执行
                    while let Some(item) = abortable_stream.next().await {
                        let forward_blocks = flight_scatter.execute(&item?)?; // 散射

                        assert_eq!(forward_blocks.len(), sinks_tx_ref.len());

                        for (index, forward_block) in forward_blocks.iter().enumerate() {
                            let tx: &Sender<Result<DataBlock>> = &sinks_tx_ref[index];
                            tx.send(Ok(forward_block.clone())) // 把数据发送到sink对应的tx
                                .await
                                .map_err_to_code(ErrorCode::LogicalError, || {
                                    "Cannot push data when run_action"
                                })?;
                        }
                    }

                    Result::Ok(())
                };

                if let Err(error) = forward_blocks.await {
                    for tx in &sinks_tx {
                        if !tx.is_closed() {
                            let send_error_message = tx.send(Err(error.clone()));
                            let _ignore_send_error = send_error_message.await;
                        }
                    }
                }
            }
            .instrument(Span::current()),
        )?;

        Ok(())
    }
```

##### FlightScatter

```rust
// file: query/src/api/rpc/flight_scatter.rs
pub trait FlightScatter: Sized {
    fn try_create(
        ctx: Arc<QueryContext>,
        schema: DataSchemaRef,
        expr: Option<Expression>,
        num: usize,
    ) -> Result<Self>;

    fn execute(&self, data_block: &DataBlock) -> Result<Vec<DataBlock>>;
}
```

##### HashFligthScatter

```rust
// file: query/src/api/rpc/flight_scatter_hash.rs
pub struct HashFlightScatter {
    scatter_expression_executor: Arc<ExpressionExecutor>,
    scatter_expression_name: String,
    scattered_size: usize,
}

// func: 
// 把输入的data_block Hash到多个sinks
    fn execute(&self, data_block: &DataBlock) -> common_exception::Result<Vec<DataBlock>> {
        let expression_executor = self.scatter_expression_executor.clone();
        let evaluated_data_block = expression_executor.execute(data_block)?;
        let indices = evaluated_data_block.try_column_by_name(&self.scatter_expression_name)?;

        let col: &PrimitiveColumn<u64> = Series::check_get(indices)?;
        let indices: Vec<usize> = col.iter().map(|c| *c as usize).collect();
        DataBlock::scatter_block(data_block, &indices, self.scattered_size)
    }
```

```rust
// file: common/datablocks/src/kernels/data_block_scatter.rs
// 散列DataBlock
    pub fn scatter_block(
        block: &DataBlock,
        indices: &[usize],
        scatter_size: usize,
    ) -> Result<Vec<DataBlock>> {
        let columns_size = block.num_columns();
        let mut scattered_columns = Vec::with_capacity(scatter_size);

        for column_index in 0..columns_size { // 先把每一列打散
            let column = block.column(column_index).scatter(indices, scatter_size);
            scattered_columns.push(column);
        }

        let mut scattered_blocks = Vec::with_capacity(scatter_size);
        for index in 0..scatter_size { // 把属于同一个sink的所有列组装成DataBlock
            let mut block_columns = vec![];

            for item in scattered_columns.iter() {
                block_columns.push(item[index].clone())
            }
            scattered_blocks.push(DataBlock::create(block.schema().clone(), block_columns));
        }

        Ok(scattered_blocks)
    }
```

##### BroadcastFlightScatter

```rust
// file: query/src/api/rpc/flight_scatter_broadcast.rs
pub struct BroadcastFlightScatter {
    scattered_size: usize,
}
// func: execute(...)
// 广播数据，把输入的data_block广播到下游多个sink
    fn execute(&self, data_block: &DataBlock) -> Result<Vec<DataBlock>> {
        let mut data_blocks = vec![];
        for _ in 0..self.scattered_size {
            data_blocks.push(data_block.clone());
        }

        Ok(data_blocks)
    }
```

### 10 Cluster Register

#### Woker节点注册

```rust
// file: query/bin/databend-query.rs
// func: main()
// Cluster register.
    {
        let cluster_discovery = session_manager.get_cluster_discovery(); // 从SessionManager获取ClusterDiscovery
        let register_to_metastore = cluster_discovery.register_to_metastore(&conf); // 将本地节点注册到meta服务
        register_to_metastore.await?;
        // ...
    }
```

#### 关键模块

##### SessionManager

```rust
// file: query/src/sessions/session_mgr.rs
/// SessisionManager管理模块，这里只列出集群管理相关的代码
pub struct SessionManager {
    ...
    pub(in crate::sessions) discovery: RwLock<Arc<ClusterDiscovery>>,
    ...
}
// func: from_conf(...) -> Result<Arc<SessionManager>>
// 创建ClusterDiscovery
    let discovery = ClusterDiscovery::create_global(conf.clone()).await?; // 初始化ClusterDiscovery
    SessionManager {
        ...
        discovery: RwLock::new(discovery),
    }
// func: get_cluster_discovery()
// 获取ClusterDiscovery
    pub fn get_cluster_discovery(self: &Arc<Self>) -> Arc<ClusterDiscovery> {
        self.discovery.read().clone() // 获取读锁
    }
```

##### ClusterDiscovery/ClusterHeartbeat/Cluster/ClusterMgr

```rust
// file: query/src/clusters/cluster.rs
/// 服务发现模块，负责维护本地节点的ID和与meta服务通信，包括当前节点的心跳、注册节点、删除节点、获取集群信息
pub struct ClusterDiscovery {
    local_id: String, // 当前Woker节点的UUID
    heartbeat: Mutex<ClusterHeartbeat>, // 心跳服务，用于worker与meta service间的心跳
    api_provider: Arc<dyn ClusterApi>, // trait object, 可以是任意实现ClusterApi trait的类，目前只有ClusterApi struct
}
// func: register_to_metastore(...)
// 注册当前节点到meta服务
    pub async fn register_to_metastore(self: &Arc<Self>, cfg: &Config) -> Result<()> {
        let cpus = cfg.query.num_cpus;
        // TODO: 127.0.0.1 || ::0
        let address = cfg.query.flight_api_address.clone();
        let node_info = NodeInfo::create(self.local_id.clone(), cpus, address); // 创建节点信息

        self.drop_invalid_nodes(&node_info).await?; // 删除当前节点过期的信息
        match self.api_provider.add_node(node_info.clone()).await { // 通过ClusterMgr注册当前节点信息
            Ok(_) => self.start_heartbeat(node_info).await, // 如果注册成功，则开启心跳
            Err(cause) => Err(cause.add_message_back("(while cluster api add_node).")),
        }
    }


/// 心跳模块
struct ClusterHeartbeat {
    timeout: Duration,
    shutdown: Arc<AtomicBool>,
    shutdown_notify: Arc<Notify>,
    cluster_api: Arc<dyn ClusterApi>,
    shutdown_handler: Option<JoinHandle<()>>,
}
// func: start(...)
// 启动心跳loop
    pub fn start(&mut self, node_info: NodeInfo) {
        self.shutdown_handler = Some(tokio::spawn(self.heartbeat_loop(node_info))); // 放到一个线程中loop
    }

/// 集群模块，表示了一个集群
pub struct Cluster {
    local_id: String, // 当前节点的ID
    nodes: Vec<Arc<NodeInfo>>, // 集群内所有节点信息
}
// func: create_node_conn(&self, name: &str, config: &Config) -> Result<FlightClient>
// 创建当前节点访问name节点的Client
    for node in &self.nodes {
        if node.id == name {
            return match config.tls_query_cli_enabled() {
                true => // 创建tls client
                false => // 创建普通client
            }
        }
    }
```

```rust
// file: common/management/src/cluster/cluster_mgr.rs
/// 集群管理模块，实现了ClusterApi trait，与meta服务通信，包括心跳、添加、获取、删除节点信息
pub struct ClusterMgr {
    kv_api: Arc<dyn KVApi>, // KVDB
    lift_time: Duration, // 生命周期
    cluster_prefix: String,
}

impl ClusterApi for ClusterMgr {
    async fn add_node(&self, node: NodeInfo) -> Result<u64> {...}
    async fn get_nodes(&self) -> Result<Vec<NodeInfo>> {...}
    async fn drop_node(&self, node_id: String, seq: Option<u64>) -> Result<()> {...}
    async fn heartbeat(&self, node: &NodeInfo, seq: Option<u64>) -> Result<u64> {...}
}
```
