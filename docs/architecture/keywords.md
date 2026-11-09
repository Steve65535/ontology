# 编程语言与关键字系统

Autelogy 的本体建模语言使用领域关键字表达对象操作、约束、关系和接口，再编译为七原语。本文聚焦平台自身语言的词汇、用法与编译语义。

本文按技术笔记能够确认的拼写与用法整理。当前仓库不含解析器源码，以下是已知词汇与语义索引，不是经解析器验证的完整保留字表；未提供准确句法的部分不补造关键字。完整编译流程见[编译器](compiler.md)，脚本运行见 [Function](functions.md)。

## 1. 语言分为哪些部分

| 语言部分 | 使用者与用途 | 编译或运行结果 |
| --- | --- | --- |
| Action DSL | 定义业务操作、参数、权限、前置条件和步骤 | `ActionType` 与相关 `Constraint` |
| Constraint DSL | 定义规则、作用域和判定时机 | `Constraint` |
| Link DSL | 定义业务关系、方向、基数及实现 | `LinkType`，必要时附带关系属性 |
| Interface DSL | 定义多个对象共享的契约 | `Interface` 及实现映射相关定义 |
| 共享表达式与类型语法 | 在上述 DSL 中表达值、引用、计算和类型 | `ExprNode`、`BaseType` |

四类 DSL 共享表达式与类型解析能力。七原语均有标准结构化定义，但笔记只明确了四类专用 DSL，不能据此宣称每个原语都有独立文本语言。

## 2. Action DSL 的已知关键字

### 2.1 操作声明与描述

| 词汇 | 用途 | 笔记中的用法或语义 |
| --- | --- | --- |
| `modify` | 在表头声明修改型操作 | `modify purchase_order as approve(...)` |
| `as` | 命名操作或绑定步骤结果 | 表头的 `as approve`；创建步骤的 `as log` |
| `display` | 声明面向用户的显示名称 | `display "审批采购订单"` |
| `roles` | 声明操作所需角色 | `roles approver_role` |
| `emits` | 声明操作可以发出的业务事件 | `emits order_approved` |
| `idempotent` | 声明操作的幂等属性 | `idempotent true` |
| `automatable` | 声明是否允许自动化执行 | `automatable false` |
| `risk` | 声明影响范围、可逆性与外部副作用等风险信息 | 笔记示例为 `risk cascade true false`；参数顺序以解析器契约为准 |

`approve`、`purchase_order`、`approver_role` 和 `order_approved` 是业务名称，不是语言关键字。`batch`、`cascade` 等词还可能作为风险取值出现，不能将所有枚举值混称为语句关键字。

### 2.2 条件与反馈

| 关键字 | 用途 | 示例 |
| --- | --- | --- |
| `require` | 声明操作执行前必须满足的条件 | `require target.status == 'submitted' ...` |
| `else` | 为条件不成立提供业务说明 | `else "只能审批已提交的订单"` |
| `hint` | 给出修正或下一步建议 | `hint "先提交"` |

`require` 会参与编译为前置约束，执行时由约束链判定。`else` 声明条件不成立时的业务消息，`hint` 提供用户可采用的处理建议。

### 2.3 业务步骤

| 关键字 | 用途 | 示例或边界 |
| --- | --- | --- |
| `set` | 设置可写属性 | `set target.status = 'approved'` |
| `create` | 创建业务对象，可用 `as` 绑定结果 | `create approval_log as log { po_id: target.po_id }` |
| `link` | 声明建立业务关系的步骤 | `link po_approvals target -> log` |
| `call_action` | 调用另一个声明的业务操作 | `call_action supplier.notify_approved(po: target)` |
| `emit` | 生成业务事件 | `emit order_approved { po_id: target.po_id }` |
| `for` | 表达受限循环块 | 笔记确认存在循环语法，但未给出完整循环表头 |

步骤按书写顺序展开。`create ... as log` 之后才能引用 `log`；循环内的绑定不泄漏到外部。循环体不能再嵌套循环，也不能容纳外部调用。

操作头的其他形式、删除／解除关系／外部调用的完整 DSL 拼写，现有笔记没有逐项列出，应在核实 parser 后补齐，不能只靠底层 effect 名称猜测。

## 3. 一个 Action 程序如何阅读

以下整理自笔记中的 Action 示例，展示已出现的关键字，尚未在当前仓库通过真实解析器验证。业务实体及角色需要在模型中实际定义。

```text
modify purchase_order as approve(approver: object_ref<employee>, note: text):
    display "审批采购订单"
    roles approver_role
    emits order_approved
    idempotent true
    automatable false
    risk cascade true false

    require target.status == 'submitted' else "只能审批已提交的订单" hint "先提交"
    set target.status = 'approved'
    set target.approver_id = approver.employee_id
    create approval_log as log { po_id: target.po_id, by: approver.employee_id }
    link po_approvals target -> log
    call_action supplier.notify_approved(po: target)
    emit order_approved { po_id: target.po_id }
```

表头说明操作作用于什么对象、叫什么、接受什么类型的参数；声明部分描述权限、事件和自动化属性；条件部分定义能否执行；步骤部分定义执行后发生什么。`automatable false` 表示即便工具能够被发现，也不能由自动化执行者绕过限制。

这不是任意执行的脚本。编译器先解析声明、检查作用域并转成模型，真正的对象修改、子操作及事件都由执行器放入受控事务完成。

## 4. DSL 词汇如何对应底层模型

| 文本层表达 | 模型或运行时含义 |
| --- | --- |
| `modify ... as ...` 与参数表 | Action 的绑定对象、操作类型、名称与参数 |
| `display`、`roles`、`emits` | 显示名称、所需角色、允许发出的事件声明 |
| `idempotent`、`automatable`、`risk` | 幂等、自动化与风险属性 |
| `require ... else ... hint ...` | 带说明和建议的前置 `Constraint` |
| `set` | `set_fields` effect |
| `create` | `create_object` effect |
| `link` | `link` effect；高层草案中可由 `link_objects` 的 mode 表达 |
| `call_action` | `call_action` effect |
| `emit` | `emit_event` effect |
| `for` | `for_each` effect |

`set_fields`、`create_object`、`delete_object`、`unlink`、`emit_event`、`for_each`、`call_external` 是模型或草案层的操作名称。它们是否以完全相同的拼写出现在某类 DSL 中，应以该 DSL 的解析器为准。关键字文档需要区分“文本中怎么写”和“编译后叫什么”。

## 5. 其他 DSL 的已知语义词汇

现有笔记确认 Constraint、Link、Interface 具有各自的 parser、草案转换、格式化与语言服务，但未给出完整文本语法。下面列的是编译目标中的语义取值，供理解语言表达能力，不将其冒充完整关键字清单。

| DSL | 已确认的语义内容 | 标准模型中的相关名称 |
| --- | --- | --- |
| Constraint | 必填、范围、长度、正则、枚举、唯一、状态迁移、表达式、关系基数、引用完整性、无环、频率 | `required`、`range`、`length`、`regex`、`enum_member`、`unique`、`state_machine`、`expression`、`cardinality`、`referential_integrity`、`acyclic`、`rate_limit` |
| Constraint | 在何处、何时、以何种严重度判定 | `property`／`object`／`link`／`action`；`pre_effect`／`pre_commit`／`in_tx_post`／`async_drift`；`error`／`warning` |
| Link | 外键、连接表、派生关系，关系角色与基数 | `foreign_key`、`join_table`、`derived`、关系两端定义 |
| Interface | 共享属性与关系、继承、对象实现映射 | `extends`、`implements`、`propertyMapping`、`linkMapping` 等模型概念 |

高层草案模板与标准规则名也不总相同，例如 `enum` 模板对应 `enum_member`，`state_transition` 模板对应 `state_machine`。文档与编辑器不能将这几个层次混用。

## 6. 表达式、类型与名字

### 6.1 内置函数与类型词汇

表达式的已知内置函数包括 `now`、`sum`、`contains`、`starts_with`、`ends_with`、`array_of`。这些是可调用函数名，区别于 `require`、`set` 等语句关键字。自定义 Function 按本体定义解析，不能将任意名字当作运行时函数执行。

类型语法涉及标量、`decimal(p,s)`、`enum<...>`、`array<T>`、`object_ref<T>` 和 `interface_ref<I>`。笔记示例使用 `text` 作为参数类型拼写；DSL 类型名称与标准 `BaseType` 的映射应由共享类型解析器定义，不能凭文本名称自行推导。

`true`、`false`、字符串和数字是值；`.` 表达路径访问，`==` 表达示例中的比较，`=` 在 `set` 中表示赋值，`->` 在关系步骤中表达方向。它们的词法类别应分别说明，不与业务名称混成一组关键字。

### 6.2 三类名称不能混用

| 名称类别 | 用途 | 规则或边界 |
| --- | --- | --- |
| `apiName` | 持久化的业务实体名称 | `snake_case`、不超过 63 字节、禁止 `pg_` 前缀 |
| `bindingName` | 参数、步骤结果和循环变量 | 局部作用域，可使用 `camelCase`，不作为持久化实体身份 |
| 展示名称 | 人可读业务含义 | 与稳定的 API 身份分开，按相应字段契约校验 |

`target`、`before`、`after`、`actor` 是上下文中的绑定名，是否可用取决于当前规则与时机。它们不是任意位置都有效的全局变量。完整引用按 FQN 契约解析，要求完整引用的入口不能自动猜测裸名所属业务域。

笔记未提供所有关键字在每种命名位置的保留策略，也未说明完整转义规则，因此本版不宣称这些词在所有上下文一律不可用作名字。

## 7. 补全、诊断与格式化

编辑器根据当前 DSL、光标位置、本体和作用域提供补全。输入到 `target.` 时应建议当前目标的合法成员，输入到步骤位置时应提供相应关键字，未知函数或错误类型应给出定位诊断。

笔记描述了逐行容错：一行语法错误不应阻止后续行的诊断；表头未输入完整时也尽量提供最小上下文补全。这种容错用于编辑反馈，不能让非法程序成为可保存、可执行的模型。

已有实体可以格式化回 DSL 再编辑。关键的 `idempotent`、`automatable`、`risk` 以及关系的 `acyclic` 等信息应在往返过程中保持语义，不能因格式化省略默认值而改变权限或风险。

后续代码核实应补齐：四类 DSL 的完整语法与保留字表、每个关键字的参数形式、类型别名、名称转义与冲突规则、外部调用和删除步骤的真实文本拼写，以及能被真实解析器验证的示例。
