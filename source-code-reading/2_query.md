## 二 查询流程

### 1 查询示例（Standalone）

```rust
    #[tokio::test(flavor = "multi_thread", worker_threads = 1)]
    async fn run_sql() -> Result<()> {
        common_tracing::init_default_ut_tracing();
        let sql = "SELECT * FROM numbers_mt(10000) where number = 10 limit 10";
        select_executor(sql).await?;
        Ok(())
    }

    pub async fn select_executor(sql: &str) -> Result<()> {
        // 1 创建Config
        let mut config = Config::default();
        config.query.tenant_id = "admin".to_string();
        config.query.num_cpus = 2; // 并行度
  
        // 2 创建Session
        let sessions = SessionManager::from_conf(config).await?;
        let executor_session = sessions.create_session(SessionType::Dummy).await?;
  
        // 3 创建QueryContext
        let ctx = executor_session.create_query_context().await?;

        // 4 词法、语法解析，生成查询计划
        let plan_node = PlanParser::parse(ctx.clone(), sql).await?;

        // 5 创建Interpreter，这里是SelectInterpreter
        let interpreter = InterpreterFactory::get(ctx.clone(), plan_node)?;

        // 6 执行查询
        let mut stream = interpreter.execute(None).await?;
  
        // 7 打印结果
        // 7.1 print pretty results
        // let results = stream.try_collect::<Vec<_>>().await?;
        // let pretty_results = common_datablocks::pretty_format_blocks(&results)?;
        // println!("{}", pretty_results);
        // 7.2 print result stream
        while let Some(Ok(block)) = stream.next().await {
            println!("{:?}", block);
        }
        Ok(())
    }
```

其中，第4、5步，可以选择新的Planner和InterpreterFactoryV2

```rust
// 4 词法、语法解析，生成查询计划
let mut planner = Planner::new(context.clone());
let (plan_node, _) = planner.plan_sql(sql).await?;

// 5 创建Interpreter，这里是SelectInterpreter
let interpreter = InterpreterFactoryV2::get(ctx.clone(), &plan_node)?;
```

### 2 词法、语法分析，生成查询计划

#### V1

```rust
let plan_node = PlanParser::parse(ctx.clone(), sql).await?;
```

##### PlanParser

```rust
// file: query/src/sql/plan_parser.rs
pub struct PlanParser;
// func: parse(ctx: Arc<QueryContext>, query: &str) -> Result<PlanNode>
// 词法、语法解析，生成查询计划
    pub async fn parse(ctx: Arc<QueryContext>, query: &str) -> Result<PlanNode> {
        let (statements, _) = DfParser::parse_sql(query, ctx.get_current_session().get_type())?; // 词法、语法解析
        PlanParser::build_plan(statements, ctx).await // 生成查询计划
    }
// func: build_plan(statements: Vec<DfStatement<'_>>, ctx: Arc<QueryContext>,) -> Result<PlanNode>
// 生成查询计划
    pub async fn build_plan(
        statements: Vec<DfStatement<'_>>,
        ctx: Arc<QueryContext>,
    ) -> Result<PlanNode> {
        // assert_eq!(statements.len(), 1);
        if statements.is_empty() {
            return Ok(PlanNode::Empty(EmptyPlan::create()));
        } else if statements.len() > 1 {
            return Err(ErrorCode::SyntaxException("Only support single query"));
        }

        match statements[0].analyze(ctx.clone()).await? { // 解析
            AnalyzedResult::SimpleQuery(plan) => Ok(*plan),
            AnalyzedResult::SelectQuery(data) => Self::build_query_plan(&data), // 生成Select计划
            AnalyzedResult::ExplainQuery((typ, data)) => {
                let res = Self::build_query_plan(&data)?;
                Ok(PlanNode::Explain(ExplainPlan { // 生成Explain计划
                    typ,
                    input: Arc::new(res),
                }))
            }
        }
    }
// func: build_query_plan() -> Result<PlanNode>
// 构造Select计划，主要是调用PlanBuilder中的各函数，生成AST
    pub fn build_query_plan(data: &QueryAnalyzeState) -> Result<PlanNode> {
        let from = Self::build_from_plan(data)?;
        let filter = Self::build_filter_plan(from, data)?;
        let group_by = Self::build_group_by_plan(filter, data)?;
        let before_order = Self::build_before_order(group_by, data)?;
        let having = Self::build_having_plan(before_order, data)?;
        let distinct = Self::build_distinct_plan(having, data)?;
        let order_by = Self::build_order_by_plan(distinct, data)?;
        let projection = Self::build_projection_plan(order_by, data)?;
        let limit = Self::build_limit_plan(projection, data)?;
        Ok(PlanNode::Select(SelectPlan {
            input: Arc::new(limit),
        }))
    }
```

##### PlanBuilder

```rust
// file: common/planners/src/plan_node_builder.rs
/// 构造AST
pub struct PlanBuilder {
    plan: PlanNode,
}
```

##### DfParser

```rust
// file: query/src/sql/sql_parser.rs
/// SQL parser
pub struct DfParser<'a> {
    pub(crate) parser: Parser<'a>,
    pub(crate) sql: &'a str,
}
// func: parse_sql(...)
// Parse a SQL statement and produce a set of statements with dialect
    pub fn parse_sql(
        sql: &'a str,
        typ: SessionType,
    ) -> Result<(Vec<DfStatement<'a>>, Vec<DfHint>), ErrorCode> {
        match typ {
            SessionType::MySQL => {
                let dialect = &MySqlDialect {};
                let start = Instant::now();
                let result = DfParser::parse_sql_with_dialect(sql, dialect)?; // 解析
                histogram!(super::metrics::METRIC_PARSER_USEDTIME, start.elapsed());
                Ok(result)
            }
            _ => {
                ...
            }
        }
    }
// func: parse_sql_with_dialect(...)
// Parse a SQL statement and produce a set of statements
    pub fn parse_sql_with_dialect(
        sql: &'a str,
        dialect: &'a dyn Dialect,
    ) -> Result<(Vec<DfStatement<'a>>, Vec<DfHint>), ParserError> {
        // 1 parse statements
        let mut parser = DfParser::new_with_dialect(sql, dialect)?; // 创建DfParser
        let mut stmts = Vec::new(); // 初始化statements

        let mut expecting_statement_delimiter = false;
        loop {
            // ignore empty statements (between successive statement delimiters)
            while parser.parser.consume_token(&Token::SemiColon) {
                expecting_statement_delimiter = false;
            }

            if parser.parser.peek_token() == Token::EOF {
                break;
            }
            if expecting_statement_delimiter {
                return parser.expected("end of statement", parser.parser.peek_token());
            }

            let statement = parser.parse_statement()?;
            stmts.push(statement);
            expecting_statement_delimiter = true;
        }

        // 2 parse hints
        let mut hints = Vec::new();

        let mut parser = DfParser::new_with_dialect(sql, dialect)?;
        loop {
            let token = parser.parser.next_token_no_skip();
            match token {
                Some(Token::Whitespace(Whitespace::SingleLineComment { comment, prefix })) => {
                    hints.push(DfHint::create_from_comment(comment, prefix));
                }
                Some(Token::Whitespace(Whitespace::Newline)) | Some(Token::EOF) | None => break,
                _ => continue,
            }
        }
        Ok((stmts, hints))
    }
```

##### DfStatement

```rust
// file: query/src/sql/sql_statement.rs
pub enum DfStatement<'a> {
    // ANSI SQL AST node
    Query(Box<DfQueryStatement>),
    Explain(DfExplain<'a>),
    ...
}
```

##### AnalyzedResult

```rust
// file: query/src/sql/statements/analyzer_statement.rs
pub enum AnalyzedResult {
    SimpleQuery(Box<PlanNode>),
    SelectQuery(Box<QueryAnalyzeState>),
    ExplainQuery((ExplainType, Box<QueryAnalyzeState>)),
}
```

##### PlanNode

```rust
// file: common/planners/src/plan_node.rs
pub enum PlanNode {
    // Base.
    Empty(EmptyPlan),
    Stage(StagePlan),
    Broadcast(BroadcastPlan),
    Remote(RemotePlan),
    Projection(ProjectionPlan),
    Expression(ExpressionPlan),
    AggregatorPartial(AggregatorPartialPlan),
    AggregatorFinal(AggregatorFinalPlan),
    Filter(FilterPlan),
    Having(HavingPlan),
    Sort(SortPlan),
    Limit(LimitPlan),
    LimitBy(LimitByPlan),
    ReadSource(ReadDataSourcePlan),
    SubQueryExpression(SubQueriesSetPlan),
    Sink(SinkPlan),

    // Explain.
    Explain(ExplainPlan),

    // Query.
    Select(SelectPlan),

    // Insert.
    Insert(InsertPlan),

    // Copy.
    Copy(CopyPlan),

    // Call.
    Call(CallPlan),

    // List
    List(ListPlan),

    // Show.
    Show(ShowPlan),

    // Database.
    CreateDatabase(CreateDatabasePlan),
    DropDatabase(DropDatabasePlan),
    RenameDatabase(RenameDatabasePlan),
    ShowCreateDatabase(ShowCreateDatabasePlan),

    // Table.
    CreateTable(CreateTablePlan),
    DropTable(DropTablePlan),
    UnDropTable(UnDropTablePlan),
    RenameTable(RenameTablePlan),
    TruncateTable(TruncateTablePlan),
    OptimizeTable(OptimizeTablePlan),
    DescribeTable(DescribeTablePlan),
    ShowCreateTable(ShowCreateTablePlan),

    // View.
    CreateView(CreateViewPlan),
    DropView(DropViewPlan),
    AlterView(AlterViewPlan),

    // User.
    CreateUser(CreateUserPlan),
    AlterUser(AlterUserPlan),
    DropUser(DropUserPlan),

    // Grant.
    GrantPrivilege(GrantPrivilegePlan),
    GrantRole(GrantRolePlan),

    // Revoke.
    RevokePrivilege(RevokePrivilegePlan),
    RevokeRole(RevokeRolePlan),

    // Role.
    CreateRole(CreateRolePlan),
    DropRole(DropRolePlan),

    // Stage.
    CreateUserStage(CreateUserStagePlan),
    DropUserStage(DropUserStagePlan),
    DescribeUserStage(DescribeUserStagePlan),

    // UDF.
    CreateUserUDF(CreateUserUDFPlan),
    DropUserUDF(DropUserUDFPlan),
    AlterUserUDF(AlterUserUDFPlan),

    // Use.
    UseDatabase(UseDatabasePlan),

    // Set.
    SetVariable(SettingPlan),

    // Kill.
    Kill(KillPlan),
}
```

##### Optimizer

v1在生成PlanNode时，并未执行Optimizer逻辑，而是在SelectInterpreter的execute()函数中执行的，下文中会详细介绍。

#### V2

```rust
let mut planner = Planner::new(context.clone());
let (plan_node, _) = planner.plan_sql(sql).await?;
```

##### Planner

```rust
// file: query/src/sql/planner/mod.rs
pub struct Planner {
    ctx: Arc<QueryContext>,
}
// func: plan_sql(...)
// 词法、语法解析，生成查询计划，只要使用common/ast/src中的方法
// 解析过程: SQL -> Token -> Statement -> SExpr -> Plan(contains SExpr)
    pub async fn plan_sql(&mut self, sql: &str) -> Result<(Plan, MetadataRef)> {
        // Step 1: parse SQL text into AST
        let tokens = tokenize_sql(sql)?; // 自己实现的词法、语法解析，common/ast/src/parser/mod.rs

        let backtrace = Backtrace::new();
        let stmts = parse_sql(&tokens, &backtrace)?; // common/ast/src/parser/mod.rs，这里的stmt与V1不是同一个
        if stmts.len() > 1 {
            return Err(ErrorCode::UnImplement("unsupported multiple statements"));
        }

        // Step 2: bind AST with catalog, and generate a pure logical SExpr
        let metadata = Arc::new(RwLock::new(Metadata::create()));
        let binder = Binder::new(self.ctx.clone(), self.ctx.get_catalogs(), metadata.clone());
        let plan = binder.bind(&stmts[0]).await?;

        // Step 3: optimize the SExpr with optimizers, and generate optimized physical SExpr
        let optimized_plan = optimize(plan)?;

        Ok((optimized_plan, metadata.clone()))
    }

```

##### Binder

```rust
// file: query/src/sql/planner/binder/mod.rs
/// Binder负责将查询的AST转换为规范的逻辑SExpr。
///
///在此阶段，它将:
///使用Catalog解析列和表
///检查查询的语义
///验证表达式
///创建元数据
pub struct Binder {
    ctx: Arc<QueryContext>,
    catalogs: Arc<CatalogManager>,
    metadata: MetadataRef,
}
// func: bind(...)
// 使用Statement生成Plan
    pub async fn bind(mut self, stmt: &Statement<'a>) -> Result<Plan> {
        let init_bind_context = BindContext::new();
        let plan = self.bind_statement(&init_bind_context, stmt).await?;
        Ok(plan)
    }
// func: bind_statement(...)
    async fn bind_statement(
        &mut self,
        bind_context: &BindContext,
        stmt: &Statement<'a>,
    ) -> Result<Plan> {
        match stmt {
            Statement::Query(query) => {
                let (mut s_expr, bind_context) = self.bind_query(bind_context, query).await?; // 生成s_expr
                let mut rewriter = SubqueryRewriter::new(self.metadata.clone());
                s_expr = rewriter.rewrite(&s_expr)?; // 重写s_expr
                Ok(Plan::Query {
                    s_expr,
                    metadata: self.metadata.clone(),
                    bind_context: Box::new(bind_context),
                })
            }
            Statement::Explain { query, kind } => {
                ...
            }
            _ => Err(ErrorCode::UnImplement(format!(
                "UnImplemented stmt {stmt} in binder"
            ))),
        }
    }
// func: bind_query(...)
// bind Select, Limit and Offset
        // 1 bind Select   
        let (mut s_expr, bind_context) = match &query.body {
            SetExpr::Select(stmt) => {
                self.bind_select_stmt(bind_context, stmt, &query.order_by) // 重点函数
                    .await
            }
            SetExpr::Query(stmt) => self.bind_query(bind_context, stmt).await,
            _ => Err(ErrorCode::UnImplement("Unsupported query type")),
        }?;
        // 2 bind Limit and Offset
        self.bind_limit(&bind_context, s_expr, ...)
// func: bind_select_stmt(...)
// 构造新的SExpr Tree

```

##### Token

```rust
// file: common/ast/src/parser/token.rs
// Using in tokenize_sql(), Tokenizer::new(sql).collect::<Result<Vec<_>>>()
pub struct Token<'a> {
    pub source: &'a str,
    pub kind: TokenKind,
    pub span: Span,
}
```

##### Statement

```rust
// file: common/ast/src/ast/statement.rs
// Using in parse_sql() -> statements(), transforms Token into Statement
pub enum Statement<'a> {
    Explain {
        kind: ExplainKind,
        query: Box<Statement<'a>>,
    },
    Query(Box<Query<'a>>),
    ...
}
```

##### SExpr

```rust
// file: query/src/sql/optimizer/s_expr.rs
// “SExpr”是single expression的缩写，它是一个关系操作符树。
pub struct SExpr {
    plan: RelOperator,
    children: Vec<SExpr>,

    original_group: Option<IndexType>,
}
```

##### RelOperator

```rust
// file: query/src/sql/planner/plans/operator.rs
pub enum RelOperator {
    LogicalGet(LogicalGet),
    LogicalInnerJoin(LogicalInnerJoin),

    PhysicalScan(PhysicalScan),
    PhysicalHashJoin(PhysicalHashJoin),

    Project(Project),
    EvalScalar(EvalScalar),
    Filter(FilterPlan),
    Aggregate(AggregatePlan),
    Sort(SortPlan),
    Limit(LimitPlan),
    CrossApply(CrossApply),
    Max1Row(Max1Row),

    Pattern(PatternPlan),
}
```

##### Plan

```rust
// file: query/src/sql/planner/plans/mod.rs
pub enum Plan {
    // Query statement, `SELECT`
    Query {
        s_expr: SExpr,
        metadata: MetadataRef,
        bind_context: Box<BindContext>,
    },

    // Explain query statement, `EXPLAIN`
    Explain {
        kind: ExplainKind,
        plan: Box<Plan>,
    },
}
```

##### Optimizer

```rust
// file: query/src/sql/optimizer/mod.rs
// func: optimize(plan: Plan) -> Result<Plan>
pub fn optimize(plan: Plan) -> Result<Plan> {
    match plan {
        Plan::Query {
            s_expr,
            bind_context,
            metadata,
        } => Ok(Plan::Query {
            s_expr: optimize_query(s_expr)?,
            bind_context,
            metadata,
        }),
        Plan::Explain { kind, plan } => Ok(Plan::Explain {
            kind,
            plan: Box::new(optimize(*plan)?),
        }),
    }
}


pub fn optimize_query(expression: SExpr) -> Result<SExpr> {
    let mut heuristic = HeuristicOptimizer::create()?; // 创建启发式优化器
    let s_expr = heuristic.optimize(expression)?;
    // TODO: enable cascades optimizer
    // let mut cascades = CascadesOptimizer::create(ctx);
    // cascades.optimize(s_expr)

    Ok(s_expr)
}
```

### 3 创建Interperter和Pipeline

#### V1

```rust
let interpreter = InterpreterFactory::get(ctx.clone(), plan_node)?;
let mut stream = interpreter.execute(None).await?;
```

##### InterpreterFactory

```rust
// file: query/src/interpreters/interpreter_factory.rs
// func: get() -> Result<Arc<dyn Interpreter>>
pub fn get(ctx: Arc<QueryContext>, plan: PlanNode) -> Result<Arc<dyn Interpreter>> {
        let ctx_clone = ctx.clone();
        let inner = match plan.clone() {
            PlanNode::Select(v) => SelectInterpreter::try_create(ctx_clone, v),
            PlanNode::Explain(v) => ExplainInterpreter::try_create(ctx_clone, v),
            ...
        }
}
```

##### SelectInterpreter

```rust
// file: query/src/interpreters/interpreter_select.rs
/// SelectInterpreter struct which interprets SelectPlan
pub struct SelectInterpreter {
    ctx: Arc<QueryContext>,
    select: SelectPlan,
}
// func: try_create() -> Result<InterpreterPtr>
    pub fn try_create(ctx: Arc<QueryContext>, select: SelectPlan) -> Result<InterpreterPtr> {
        Ok(Arc::new(SelectInterpreter { ctx, select }))
    }
// func: excute() -> Result<SendableDataBlockStream>
// 执行查询
    async fn execute(
        &self,
        _input_stream: Option<SendableDataBlockStream>,
    ) -> Result<SendableDataBlockStream> {
        let settings = self.ctx.get_settings();
        // 1 非集群环境下，可以选择使用new_processor_framework
        if settings.get_enable_new_processor_framework()? != 0 && self.ctx.get_cluster().is_empty()
        {
            let async_runtime = self.ctx.get_storage_runtime();
            let new_pipeline = self.create_new_pipeline()?; // 创建new pipeline
            let executor = PipelinePullingExecutor::try_create(async_runtime, new_pipeline)?; // 创建executor
            let executor_stream = Box::pin(ProcessorExecutorStream::create(executor)?); // 创建stream
            return Ok(Box::pin(self.ctx.try_create_abortable(executor_stream)?));
        }
        // 2 old processor
        let optimized_plan = self.rewrite_plan()?; // 优化查询计划
        plan_schedulers::schedule_query(&self.ctx, &optimized_plan).await // 执行分布式查询
    }
// func: 
// 创建 new pipeline
// The QueryPipelineBuilder will use the optimized plan to generate a NewPipeline
    fn create_new_pipeline(&self) -> Result<NewPipeline> {
        let settings = self.ctx.get_settings();
        let builder = QueryPipelineBuilder::create(self.ctx.clone());

        let optimized_plan = self.rewrite_plan()?; // 优化
        let select_plan = SelectPlan {
            input: Arc::new(optimized_plan),
        };
        let mut new_pipeline = builder.finalize(&select_plan)?; // 创建new pipeline
        new_pipeline.set_max_threads(settings.get_max_threads()? as usize); // 设置并行度
        Ok(new_pipeline)
    }
// func: rewrite_plan() -> Result<PlanNode>
// 优化查询计划，注意，这里保护将单机查询计划重写成分布式查询计划，这个是通过ScattersOptimizer实现的!!!
    fn rewrite_plan(&self) -> Result<PlanNode> {
        plan_schedulers::apply_plan_rewrite(
            Optimizers::create(self.ctx.clone()),
            &self.select.input,
        )
    }
```

##### QueryPipelineBuilder

```rust
// file: query/src/pipelines/new/pipeline_builder.rs
pub struct QueryPipelineBuilder {
    ctx: Arc<QueryContext>,
    pipeline: NewPipeline,
    limit: Option<usize>,
    offset: usize,
}
```

#### V2

```rust
let interpreter = InterpreterFactoryV2::get(ctx.clone(), &plan_node)?;
let mut stream = interpreter.execute(None).await?;
```

##### InterpreterFactoryV2

```rust
// file: query/src/interpreters/interpreter_factory_v2.rs
// func: get(...) -> Result<InterpreterPtr>
    pub fn get(ctx: Arc<QueryContext>, plan: &Plan) -> Result<InterpreterPtr> {
        let inner = match plan {
            Plan::Query {
                s_expr,
                bind_context,
                metadata,
            } => SelectInterpreterV2::try_create(
                ctx,
                *bind_context.clone(),
                s_expr.clone(),
                metadata.clone(),
            ),
            Plan::Explain { kind, plan } => {
                ExplainInterpreterV2::try_create(ctx, *plan.clone(), kind.clone())
            }
        }?;
        Ok(inner)
    }
```

##### SelectInterpreterV2

```rust
// file: query/src/interpreters/interpreter_select_v2.rs
// Interpret SQL query with new SQL planner
pub struct SelectInterpreterV2 {
    ctx: Arc<QueryContext>,
    s_expr: SExpr,
    bind_context: BindContext,
    metadata: MetadataRef,
}
// func: try_create(...) -> Result<InterpreterPtr>
    pub fn try_create(
        ctx: Arc<QueryContext>,
        bind_context: BindContext,
        s_expr: SExpr,
        metadata: MetadataRef,
    ) -> Result<InterpreterPtr> {
        Ok(Arc::new(SelectInterpreterV2 {
            ctx,
            s_expr,
            bind_context,
            metadata,
        }))
    }
// func: execute(...) -> Result<SendableDataBlockStream>
    async fn execute(
        &self,
        _input_stream: Option<SendableDataBlockStream>,
    ) -> Result<SendableDataBlockStream> {
        let pb = PipelineBuilder::new(
            self.ctx.clone(),
            self.bind_context.result_columns(),
            self.metadata.clone(),
            self.s_expr.clone(),
        );

        if let Some(handle) = self.ctx.get_http_query() {
            return handle.execute(self.ctx.clone(), pb).await;
        }
        let (root_pipeline, pipelines, _) = pb.spawn()?; // 创建pipeline
        let async_runtime = self.ctx.get_storage_runtime();

        // Spawn sub-pipelines
        for pipeline in pipelines {
            let executor = PipelineExecutor::create(async_runtime.clone(), pipeline)?;
            executor.execute()?;
        }

        // Spawn root pipeline
        let executor = PipelinePullingExecutor::try_create(async_runtime, root_pipeline)?;
        let executor_stream = Box::pin(ProcessorExecutorStream::create(executor)?);
        Ok(Box::pin(self.ctx.try_create_abortable(executor_stream)?))
    }
```

##### PipelineBuilder

```rust
// file: query/src/sql/exec/mod.rs
// Helper to build a `Pipeline` from `SExpr`
pub struct PipelineBuilder {
    ctx: Arc<QueryContext>,
    metadata: MetadataRef,
    result_columns: Vec<(IndexType, String)>,
    expression: SExpr,
    pub pipelines: Vec<NewPipeline>,
    limit: Option<usize>,
    offset: usize,
}
```

### 4 Pipeline视图

##### NewPipeline

```rust
// file: query/src/pipelines/new/pipeline.rs
pub struct NewPipeline {
    max_threads: usize,
    pub pipes: Vec<NewPipe>,
}
```

##### NewPipe

```rust
// file: query/src/pipelines/new/pipe.rs
pub enum NewPipe {
    SimplePipe {
        processors: Vec<ProcessorPtr>,
        inputs_port: Vec<Arc<InputPort>>,
        outputs_port: Vec<Arc<OutputPort>>,
    },
    ResizePipe {
        processor: ProcessorPtr,
        inputs_port: Vec<Arc<InputPort>>,
        outputs_port: Vec<Arc<OutputPort>>,
    },
}
```

##### ProcessorPtr/Processor

```rust
// file: query/src/pipelines/new/processors/processor.rs
pub struct ProcessorPtr {
    id: Arc<UnsafeCell<NodeIndex>>, // ProcessorPtr在DAG中节点的序号
    inner: Arc<UnsafeCell<Box<dyn Processor>>>, // 实际的Processor
}

// trait: Processor
// 算子的抽象
pub trait Processor: Send {
    fn name(&self) -> &'static str;

    fn event(&mut self) -> Result<Event>;

    // Synchronous work.
    fn process(&mut self) -> Result<()> {
        Err(ErrorCode::UnImplement("Unimplemented process."))
    }

    // Asynchronous work.
    async fn async_process(&mut self) -> Result<()> {
        Err(ErrorCode::UnImplement("Unimplemented async_process."))
    }
}
```

##### InputPort/OutputPort/SharedStatus

```rust
// file: query/src/pipelines/new/processors/port.rs
// 数据流: OutputPort -> SharedStatus -> InputPort
pub struct InputPort {
    shared: UnSafeCellWrap<Arc<SharedStatus>>,
    update_trigger: UnSafeCellWrap<*mut UpdateTrigger>,
}

pub struct OutputPort {
    shared: UnSafeCellWrap<Arc<SharedStatus>>,
    update_trigger: UnSafeCellWrap<*mut UpdateTrigger>,
}

pub struct SharedStatus {
    data: AtomicPtr<SharedData>,
}

pub struct SharedData(pub Result<DataBlock>);
```

##### UpdateTrigger/UpdateList/UpdateListMutable/DirectedEdge

```rust
// file: query/src/pipelines/new/processors/port_trigger.rs
pub struct UpdateTrigger {
    index: EdgeIndex,
    update_list: Arc<UpdateList>,
    version: usize,
    prev_version: usize,
}
// func: trigger_version(...)
// 更新version
    pub unsafe fn trigger_version(self_: *mut UpdateTrigger) {
        (*self_).prev_version = (*self_).version;
    }
// func: update_input(...)
// 把边放入update_list，被InputPort的set_need_data()、pull_data()和finish()函数调用
    pub unsafe fn update_input(self_: &*mut UpdateTrigger) {
        if !self_.is_null() {
            let self_ = &mut **self_;
            if self_.version == self_.prev_version {
                self_.version += 1;
                self_
                    .update_list
                    .update_edge(DirectedEdge::Target(self_.index));
            }
        }
    }
// func: update_output(...)
// 把边放入update_list，被OutputPort的push_data()和finish()函数调用
    pub unsafe fn update_output(self_: &*mut UpdateTrigger) {
        if !self_.is_null() {
            let self_ = &mut **self_;
            if self_.version == self_.prev_version {
                self_.version += 1;
                self_
                    .update_list
                    .update_edge(DirectedEdge::Source(self_.index));
            }
        }
    }

pub struct UpdateList {
    inner: UnsafeCell<UpdateListMutable>,
}
// func: update_edge(...)
// 把被触发的边放入inner，在UpdateTrigger::update_input()和update_output()中调用
    pub unsafe fn update_edge(&self, edge: DirectedEdge) {
        let inner = &mut *self.inner.get();
        inner.updated_edges.push(edge);
    }
// func: trigger(...)
// 从inner中取出被触发的边，放到queue中，用于调度下一个节点
    pub unsafe fn trigger(&self, queue: &mut VecDeque<DirectedEdge>) {
        let inner = &mut *self.inner.get();

        for trigger in &inner.updated_triggers {
            UpdateTrigger::trigger_version(trigger.get());
        }
        // 把inner中的所有边放到队列中
        while let Some(index) = inner.updated_edges.pop() {
            queue.push_front(index);
        }
    }
// func: create_trigger() -> UpdateTrigger
// 创建边对应的trigger
    pub unsafe fn create_trigger(self: &Arc<Self>, edge_index: EdgeIndex) -> *mut UpdateTrigger {
        let inner = &mut *self.inner.get();
        let update_trigger = UpdateTrigger::create(edge_index, self.clone());
        inner
            .updated_triggers
            .push(Arc::new(UnsafeCell::new(update_trigger)));
        inner.updated_triggers.last().unwrap().get()
        // let update_trigger = UnsafeCell::new(UpdateTrigger::create(edge_index, self.clone()));
        // let res = update_trigger.get();
        // inner.updated_triggers.push(update_trigger);
        // res
    }

struct UpdateListMutable {
    updated_edges: Vec<DirectedEdge>,
    updated_triggers: Vec<Arc<UnsafeCell<UpdateTrigger>>>,
}

pub enum DirectedEdge {
    Source(EdgeIndex),
    Target(EdgeIndex),
}
```

### 5 执行Pipeline

```rust
let executor = PipelinePullingExecutor::try_create(async_runtime, root_pipeline)?;
let executor_stream = Box::pin(ProcessorExecutorStream::create(executor)?);
Ok(Box::pin(self.ctx.try_create_abortable(executor_stream)?))
```

1. 初始化: PipelinePullingExecutor::try_create() -> PipelineExecutor::create() -> {RunningGraph::create() and init_schedule_queue(), ExecutorTasksQueue::create()}
2. 启动: ProcessorExecutorStream::create() -> PipelinePullingExecutor::start() -> PipelineExecutor::execute() -> {ExecutorWorkerContext::create() and execute_task(), RunningGraph::schedule_queue(), ExecutorTasksQueue::steal_task_to_context()}
3. 获取数据: ProcessorExecutorStream::poll_next() -> PipelinePullingExecutor::pull_data() -> mpsc -> PullSink::consume()

##### ProcessorExecutorStream

```rust
// file: query/src/interpreters/stream/processor_executor_stream.rs
// 封装PipelinePullingExecutor
pub struct ProcessorExecutorStream {
    executor: PipelinePullingExecutor,
}
// func: create(...) -> Result<Self>
// 创建ProcessorExecutorStream
    pub fn create(mut executor: PipelinePullingExecutor) -> Result<Self> {
        executor.start(); // 启动PipelinePullingExecutor
        Ok(Self { executor })
    }
// func: poll_next()
impl Stream for ProcessorExecutorStream {
    type Item = Result<DataBlock>;

    fn poll_next(self: Pin<&mut Self>, _: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let self_ = Pin::get_mut(self);
        match self_.executor.pull_data() { // 获取数据
            Ok(None) => Poll::Ready(None),
            Err(cause) => Poll::Ready(Some(Err(cause))),
            Ok(Some(data)) => Poll::Ready(Some(Ok(data))),
        }
    }
}
```

##### PipelinePullingExecutor

```rust
// file: query/src/pipelines/new/executor/pipeline_pulling_executor.rs
// Use this executor when the pipeline is pulling pipeline (exists source but not exists sink)
pub struct PipelinePullingExecutor {
    state: Arc<State>,
    executor: Arc<PipelineExecutor>,
    receiver: Receiver<Result<Option<DataBlock>>>,
}
// func: try_create() -> Result<PipelinePullingExecutor>
// 创建PipelinePullingExecutor
    pub fn try_create(
        async_runtime: Arc<Runtime>,
        mut pipeline: NewPipeline,
    ) -> Result<PipelinePullingExecutor> {
        let (sender, receiver) = std::sync::mpsc::sync_channel(pipeline.output_len());
        let state = State::create(sender.clone());

        Self::wrap_pipeline(&mut pipeline, sender)?; // 包一层NewPipe::SimplePipe，里边是PullSink Processor
        let executor = PipelineExecutor::create(async_runtime, pipeline)?; // 创建PipelineExecutor
        Ok(PipelinePullingExecutor {
            receiver,
            state,
            executor,
        })
    }
// func: start()
// 启动
    pub fn start(&mut self) {
        let state = self.state.clone();
        let threads_executor = self.executor.clone();
        let thread_function = Self::thread_function(state, threads_executor);
        std::thread::spawn(thread_function);
    }
```

##### PipelineExecutor ⭐️

```rust
// file: query/src/pipelines/new/executor/pipeline_executor.rs
pub struct PipelineExecutor {
    threads_num: usize, // 并行度
    graph: RunningGraph, // 图调度器
    workers_condvar: Arc<WorkersCondvar>, // 封装的全部Woker的条件变量
    pub async_runtime: Arc<Runtime>, // 运行时
    pub global_tasks_queue: Arc<ExecutorTasksQueue>, // 全局任务队列
}
// func: create(...) -> Result<Arc<PipelineExecutor>>
// 创建PipelineExecutor
    pub fn create(async_rt: Arc<Runtime>, pipeline: NewPipeline) -> Result<Arc<PipelineExecutor>> {
        unsafe {
            let threads_num = pipeline.get_max_threads();
            let workers_condvar = WorkersCondvar::create(threads_num);
            let global_tasks_queue = ExecutorTasksQueue::create(threads_num);
  
            let graph = RunningGraph::create(pipeline)?; // 创建RunningGraph
            let mut init_schedule_queue = graph.init_schedule_queue()?; // 初始化DAG任务队列，获取起始任务
  
            let mut tasks = VecDeque::new();
            while let Some(task) = init_schedule_queue.pop_task() {
                tasks.push_back(task);
            }
            global_tasks_queue.init_tasks(tasks); // 把起始任务放入全局队列

            Ok(Arc::new(PipelineExecutor {
                graph,
                threads_num,
                workers_condvar,
                global_tasks_queue,
                async_runtime: async_rt,
            }))
        }
    }
// func: execute() 
    let mut thread_join_handles = self.execute_threads(self.threads_num);
// func: execute_threads(threads_size)
// 并行执行Pipeline
for thread_num in 0..threads_size {
    this.execute_single_thread(thread_num) // thread_num是线程序号
}
// func: execute_single_thread() ⭐️
// 单线程执行Pipeline 
pub unsafe fn execute_single_thread(&self, thread_num: usize) -> Result<()> {
        let workers_condvar = self.workers_condvar.clone();
        // 创建当前线程的Context
        let mut context = ExecutorWorkerContext::create(thread_num, workers_condvar);
        // 如果全局任务队列没有结束
        while !self.global_tasks_queue.is_finished() {
            // When there are not enough tasks, the thread will be blocked, so we need loop check.
            while !self.global_tasks_queue.is_finished() && !context.has_task() {
                self.global_tasks_queue.steal_task_to_context(&mut context); // 从全局任务队列窃取任务到当前Context
            }

            while context.has_task() {
                if let Some(executed_pid) = context.execute_task(self)? { // 执行任务，调用processor的process()函数，执行数据处理逻辑
                    // We immediately schedule the processor again.
                    let schedule_queue = self.graph.schedule_queue(executed_pid)?; // 调度任务，调用processor的event()函数，执行数据流转逻辑
                    // 把任务队列中的栈顶任务放入Context，剩余的任务放入global_tasks_queue中
                    schedule_queue.schedule(&self.global_tasks_queue, &mut context);
                }
            }
        }

        Ok(())
    }

```

##### RunningGraph/ExecutingGraph ⭐️

负责任务的调度

```rust
// file: query/src/pipelines/new/executor/executor_graph.rs
// 对ExecutingGraph的封装
pub struct RunningGraph(ExecutingGraph);
// func: 
    pub fn create(pipeline: NewPipeline) -> Result<RunningGraph> {
        let graph_state = ExecutingGraph::create(pipeline)?;
        tracing::debug!("Create running graph:{:?}", graph_state);
        Ok(RunningGraph(graph_state))
    }
// func: 
    pub unsafe fn init_schedule_queue(&self) -> Result<ScheduleQueue> {
        ExecutingGraph::init_schedule_queue(&self.0)
    }
// func: 
    pub unsafe fn schedule_queue(&self, node_index: NodeIndex) -> Result<ScheduleQueue> {
        let mut schedule_queue = ScheduleQueue::create();
        ExecutingGraph::schedule_queue(&self.0, node_index, &mut schedule_queue)?;
        Ok(schedule_queue)
    }

// 基于StableGraph的DAG
struct ExecutingGraph {
    graph: StableGraph<Arc<Node>, ()>,
}
// func:
// 基于Pipeline构建DAG
pub fn create(pipeline: NewPipeline) -> Result<ExecutingGraph> {
    let mut graph = StableGraph::new();
    for query_pipe in &pipeline.pipes {
        // 给DAG添加节点和边, graph.add_node(), graph.add_edge()
        // 构建顺序: Source -> Sink
        match query_pipe {
            NewPipe::ResizePipe {...} => {...}
            NewPipe::SimplePipe {...} => {...}
        }
    }
    Ok(ExecutingGraph { graph })
}
/*  
对于示例中的查询: 
SELECT * FROM numbers_mt(10000) where number = 10 limit 10

Pipeline如下:
PullingExecutorSink × 1 processor
  LimitTransform × 1 processor
    Resize × 1 processor
      ProjectionTransform × 2 processors
        FilterTransform × 2 processors
          numbers × 2 processors

DAG如下:
    0 [ label = "numbers" ]
    1 [ label = "numbers" ]
    2 [ label = "FilterTransform" ]
    3 [ label = "FilterTransform" ]
    4 [ label = "ProjectionTransform" ]
    5 [ label = "ProjectionTransform" ]
    6 [ label = "Resize" ]
    7 [ label = "LimitTransform" ]
    8 [ label = "PullingExecutorSink" ]
    0 -> 2 [ ]
    1 -> 3 [ ]
    2 -> 4 [ ]
    3 -> 5 [ ]
    4 -> 6 [ ]
    5 -> 6 [ ]
    6 -> 7 [ ]
    7 -> 8 [ ]
*/


```

介绍一下构建流程，填加完numbers节点的DAG是如下所示，其中节点表示为"ProcessorName_NodeIndexInDAG"

```shell
numbers_0 

numbers_1
```

现在需要增加FilterTransform节点，首先创建FilterTransform_2，并加入DAG中；然后确定node_stack（里边存的是先前节点在DAG中的序号）不为空，即存在source节点numbers_0，需要在二者间建立边，并获得边的编号edge_index；第三步是给FilterTransform_2的InputPort设置与edge_index相关的trigger；第四步是给numbers_0的OutputPort设置与edge_index相关的trigger；第五步是连接FilterTransform_2的InputPort和numbers_0的OutputPort；最后，因为FilterTransform_2存在下游节点（ProjectionTransform），需要将其记录到new_node_stack中，在填加ProjectionTransform时使用。所有的FilterTransform节点（个数与并行度有关）填加完之后，生成的DAG如下：

```shell
numbers_0 -> FilterTransform_2

numbers_1 -> FilterTransform_3
```

所有节点填加完之后，DAG如下：

```shell
numbers_0 -> FilterTransform_2 -> ProjectionTransform_4 ↘

                                                           Resize_6 -> LimitTransform_7 -> PullingExecutorSink_8
numbers_1 -> FilterTransform_3 -> ProjectionTransform_5 ↗
```

```rust
// type: 
   type StateLockGuard = ExecutingGraph;

// func: init_schedule_queue(...) -> Result<ScheduleQueue>
// 初始化任务队列，获取第一个任务
   pub unsafe fn init_schedule_queue(locker: &StateLockGuard) -> Result<ScheduleQueue> {
        let mut schedule_queue = ScheduleQueue::create(); // 创建ScheduleQueue
        for sink_index in locker.graph.externals(Direction::Outgoing) { // 通过Direction::Outgoing获取Sink, 即上述的PullingExecutorSink_8
            ExecutingGraph::schedule_queue(locker, sink_index, &mut schedule_queue)?; // 调度后放入schedule_queue
        }

        Ok(schedule_queue)
    }
// func: schedule_queue()
// 调度节点
    pub unsafe fn schedule_queue(
        locker: &StateLockGuard, // &Self
        index: NodeIndex, // 待调度节点的索引
        schedule_queue: &mut ScheduleQueue, // 调度队列
    ) -> Result<()> {
        let mut need_schedule_nodes = VecDeque::new();
        let mut need_schedule_edges = VecDeque::new();

        need_schedule_nodes.push_back(index); // 加入待调度节点
        while !need_schedule_nodes.is_empty() || !need_schedule_edges.is_empty() {
            // To avoid lock too many times, we will try to cache lock.
            let mut state_guard_cache = None;

            // 如果待调度的边不为空
            if need_schedule_nodes.is_empty() {
                let edge = need_schedule_edges.pop_front().unwrap(); // 取出栈顶的边
                let target_index = DirectedEdge::get_target(&edge, &locker.graph)?; // 获取边目标节点的索引

                let node = &locker.graph[target_index]; // 获取目标节点
                let node_state = node.state.lock().unwrap(); // 节点状态

                if matches!(*node_state, State::Idle) { // 如果节点的状态是Idle
                    state_guard_cache = Some(node_state);
                    need_schedule_nodes.push_back(target_index); // 把节点放入待调度队列
                }
            }
  
            // 取出栈顶的节点索引
            if let Some(schedule_index) = need_schedule_nodes.pop_front() {
                let node = &locker.graph[schedule_index]; // 从DAG中获取schedule_index对应的Node

                if state_guard_cache.is_none() {
                    state_guard_cache = Some(node.state.lock().unwrap());
                }

                *state_guard_cache.unwrap() = match node.processor.event()? {
                    Event::Finished => State::Finished,
                    Event::NeedData | Event::NeedConsume => State::Idle,
                    Event::Sync => {
                        schedule_queue.push_sync(node.processor.clone()); // 把当前Node放入schedule_queue的同步队列
                        State::Processing
                    }
                    Event::Async => {
                        schedule_queue.push_async(node.processor.clone()); // 把当前Node放入schedule_queue的异步队列
                        State::Processing
                    }
                };
  
                node.trigger(&mut need_schedule_edges); // 节点触发，尝试把边索引加入need_schedule_edges
            }
        }

        Ok(())
    }
```

```shell
numbers_0 -> FilterTransform_2 -> ProjectionTransform_4 ↘

                                                           Resize_6 -> LimitTransform_7 -> PullingExecutorSink_8
numbers_1 -> FilterTransform_3 -> ProjectionTransform_5 ↗

上述DAG的执行顺序：
确定第一个调度节点：PullingExecutorSink_8
调度：PullingExecutorSink_8.event() -> Event::Sync          // 获得一个可执行节点：PullingExecutorSink_8
执行：PullingExecutorSink_8.process()                       // do nothing
调度：PullingExecutorSink_8.event() -> Event::NeedData      // 向上游调度，self.input.set_need_data()
     LimitTransform_7.event() -> Event::NeedData           // 向上游调度，self.input.set_need_data()
     Resize_6.event() -> Event::NeedData                   // 向上游调度，input.set_need_data()
         [ProjectionTransform_4.event() -> Event::NeedData // 向上游调度，self.input.set_need_data()
          FilterTransform_2.event() -> Event::NeedData     // 向上游调度，self.input.set_need_data()
          numbers_0.event() -> Event::Sync]                // 获得一个可执行节点：numbers_0
         [ProjectionTransform_5.event() -> Event::NeedData // 向上游调度，self.input.set_need_data()
          FilterTransform_3.event() -> Event::NeedData     // 向上游调度，self.input.set_need_data()
          numbers_1.event() -> Event::Sync]                // 获得一个可执行节点：numbers_1
并行执行：numbers_0.process()                                // 读取数据
        numbers_1.process()                                // 读取数据
并行调度：numbers_0.event() -> Event::NeedConsume            // 向下游调度，self.output.push_data(Ok(data_block))
        FilterTransform_2.event() -> Event::Sync           // 获取一个可执行节点：FilterTransform_2，self.input.pull_data().unwrap()?这里同时会调度上游游
        numbers_0.event() -> Event::NeedConsume            // 未触发边，结束调度
        [numbers_1 同理，这里省略]
并行执行：FilterTransform_2.process()                        // 过滤数据
并行调度：FilterTransform_2.event() -> Event::NeedConsume    // 向下游调度，self.output.push_data(Ok(data))
...
```

##### ExecutorWorkerContext

负责任务的执行

```rust
// file: query/src/pipelines/new/executor/executor_worker_context.rs
pub enum ExecutorTask {
    None,
    Sync(ProcessorPtr),
    Async(ProcessorPtr),
    // AsyncSchedule(ExecutingAsyncTask),
    AsyncCompleted(CompletedAsyncTask),
}

pub struct ExecutorWorkerContext {
    worker_num: usize, // woker序号
    task: ExecutorTask, // 当前任务
    workers_condvar: Arc<WorkersCondvar>,
}

// func: set_task()
// 给当前Context设置Task
    pub fn set_task(&mut self, task: ExecutorTask) {
        dbg!(&task);
        self.task = task
    }
// func: take_task() -> ExecutorTask
// 获取当前Context的Task
    pub fn take_task(&mut self) -> ExecutorTask {
        std::mem::replace(&mut self.task, ExecutorTask::None)
    }
// func: execute_task() -> Result<Option<NodeIndex>>
// 执行当前Context的Task
    pub unsafe fn execute_task(&mut self, exec: &PipelineExecutor) -> Result<Option<NodeIndex>> {
        match std::mem::replace(&mut self.task, ExecutorTask::None) {
            ExecutorTask::None => Err(ErrorCode::LogicalError("Execute none task.")),
            ExecutorTask::Sync(processor) => self.execute_sync_task(processor), // 执行同步任务
            ExecutorTask::Async(processor) => self.execute_async_task(processor, exec), // 执行异步任务
            ExecutorTask::AsyncCompleted(task) => match task.res {
                Ok(_) => Ok(Some(task.id)),
                Err(cause) => Err(cause),
            },
        }
    }

    unsafe fn execute_sync_task(&mut self, processor: ProcessorPtr) -> Result<Option<NodeIndex>> {
        processor.process()?;
        Ok(Some(processor.id()))
    }

    unsafe fn execute_async_task(
        &mut self,
        processor: ProcessorPtr,
        executor: &PipelineExecutor,
    ) -> Result<Option<NodeIndex>> {
        let worker_id = self.worker_num;
        let workers_condvar = self.get_workers_condvar().clone();
        let tasks_queue = executor.global_tasks_queue.clone();
        let clone_processor = processor.clone();
        let join_handle = executor
            .async_runtime
            .spawn(async move { processor.async_process().await });
        executor.async_runtime.spawn(async move {
            let res = match join_handle.await {
                Ok(Ok(_)) => Ok(()),
                Ok(Err(cause)) => Err(cause),
                Err(cause) => Err(ErrorCode::PanicError(format!(
                    "Panic error, cause {}",
                    cause
                ))),
            };
            // 把执行完的任务放入global_tasks_queue中
            tasks_queue.completed_async_task(
                workers_condvar,
                CompletedAsyncTask::create(clone_processor, worker_id, res),
            );
        });

        Ok(None)
    }
```

##### Node

```rust
// file: query/src/pipelines/new/executor/executor_graph.rs
struct Node {
    state: std::sync::Mutex<State>,
    processor: ProcessorPtr,

    updated_list: Arc<UpdateList>,
    #[allow(dead_code)]
    inputs_port: Vec<Arc<InputPort>>,
    #[allow(dead_code)]
    outputs_port: Vec<Arc<OutputPort>>,
}
// func: trigger()
// 触发相关的边
    pub unsafe fn trigger(&self, queue: &mut VecDeque<DirectedEdge>) {
        self.updated_list.trigger(queue)
    }
```

##### State

```rust
// file: query/src/pipelines/new/executor/executor_graph.rs
enum State {
    Idle,
    // Preparing,
    Processing,
    Finished,
}
```

##### ScheudleQueue

```rust
// 调度队列
pub struct ScheduleQueue {
    sync_queue: VecDeque<ProcessorPtr>,
    async_queue: VecDeque<ProcessorPtr>,
}

impl ScheduleQueue {
    pub fn create() -> ScheduleQueue;
    pub fn push_sync(&mut self, processor: ProcessorPtr);
    pub fn push_async(&mut self, processor: ProcessorPtr);
    pub fn pop_task(&mut self) -> Option<ExecutorTask>;
    pub fn schedule_tail(mut self, global: &ExecutorTasksQueue, ctx: &mut ExecutorWorkerContext);
    pub fn schedule(mut self, global: &ExecutorTasksQueue, context: &mut ExecutorWorkerContext);
    pub fn sync_queue(&self) -> &VecDeque<ProcessorPtr>;
}
```

##### ExecutorTasksQueue

```rust
// file: query/src/pipelines/new/executor/executor_tasks.rs
// 可执行任务队列, 封装的ExecutorTasks
pub struct ExecutorTasksQueue {
    finished: Arc<AtomicBool>,
    workers_tasks: Mutex<ExecutorTasks>,
}
```

##### ExecutorTasks

```rust
// file: query/src/pipelines/new/executor/executor_tasks.rs
// 可执行任务队列
struct ExecutorTasks {
    tasks_size: usize,
    workers_waiting_status: WorkersWaitingStatus,
    workers_sync_tasks: Vec<VecDeque<ProcessorPtr>>,
    workers_async_tasks: Vec<VecDeque<ProcessorPtr>>,
    workers_completed_async_tasks: Vec<VecDeque<CompletedAsyncTask>>,
}
```

##### CompleteAsyncTask

```rust
// file: query/src/pipelines/new/executor/executor_tasks.rs
// 可执行任务
pub struct CompletedAsyncTask {
    pub id: NodeIndex,
    pub worker_id: usize,
    pub res: Result<()>,
}
```
