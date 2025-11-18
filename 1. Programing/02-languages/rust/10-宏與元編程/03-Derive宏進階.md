# Derive 宏進階

> 基於 Rust 1.90+ (2025) | 構建強大的派生宏

## 📋 概述

Derive 宏是最常用的過程宏類型,用於自動實現 trait。本章介紹如何構建功能完整、錯誤處理友好的 derive 宏。

---

## 🎯 Helper Attributes

### 定義輔助屬性

```rust
use proc_macro::TokenStream;
use syn::{parse_macro_input, DeriveInput, Attribute};

#[proc_macro_derive(MyMacro, attributes(my_attr))]
pub fn derive_my_macro(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    
    // 現在可以使用 #[my_attr] 屬性
    for attr in &input.attrs {
        if attr.path().is_ident("my_attr") {
            // 處理屬性
        }
    }
    
    TokenStream::new()
}
```

### 使用 Helper Attributes

```rust
#[derive(MyMacro)]
#[my_attr(option1 = "value1")]
struct MyStruct {
    #[my_attr(skip)]
    field1: String,
    
    #[my_attr(rename = "new_name")]
    field2: i32,
}
```

---

## 🔧 處理不同數據類型

### 完整的數據類型處理

```rust
use syn::{Data, Fields, DeriveInput};

#[proc_macro_derive(Display)]
pub fn derive_display(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    
    let display_impl = match input.data {
        // 結構體
        Data::Struct(data) => match data.fields {
            Fields::Named(fields) => {
                // 命名字段結構體
                generate_struct_display(name, &fields.named)
            }
            Fields::Unnamed(fields) => {
                // 元組結構體
                generate_tuple_display(name, &fields.unnamed)
            }
            Fields::Unit => {
                // 單元結構體
                quote! {
                    impl std::fmt::Display for #name {
                        fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
                            write!(f, "{}", stringify!(#name))
                        }
                    }
                }
            }
        },
        
        // 枚舉
        Data::Enum(data) => {
            generate_enum_display(name, &data.variants)
        }
        
        // 聯合體 (不支持)
        Data::Union(_) => {
            return Error::new(
                name.span(),
                "Display cannot be derived for unions"
            ).to_compile_error().into();
        }
    };
    
    TokenStream::from(display_impl)
}
```

---

## 🎨 實戰範例: Validate 宏

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, DeriveInput, Data, Fields, Meta};

#[proc_macro_derive(Validate, attributes(validate))]
pub fn derive_validate(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as DeriveInput);
    let name = &input.ident;
    
    let fields = match input.data {
        Data::Struct(data) => match data.fields {
            Fields::Named(fields) => fields.named,
            _ => panic!("Validate only works with named fields"),
        },
        _ => panic!("Validate only works with structs"),
    };
    
    let validations = fields.iter().map(|f| {
        let field_name = &f.ident;
        let field_name_str = field_name.as_ref().unwrap().to_string();
        
        // 查找 validate 屬性
        let mut checks = Vec::new();
        
        for attr in &f.attrs {
            if attr.path().is_ident("validate") {
                if let Ok(Meta::List(meta)) = attr.parse_args() {
                    // 解析驗證規則
                    // #[validate(min_length = 5, max_length = 20)]
                    for nested in meta.tokens {
                        // 處理驗證邏輯...
                    }
                }
            }
        }
        
        quote! {
            // 生成驗證代碼
        }
    });
    
    let expanded = quote! {
        impl #name {
            pub fn validate(&self) -> Result<(), String> {
                #(#validations)*
                Ok(())
            }
        }
    };
    
    TokenStream::from(expanded)
}
```

完整文檔請參考完整版本...

---

*最後更新: 2025-01-17*  
*Rust 版本: 1.90+*
