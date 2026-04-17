---
title: Rust宏：从macro_rules到过程宏
date: 2026-04-17 18:40:45
tags: rust
categories: TechMagic
index_img: ./rustmacro/rustdocs.png
banner_img:
---

写 Rust 写到一定阶段，你会发现“宏”几乎绕不过去。

从 `println!`、`vec!` 到 `#[tokio::main]`、`#[derive(Debug)]`，它们都在编译期改写代码，只是方式不同。本文对各类宏进行深入解析，带你从 `macro_rules!` 的模式匹配到过程宏的 TokenStream 转换，全面理解 Rust 宏的强大与灵活。

<!-- more -->
## 声明宏

### 例1：println! 
```rust
macro_rules! println {
    () => {
        $crate::print!("\n")
    };
    ($($arg:tt)*) => {{
        $crate::io::_print($crate::format_args_nl!($($arg)*));
    }};
}
```

`macro_rules!` 类似于一个巨大的 `match` 语句，对于第一种模式，`println!()` 没有参数，直接输出一个换行符；对于第二种模式，`println!()` 接受任意数量的参数，并将它们传递给 `format_args_nl!` 宏进行格式化，然后调用 `_print` 函数输出结果。

`$($arg:tt)*` 是一个特殊的语法，`tt` 代表一个 token tree，可以匹配任意的 Rust 代码片段。`$()*` 表示匹配零个或多个这样的 token tree，并将它们捕获到一个变量 `$arg` 中。

`$crate` 是一个特殊的变量，指向当前宏所在的 crate。这使得宏能够正确地引用 crate 内部的项，无论宏被调用的位置在哪里。

### 例2：lazy_static!

```rust
macro_rules! lazy_static {
    ($(#[$attr:meta])* static ref $N:ident : $T:ty = $e:expr; $($t:tt)*) => {
        // use `()` to explicitly forward the information about private items
        __lazy_static_internal!($(#[$attr])* () static ref $N : $T = $e; $($t)*);
    };
    ($(#[$attr:meta])* pub static ref $N:ident : $T:ty = $e:expr; $($t:tt)*) => {
        __lazy_static_internal!($(#[$attr])* (pub) static ref $N : $T = $e; $($t)*);
    };
    ($(#[$attr:meta])* pub ($($vis:tt)+) static ref $N:ident : $T:ty = $e:expr; $($t:tt)*) => {
        __lazy_static_internal!($(#[$attr])* (pub ($($vis)+)) static ref $N : $T = $e; $($t)*);
    };
    () => ()
}
```

`lazy_static!` 宏定义了三种模式，分别用于处理不同的可见性修饰符（无修饰符、`pub` 修饰符和带有自定义可见性的 `pub` 修饰符）。每种模式都将捕获的参数传递给一个内部宏 `__lazy_static_internal!`，并使用 `()` 来显式地转发关于私有项的信息。

`$(#[$attr:meta])*` 用于匹配零个或多个属性（如 `#[derive(Debug)]`），`$N:ident` 匹配一个标识符作为静态变量的名称，`$T:ty` 匹配一个类型，`$e:expr` 匹配一个表达式作为静态变量的初始化值，`$($t:tt)*` 匹配剩余的 token tree 以允许定义多个静态变量。

```rust
macro_rules! __lazy_static_internal {
    // optional visibility restrictions are wrapped in `()` to allow for
    // explicitly passing otherwise implicit information about private items
    ($(#[$attr:meta])* ($($vis:tt)*) static ref $N:ident : $T:ty = $e:expr; $($t:tt)*) => {
        __lazy_static_internal!(@MAKE TY, $(#[$attr])*, ($($vis)*), $N);
        __lazy_static_internal!(@TAIL, $N : $T = $e);
        lazy_static!($($t)*);
    };
    (@TAIL, $N:ident : $T:ty = $e:expr) => {
        impl $crate::__Deref for $N {
            type Target = $T;
            fn deref(&self) -> &$T {
                #[inline(always)]
                fn __static_ref_initialize() -> $T { $e }

                #[inline(always)]
                fn __stability() -> &'static $T {
                    __lazy_static_create!(LAZY, $T);
                    LAZY.get(__static_ref_initialize)
                }
                __stability()
            }
        }
        impl $crate::LazyStatic for $N {
            fn initialize(lazy: &Self) {
                let _ = &**lazy;
            }
        }
    };
    // `vis` is wrapped in `()` to prevent parsing ambiguity
    (@MAKE TY, $(#[$attr:meta])*, ($($vis:tt)*), $N:ident) => {
        #[allow(missing_copy_implementations)]
        #[allow(non_camel_case_types)]
        #[allow(dead_code)]
        $(#[$attr])*
        $($vis)* struct $N {__private_field: ()}
        #[doc(hidden)]
        #[allow(non_upper_case_globals)]
        $($vis)* static $N: $N = $N {__private_field: ()};
    };
    () => ()
}
```

前面的模式是递归调用来处理多个静态变量的定义，同时使用 `@TAIL` 和 `@MAKE` 来分别处理静态变量的实现和结构体定义。


`@MAKE` 模式定义了一个零大小同名结构体来表示静态变量，并创建一个同名静态实例。

`@TAIL` 模式实现了 `Deref` 和 `LazyStatic` trait，前者允许通过自动解引用访问静态变量的对应类型的值，并且在 `deref` 的实现中处理初始化，后者提供了一个 `initialize` 方法来显式地触发静态变量的初始化。

### 例3: vec!

```rust
macro_rules! vec {
    () => (
        $crate::vec::Vec::new()
    );
    ($elem:expr; $n:expr) => (
        $crate::vec::from_elem($elem, $n)
    );
    ($($x:expr),+ $(,)?) => (
        $crate::boxed::box_assume_init_into_vec_unsafe(
            $crate::intrinsics::write_box_via_move($crate::boxed::Box::new_uninit(), [$($x),+])
        )
    );
}
```

第一个模式匹配 `vec!()`，没有参数时返回一个新的空 `Vec`。

第二个模式匹配 `vec![elem; n]`，接受一个元素和一个数量，使用 `from_elem` 函数创建一个包含 `n` 个 `elem` 的 `Vec`。

第三个模式匹配 `vec![x1, x2, ...]`，接受一个或多个表达式，并使用一些内部函数来创建一个包含这些元素的 `Vec`。这里使用了 `box_assume_init_into_vec_unsafe` 和 `write_box_via_move` 来高效地构建 `Vec`，避免了不必要的复制。


## 过程宏

输入 TokenStream，输出 TokenStream。

### 属性宏 例：my tokio main

```rust
#[proc_macro_attribute] // 声明这是一个属性宏
pub fn my_tokio_main(_attr: TokenStream, item: TokenStream) -> TokenStream {
    // 1. 解析输入的函数（item 是你写的 async fn main）
    let input_fn = parse_macro_input!(item as ItemFn);

    // 保存函数名（通常是 main）
    let name = &input_fn.sig.ident;
    // 保存函数体
    let body = &input_fn.block;
    // 保存函数的属性、可见性等
    let attrs = &input_fn.attrs;
    let vis = &input_fn.vis;

    // 2. 生成替换代码
    // 我们把异步函数变成同步函数，并在内部启动运行时
    let expanded = quote! {
        #(#attrs)*
        #vis fn #name() {
            // 这里模拟 tokio 的启动逻辑
            let rt = tokio::runtime::Runtime::new().unwrap();
            rt.block_on(async {
                #body
            });
        }
    };

    // 3. 返回生成的标记流
    TokenStream::from(expanded)
}
```

实际上的 `#[tokio::main]` 比这个例子复杂得多，因为它需要处理不同的运行时选项（如 `multi_thread`、`current_thread` 等），并且要确保生成的代码能够正确地处理异步函数的输入参数和返回值。

但这个例子展示了属性宏的基本结构：解析输入的函数，生成新的代码，并返回一个新的 `TokenStream`。通过这种方式，我们可以在编译时修改函数的行为，实现类似于 `#[tokio::main]` 的功能。

其中 `syn` crate 用于解析 Rust 代码，`quote` crate 用于生成 Rust 代码。

使用上，举个例子

```rust
#[my_macros::my_tokio_main]
async fn main() {
    let a = 1;
    let b = vec![1; 2];
    println!("Hello, world! a = {}, b = {:?}", a, b);
}
```

其展开后会变成

```rust
#[my_macros::my_tokio_main]
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        {
            let a = 1;
            let b = ::alloc::vec::from_elem(1, 2);
            {
                ::std::io::_print(
                    format_args!("Hello, world! a = {0}, b = {1:?}\n", a, b),
                );
            };
        }
    });
}
```

### 派生宏 例：derive(LazyGetter)

```rust

#[proc_macro_derive(LazyGetter)]
pub fn lazy_getter_derive(input: TokenStream) -> TokenStream {
    let input_clone = input.clone();
    // 解析输入的结构体
    let derive_input = parse_macro_input!(input_clone as syn::DeriveInput);

    match derive_input.data {
        syn::Data::Struct(data_struct) => {
            let fields = data_struct.fields;
            let impls = fields
                .iter()
                .map(|field| {
                    let field_name = &field.ident;
                    let field_type = &field.ty;
                    let setter_ident = format_ident!("set_{}", field_name.as_ref().unwrap());
                    let getter_ident = field_name.as_ref().unwrap();
                    let inner_type = get_inner_type_from_option(field_type)
                        .expect("Field must be of type Option<T>");
                    quote! {
                        pub fn #setter_ident(&mut self, value: #inner_type) {
                            self.#field_name = Some(value);
                        }

                        pub fn #getter_ident(&self) -> &#inner_type {
                            self.#field_name.as_ref().expect("Field is not initialized")
                        }
                    }
                })
                .collect::<Vec<_>>();
            let struct_name = derive_input.ident;
            quote! {
                impl #struct_name {
                    #(#impls)*
                }
            }
            .into()
        }
        _ => {
            // 其他类型（枚举、联合）不处理
            return input;
        }
    }
}

fn get_inner_type_from_option(ty: &syn::Type) -> Option<&syn::Type> {
    if let syn::Type::Path(type_path) = ty {
        if type_path.path.segments.len() == 1 && type_path.path.segments[0].ident == "Option" {
            if let syn::PathArguments::AngleBracketed(args) = &type_path.path.segments[0].arguments
            {
                if args.args.len() == 1 {
                    if let syn::GenericArgument::Type(inner_type) = &args.args[0] {
                        return Some(inner_type);
                    }
                }
            }
        }
    }
    None
}
```

设想了一个何意味的场景，我们有一个结构体，其中的字段都是 `Option<T>` 类型，我们希望通过一个派生宏自动生成对应的 getter 和 setter 方法。getter 方法会返回字段的值，如果字段未初始化（即为 `None`），则会 panic；setter 方法会接受一个值并将其包装在 `Some` 中赋值给字段。

在这个例子中，`lazy_getter_derive` 函数首先解析输入的结构体，然后检查它是否是一个结构体类型。如果是结构体，它会遍历字段，生成对应的 getter 和 setter 方法，并将它们组合成一个 `impl` 块返回。如果输入的类型不是结构体（例如枚举或联合），则直接返回原始输入，表示不进行任何修改。

`get_inner_type_from_option` 函数用于从 `Option<T>` 类型中提取出 `T` 的类型，以便在生成 getter 和 setter 方法时使用正确的类型签名。

使用上，我们可以这样定义一个结构体：

```rust
#[derive(my_macros::LazyGetter)]
struct MyStruct {
    a: Option<i32>,
    b: Option<Vec<i32>>,
}
```
编译后会自动生成如下代码：

```rust
struct MyStruct {
    a: Option<i32>,
    b: Option<Vec<i32>>,
}
impl MyStruct {
    pub fn set_a(&mut self, value: i32) {
        self.a = Some(value);
    }
    pub fn a(&self) -> &i32 {
        self.a.as_ref().expect("Field is not initialized")
    }
    pub fn set_b(&mut self, value: Vec<i32>) {
        self.b = Some(value);
    }
    pub fn b(&self) -> &Vec<i32> {
        self.b.as_ref().expect("Field is not initialized")
    }
}
```

### 类函数宏 例：foreach!(field, |x| { ... })

```rust
// 1. 定义一个结构体来保存解析后的参数
struct ForEachInput {
    field: Expr,          // 存储变量
    _comma: Token![,],    // 存储逗号分隔符
    closure: ExprClosure, // 存储闭包，如 |x| { ... }
}

// 2. 实现 Parse Trait，告诉 syn 怎么按顺序读 Token
impl Parse for ForEachInput {
    fn parse(input: ParseStream) -> syn::Result<Self> {
        Ok(ForEachInput {
            field: input.parse()?,   // 尝试解析为一个表达式（变量）
            _comma: input.parse()?,  // 必须跟着一个逗号
            closure: input.parse()?, // 尝试解析为一个闭包
        })
    }
}

// 3. 宏入口
#[proc_macro]
pub fn foreach(input: TokenStream) -> TokenStream {
    // 解析输入
    let ForEachInput { field, closure, .. } = parse_macro_input!(input as ForEachInput);

    let expanded = quote! {
        {
            #field.iter().for_each(#closure);
        }
    };

    TokenStream::from(expanded)
}
```

这个例子展示了一个类函数宏 `foreach!`，它接受一个字段和一个闭包，并生成一个迭代该字段的代码块。输入的格式是 `foreach!(field, |x| { ... })`，其中 `field` 是一个可迭代的变量，`|x| { ... }` 是一个闭包。

这个宏的用例如下：

```rust
let my_vec = vec![1, 2, 3];
foreach!(my_vec, |x| {
    println!("Value: {}", x);
});
```

编译后会展开成：

```rust
let my_vec = vec![1, 2, 3];
{
    my_vec.iter().for_each(|x| {
        println!("Value: {}", x);
    });
}
```

### 对比

派生宏不可以修改原本的代码，而属性宏和类函数宏可以修改原本的代码。

所以派生宏适合在结构体、枚举等类型上自动生成一些方法或实现某些 trait，而属性宏和类函数宏更适合在函数或表达式级别进行代码转换和增强。