## 三 分布式查询

### 1 执行入口

##### SelectInterpreter

```rust
func: execute(...) -> Result<SendableDataBlockStream>
// 执行查询
    async fn execute(
        &self,
        _input_stream: Option<SendableDataBlockStream>,
    ) -> Result<SendableDataBlockStream> {
        let settings = self.ctx.get_settings();

        if settings.get_enable_new_processor_framework()? != 0 && self.ctx.get_cluster().is_empty()
        {
            ...
        }
        let optimized_plan = self.rewrite_plan()?; // 1. 这里会生成分布式查询计划
        plan_schedulers::schedule_query(&self.ctx, &optimized_plan).await // 2. 分布式调度
    }
// func: rewrite_plan() -> Result<PlanNode>
// 优化逻辑查询计划，通过ScattersOptimizer生成分布式逻辑查询计划
    fn rewrite_plan(&self) -> Result<PlanNode> {
        plan_schedulers::apply_plan_rewrite(
            Optimizers::create(self.ctx.clone()),
            &self.select.input,
        )
    }
```

### 2 创建分布式逻辑计划

##### Optimizers

```rust
// file: query/src/optimizers/optimizer.rs
pub struct Optimizers {
    inner: Vec<Box<dyn Optimizer>>,
}
// func: create(...)
// 创建优化规则集合
    pub fn create(ctx: Arc<QueryContext>) -> Self {
        let mut optimizers = Self::without_scatters(ctx.clone());
        optimizers
            .inner
            .push(Box::new(ScattersOptimizer::create(ctx))); // 产生分布式计划的优化规则
        optimizers
    }
// func: without_scatters(...)
    pub fn without_scatters(ctx: Arc<QueryContext>) -> Self {
        Optimizers {
            inner: vec![
                Box::new(ConstantFoldingOptimizer::create(ctx.clone())),
                Box::new(ExprTransformOptimizer::create(ctx.clone())),
                Box::new(TopNPushDownOptimizer::create(ctx.clone())),
                Box::new(StatisticsExactOptimizer::create(ctx)),
            ],
        }
    }
```

##### ScattersOptimizer

```rust
// file: query/src/optimizers/optimizer_scatters.rs
pub struct ScattersOptimizer {
    ctx: Arc<QueryContext>,
}
// func: optimize(...) -> Result<PlanNode>
// 优化逻辑
    fn optimize(&mut self, plan: &PlanNode) -> Result<PlanNode> {
        if self.ctx.get_cluster().is_empty() {
            // Standalone mode.
            return Ok(plan.clone());
        }

        let mut optimizer_impl = ScattersOptimizerImpl::create(self.ctx.clone());
        let rewrite_plan = optimizer_impl.rewrite_plan_node(plan)?;

        // We need to converge at the end
        match optimizer_impl.running_mode {
            RunningMode::Standalone => Ok(rewrite_plan),
            RunningMode::Cluster => Ok(PlanNode::Stage(StagePlan { // 创建StagePlan
                kind: StageKind::Convergent,
                scatters_expr: Expression::create_literal(DataValue::UInt64(0)),
                input: Arc::new(rewrite_plan),
            })),
        }
    }

enum RunningMode {
    Standalone,
    Cluster,
}

struct ScattersOptimizerImpl {
    ctx: Arc<QueryContext>,
    running_mode: RunningMode, // 表示当前重写的PlanNode应该在单节点或者分布式运行
    before_group_by_schema: Option<DataSchemaRef>,

    // temporary node
    input: Option<Arc<PlanNode>>,
}
// func: create(...) -> ScattersOptimizerImpl
// 创建ScattersOptimizerImpl
    pub fn create(ctx: Arc<QueryContext>) -> ScattersOptimizerImpl {
        ScattersOptimizerImpl {
            ctx,
            running_mode: RunningMode::Standalone, // 初始化为Standalone
            before_group_by_schema: None,
            input: None,
        }
    }
// impl trait:
// 这里的逻辑比较简单，之间看源码即可，这里只列出函数定义
impl PlanRewriter for ScattersOptimizerImpl {
    fn rewrite_subquery_plan(&mut self, subquery_plan: &PlanNode) -> Result<PlanNode> {
        let subquery_ctx = QueryContext::create_from(self.ctx.clone());
        let mut subquery_optimizer = ScattersOptimizerImpl::create(subquery_ctx);
        let rewritten_subquery = subquery_optimizer.rewrite_plan_node(subquery_plan)?;

        match (&self.running_mode, &subquery_optimizer.running_mode) {
            (RunningMode::Standalone, RunningMode::Standalone) => Ok(rewritten_subquery),
            // 填加PlanNode::Stage
            (RunningMode::Standalone, RunningMode::Cluster) => {
                Ok(Self::convergent_shuffle_stage(rewritten_subquery)?)
            }
            // 填加PlanNode::Broadcast
            (RunningMode::Cluster, RunningMode::Standalone) => {
                Ok(PlanNode::Broadcast(BroadcastPlan {
                    input: Arc::new(rewritten_subquery),
                }))
            }
            // 填加PlanNode::Broadcast
            (RunningMode::Cluster, RunningMode::Cluster) => {
                Ok(PlanNode::Broadcast(BroadcastPlan {
                    input: Arc::new(rewritten_subquery),
                }))
            }
        }
    }
    fn rewrite_aggregate_partial(&mut self, plan: &AggregatorPartialPlan) -> Result<PlanNode>;
    fn rewrite_aggregate_final(&mut self, plan: &AggregatorFinalPlan) -> Result<PlanNode>;
    fn rewrite_sort(&mut self, plan: &SortPlan) -> Result<PlanNode>;
    fn rewrite_limit(&mut self, plan: &LimitPlan) -> Result<PlanNode>;
    fn rewrite_limit_by(&mut self, plan: &LimitByPlan) -> Result<PlanNode>;
    fn rewrite_read_data_source(&mut self, plan: &ReadDataSourcePlan) -> Result<PlanNode>;
}
// 会创建PlanNode::Stage的函数
// func: convergent_shuffle_stage_builder(...) -> PlanBuilder
// 创建收敛于一个节点的StagePlan，cluster_sort()、cluster_limit()、cluster_limit_by()都会调用此函数，即 n -> 1
    fn convergent_shuffle_stage_builder(input: Arc<PlanNode>) -> PlanBuilder {
        PlanBuilder::from(&PlanNode::Stage(StagePlan {
            kind: StageKind::Convergent,
            scatters_expr: Expression::create_literal(DataValue::UInt64(0)),
            input,
        }))
    }
// func: convergent_shuffle_stage_builder(...) -> Result<PlanNode>
// 同convergent_shuffle_stage_builder()，cluster_aggregate_without_key()和rewrite_subquery_plan()在(Standalone,Cluster)时会调用此函数
    fn convergent_shuffle_stage(input: PlanNode) -> Result<PlanNode> {
        Ok(PlanNode::Stage(StagePlan {
            kind: StageKind::Convergent,
            scatters_expr: Expression::create_literal(DataValue::UInt64(0)),
            input: Arc::new(input),
        }))
    }
// func: normal_shuffle_stage(...) -> Result<PlanNode>
// 创建普通shuffle stage，cluster_aggregate_with_key()会调用此函数，即 n -> m，比如Hash shuffle
    fn normal_shuffle_stage(key: impl Into<String>, input: PlanNode) -> Result<PlanNode> {
        let scatters_expr = Expression::ScalarFunction {
            op: String::from("sipHash"),
            args: vec![Expression::Column(key.into())],
        };

        Ok(PlanNode::Stage(StagePlan {
            scatters_expr,
            kind: StageKind::Normal,
            input: Arc::new(input),
        }))
    }

```

### 3 分布式调度与执行

##### fn schedule_query

```rust
// file: query/src/interpreters/plan_schedulers/plan_scheduler_query.rs
pub async fn schedule_query(
    ctx: &Arc<QueryContext>,
    plan: &PlanNode,
) -> Result<SendableDataBlockStream> {
    let scheduler = PlanScheduler::try_create(ctx.clone())?; // 创建PlanScheduler
    let scheduled_tasks = scheduler.reschedule(plan)?; // 生成Tasks
    let remote_stage_actions = scheduled_tasks.get_tasks()?; // 获取每个节点和其对应的FlightAction

    let config = ctx.get_config();
    let cluster = ctx.get_cluster();
    let timeout = ctx.get_settings().get_flight_client_timeout()?;
    let mut scheduled = Scheduled::new();
    for (node, action) in remote_stage_actions { // 执行远程节点任务
        let mut flight_client = cluster.create_node_conn(&node.id, &config).await?; // 创建链接
        let executing_action = flight_client.execute_action(action.clone(), timeout); // 在各节点执行action

        executing_action.await?;
        scheduled.insert(node.id.clone(), node.clone());
    }

    let pipeline_builder = PipelineBuilder::create(ctx.clone());
    let mut in_local_pipeline = pipeline_builder.build(&scheduled_tasks.get_local_task())?; // 创建本地节点pipeline，这里会创建RemoteTransform，从其它节点获取数据

    match in_local_pipeline.execute().await { // 执行本地节点任务
        Ok(stream) => Ok(ScheduledStream::create(ctx.clone(), scheduled, stream)),
        Err(error) => {
            plan_schedulers::handle_error(ctx, scheduled, timeout).await;
            Err(error)
        }
    }
}
```

#### 调度

##### PlanScheduler

```rust
// file: query/src/interpreters/plan_schedulers/plan_scheduler.rs
pub struct PlanScheduler {
    stage_id: String,
    cluster_nodes: Vec<String>, // 集群节点名称

    local_pos: usize, // 本地节点名称的序号
    nodes_plan: Vec<PlanNode>, // 记录每个节点临时的PlanNode，在Visit的时候对其进行update，并生成Task。
    running_mode: RunningMode, // 执行模式，单机或集群
    query_context: Arc<QueryContext>,
    subqueries_expressions: Vec<Expressions>,
}
// func: reschedule(...) -> Result<Tasks>
// 根据PlanNode生成Tasks
    pub fn reschedule(mut self, plan: &PlanNode) -> Result<Tasks> {
        let context = self.query_context.clone();
        let cluster = context.get_cluster();
        let mut tasks = Tasks::create(context); // 初始化Tasks

        match cluster.is_empty() {
            true => tasks.finalize(plan), // 无其它节点
            false => {
                self.visit_plan_node(plan, &mut tasks)?;
                tasks.finalize(&self.nodes_plan[self.local_pos])
            }
        }
    }
// func: visit_plan_node(...) 
    fn visit_plan_node(&mut self, node: &PlanNode, tasks: &mut Tasks) -> Result<()> {
        match node {
            PlanNode::AggregatorPartial(plan) => self.visit_aggr_part(plan, tasks),
            PlanNode::AggregatorFinal(plan) => self.visit_aggr_final(plan, tasks),
            PlanNode::Empty(plan) => self.visit_empty(plan, tasks),
            PlanNode::Projection(plan) => self.visit_projection(plan, tasks),
            PlanNode::Filter(plan) => self.visit_filter(plan, tasks),
            PlanNode::Sort(plan) => self.visit_sort(plan, tasks),
            PlanNode::Limit(plan) => self.visit_limit(plan, tasks),
            PlanNode::LimitBy(plan) => self.visit_limit_by(plan, tasks),
            PlanNode::ReadSource(plan) => self.visit_data_source(plan, tasks),
            PlanNode::Sink(plan) => self.visit_sink(plan, tasks),
            PlanNode::Select(plan) => self.visit_select(plan, tasks),
            PlanNode::Stage(plan) => self.visit_stage(plan, tasks),         // 这里会创建FlightAction
            PlanNode::Broadcast(plan) => self.visit_broadcast(plan, tasks), // 这里会创建FlightAction
            PlanNode::Having(plan) => self.visit_having(plan, tasks),
            PlanNode::Expression(plan) => self.visit_expression(plan, tasks),
            PlanNode::SubQueryExpression(plan) => self.visit_subqueries_set(plan, tasks),
            _ => Err(ErrorCode::UnImplement("")),
        }
    }
// func: visit_limit(...)
    fn visit_limit(&mut self, plan: &LimitPlan, tasks: &mut Tasks) -> Result<()> {
        self.visit_plan_node(plan.input.as_ref(), tasks)?; // 先遍历孩子节点
        match self.running_mode {
            RunningMode::Cluster => self.visit_cluster_limit(plan), // 集群模式
            RunningMode::Standalone => self.visit_local_limit(plan), // 单机模式
        };
        Ok(())
    }
// func: visit_local_limit(...)
    fn visit_local_limit(&mut self, plan: &LimitPlan) {
        self.nodes_plan[self.local_pos] = PlanNode::Limit(LimitPlan {
            n: plan.n,
            offset: plan.offset,
            input: Arc::new(self.nodes_plan[self.local_pos].clone()), // 只处理当前节点的PlanNode
        }); // 更新节点的PlanNode
    }
// func: visit_cluster_limit(...)
    fn visit_cluster_limit(&mut self, plan: &LimitPlan) {
        for index in 0..self.nodes_plan.len() { // 处理所有节点的PlanNode
            self.nodes_plan[index] = PlanNode::Limit(LimitPlan {
                n: plan.n,
                offset: plan.offset,
                input: Arc::new(self.nodes_plan[index].clone()),
            }); // 更新节点的PlanNode
        }
    }
// func: visit_stage(...)
// 处理StagePlan
    fn visit_stage(&mut self, stage: &StagePlan, tasks: &mut Tasks) -> Result<()> {
        self.visit_plan_node(stage.input.as_ref(), tasks)?;

        // Entering new stage
        self.stage_id = uuid::Uuid::new_v4().to_string();

        match stage.kind {
            StageKind::Normal => self.schedule_normal_tasks(stage, tasks),
            StageKind::Expansive => self.schedule_expansive_tasks(stage, tasks),
            StageKind::Convergent => self.schedule_converge_tasks(stage, tasks),
        }
    }
// func: schedule_normal_tasks(...)
    fn schedule_normal_tasks(&mut self, stage: &StagePlan, tasks: &mut Tasks) -> Result<()> {
        if let RunningMode::Standalone = self.running_mode { // 必须是集群
            return Err(ErrorCode::LogicalError(
                "Normal stage cannot work on standalone mode",
            ));
        }

        for index in 0..self.nodes_plan.len() { // 遍历所有节点
            let node_name = &self.cluster_nodes[index]; // 获取节点名
            let shuffle_action = self.normal_action(stage, &self.nodes_plan[index]); // 创建ShuffleAction
            let remote_plan_node = self.normal_remote_plan(node_name, &shuffle_action); // 创建RemotePlan
            let shuffle_flight_action = FlightAction::PrepareShuffleAction(shuffle_action); // 创建FlightAction

            tasks.add_task(node_name, shuffle_flight_action); // 加入tasks，启动任务时会用
            self.nodes_plan[index] = PlanNode::Remote(remote_plan_node); // 更新节点的PlanNode
        }

        Ok(())
    }
// func: schedule_expansive_tasks() 
    fn schedule_expansive_tasks(&mut self, stage: &StagePlan, tasks: &mut Tasks) -> Result<()> {
        if let RunningMode::Cluster = self.running_mode { // 必须是单节点
            return Err(ErrorCode::LogicalError(
                "Expansive stage cannot work on Cluster mode",
            ));
        }

        self.running_mode = RunningMode::Cluster;
        let node_name = &self.cluster_nodes[self.local_pos]; // 本地节点名称
        let shuffle_action = self.expansive_action(stage, &self.nodes_plan[self.local_pos]); // 创建ShuffleAction
        tasks.add_task(
            node_name,
            FlightAction::PrepareShuffleAction(shuffle_action.clone()),
        ); // 加入tasks，启动任务时会用

        for index in 0..self.nodes_plan.len() {
            let node_name = &self.cluster_nodes[index];
            self.nodes_plan[index] = self.expansive_remote_plan(node_name, &shuffle_action); // 更新节点的PlanNode
        }

        Ok(())
    }
// func: schedule_converge_tasks(...)
    fn schedule_converge_tasks(&mut self, stage: &StagePlan, tasks: &mut Tasks) -> Result<()> {
        if let RunningMode::Standalone = self.running_mode { // 必须是集群
            return Err(ErrorCode::LogicalError(
                "Converge stage cannot work on standalone mode",
            ));
        }

        for index in 0..self.nodes_plan.len() {
            let node_name = &self.cluster_nodes[index]; // 节点名
            let shuffle_action = self.converge_action(stage, &self.nodes_plan[index]); // 创建ShuffleAction
            let shuffle_flight_action = FlightAction::PrepareShuffleAction(shuffle_action); // 创建FlightAction

            tasks.add_task(node_name, shuffle_flight_action);  // 加入tasks，启动任务时会用
        }

        self.running_mode = RunningMode::Standalone;
        let node_name = &self.cluster_nodes[self.local_pos];
        let remote_plan_node = self.converge_remote_plan(node_name, stage);
        self.nodes_plan[self.local_pos] = PlanNode::Remote(remote_plan_node); // 更新节点的PlanNode

        Ok(())
    }
// func: visit_broadcast(...)
// 处理BroadcastPlan
    fn visit_broadcast(&mut self, plan: &BroadcastPlan, tasks: &mut Tasks) -> Result<()> {
        self.visit_plan_node(plan.input.as_ref(), tasks)?;

        // Entering new stage
        self.stage_id = uuid::Uuid::new_v4().to_string();

        match self.running_mode {
            RunningMode::Cluster => self.visit_cluster_broadcast(tasks),
            RunningMode::Standalone => self.visit_local_broadcast(tasks),
        };

        Ok(())
    }
// func: visit_cluster_broadcast(...)
    fn visit_cluster_broadcast(&mut self, tasks: &mut Tasks) {
        self.running_mode = RunningMode::Cluster;
        for index in 0..self.nodes_plan.len() {
            let node_name = &self.cluster_nodes[index];
            let action = self.broadcast_action(&self.nodes_plan[index]);
            let remote_plan_node = self.broadcast_remote(node_name, &action);

            tasks.add_task(node_name, FlightAction::BroadcastAction(action));
            self.nodes_plan[index] = PlanNode::Remote(remote_plan_node);
        }
    }
```

##### Tasks

```rust
// file: query/src/interpreters/plan_schedulers/plan_scheduler.rs
pub struct Tasks {
    plan: PlanNode,
    context: Arc<QueryContext>,
    actions: HashMap<String, VecDeque<FlightAction>>, // 一个节点可能有多个FlightAction
}
// func: finalize（...）
// 设置本地节点PlanNode
    pub fn finalize(mut self, plan: &PlanNode) -> Result<Self> {
        self.plan = plan.clone();
        Ok(self)
    }
// func: get_tasks() -> Result<Vec<(Arc<NodeInfo>, FlightAction)>>
// 获取集群各节点对应的FlightAction
    pub fn get_tasks(&self) -> Result<Vec<(Arc<NodeInfo>, FlightAction)>> {
        let cluster = self.context.get_cluster();

        let mut tasks = Vec::new();
        for cluster_node in &cluster.get_nodes() {
            if let Some(actions) = self.actions.get(&cluster_node.id) {
                for action in actions {
                    tasks.push((cluster_node.clone(), action.clone()));
                }
            }
        }

        Ok(tasks)
    }
// func: add_task(...) ⭐️
// 加入节点对应的FlightAction，会被visit_stage()、visit_local_broadcast()、visit_cluster_broadcast()函数调用
    #[allow(clippy::ptr_arg)]
    pub fn add_task(&mut self, node_name: &String, action: FlightAction) {
        match self.actions.entry(node_name.to_string()) {
            Entry::Occupied(mut entry) => entry.get_mut().push_back(action), // 加入已有的队列中
            Entry::Vacant(entry) => {
                let mut node_tasks = VecDeque::new(); // 创建新的队列
                node_tasks.push_back(action);
                entry.insert(node_tasks);
            }
        };
    }
```

##### PipelineBuilder

```rust
// file: query/src/pipelines/processors/pipeline_builder.rs
// func: build(...) -> Result<Pipeline>
    pub fn build(mut self, node: &PlanNode) -> Result<Pipeline> {
        let pipeline = self.visit(node)?;
        Ok(pipeline)
    }
// func: visit(...)
    fn visit(&mut self, node: &PlanNode) -> Result<Pipeline> {
        match node {
            ...
            PlanNode::Remote(node) => self.visit_remote(node),
            ...
        }
    }
// func: visit_remote(...) -> Result<Pipeline>
    fn visit_remote(&self, plan: &RemotePlan) -> Result<Pipeline> {
        let mut pipeline = Pipeline::create(self.ctx.clone());

        for fetch_node in &plan.fetch_nodes { // 遍历其它节点
            let flight_ticket =
                FlightTicket::stream(&plan.query_id, &plan.stage_id, &plan.stream_id); // 创建FlightTicket，用于创建Request

            pipeline.add_source(Arc::new(RemoteTransform::try_create( // 创建RemoteTransform，并作为Source加入pipeline
                flight_ticket,
                self.ctx.clone(),
                /* fetch_node_name */ fetch_node.clone(),
                /* fetch_stream_schema */ plan.schema.clone(),
            )?))?;
        }

        Ok(pipeline)
    }
```

#### 执行

##### RemoteTransform

```rust
// file: query/src/pipelines/transforms/transform_remote.rs
pub struct RemoteTransform {
    ticket: FlightTicket,
    fetch_node_name: String,
    schema: DataSchemaRef,
    pub ctx: Arc<QueryContext>,
}
// func: execute() -> Result<SendableDataBlockStream>
// 执行
    async fn execute(&self) -> Result<SendableDataBlockStream> {
        tracing::debug!(
            "execute, flight_ticket {:?}, node name:{:#}...",
            self.ticket,
            self.fetch_node_name
        );

        let data_schema = self.schema.clone();
        let timeout = self.ctx.get_settings().get_flight_client_timeout()?;

        let fetch_ticket = self.ticket.clone();
        let mut flight_client = self.flight_client().await?; // 创建Cliet
        let fetch_stream = flight_client
            .fetch_stream(fetch_ticket, data_schema, timeout)
            .await?; // 从远端获取数据流
        Ok(Box::pin(self.ctx.try_create_abortable(fetch_stream)?))
    }
// func: flight_client() -> Result<FlightClient>
// 创建FlightClient
    async fn flight_client(&self) -> Result<FlightClient> {
        let context = self.ctx.clone();
        let node_name = self.fetch_node_name.clone();

        let cluster = context.get_cluster();
        cluster
            .create_node_conn(&node_name, &self.ctx.get_config())
            .await
    }
```

##### FlightAction

```rust
// file: query/src/api/rpc/flight_actions.rs
pub enum FlightAction {
    PrepareShuffleAction(ShuffleAction),
    BroadcastAction(BroadcastAction),
    CancelAction(CancelAction),
}
```

##### FlightTicket

记录节点和query任务信息，从远程节点获取对应的数据流。FlightClient -> (FlightTicket -> Ticket) -> DatabendQueryFlightService

```rust
// file: query/src/api/rpc/flight_tickets.rs
pub enum FlightTicket {
    StreamTicket(StreamTicket),
}
impl TryInto<FlightTicket> for Ticket {...}
impl TryInto<Ticket> for FlightTicket {...}

pub struct StreamTicket {
    pub query_id: String,
    pub stage_id: String,
    pub stream: String,
}
```

##### FlightClient

两个功能：

* 启动远程节点的查询任务，execute_action(FlightAction) -> do_action(Action)
* 获取远程节点的数据流，fetch_stream(FlightTicket) -> do_get(Ticket) -> Streaming\<FlightData\> -> SendableDataBlockStream

```rust
// file: query/src/api/rpc/flight_client.rs
pub struct FlightClient {
    inner: FlightServiceClient<Channel>,
}
// func: new() -> Self
// 创建FlightClient
    pub fn new(inner: FlightServiceClient<Channel>) -> FlightClient {
        FlightClient { inner }
    }
// func: execute_action()
// 执行查询计划
    pub async fn execute_action(&mut self, action: FlightAction, timeout: u64) -> Result<()> {
        self.do_action(action, timeout).await?;
        Ok(())
    }
    async fn do_action(&mut self, action: FlightAction, timeout: u64) -> Result<Vec<u8>> {
        let action: Action = action.try_into()?;
        let action_type = action.r#type.clone();
        let request = Request::new(action);
        let mut request = common_tracing::inject_span_to_tonic_request(request);
        request.set_timeout(Duration::from_secs(timeout));

        let response = self.inner.do_action(request).await?;

        match response.into_inner().message().await? {
            Some(response) => Ok(response.body),
            None => Result::Err(ErrorCode::EmptyDataFromServer(format!(
                "Can not receive data from flight server, action: {action_type:?}",
            ))),
        }
    }
// func: fetch_stream() -> Result<SendableDataBlockStream>
// 获取数据流
    pub async fn fetch_stream(
        &mut self,
        ticket: FlightTicket,
        schema: DataSchemaRef,
        timeout: u64,
    ) -> Result<SendableDataBlockStream> {
        let ticket = ticket.try_into()?;
        let inner = self.do_get(ticket, timeout).await?;
        Ok(Box::pin(FlightDataStream::from_remote(schema, inner)))
    }
    async fn do_get(&mut self, ticket: Ticket, timeout: u64) -> Result<Streaming<FlightData>> {
        let request = Request::new(ticket);
        let mut request = common_tracing::inject_span_to_tonic_request(request);
        request.set_timeout(Duration::from_secs(timeout));

        let response = self.inner.do_get(request).await?;
        Ok(response.into_inner())
    }
```

##### DatabendQueryFlightService

```rust
// file: query/src/api/rpc/flight_service.rs
pub struct DatabendQueryFlightService {
    sessions: Arc<SessionManager>,
    dispatcher: Arc<DatabendQueryFlightDispatcher>,
}
```

详细逻辑见《9 RPC API Service》
