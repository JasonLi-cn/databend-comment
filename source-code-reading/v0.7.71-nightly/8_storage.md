* [八 Storage](#八-storage)
  * [Init Storeage and Table](#init-storeage-and-table)
    * [GlobalServices](#globalservices)
    * [Operator](#operator)
    * [CatalogManager](#catalogmanager)
    * [CatalogManagerHelper](#catalogmanagerhelper)
    * [DatabaseCatalog](#databasecatalog)
    * [MutableCatalog](#mutablecatalog)
    * [StorageFactory](#storagefactory)
    * [FuseTable](#fusetable)
  * [Create Table](#create-table)
    * [Binder](#binder)
    * [PhysicalPlanBuilder](#physicalplanbuilder)
    * [PipelineBuilder](#pipelinebuilder)
    * [QueryContext](#querycontext)
    * [DatabaseCatalog](#databasecatalog)
    * [MutableCatalog](#mutablecatalog)
    * [StorageFactory](#storagefactory)
    * [Table trait](#table-trait)
    * [FuseTable](#fusetable)
    * [FuseTableSource](#fusetablesource)
  * [Reference](#reference)

## 八 Storage

### Init Storeage and Table

```rust
// file: src/binaries/query/main.rs
GlobalServices::init(conf.clone()).await?;
```

#### GlobalServices

```rust
// src/query/service/src/global_services.rs
    pub async fn init(config: Config) -> Result<()> {
        let global_services = Arc::new(GlobalServices {
            ...
            storage_operator: UnsafeCell::new(None),
            ...
        });
        // 初始化DataOperator，即Storage
        DataOperator::init(&config.storage, global_services.clone()).await?;
        // 初始化CatalogManager，即Catalog，依赖Storage
        CatalogManager::init(&config, global_services.clone()).await?;
    }

impl SingletonImpl<DataOperator> for GlobalServices {
    fn get(&self) -> DataOperator {
        unsafe {
            match &*self.storage_operator.get() {
                None => panic!("StorageOperator is not init"),
                Some(storage_operator) => storage_operator.clone(),
            }
        }
    }

    fn init(&self, value: DataOperator) -> Result<()> {
        unsafe {
            *(self.storage_operator.get() as *mut Option<DataOperator>) = Some(value);
            Ok(())
        }
    }
}
```

#### Operator

```rust
// src/common/storage/src/operator.rs

// 单例
static DATA_OPERATOR: OnceCell<Singleton<DataOperator>> = OnceCell::new();

    // 初始化单例
    pub async fn init(
        conf: &StorageConfig,
        v: Singleton<DataOperator>,
    ) -> common_exception::Result<()> {
        v.init(Self::try_create(conf).await?)?;

        DATA_OPERATOR.set(v).ok();
        Ok(())
    }

    // 获取单例
    pub fn instance() -> DataOperator {
        match DATA_OPERATOR.get() {
            None => panic!("StorageOperator is not init"),
            Some(storage_operator) => storage_operator.get(),
        }
    }

    pub async fn try_create(conf: &StorageConfig) -> common_exception::Result<DataOperator> {
        Self::try_create_with_storage_params(&conf.params).await
    }

    pub async fn try_create_with_storage_params(
        sp: &StorageParams,
    ) -> common_exception::Result<DataOperator> {
        let operator = init_operator(sp)?; // 初始化operator

        // OpenDAL will send a real request to underlying storage to check whether it works or not.
        // If this check failed, it's highly possible that the users have configured it wrongly.
        //
        // Make sure the check is called inside GlobalIORuntime to prevent
        // IO hang on reuse connection.
        let op = operator.clone();
        if let Err(cause) = GlobalIORuntime::instance()
            .spawn(async move { op.check().await })
            .await
            .expect("join must succeed")
        {
            return Err(ErrorCode::StorageUnavailable(format!(
                "current configured storage is not available: config: {:?}, cause: {cause}",
                sp
            )));
        }

        Ok(DataOperator {
            operator,
            params: sp.clone(),
        })
    }

/// init_operator will init an opendal operator based on storage config.
pub fn init_operator(cfg: &StorageParams) -> Result<Operator> {
    let op = match &cfg {
        StorageParams::Azblob(cfg) => init_azblob_operator(cfg)?,
        StorageParams::Fs(cfg) => init_fs_operator(cfg)?,
        StorageParams::Ftp(cfg) => init_ftp_operator(cfg)?,
        StorageParams::Gcs(cfg) => init_gcs_operator(cfg)?,
        #[cfg(feature = "storage-hdfs")]
        StorageParams::Hdfs(cfg) => init_hdfs_operator(cfg)?,
        StorageParams::Http(cfg) => init_http_operator(cfg)?,
        StorageParams::Ipfs(cfg) => init_ipfs_operator(cfg)?,
        StorageParams::Memory => init_memory_operator()?,
        StorageParams::Moka(cfg) => init_moka_operator(cfg)?,
        StorageParams::Obs(cfg) => init_obs_operator(cfg)?,
        StorageParams::S3(cfg) => init_s3_operator(cfg)?, // 初始化s3
        StorageParams::Oss(cfg) => init_oss_operator(cfg)?,
    };

    let op = op
        // Add retry
        .layer(RetryLayer::new(ExponentialBackoff::default().with_jitter()))
        // Add metrics
        .layer(MetricsLayer)
        // Add logging
        .layer(LoggingLayer)
        // Add tracing
        .layer(TracingLayer)
        // NOTE
        //
        // Magic happens here. We will add a layer upon original
        // storage operator so that all underlying storage operations
        // will send to storage runtime.
        .layer(RuntimeLayer::new(GlobalIORuntime::instance().inner()));

    Ok(op)
}

/// init_s3_operator will init a opendal s3 operator with input s3 config.
fn init_s3_operator(cfg: &StorageS3Config) -> Result<Operator> {
    let mut builder = s3::Builder::default();

    // Endpoint.
    builder.endpoint(&cfg.endpoint_url);

    // Region
    builder.region(&cfg.region);

    // Credential.
    builder.access_key_id(&cfg.access_key_id);
    builder.secret_access_key(&cfg.secret_access_key);
    builder.security_token(&cfg.security_token);
    builder.role_arn(&cfg.role_arn);
    builder.external_id(&cfg.external_id);

    // Bucket.
    builder.bucket(&cfg.bucket);

    // Root.
    builder.root(&cfg.root);

    // Disable credential loader
    if cfg.disable_credential_loader {
        builder.disable_credential_loader();
    }

    // Enable virtual host style
    if cfg.enable_virtual_host_style {
        builder.enable_virtual_host_style();
    }

    Ok(Operator::new(builder.build()?))
}
```

#### CatalogManager

```rust
// file: src/query/catalog/src/catalog.rs
static CATALOG_MANAGER: OnceCell<Singleton<Arc<CatalogManager>>> = OnceCell::new(); // 全局单例

pub struct CatalogManager {
    pub catalogs: HashMap<String, Arc<dyn Catalog>>,
}

// file: src/query/service/src/catalogs/catalog_manager.rs
#[async_trait::async_trait]
impl CatalogManagerHelper for CatalogManager {
    async fn init(conf: &Config, v: Singleton<Arc<CatalogManager>>) -> Result<()> {
        v.init(Self::try_create(conf).await?)?;
        CatalogManager::set_instance(v);
        Ok(())
    }

    async fn try_create(conf: &Config) -> Result<Arc<CatalogManager>> {
        let mut catalog_manager = CatalogManager {
            catalogs: HashMap::new(),
        };

        catalog_manager.register_build_in_catalogs(conf).await?;

        ...

        Ok(Arc::new(catalog_manager))
    }

    async fn register_build_in_catalogs(&mut self, conf: &Config) -> Result<()> {
        let default_catalog: Arc<dyn Catalog> =
            Arc::new(DatabaseCatalog::try_create_with_config(conf.clone()).await?); // 创建DatabaseCatalog
        self.catalogs
            .insert(CATALOG_DEFAULT.to_owned(), default_catalog);
        Ok(())
    }
}
```

#### CatalogManagerHelper

```rust
// file: src/query/service/src/catalogs/catalog_manager.rs
pub trait CatalogManagerHelper {
    async fn init(conf: &Config, v: Singleton<Arc<CatalogManager>>) -> Result<()>;

    async fn try_create(conf: &Config) -> Result<Arc<CatalogManager>>;

    async fn register_build_in_catalogs(&mut self, conf: &Config) -> Result<()>;

    #[cfg(feature = "hive")]
    fn register_external_catalogs(&mut self, conf: &Config) -> Result<()>;
}
```

#### DatabaseCatalog

```rust
// file: src/query/service/src/catalogs/default/database_catalog.rs
pub struct DatabaseCatalog {
    /// the upper layer, read only
    immutable_catalog: Arc<dyn Catalog>,
    /// bottom layer, writing goes here
    mutable_catalog: Arc<dyn Catalog>,
    /// table function engine factories
    table_function_factory: Arc<TableFunctionFactory>,
}

    pub async fn try_create_with_config(conf: Config) -> Result<DatabaseCatalog> {
        let immutable_catalog = ImmutableCatalog::try_create_with_config(&conf).await?;
        let mutable_catalog = MutableCatalog::try_create_with_config(conf).await?; // 创建MutableCatalog
        let table_function_factory = TableFunctionFactory::create();
        let res = DatabaseCatalog::create(
            Arc::new(immutable_catalog),
            Arc::new(mutable_catalog),
            Arc::new(table_function_factory),
        );
        Ok(res)
    }
```

#### MutableCatalog

```rust
// file: src/query/service/src/catalogs/default/mutable_catalog.rs
pub struct MutableCatalog {
    ctx: CatalogContext,
}

    pub async fn try_create_with_config(conf: Config) -> Result<Self> {
        ...
        // Storage factory.
            let storage_factory = StorageFactory::create(conf.clone());
        // Database factory.
            let database_factory = DatabaseFactory::create(conf.clone());
        ...
    }
```

#### StorageFactory

```rust
// file: src/query/storages/factory/src/storage_factory.rs
pub struct StorageFactory {
    storages: RwLock<HashMap<String, Storage>>,
}

    pub fn create(conf: Config) -> Self {
        ...
        // Register FUSE table engine.
        creators.insert("FUSE".to_string(), Storage {
            creator: Arc::new(FuseTable::try_create), // 创建Fuze Creator
            descriptor: Arc::new(FuseTable::description),
        });
        ...
    }
```

#### FuseTable

```rust
// file: src/query/storages/fuse/src/fuse_table.rs
pub struct FuseTable {
    pub(crate) table_info: TableInfo,
    pub(crate) meta_location_generator: TableMetaLocationGenerator,

    pub(crate) cluster_keys: Vec<LegacyExpression>,
    pub(crate) cluster_key_meta: Option<ClusterKey>,
    pub(crate) read_only: bool,

    pub(crate) operator: Operator,
    pub(crate) data_metrics: Arc<StorageMetrics>,
}

    // 创建FuseTable，该函数会被注册到StorageFactory中
    pub fn try_create(table_info: TableInfo) -> Result<Box<dyn Table>> {
        let r = Self::do_create(table_info, false)?;
        Ok(r)
    }

    fn init_operator(table_info: &TableInfo) -> Result<Operator> {
        let operator = match table_info.from_share {
            Some(ref from_share) => create_share_table_operator(
                ShareTableConfig::share_endpoint_address(),
                &table_info.tenant,
                &from_share.tenant,
                &from_share.share_name,
                &table_info.name,
            ),
            None => {
                let storage_params = table_info.meta.storage_params.clone();
                match storage_params {
                    Some(sp) => init_operator(&sp)?,
                    None => {
                        let op = &*(DataOperator::instance()); // 获取DataOperator单例
                        op.clone()
                    }
                }
            }
        };
        Ok(operator)
    }

    pub fn do_create(table_info: TableInfo, read_only: bool) -> Result<Box<FuseTable>> {
        let operator = Self::init_operator(&table_info)?; // 获取DataOperator单例
        Self::do_create_with_operator(table_info, operator, read_only)
    }

    pub fn do_create_with_operator(
        table_info: TableInfo,
        operator: Operator,
        read_only: bool,
    ) -> Result<Box<FuseTable>> {
        let storage_prefix = Self::parse_storage_prefix(&table_info)?;
        let cluster_key_meta = table_info.meta.cluster_key();
        let mut cluster_keys = Vec::new();
        if let Some((_, order)) = &cluster_key_meta {
            cluster_keys = ExpressionParser::parse_exprs(order)?;
        }
        let data_metrics = Arc::new(StorageMetrics::default());
        let operator = operator.layer(StorageMetricsLayer::new(data_metrics.clone())); // Create a new layer.
        Ok(Box::new(FuseTable {
            table_info,
            cluster_keys,
            cluster_key_meta,
            meta_location_generator: TableMetaLocationGenerator::with_prefix(storage_prefix),
            read_only,
            operator,
            data_metrics,
        }))
    }
```

### Create Table

#### Binder

```rust
// file: src/query/sql/src/planner/binder/select.rs
pub(super) async fn bind_select_stmt() {
        let (mut s_expr, mut from_context) = if stmt.from.is_empty() {
            self.bind_one_table(bind_context, stmt).await?
        } else {
            ...
            self.bind_table_reference(bind_context, &cross_joins)
        }
}
```

```rust
// file: src/query/sql/src/planner/binder/table.rs
    pub(super) async fn bind_table_reference(
        &mut self,
        bind_context: &BindContext,
        table_ref: &TableReference<'a>,
    ) -> Result<(SExpr, BindContext)> {
                // Resolve table with catalog
                let table_meta: Arc<dyn Table> = self
                    .resolve_data_source(
                        tenant.as_str(),
                        catalog.as_str(),
                        database.as_str(),
                        table_name.as_str(),
                        &navigation_point,
                    )
                    .await?;
                match table_meta.engine() {
                    "VIEW" => {
                    }
                    _ => {
                        let table_index =
                            self.metadata
                                .write()
                                .add_table(catalog, database.clone(), table_meta); // 把Table放入Meta，这里会在创建PhysicalPlan时使用

                        let (s_expr, mut bind_context) =
                            self.bind_base_table(bind_context, database.as_str(), table_index)?;
                        if let Some(alias) = alias {
                            bind_context.apply_table_alias(alias, &self.name_resolution_ctx)?;
                        }
                    }
                }

    }

    async fn resolve_data_source(
        &self,
        tenant: &str,
        catalog_name: &str,
        database_name: &str,
        table_name: &str,
        travel_point: &Option<NavigationPoint>,
    ) -> Result<Arc<dyn Table>> {
        // Resolve table with catalog
        let catalog = self.catalogs.get_catalog(catalog_name)?;
        let mut table_meta = catalog.get_table(tenant, database_name, table_name).await?; // 获取Table

        if let Some(tp) = travel_point {
            table_meta = table_meta.navigate_to(tp).await?;
        }
        Ok(table_meta)
    }
```

#### PhysicalPlanBuilder

```rust
// file: src/query/sql/src/executor/physical_plan_builder.rs

    pub async fn build(&self, s_expr: &SExpr) -> Result<PhysicalPlan> {
        debug_assert!(check_physical(s_expr));

        match s_expr.plan() {
            RelOperator::PhysicalScan(scan) => {
                ...
                let table_entry = metadata.table(scan.table_index); // 从meta中获取table
                let table = table_entry.table();
                let table_schema = table.schema();

                let push_downs = self.push_downs(scan, &table_schema, has_inner_column)?;

                let source = table
                    .read_plan_with_catalog(
                        self.ctx.clone(),
                        table_entry.catalog().to_string(),
                        Some(push_downs),
                    )
                    .await?; // 创建ReadDataSourcePlan，这里会在创建Pipeline时使用
                Ok(PhysicalPlan::TableScan(TableScan {
                    name_mapping,
                    source: Box::new(source),
                    table_index: scan.table_index,
                }))
            }
        }
    }
```

#### PipelineBuilder

```rust
// file: src/query/service/src/pipelines/pipeline_builder.rs
pub struct PipelineBuilder {
    ctx: Arc<QueryContext>,
    main_pipeline: Pipeline,
    pub pipelines: Vec<Pipeline>,
}

    fn build_table_scan(&mut self, scan: &TableScan) -> Result<()> {
        let table = self.ctx.build_table_from_source_plan(&scan.source)?; // 创建Table，scan.source是在PhysicalPlanBuilder中创建的
        self.ctx.try_set_partitions(scan.source.parts.clone())?; // set partition
        table.read_data(self.ctx.clone(), &scan.source, &mut self.main_pipeline)?; // 读取数据，详见下节
        let schema = scan.source.schema();
        let projections = scan
            .name_mapping
            .iter()
            .map(|(name, _)| schema.index_of(name.as_str()))
            .collect::<Result<Vec<usize>>>()?;

        let func_ctx = self.ctx.try_get_function_context()?;
        self.main_pipeline.add_transform(|input, output| {
            Ok(CompoundChunkOperator::create(
                input,
                output,
                func_ctx.clone(),
                vec![ChunkOperator::Project {
                    offsets: projections.clone(),
                }],
            ))
        })?;

        self.main_pipeline.add_transform(|input, output| {
            Ok(CompoundChunkOperator::create(
                input,
                output,
                func_ctx.clone(),
                vec![ChunkOperator::Rename {
                    output_schema: scan.output_schema()?,
                }],
            ))
        })
    }
```

#### QueryContext

```rust
// file: src/query/service/src/sessions/query_ctx.rs
impl TableContext for QueryContext {
    /// Build a table instance the plan wants to operate on.
    ///
    /// A plan just contains raw information about a table or table function.
    /// This method builds a `dyn Table`, which provides table specific io methods the plan needs.
    fn build_table_from_source_plan(&self, plan: &ReadDataSourcePlan) -> Result<Arc<dyn Table>> {
        match &plan.source_info {
            SourceInfo::TableSource(table_info) => {
                self.build_table_by_table_info(&plan.catalog, table_info, plan.tbl_args.clone()) // 创建Table
            }
            SourceInfo::StageSource(stage_info) => {
                self.build_external_by_table_info(&plan.catalog, stage_info, plan.tbl_args.clone())
            }
        }
    }
}

    // Build fuse/system normal table by table info.
    fn build_table_by_table_info(
        &self,
        catalog_name: &str,
        table_info: &TableInfo,
        table_args: Option<Vec<LegacyExpression>>,
    ) -> Result<Arc<dyn Table>> {
        let catalog = self.get_catalog(catalog_name)?;
        if table_args.is_none() {
            catalog.get_table_by_info(table_info) // case1，
        } else {
            Ok(catalog
                .get_table_function(&table_info.name, table_args)? // case2，内置的table，e.g. numbers, numbers_mt, fuse_segment等，详见TableFunctionFactory
                .as_table())
        }
    }
```

#### DatabaseCatalog

```rust
// file: src/query/service/src/catalogs/default/database_catalog.rs
    fn get_table_by_info(&self, table_info: &TableInfo) -> Result<Arc<dyn Table>> {
        let res = self.immutable_catalog.get_table_by_info(table_info); // 先从immutable_catalog中获取
        match res {
            Ok(t) => Ok(t),
            Err(e) => {
                if e.code() == ErrorCode::unknown_table_code() {
                    self.mutable_catalog.get_table_by_info(table_info) // 再从mutable_catalog中获取
                } else {
                    Err(e)
                }
            }
        }
    }
```

#### MutableCatalog

```rust
// file: src/query/service/src/catalogs/default/mutable_catalog.rs

    async fn get_table(
        &self,
        tenant: &str,
        db_name: &str,
        table_name: &str,
    ) -> Result<Arc<dyn Table>> {
        let db = self.get_database(tenant, db_name).await?;
        db.get_table(table_name).await
    }

    fn get_table_by_info(&self, table_info: &TableInfo) -> Result<Arc<dyn Table>> {
        let storage = self.ctx.storage_factory.clone(); // 获取storage
        storage.get_table(table_info) // 获取Table
    }
```

#### StorageFactory

```rust
// file: src/query/storages/factory/src/storage_factory.rs
    pub fn get_table(&self, table_info: &TableInfo) -> Result<Arc<dyn Table>> {
        let engine = table_info.engine().to_uppercase();
        let lock = self.storages.read();
        let factory = lock.get(&engine).ok_or_else(|| { // 获取Engine
            ErrorCode::UnknownTableEngine(format!("Unknown table engine {}", engine))
        })?;

        let table: Arc<dyn Table> = factory.creator.try_create(table_info.clone())?.into(); // 创建Table
        Ok(table)
    }
```

#### Table trait

```rust
// file: src/query/catalog/src/table.rs
    /// Gather partitions to be scanned according to the push_downs
    /// 在PhysicalPlanBuilder中被调用，获取要查询的分片
    async fn read_partitions(
        &self,
        ctx: Arc<dyn TableContext>,
        push_downs: Option<Extras>,
    ) -> Result<(Statistics, Partitions)> {
        let (_, _) = (ctx, push_downs);
        Err(ErrorCode::UnImplement(format!(
            "read_partitions operation for table {} is not implemented. table engine : {}",
            self.name(),
            self.get_table_info().meta.engine
        )))
    }

    /// Assembly the pipeline of reading data from storage, according to the plan
    // 在PipelineBuilder中被调用
    fn read_data(
        &self,
        ctx: Arc<dyn TableContext>,
        plan: &ReadDataSourcePlan,
        pipeline: &mut Pipeline,
    ) -> Result<()> {
        let (_, _, _) = (ctx, plan, pipeline);

        Err(ErrorCode::UnImplement(format!(
            "read_data operation for table {} is not implemented. table engine : {}",
            self.name(),
            self.get_table_info().meta.engine
        )))
    }
```

#### FuseTable

```rust
// file: src/query/storages/fuse/src/fuse_table.rs
#[async_trait::async_trait]
impl Table for FuseTable {
    #[tracing::instrument(level = "debug", name = "fuse_table_read_partitions", skip(self, ctx), fields(ctx.id = ctx.get_id().as_str()))]
    async fn read_partitions(
        &self,
        ctx: Arc<dyn TableContext>,
        push_downs: Option<Extras>,
    ) -> Result<(Statistics, Partitions)> {
        self.do_read_partitions(ctx, push_downs).await
    }

    #[tracing::instrument(level = "debug", name = "fuse_table_read_data", skip(self, ctx, pipeline), fields(ctx.id = ctx.get_id().as_str()))]
    fn read_data(
        &self,
        ctx: Arc<dyn TableContext>,
        plan: &ReadDataSourcePlan,
        pipeline: &mut Pipeline,
    ) -> Result<()> {
        let max_io_requests = ctx.get_settings().get_max_storage_io_requests()? as usize;
        self.do_read_data(ctx, plan, pipeline, max_io_requests)
    }
}

impl FuseTable {
    #[inline]
    pub fn do_read_data(
        &self,
        ctx: Arc<dyn TableContext>,
        plan: &ReadDataSourcePlan,
        pipeline: &mut Pipeline,
        max_io_requests: usize,
    ) -> Result<()> {
        ...
        // Add source pipe.
        pipeline.add_source(
            |output| {
                FuseTableSource::create(
                    ctx.clone(),
                    output,
                    block_reader.clone(),
                    prewhere_reader.clone(),
                    prewhere_filter.clone(),
                    remain_reader.clone(),
                )
            },
            max_io_requests,
        )?;

        // Resize pipeline to max threads.
        let max_threads = ctx.get_settings().get_max_threads()? as usize;
        let resize_to = std::cmp::min(max_threads, max_io_requests);
        info!(
            "read block pipeline resize from:{} to:{}",
            max_io_requests, resize_to
        );
        pipeline.resize(resize_to)
    }
}
```

#### FuseTableSource

```rust
// file: src/query/storages/fuse/src/operations/fuse_source.rs
pub struct FuseTableSource {
    state: State,
    ctx: Arc<dyn TableContext>,
    scan_progress: Arc<Progress>,
    output: Arc<OutputPort>,
    output_reader: Arc<BlockReader>,

    prewhere_reader: Arc<BlockReader>,
    prewhere_filter: Arc<Option<ExpressionExecutor>>,
    remain_reader: Arc<Option<BlockReader>>,

    support_blocking: bool,
}

impl Processor for FuseTableSource {
    fn name(&self) -> String {
        "FuseEngineSource".to_string()
    }

    fn event(&mut self) -> Result<Event> {
        ...
    }

    fn process(&mut self) -> Result<()> {
        ...
    }
}
```

### Reference

* [《Databend存储架构总览》](https://mp.weixin.qq.com/s/jXAu3mSmJF80TwK3xeFlcg)
