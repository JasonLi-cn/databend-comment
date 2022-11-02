* [六 Join](#六-join)
  * [Build Pipeline](#build-pipeline)
    * [V2](#v2)
      * [PhysicalPlan](#physicalplan)
      * [HashJoin](#hashjoin)
      * [JoinType](#jointype)
      * [SelectInterpreterV2](#selectinterpreterv2)
      * [PipelineBuilder](#pipelinebuilder)
      * [JoinHashTable](#joinhashtable)
      * [SinkBuildHashTable](#sinkbuildhashtable)
      * [TransformHashJoinProbe](#transformhashjoinprobe)

## 六 Join

### Build Pipeline

#### V2

##### PhysicalPlan

```rust
// file: src/query/service/src/sql/executor/physical_plan.rs
pub enum PhysicalPlan {
    TableScan(TableScan),
    Filter(Filter),
    Project(Project),
    EvalScalar(EvalScalar),
    AggregatePartial(AggregatePartial),
    AggregateFinal(AggregateFinal),
    Sort(Sort),
    Limit(Limit),
    HashJoin(HashJoin),
    Exchange(Exchange),
    UnionAll(UnionAll),

    /// Synthesized by fragmenter
    ExchangeSource(ExchangeSource),
    ExchangeSink(ExchangeSink),
}
```

##### HashJoin

```rust
// file: src/query/service/src/sql/executor/physical_plan.rs
pub struct HashJoin {
    pub build: Box<PhysicalPlan>,
    pub probe: Box<PhysicalPlan>,
    pub build_keys: Vec<PhysicalScalar>,
    pub probe_keys: Vec<PhysicalScalar>,
    pub other_conditions: Vec<PhysicalScalar>, // 目前好像没用上
    pub join_type: JoinType,
    pub marker_index: Option<IndexType>,
    pub from_correlated_subquery: bool,
}
```

##### JoinType

```rust
// file: src/query/service/src/sql/planner/plans/logical_join.rs
pub enum JoinType {
    Inner,
    Left,
    Right,
    Full,
    Semi,
    Anti,
    Cross,
    /// Mark Join is a special case of join that is used to process Any subquery and correlated Exists subquery.
    Mark,
    /// Single Join is a special kind of join that is used to process correlated scalar subquery.
    Single,
}
```

##### SelectInterpreterV2

入口，调用流程：

```rust
// file: src/query/service/src/interpreters/interpreter_select_v2.rs
SelectInterpreterV2::execute() -> Self::build_pipeline() -> PhysicalPlanBuilder::new()
                                                         -> PhysicalPlanBuilder::build() // build physical_plan
                                                         -> PipelineBuilder::create()
                                                         -> PipelineBuilder::finalize(physical_plan) // build pipeline
```

##### PipelineBuilder

负责构造Pipeline

```rust
// file: src/query/service/src/sql/executor/pipeline_builder.rs
pub struct PipelineBuilder {
    ctx: Arc<QueryContext>,
    main_pipeline: Pipeline,
    pub pipelines: Vec<Pipeline>,
}
// func: finalize()
    pub fn finalize(mut self, plan: &PhysicalPlan) -> Result<PipelineBuildResult> {
        self.build_pipeline(plan)?; // 构造 Pipeline，把产生的Pipeline放入self.pipelines和self.pipelines

        for source_pipeline in &self.pipelines {
            if !source_pipeline.is_complete_pipeline()? {
                return Err(ErrorCode::IllegalPipelineState(
                    "Source pipeline must be complete pipeline.",
                ));
            }
        }

        Ok(PipelineBuildResult {
            main_pipeline: self.main_pipeline,
            sources_pipelines: self.pipelines,
        })
    }

// func: build_pipeline()
    fn build_pipeline(&mut self, plan: &PhysicalPlan) -> Result<()> {
            ...
            PhysicalPlan::Limit(limit) => self.build_limit(limit),
            PhysicalPlan::HashJoin(join) => self.build_join(join),
            ...
    }

// func: build_join() ⭐️
    fn build_join(&mut self, join: &HashJoin) -> Result<()> {
        let state = self.build_join_state(join)?; // 创建HashMap
        self.expand_build_side_pipeline(&join.build, state.clone())?; // 创建Build表的pipeline
        self.build_join_probe(join, state) // 创建probe的pipeline
    }

// func: build_join_state()
// 创建HashMap
    fn build_join_state(&mut self, join: &HashJoin) -> Result<Arc<JoinHashTable>> {
        JoinHashTable::create_join_state(
            self.ctx.clone(),
            &join.build_keys,
            join.build.output_schema()?,
            HashJoinDesc::create(join)?,
        )
    }

// func: expand_build_side_pipeline()
    fn expand_build_side_pipeline(
        &mut self,
        build: &PhysicalPlan,
        join_state: Arc<JoinHashTable>,
    ) -> Result<()> {
        let build_side_context = QueryContext::create_from(self.ctx.clone());
        let build_side_builder = PipelineBuilder::create(build_side_context);
        let mut build_res = build_side_builder.finalize(build)?;

        assert!(build_res.main_pipeline.is_pulling_pipeline()?);
        let mut sink_pipeline_builder = SinkPipeBuilder::create();
        for _index in 0..build_res.main_pipeline.output_len() {
            let input_port = InputPort::create();
            sink_pipeline_builder.add_sink(
                input_port.clone(),
                Sinker::<SinkBuildHashTable>::create(
                    input_port,
                    SinkBuildHashTable::try_create(join_state.clone())?,
                ),
            );
        }

        build_res
            .main_pipeline
            .add_pipe(sink_pipeline_builder.finalize()); // build side pipeline add sink

        self.pipelines.push(build_res.main_pipeline);    // 把build side main pipeline加入self.pipelines
        self.pipelines
            .extend(build_res.sources_pipelines.into_iter()); // 把build side sources_pipelines加入self.pipelines
        Ok(())
    }

// func: build_join_probe()
// 创建probe side pipeline
    fn build_join_probe(&mut self, join: &HashJoin, state: Arc<JoinHashTable>) -> Result<()> {
        self.build_pipeline(&join.probe)?;

        self.main_pipeline.add_transform(|input, output| { // 把join probe tranform加入self.main_pipeline
            Ok(TransformHashJoinProbe::create(
                self.ctx.clone(),
                input,
                output,
                state.clone(),
                join.output_schema()?,
            ))
        })?;

        if join.join_type == JoinType::Mark {
            self.main_pipeline.resize(1)?;
            self.main_pipeline.add_transform(|input, output| {
                TransformMarkJoin::try_create(
                    input,
                    output,
                    MarkJoinCompactor::create(state.clone()),
                )
            })?;
        }

        Ok(())
    }
```

##### JoinHashTable

```rust
// file: src/query/service/src/pipelines/processors/transforms/hash_join/join_hash_table.rs
pub struct JoinHashTable {
    pub(crate) ctx: Arc<QueryContext>,
    /// Reference count
    ref_count: Mutex<usize>,
    is_finished: Mutex<bool>,
    /// A shared big hash table stores all the rows from build side
    pub(crate) hash_table: RwLock<HashTable>,
    pub(crate) row_space: RowSpace,
    pub(crate) hash_join_desc: HashJoinDesc,
    pub(crate) row_ptrs: RwLock<Vec<RowPtr>>,
    finished_notify: Arc<Notify>,
}
```

##### SinkBuildHashTable

```rust
// file: src/query/service/src/pipelines/processors/transforms/transform_hash_join.rs
pub struct SinkBuildHashTable {
    join_state: Arc<dyn HashJoinState>,
}

// func: try_create()
    pub fn try_create(join_state: Arc<dyn HashJoinState>) -> Result<Self> {
        join_state.attach()?;
        Ok(Self { join_state })
    }
// implement Sink
impl Sink for SinkBuildHashTable {
    const NAME: &'static str = "BuildHashTable";

    fn on_finish(&mut self) -> Result<()> {
        self.join_state.detach()
    }

    fn consume(&mut self, data_block: DataBlock) -> Result<()> {
        self.join_state.build(data_block)
    }
}
```

##### TransformHashJoinProbe

```rust
// file: src/query/service/src/pipelines/processors/transforms/transform_hash_join.rs
pub struct TransformHashJoinProbe {
    input_data: Option<DataBlock>,
    output_data_blocks: VecDeque<DataBlock>,

    input_port: Arc<InputPort>,
    output_port: Arc<OutputPort>,
    step: HashJoinStep,
    join_state: Arc<dyn HashJoinState>,
    probe_state: ProbeState,
}

// func: probe()
    fn probe(&mut self, block: &DataBlock) -> Result<()> {
        self.probe_state.clear();
        self.output_data_blocks
            .extend(self.join_state.probe(block, &mut self.probe_state)?); // 调用JoinHashTable的probe函数
        Ok(())
    }

impl Processor for TransformHashJoinProbe {
// func: event()
// 输出驱动
    fn event(&mut self) -> Result<Event> {
        match self.step {
            HashJoinStep::Build => Ok(Event::Async),
            HashJoinStep::Probe => {
                // 如果output_port结束，则结束input_port，同时返回Event::Finished
                if self.output_port.is_finished() {
                    self.input_port.finish();
                    return Ok(Event::Finished);
                }
                // 如果output_port当前不能接受数据，则告知input_port当前不需要数据，同时返回Event::NeedConsume，
                // 调度器会执行下游算子
                if !self.output_port.can_push() {
                    self.input_port.set_not_need_data();
                    return Ok(Event::NeedConsume);
                }
                // 如果当前算子的输出缓存不为空，则把缓存中的数据push到output_port，同时返回Event::NeedConsume
                if !self.output_data_blocks.is_empty() {
                    let data = self.output_data_blocks.pop_front().unwrap();
                    self.output_port.push_data(Ok(data));
                    return Ok(Event::NeedConsume);
                }
                // 如果当前输入缓存不为空，则返回Event::Sync，说明需要处理当前的输入数据。
                if self.input_data.is_some() {
                    return Ok(Event::Sync);
                }
                // 如果input_port有数据，则从中pull数据，并放到self.input_data，返回Event::Sync，等待下一步处理
                if self.input_port.has_data() {
                    let data = self.input_port.pull_data().unwrap()?;
                    self.input_data = Some(data);
                    return Ok(Event::Sync);
                }
                // 如果input_port结束了，则结束output_port，并返回Event::Finished
                if self.input_port.is_finished() {
                    self.output_port.finish();
                    return Ok(Event::Finished);
                }
                // 设置input_port需要数据，从而其对应的上游的output_port调用can_push时，就会返回true.
                self.input_port.set_need_data();
                Ok(Event::NeedData)
            }
        }
    }
// func: process()
// 处理数据，这里主要是进行probe
    fn process(&mut self) -> Result<()> {
        match self.step {
            HashJoinStep::Build => Ok(()),
            HashJoinStep::Probe => {
                if let Some(data) = self.input_data.take() {
                    self.probe(&data)?;
                }
                Ok(())
            }
        }
    }
// func: async_process()
// 等待build表构建完毕
    async fn async_process(&mut self) -> Result<()> {
        if let HashJoinStep::Build = &self.step {
            self.join_state.wait_finish().await?;
            self.step = HashJoinStep::Probe;
        }

        Ok(())
    }
}
```
