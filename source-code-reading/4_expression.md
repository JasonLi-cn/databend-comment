* [四 Expression](#四-expression)
  * [V2](#v2)
    * [Expr](#expr)
    * [ScalarBinder](#scalarbinder)
    * [Scalar](#scalar)
    * [ExpressionBuilder](#expressionbuilder)
    * [Expression](#expression)
    * [ExpressionExecutor](#expressionexecutor)
    * [ExpressionChain](#expressionchain)
    * [ExpressionAction](#expressionaction)
    * [Function](#function)
    * [FunctionFactory](#functionfactory)

## 四 Expression

#### V2

```rust
let mut planner = Planner::new(ctx.clone());
let (plan_node, _) = planner.plan_sql(sql).await?;

// 从sql创建逻辑计划的流程
Planner::plan_sql(sql: &str) -> Plan                                             // query/src/sql/planner/mod.rs
    -> tokenize_sql(&str) -> Token                                               // common/ast/src/parser/mod.rs
       parse_sql(Token) -> Vec<Statement>                                        // common/ast/src/parser/mod.rs
        -> statements(Token) -> Vec<Statement>                                   // common/ast/src/parser/statement.rs
            -> query(Token) -> Query                                             // common/ast/src/parser/query.rs
                -> select_target(Token) -> SelectTarget                          // common/ast/src/parser/query.rs
                    -> expr(Token) -> Expr                                       // common/ast/src/parser/expr.rs
                        -> subexpr()                                             // common/ast/src/parser/expr.rs
       Binder::bind(Statement) -> Plan                                           // query/src/sql/planner/mod.rs
           -> bind_statement(Statement) -> Plan                                  // query/src/sql/planner/binder/mod.rs
               -> bind_query(Query) -> SExpr                                     // query/src/sql/planner/binder/select.rs
                      -> bind_select_stmt(SelectStmt, OrderByExpr) -> SExpr      // query/src/sql/planner/binder/select.rs
                          -> bind_where(Expr, SExpr) -> SExpr                    // query/src/sql/planner/binder/select.rs
                              -> ScalarBinder::bind(Expr) -> Scalar              // query/src/sql/planner/binder/select.rs
                                 Scalar -> FilterPlan impl Operator              // query/src/sql/planner/binder/select.rs
                                 SExpr::create_unary(Plan) -> SExpr              // query/src/sql/optimizer/s_expr.rs
                  SubqueryRewriter::rewrite(SExpr) -> SExpr                      // query/src/sql/planner/binder/subquery.rs
                  SExpr -> Plan::Query                                           // query/src/sql/planner/binder/subquery.rs
```

```shell
str -> Token -> ExprElement ->    WithSpan    ->    Expr    ->    Scalar  |   Expression    ->    ExpressionChain    ->    ExpressionAction
                                         ExprParser     ScalarBinder      |   ExpressionBuilder
                                                Before build pipeline     |   Building pipeline
```

##### Expr

AST中的表达式

```rust
// file: common/ast/src/ast/expr.rs
pub enum Expr<'a> {
    /// Column reference, with indirection like `table.column`
    ColumnRef {
        span: &'a [Token<'a>],
        database: Option<Identifier<'a>>,
        table: Option<Identifier<'a>>,
        column: Identifier<'a>,
    },
    ...

    /// `BETWEEN ... AND ...`
    Between {
        span: &'a [Token<'a>],
        expr: Box<Expr<'a>>,
        low: Box<Expr<'a>>,
        high: Box<Expr<'a>>,
        not: bool,
    },
    /// Binary operation
    BinaryOp {
        span: &'a [Token<'a>],
        op: BinaryOperator,
        left: Box<Expr<'a>>,
        right: Box<Expr<'a>>,
    },
    ...
}

pub enum Literal {
    Integer(u64),
    Float(f64),
    BigInt { lit: String, is_hex: bool },
    // Quoted string literal value
    String(String),
    Boolean(bool),
    CurrentTimestamp,
    Null,
}

#[derive(Debug, Clone, PartialEq)]
pub enum TypeName {
    Boolean,
    UInt8,
    UInt16,
    UInt32,
    UInt64,
    Int8,
    Int16,
    Int32,
    Int64,
    Float32,
    Float64,
    Date,
    DateTime { precision: Option<u64> },
    Timestamp,
    String,
    Array { item_type: Option<Box<TypeName>> },
    Object,
    Variant,
}

pub enum BinaryOperator {
    Plus,
    Minus,
    Multiply,
    Div,
    Divide,
    Modulo,
    StringConcat,
    // `>` operator
    Gt,
    // `<` operator
    Lt,
    ...
}
```

##### ScalarBinder

Expr -> Scalar

```rust
// file: query/src/sql/planner/binder/scalar.rs
    pub async fn bind(&mut self, expr: &Expr<'a>) -> Result<(Scalar, DataTypeImpl)> {
        let mut type_checker =
            TypeChecker::new(self.bind_context, self.ctx.clone(), self.metadata.clone());
        type_checker.resolve(expr, None).await
    }
```

##### Scalar

```rust
// file: query/src/sql/planner/plans/scalar.rs
pub enum Scalar {
    BoundColumnRef(BoundColumnRef),
    ConstantExpr(ConstantExpr),
    AndExpr(AndExpr),
    OrExpr(OrExpr),
    ComparisonExpr(ComparisonExpr),
    AggregateFunction(AggregateFunction),
    FunctionCall(FunctionCall),
    // TODO(leiysky): maybe we don't need this variant any more
    // after making functions static typed?
    Cast(CastExpr),
    SubqueryExpr(SubqueryExpr),
}
```

##### ExpressionBuilder

Scalar -> Expression

```rust
// file: query/src/sql/exec/expression_builder.rs
// 在PipelineBuilder中被用于创建Expression
pub struct ExpressionBuilder {
    metadata: MetadataRef,
}
// func: create(...) -> ExpressionBuilder
// 创建ExpressionBuilder
    pub fn create(metadata: MetadataRef) -> Self {
        ExpressionBuilder { metadata }
    }
// func: build(...) -> Expression
// 生成Expression
    pub fn build(&self, scalar: &Scalar) -> Result<Expression> {
        match scalar {
            Scalar::BoundColumnRef(BoundColumnRef { column }) => {
                self.build_column_ref(column.index)
            }
            Scalar::ConstantExpr(ConstantExpr { value, data_type }) => {
                self.build_literal(value, data_type)
            }
            ...
        }  
    }
```

##### Expression

```rust
// file: common/planners/src/plan_expression.rs
#[derive(serde::Serialize, serde::Deserialize, Clone, PartialEq)]
pub enum Expression {
    /// An expression with a alias name.
    Alias(String, Box<Expression>),

    /// Column name.
    Column(String),
    /// Qualified column name.
    QualifiedColumn(Vec<String>),
  
    /// Constant value.
    /// Note: When literal represents a column, its column_name will not be None
    Literal {
        value: DataValue,
        column_name: Option<String>,

        // Logic data_type for this literal
        data_type: DataTypeImpl,
    },

    ...
}
```

##### ExpressionExecutor

```rust
// file: query/src/pipelines/transforms/transform_expression_executor.rs
// 表达式执行器，在TransformFilter和TransformHaving等地方会使用
pub struct ExpressionExecutor {
    // description of this executor
    description: String,
    _input_schema: DataSchemaRef,
    output_schema: DataSchemaRef,
    chain: Arc<ExpressionChain>,
    // whether to perform alias action in executor
    alias_project: bool,
    ctx: Arc<QueryContext>,
}
// func: try_create() -> ExpressionExecutor
// 创建ExpressionExecutor
    ...
    let chain = ExpressionChain::try_create(input_schema.clone(), &exprs)?; // 创建ExpressionChain
// func: execute(...) -> DataBlock
    pub fn execute(&self, block: &DataBlock) -> Result<DataBlock> {
        ...

        let mut column_map: HashMap<&str, ColumnWithField> = HashMap::new(); // 存放参与运算的列，以及缓存临时查询结果

        let mut alias_map: HashMap<&str, &ColumnWithField> = HashMap::new(); // 如果alias_project等于true，则用于存储查询结果列

        // supported a + 1 as b, a + 1 as c
        // supported a + 1 as a, a as b
        // !currently not supported a+1 as c, b+1 as c
        let mut alias_action_map: HashMap<&str, Vec<&str>> = HashMap::new(); // 记录alias的映射

        for f in block.schema().fields().iter() {
            let column =
                ColumnWithField::new(block.try_column_by_name(f.name())?.clone(), f.clone());
            column_map.insert(f.name(), column);
        }

        let rows = block.num_rows(); // 当前DataBlock行数
        for action in self.chain.actions.iter() {
            if let ExpressionAction::Alias(alias) = action {
                if let Some(v) = alias_action_map.get_mut(alias.arg_name.as_str()) {
                    v.push(alias.name.as_str());
                } else {
                    alias_action_map.insert(alias.arg_name.as_str(), vec![alias.name.as_str()]);
                }
            }

            if column_map.contains_key(action.column_name()) { // 如果缓存中已经包含这个action的结果，则跳过
                continue;
            }

            match action {
                ExpressionAction::Input(input) => {
                    let column = block.try_column_by_name(&input.name)?.clone();
                    let column = ColumnWithField::new(
                        column,
                        block.schema().field_with_name(&input.name)?.clone(),
                    );
                    column_map.insert(input.name.as_str(), column);
                }
                ExpressionAction::Function(f) => {
                    let column_with_field = self.execute_function(&mut column_map, f, rows)?;
                    column_map.insert(f.name.as_str(), column_with_field);
                }
                ExpressionAction::Constant(constant) => {
                    let column = constant
                        .data_type
                        .create_constant_column(&constant.value, rows)?;

                    let column = ColumnWithField::new(
                        column,
                        DataField::new(constant.name.as_str(), constant.data_type.clone()),
                    );

                    column_map.insert(constant.name.as_str(), column);
                }
                _ => {}
            }
        }
  
        // 如果是alias，把结果放入alias_map
        if self.alias_project {
            for (k, v) in alias_action_map.iter() {
                let column = column_map.get(k).ok_or_else(|| {
                    ErrorCode::LogicalError("Arguments must be prepared before alias transform")
                })?;

                for name in v.iter() {
                    match alias_map.insert(name, column) {
                        Some(_) => Err(ErrorCode::UnImplement(format!(
                            "Duplicate alias name :{}",
                            name
                        ))),
                        _ => Ok(()),
                    }?;
                }
            }
        }

        let mut project_columns = Vec::with_capacity(self.output_schema.fields().len()); // 提取计算结果
        for f in self.output_schema.fields() {
            let column = match alias_map.get(f.name().as_str()) { // 优先从alias_map中获取结果
                Some(data_column) => data_column,
                None => column_map.get(f.name().as_str()).ok_or_else(|| { // 从column_map中获取结果
                    ErrorCode::LogicalError(format!(
                        "Projection column: {} not exists in {:?}, there are bugs!",
                        f.name(),
                        column_map.keys()
                    ))
                })?,
            };
            project_columns.push(column.column().clone());
        }
        // projection to remove unused columns
        Ok(DataBlock::create(
            self.output_schema.clone(),
            project_columns,
        ))
    }
// func: execute_function() -> ColumnWithField
// 执行Function
    fn execute_function(
        &self,
        column_map: &mut HashMap<&str, ColumnWithField>,
        f: &ActionFunction,
        rows: usize,
    ) -> Result<ColumnWithField> {
        // check if it's cached
        let mut arg_columns = Vec::with_capacity(f.arg_names.len());

        for arg in f.arg_names.iter() {
            let column = column_map.get(arg.as_str()).cloned().ok_or_else(|| {
                ErrorCode::LogicalError("Arguments must be prepared before function transform")
            })?;
            arg_columns.push(column);
        }

        let tz = self.ctx.get_settings().get_timezone()?;
        let tz = String::from_utf8(tz).map_err(|_| {
            ErrorCode::LogicalError("Timezone has beeen checked and should be valid.")
        })?;
        let tz = tz.parse::<Tz>().map_err(|_| {
            ErrorCode::InvalidTimezone("Timezone has been checked and should be valid")
        })?;
        let func_ctx = FunctionContext { tz };
        let column = f.func.eval(func_ctx, &arg_columns, rows)?; // 执行funcion
        Ok(ColumnWithField::new(
            column,
            DataField::new(&f.name, f.return_type.clone()),
        ))
    }
```

##### ExpressionChain

```rust
// file: common/planners/src/plan_expression_chain.rs
pub struct ExpressionChain {
    // input schema
    pub schema: DataSchemaRef,
    pub actions: Vec<ExpressionAction>,
}
// func: try_create(...) -> ExpressionChain
// 创建ExpressionChain
    pub fn try_create(schema: DataSchemaRef, exprs: &[Expression]) -> Result<Self> {
        let mut chain = Self {
            schema,
            actions: vec![],
        };

        for expr in exprs {
            chain.recursion_add_expr(expr)?;
        }

        Ok(chain)
    }
// func: recursion_add_expr(...)
// 递归填加表达式
    fn recursion_add_expr(&mut self, expr: &Expression) -> Result<()> {
        struct ExpressionActionVisitor(*mut ExpressionChain);

        impl ExpressionVisitor for ExpressionActionVisitor {
            fn pre_visit(self, _expr: &Expression) -> Result<Recursion<Self>> {
                Ok(Recursion::Continue(self))
            }

            fn post_visit(self, expr: &Expression) -> Result<Self> {
                unsafe {
                    (*self.0).add_expr(expr)?;
                    Ok(self)
                }
            }
        }

        ExpressionActionVisitor(self).visit(expr)?;
        Ok(())
    }
// func: add_expr()
// 往self.actions中填加Action
    fn add_expr(&mut self, expr: &Expression) -> Result<()> {
            ...
            Expression::BinaryExpression { op, left, right } => {
                let arg_types = vec![
                    left.to_data_type(&self.schema)?,
                    right.to_data_type(&self.schema)?,
                ];

                let arg_types2: Vec<&DataTypeImpl> = arg_types.iter().collect();
                let func = FunctionFactory::instance().get(op, &arg_types2)?; // 根据name获取Function
                let return_type = func.return_type();

                let function = ActionFunction {
                    name: expr.column_name(),
                    func_name: op.clone(),
                    func,
                    arg_names: vec![left.column_name(), right.column_name()],
                    arg_types,
                    return_type,
                };

                self.actions.push(ExpressionAction::Function(function));
            } 
            ...
    }
```

SQL:

`SELECT number as c0 FROM numbers_mt(10000) where number = 10 limit 10`

FilterTransform的ExpressionChain为：

```rust
[
    Constant(ActionConstant { name: "10", value: 10, data_type: UInt8(UInt8) }), 
    Input(ActionInput { name: "number", return_type: UInt64(UInt64) }), 
    Function(ActionFunction { name: "(number = 10)", func_name: "=", return_type: Boolean(Boolean), arg_names: ["number", "10"], arg_types: [UInt64(UInt64), UInt8(UInt8)] })
]
```

ProjectionTransform的ExpressionChain为：

```rust
[
    Input(ActionInput { name: "number", return_type: UInt64(UInt64) }), 
    Alias(ActionAlias { name: "c0", arg_name: "number", arg_type: UInt64(UInt64) })
]
```

##### ExpressionAction

```rust
// file: common/planners/src/plan_expression_action.rs
// Expression类型，细节见上述示例
pub enum ExpressionAction {
    /// Column which must be in input.
    Input(ActionInput),
    /// Constant column with known value.
    Constant(ActionConstant),
    Alias(ActionAlias),
    Function(ActionFunction),
}

#[derive(Debug, Clone)]
pub struct ActionInput {
    pub name: String, // 输入输出列名
    pub return_type: DataTypeImpl,
}

#[derive(Debug, Clone)]
pub struct ActionConstant {
    pub name: String, // 输入输出列名
    pub value: DataValue,
    pub data_type: DataTypeImpl,
}

#[derive(Debug, Clone)]
pub struct ActionAlias {
    pub name: String, // 输出列名
    pub arg_name: String, // 输入列名
    pub arg_type: DataTypeImpl,
}

#[derive(Clone)]
pub struct ActionFunction {
    pub name: String, // 输出列名
    pub func_name: String, // 函数名，eg: = 
    pub return_type: DataTypeImpl, // 输出类型
    pub func: Box<dyn Function>, // 函数

    // for functions
    pub arg_names: Vec<String>,
    pub arg_types: Vec<DataTypeImpl>,
}

```

##### Function

```rust
// file: common/functions/src/scalars/function.rs
pub trait Function: fmt::Display + Sync + Send + DynClone {
    /// Returns the name of the function, should be unique.
    fn name(&self) -> &str;

    /// Calculate the monotonicity from arguments' monotonicity information.
    /// The input should be argument's monotonicity. For binary function it should be an
    /// array of left expression's monotonicity and right expression's monotonicity.
    /// For unary function, the input should be an array of the only argument's monotonicity.
    /// The returned monotonicity should have 'left' and 'right' fields None -- the boundary
    /// calculation relies on the function.eval method.
    fn get_monotonicity(&self, _args: &[Monotonicity]) -> Result<Monotonicity> {
        Ok(Monotonicity::default())
    }

    /// The method returns the return_type of this function.
    fn return_type(&self) -> DataTypeImpl;

    /// Evaluate the function, e.g. run/execute the function.
    fn eval(
        &self,
        _func_ctx: FunctionContext,
        _columns: &ColumnsWithField,
        _input_rows: usize,
    ) -> Result<ColumnRef>;

    /// If all args are constant column, then we just return the constant result
    /// TODO, we should cache the constant result inside the context for better performance
    fn passthrough_constant(&self) -> bool {
        true
    }
}
```

##### FunctionFactory

```rust
// file: common/functions/src/scalars/function_factory.rs
pub struct FunctionFactory {
    case_insensitive_desc: HashMap<String, FunctionDescription>,
}

static FUNCTION_FACTORY: Lazy<Arc<FunctionFactory>> = Lazy::new(|| {
    let mut function_factory = FunctionFactory::create();

    ArithmeticFunction::register(&mut function_factory);
    CommonFunction::register(&mut function_factory);
    ToCastFunction::register(&mut function_factory);
    TupleClassFunction::register(&mut function_factory);
    ComparisonFunction::register(&mut function_factory);
    ContextFunction::register(&mut function_factory);
    SemiStructuredFunction::register(&mut function_factory);
    StringFunction::register(&mut function_factory);
    HashesFunction::register(&mut function_factory);
    ConditionalFunction::register(&mut function_factory);
    LogicFunction::register(&mut function_factory);
    DateFunction::register(&mut function_factory);
    OtherFunction::register(&mut function_factory);
    UUIDFunction::register(&mut function_factory);
    MathsFunction::register(&mut function_factory);

    Arc::new(function_factory)
});
// func: instance() -> FunctionFactory
// 获取单例
   pub fn instance() -> &'static FunctionFactory {
        FUNCTION_FACTORY.as_ref()
    }
// func: register(...)
// 注册函数
    pub fn register(&mut self, name: &str, desc: FunctionDescription) {
        let case_insensitive_desc = &mut self.case_insensitive_desc;
        case_insensitive_desc.insert(name.to_lowercase(), desc);
    }
// func: get(...) -> Function
// 根据函数名获取函数，在ExpressionChain::add_expr()函数中会调用，构造ActionFunction
    pub fn get(&self, name: impl AsRef<str>, args: &[&DataTypeImpl]) -> Result<Box<dyn Function>> {
        let origin_name = name.as_ref();
        let lowercase_name = origin_name.to_lowercase();

        let desc = self
            .case_insensitive_desc
            .get(&lowercase_name)
            .ok_or_else(|| {
                // TODO(Winter): we should write similar function names into error message if function name is not found.
                ErrorCode::UnknownFunction(format!("Unsupported Function: {}", origin_name))
            })?;

        FunctionAdapter::try_create(desc, origin_name, args)
    }
```
