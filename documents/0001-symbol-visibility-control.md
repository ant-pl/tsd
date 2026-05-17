- Feature Name: symbol-visibility-control
- Start Date: (填充提交的日期，格式：YYYY-MM-DD)
- TSD PR: [ant-pl/tsd#0000](https://github.com/ant-pl/tsd/pull/0001)

## 摘要
[summary]: #摘要

现在的 TypedAnt 缺少一种机制使得符号可见性是可调整的。该提案被实现之后将会有以下两种方式调整：

- pub 后紧跟符号定义的语句 针对所有 crate 公开
- pub(crate) 后紧跟符号定义的表达式
仅对当前 crate 公开

**请注意**：该提案提议符号应当默认不可见 仅当手动公开后才可 受限/不受限 的访问

## 动机
[motivation]: #动机

在当前的 TypedAnt 中，我们无法对符号进行 私有、公开、受限公开 这三种操作。  
当程序规模日渐庞大时，如果不能手动控制符号的可见性，那么很有可能出现调试半天程序错误结果是别人提交的代码里不小心修改了一个对象的字段导致程序不符合预期的情况。  
所以为了解决诸如此类的情况本提案提出在未来对于符号至少需要有三种情况并且可手动控制：  

 - 私有化（默认）
 - 针对所有 crate 公开
 - 针对当前 crate 公开
 
### 1. 私有化（默认）
私有化，即对于各种符号仅在当前模块内可访问。对于一个被私有化的符号而言，它只可被当前模块或当前模块的子模块访问  

我们来看一个示例  
```typedant
// learn/a.ta
struct Age {
    age: i32
}

pub func create_age(age: i32) -> Option<Age> {
    if age < 0 {
        Option::None
    } else {
        Option::Some(new Age {
            // age 字段可以正常的被访问，因为在同一个模块内
            age = age
        })
    }
}

// learn/b.ta

use a::create_age;

// 无法访问，因为符号私有
use a::Age;

func main -> i32 {
    // 假设 Age 结构体已公开
    new Age {
        // 无法进行初始化 因为该字段私有
        age = 9
    }
    
    let age = create_age(9);
    age.age = -4; // 无法修改 因为字段私有

    0
}
```
该示例展示了通过私有字段和私有结构体防止用户不小心修改一个变量导致后面连环错误，再到 Debug 找半天找不到为什么 age 字段是 -4 的悲剧  

### 2. 针对所有 crate 公开
顾名思义 对所有 crate 都公开

我们有一个示例可以展示
```typedant
// learn/a.ta

pub struct Age {
    // usize 先天不支持负数，不需要对年龄字段进行任何检查和限制，所以我们可以放心公开它
    pub age: usize
}

// Age 及其所有字段全部公开所以完全可以不需要辅助函数

// learn/b.ta

func main() -> i32 {
    // 可以直接使用 因为它及其它的字段完全公开
    let age = new Age {
        age = 999
    };
    
    0
}
```
请注意：符号公开应当考虑公开后用户可能的操作再考虑是否公开，或者你能承受用户乱搞的后果  

### 3. 针对当前 crate 公开
针对当前 crate 公开的符号 该 crate 的任何模块都可访问该符号。  
请注意：它并不会对子 crate 公开，因为子 crate 是一个单独的 crate  

我们有一个示例可以展示
```typedant
pub(crate) struct CompiledMetadata {
    pub(crate) version: CrateVersion
    …
}
```
上面的元数据结构体即是一个示例，很明显该结构体不应该被用在除编译器外任何地方。这时候我们就需要 pub(crate) 来明确表示仅在内部 crate 公开这样一个情况

对 TypedAnt 的任何更改都应该集中于解决 TypedAnt 用户遇到的问题。本节应详细解释要解决的问题，包括必要的背景信息。  
它还应该包含几个具体的用例，说明这个功能如何帮助用户，并解释其作用。这可以用来指导功能的设计。  
这一部分是任何 TSD 中最重要的部分之一，可能会很长。

## 指导性解释
[guide-level-explanation]: #指导性解释

我们定义以下三种可见性
 - 私有化（默认）
 - 针对所有 crate 公开
 - 针对当前 crate 公开

### 私有化（默认）
私有化可见性作用为让符号仅在当前模块内可访问，且它只可被当前模块或当前模块的子模块访问。  

我们有一个最简单的示例可以展示
```typedant
// 结构体及其字段只能在当前模块访问
sturct RawVec<T> {
    ptr: *T,
    len: usize,
    cap: usize
}
```

你要问我为什么不直接搞个 Vec 我也没辙我偏要封装你气不气。开个玩笑，但封装还是很有必要的。  

以上代码展示了一个私有化的使用场景

### 针对所有 crate 公开
顾名思义 作用为让符号对所有 crate 都公开

我们有一个最简单的示例可以展示
```typedant
// 完全公开
pub const U16_MAX: u16 = 65535
```

以上代码展示了一个针对所有 crate 公开的使用场景

### 针对当前 crate 公开
我们定义该符号可见性作用为让该 crate 的任何模块都可访问该符号，但它并不会对子 crate 公开，因为子 crate 是一个单独的 crate  

我们有一个最简单的示例可以展示
```typedant
// 仅针对当前 crate 公开，子 crate 不公开
pub(crate) const STACK_SIZE: u16 = 2048
```

以上代码展示了一个针对当前 crate 公开的使用场景

该提案对于以后所有的 crate 来讲影响很大，它会影响对于 结构体、常量、字段、内外部 api 的组织方式。而对该提案代码量、性能影响、api稳定性方面有待思考  

若该提案生效，那么以后对于私有的内部api可以放心改，因为不会有外部程序使用。对于开发者来说，这无疑是一件好事。但是与此同时，编写代码思考的要更多了，代码量也增多了，可能会显得繁琐些

**请注意**: 针对于枚举，他的所有变体完全公开。

假设这个提案已经包含在语言中，并假设你正在向另一个 TypedAnt 程序员解释它。这通常意味着：

- 引入新概念
- 主要以示例形式解释该功能。
- 解释 TypedAnt 程序员应该如何 *思考* 这个特性，以及它应该如何影响他们使用 TypedAnt 的方式。尽可能具体地解释其影响。
- 如适用，提供示例错误消息、弃用警告或迁移指南。
- 如果适用，请描述教老 TypedAnt 程序员此内容与教新 TypedAnt 程序员此内容之间的差异。
- 讨论这一变化如何影响阅读、理解和维护 TypedAnt 代码的能力。代码被阅读和修改的频率远高于编写；这项提议的功能是否会使代码更易于维护？

对于面向实现的 TSD（例如用于编译器内部），本节应侧重于技术贡献者应如何思考这一变化，并给出其实际影响的示例。对于企划，应对某种情况类等等的 TSD，本节应提供以示例为主的企划介绍，并具体解释其影响。

## 技术层解释
[reference-level-explanation]: #技术层解释

让我们以上一节的示例为例  

```typedant
// 完全公开
pub const U16_MAX: u16 = 65535
```

这段代码首先经过词法分析，然后到语法分析。在语法分析阶段遇到 pub/pub(crate) 等访问控制符时进入访问控制符的解析函数，解析完访问控制的权限之后，接着进入对应语句/表达式的解析函数，类似这样：

```rust
pub fn parse_visibility_control(parser: &mut Parser) -> ParseResult<Expression> {
    let visibility = parse_visibility(parser);

    let cur_token = parser.cur_token.clone();

    let stmt = match cur_token.token_type {
        TokenType::Const => parse_const(parser),
        TokenType::Func => parse_func(parser),
        // ...
    };

    Ok(Expression::VisibilityControl {
        visibility,
        stmt,
    })
}
```

再然后进入 NameResolver，在遇到符号定义时将可见性控制的 AST 转换为可见性枚举，并且写入符号。  
在遇到符号访问的时候，获取该符号可见性，结合当前模块ID检查是否可访问该符号。大概像这样：

```rust
// name resolver
impl<'a> NameResolver<'a> {
    /// 遍历模块的 AST，收集顶层定义
    pub fn resolve_module_definitions(
        &mut self,
        module_id: ModuleId,
        stmts: &[Statement],
        parent_def: Option<DefId>,
    ) -> ResolveResult<Vec<Option<DefId>>> {
        // ...
        for stmt in stmts {
            match stmt {
                Statement::Struct { name, generics, visibility: visibility_ast, .. } => Some({
                    // 构造原始定义数据
                    let data = StructData {
                        name: name.value.clone(),
                        visibility: self.as_visibility(visibility_ast), // 转换可见性
                        module_id,
                        generics: generics.iter().map(|g| g.to_string().into()).collect(),
                        fields: IndexMap::new(), // TypeChecker 稍后填充
                        ty: 0usize.into(),       // 同上
                        ast_index: StmtId(i),
                    };

                    let def_id = self.krate.alloc_def(Def::Struct(data));
                    local_symbols.insert(name.value.clone(), def_id);

                    def_id
                }),

                // ...
            }
        }
    }

    pub fn resolve_use(
        &mut self,
        current_module_id: ModuleId,
        path_tokens: &[Token],
        alias_token: Token,
    ) -> ResolveResult<()> {
        // ...
        
        let def = self.krate.get_def(def_id);
        if !self.is_accessible(def.visibility()) {
            return Err(Self::make_err(
                Some(&format!("symbol `{}` is private", path.last().unwrap())),
                NameResolverErrorKind::SymbolIsPrivate,
                path_tokens.last().unwrap().clone(),
            ));
        }
     
        // ...
    }
}
```

同时为了防止公开函数暴露私有数据的情况，我们还需要对函数进行暴露检查，但暴露检查往往需要得知被暴露者的类型，因此暴露检查需要在 TypeChecker 或在 TypeChecker 之后的阶段完成。就像这样：

```rust
impl<'a, 'b> TypeChecker<'a, 'b> {
    fn check_privacy_leaks(&self) -> CheckResult<()> {
        let def_count = self.name_resolver.krate.definitions.len();
        
        for i in 0..def_count {
            // ...

            // 仅对公开 API 检查
            if def.visibility() == Visibility::Public {
                let mod_id = def.module_id();

                match def {
                    Def::Function(func_data) => {
                        let ty_id = func_data.ty.get();
                        if !self.is_accessible(ty_id, mod_id) {
                            return Err(Self::make_err(
                                // ...
                            ))
                        }
                    }
                    
                    Def::Struct(struct_data) => {
                        for (_, field_ty_id) in &struct_data.fields {
                            if self.is_accessible(*field_ty_id, mod_id) {
                                continue;
                            }

                            return Err(Self::make_err(
                                // ...
                            ))
                        }
                    }

                    Def::Constant(const_data) => {
                        let ty_id = const_data.ty.get();
                        if !self.is_accessible(ty_id, mod_id) {
                            return Err(Self::make_err(
                                // ...
                            ))
                        }
                    }
                    
                    _ => {}
                }
            }
        }
        
        Ok(())
    }
}
```

这是 TSD 的技术部分。详细说明设计，以便：

- 该功能与其他功能之间的交互清晰明了。
- 功能实现方式相对清晰。
- 通过示例分析各种边界情况。

该部分应回到上一节给出的示例，并更详细地解释详细提案如何使这些示例工作。

## 缺点
[drawbacks]: #缺点

我们为什么*不*应该这样做？

1. 略显繁琐 相较于默认公开会增加代码量
2. 因为繁琐 所以可能导致用户全部写 pub，而导致前面为私有接口的重构安全全部失效
3. 变动巨大 目前现有代码库全部为默认公开，如果贸然修改可能导致所有代码库全部报错
4. 实现复杂 相较于 Python 这类无私有符号来说，编译器实现无疑会更加复杂

## 理由和其他替代方案
[rationale-and-alternatives]: #rationale-and-alternatives

该提案强制要求用户在公开时编写 pub 关键字，从而让用户在公开符号时多加思考以确认是否真的需要公开。此外，如果该提案被采用，对于私有的 api 而言则可以随意重构而不会影响外部的任何库，大大降低了用户对于重构时心里的畏惧。  

当然，我们还想过很多其他的方案，例如：

- 没有私有概念，就像 Python
- 默认公开，手动私有

但是，没有私有概念会导致用户即使有意私有也没法实施，从而导致代码库的混乱。而默认公开，手动私有虽然相比前者起来更好，但大概率仍会保持终身公开，一个内部符号直到项目生命周期结束一辈子没有被私有过。  

显然，手动指定公开相对于前两者都更好（除了繁琐之外），并且大大降低用户编写私有 api 时的心理负担。在外部使用时也能根据可见性明白是否应该调用 api

- 为什么这是在所有可能设计中最好的设计？
- 已经考虑过哪些其他设计，以及不选择它们的原因是什么？
- 不这样做会有什么影响？
- 所提出的变更是否使 TypedAnt 代码更易于阅读、理解和维护？

## 先人的足迹
[prior-art]: #先人的足迹

你说得对，但是 Rust 是由 Mozilla 基金会开发的一款安全快速的编译型编程语言。玩家将在这里扮演一个名叫作 rustacean 的角色，探索 unsafe 的秘密。
说回正题，该提案完全就是从 rust 的默认私有化和 rust rfc 0001 private-fields 演变而来的，而对于 rustacean 而言，默认私有是使用 Rust 的一部分，本段不再赘述。

讨论与该提案相关的现有技术，包括优点和缺点。以下是一些可能包括的内容示例：

- 对于技术型提案：该功能是否存在于其他编程语言中，以及它们的社区有哪些意见？
- 对于社区提案：其他社区是否做过，以及他们对此有哪些经验？
- 对于其他团队：我们能从其他社区在这里所做的事情中学到哪些经验？
- 论文：是否有关于此主题发表的论文或优秀的文章？如果你有一些相关的论文可以参考，这可以作为更详细的理论背景。

本节旨在鼓励你作为作者思考其他语言的教训，为你的 TSD 读者提供更全面的画面。如果没有先例，那也没关系——无论你的想法是全新的还是改编自其他语言，对我们来说都很有趣。

请注意，虽然其他语言的先例提供了一些动机，但这本身并不能当作一个 TSD。请同时考虑，TypedAnt 有时会故意偏离常见的语言特性。

## 未解决的问题
[unresolved-questions]: #未解决的问题

请注意，对于其他更多的受限公开例如 pub(super) 等并不属于该 TSD 范围内，应当另立 TSD

- 在这次合并之前，您期望通过 TSD 流程解决设计的哪些部分？
- 在功能稳定之前，您期望通过该功能的实现解决设计的哪些部分？
- 你认为哪些相关问题不属于此 TSD 的范围，可以在未来独立于此 TSD 产生的解决方案来解决？

## 未来可能发生的（可选）
[future-possibilities]: #未来可能发生的（可选）

暂无