# axum-valid

[![crates.io](https://img.shields.io/crates/v/axum-valid)](https://crates.io/crates/axum-valid)
[![crates.io download](https://img.shields.io/crates/d/axum-valid)](https://crates.io/crates/axum-valid)
[![LICENSE](https://img.shields.io/badge/license-MIT-blue)](https://github.com/gengteng/axum-valid/blob/main/LICENSE)
[![dependency status](https://deps.rs/repo/github/gengteng/axum-valid/status.svg)](https://deps.rs/repo/github/gengteng/axum-valid)
[![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/gengteng/axum-valid/.github/workflows/main.yml?branch=main)](https://github.com/gengteng/axum-valid/actions/workflows/ci.yml)
[![Coverage Status](https://coveralls.io/repos/github/gengteng/axum-valid/badge.svg?branch=main)](https://coveralls.io/github/gengteng/axum-valid?branch=main)

## 📑 Overview

axum-valid is a library that provides data validation extractors for the Axum web framework. It integrates several popular validation crates in the Rust ecosystem to offer convenient validation and data handling extractors for Axum applications.

## 🚀 Basic usage

### 📦 `Valid<E>`

* Install

```shell
cargo add validator --features derive
cargo add axum-valid
# validator is enabled by default
```

* Example

```rust,ignore
use validator::Validate;
use serde::Deserialize;
use axum_valid::Valid;
use axum::extract::Query;
use axum::{Json, Router};
use axum::routing::{get, post};

#[derive(Debug, Validate, Deserialize)]
pub struct Pager {
    #[validate(range(min = 1, max = 50))]
    pub page_size: usize,
    #[validate(range(min = 1))]
    pub page_no: usize,
}

pub async fn pager_from_query(
    Valid(Query(pager)): Valid<Query<Pager>>,
) {
    assert!((1..=50).contains(&pager.page_size));
    assert!((1..).contains(&pager.page_no));
}

pub async fn pager_from_json(
    pager: Valid<Json<Pager>>,
) {
    assert!((1..=50).contains(&pager.page_size));
    assert!((1..).contains(&pager.page_no));
    // NOTE: all extractors provided support automatic dereferencing
    println!("page_no: {}, page_size: {}", pager.page_no, pager.page_size);
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let router = Router::new()
        .route("/query", get(pager_from_query))
        .route("/json", post(pager_from_json));
    axum::Server::bind(&([0u8, 0, 0, 0], 8080).into())
        .serve(router.into_make_service())
        .await?;
    Ok(())
}
```

In case of inner extractor errors, it will first return the Rejection from the inner extractor. When validation errors occur, the outer extractor will automatically return 400 with validation errors as the HTTP message body.

### 📦 `Garde<E>`

* Install

```shell
cargo add garde
cargo add axum-valid --features garde,basic --no-default-features
# excluding validator
```

* Example

```rust,ignore
use axum::extract::{FromRef, Query, State};
use axum::routing::{get, post};
use axum::{Json, Router};
use axum_valid::Garde;
use garde::Validate;
use serde::Deserialize;

#[derive(Debug, Validate, Deserialize)]
pub struct Pager {
    #[garde(range(min = 1, max = 50))]
    pub page_size: usize,
    #[garde(range(min = 1))]
    pub page_no: usize,
}

pub async fn pager_from_query(Garde(Query(pager)): Garde<Query<Pager>>) {
    assert!((1..=50).contains(&pager.page_size));
    assert!((1..).contains(&pager.page_no));
}

pub async fn pager_from_json(pager: Garde<Json<Pager>>) {
    assert!((1..=50).contains(&pager.page_size));
    assert!((1..).contains(&pager.page_no));
    println!("page_no: {}, page_size: {}", pager.page_no, pager.page_size);
}

pub async fn get_state(_state: State<MyState>) {}

#[derive(Debug, Clone, FromRef)]
pub struct MyState {
    state_field: i32,
    without_validation_arguments: (),
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let router = Router::new()
        .route("/query", get(pager_from_query))
        .route("/json", post(pager_from_json));

    // WARNING: If you are using Garde and also have a state,
    // even if that state is unrelated to Garde,
    // you still need to implement FromRef<StateType> for ().
    // Tip: You can add an () field to your state and derive FromRef for it.
    let router = router.route("/state", get(get_state)).with_state(MyState {
        state_field: 1,
        without_validation_arguments: (),
    });
    axum::Server::bind(&([0u8, 0, 0, 0], 8080).into())
        .serve(router.into_make_service())
        .await?;
    Ok(())
}

```

### 📦 `Validated<E>`, `Modified<E>`, `Validified<E>` and `ValidifiedByRef<E>`

* Install

```shell
cargo add validify
cargo add axum-valid --features validify,basic --no-default-features
```

* Example

Extra dependencies of this example:

```shell
cargo add axum_typed_multipart
cargo add axum-valid --features validify,basic,typed_multipart --no-default-features
```

```rust,ignore
use axum::extract::Query;
use axum::routing::{get, post};
use axum::{Form, Json, Router};
use axum_typed_multipart::{TryFromMultipart, TypedMultipart};
use axum_valid::{Modified, Validated, Validified, ValidifiedByRef};
use serde::Deserialize;
use validify::{Validate, Validify};

#[derive(Debug, Validify, Deserialize)]
pub struct Pager {
    #[validate(range(min = 1.0, max = 50.0))]
    pub page_size: usize,
    #[validate(range(min = 1.0))]
    pub page_no: usize,
}

pub async fn pager_from_query(Validated(Query(pager)): Validated<Query<Pager>>) {
    assert!((1..=50).contains(&pager.page_size));
    assert!((1..).contains(&pager.page_no));
}

#[derive(Debug, Validify, Deserialize)]
pub struct Parameters {
    #[modify(lowercase)]
    #[validate(length(min = 1, max = 50))]
    pub v0: String,
    #[modify(trim)]
    #[validate(length(min = 1, max = 100))]
    pub v1: String,
}

pub async fn parameters_from_json(modified_parameters: Modified<Json<Parameters>>) {
    assert_eq!(
        modified_parameters.v0,
        modified_parameters.v0.to_lowercase()
    );
    assert_eq!(modified_parameters.v1, modified_parameters.v1.trim())
    // but modified_parameters may be invalid
}

// NOTE: missing required fields will be treated as validation errors.
pub async fn parameters_from_form(parameters: Validified<Form<Parameters>>) {
    assert_eq!(parameters.v0, parameters.v0.to_lowercase());
    assert_eq!(parameters.v1, parameters.v1.trim());
    assert!(parameters.validate().is_ok());
}

// NOTE: TypedMultipart doesn't using serde::Deserialize to construct data
// we should use ValidifiedByRef instead of Validified
#[derive(Debug, Validify, TryFromMultipart)]
pub struct FormData {
    #[modify(lowercase)]
    #[validate(length(min = 1, max = 50))]
    pub v0: String,
    #[modify(trim)]
    #[validate(length(min = 1, max = 100))]
    pub v1: String,
}

pub async fn parameters_from_typed_multipart(
    ValidifiedByRef(TypedMultipart(data)): ValidifiedByRef<TypedMultipart<FormData>>,
) {
    assert_eq!(data.v0, data.v0.to_lowercase());
    assert_eq!(data.v1, data.v1.trim());
    assert!(data.validate().is_ok());
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let router = Router::new()
        .route("/validated", get(pager_from_query))
        .route("/modified", post(parameters_from_json))
        .route("/validified", post(parameters_from_form))
        .route("/validified_by_ref", post(parameters_from_typed_multipart));
    axum::Server::bind(&([0u8, 0, 0, 0], 8080).into())
        .serve(router.into_make_service())
        .await?;
    Ok(())
}
```

To see how each inner extractor can be used with validation extractors, please refer to the example in the [documentation](https://docs.rs/axum-valid) of the corresponding module.

## 🚀 Argument-Based Validation

### 📦 `ValidEx<E, A>`

* Install

```shell
cargo add validator --features derive
cargo add axum-valid
# validator is enabled by default
```

* Example

```rust,ignore
use axum::routing::post;
use axum::{Form, Router};
use axum_valid::{Arguments, ValidEx};
use serde::Deserialize;
use std::ops::{RangeFrom, RangeInclusive};
use validator::{Validate, ValidateArgs, ValidationError};

// NOTE: When some fields use custom validation functions with arguments,
// `#[derive(Validate)]` will implement `ValidateArgs` instead of `Validate` for the type.
// The validation arguments will be a tuple of all the field validation args.
// In this example it is (&RangeInclusive<usize>, &RangeFrom<usize>).
// For more detailed information and understanding of `ValidateArgs` and their argument types,
// please refer to the `validator` crate documentation.
#[derive(Debug, Validate, Deserialize)]
pub struct Pager {
    #[validate(custom(function = "validate_page_size", arg = "&'v_a RangeInclusive<usize>"))]
    pub page_size: usize,
    #[validate(custom(function = "validate_page_no", arg = "&'v_a RangeFrom<usize>"))]
    pub page_no: usize,
}

fn validate_page_size(v: usize, args: &RangeInclusive<usize>) -> Result<(), ValidationError> {
    args.contains(&v)
        .then_some(())
        .ok_or_else(|| ValidationError::new("page_size is out of range"))
}

fn validate_page_no(v: usize, args: &RangeFrom<usize>) -> Result<(), ValidationError> {
    args.contains(&v)
        .then_some(())
        .ok_or_else(|| ValidationError::new("page_no is out of range"))
}

// NOTE: Clone is required, consider using Arc to reduce deep copying costs.
#[derive(Debug, Clone)]
pub struct PagerValidArgs {
    page_size_range: RangeInclusive<usize>,
    page_no_range: RangeFrom<usize>,
}

// NOTE: This implementation allows PagerValidArgs to be the second member of ValidEx, and provides arguments for actual validation.
// get() method returns the actual validation arguments to be used during validation.
impl<'a> Arguments<'a> for PagerValidArgs {
    type T = Pager;

    // NOTE: <Pager as ValidateArgs<'a>>::Args == (&RangeInclusive<usize>, &RangeFrom<usize>)
    fn get(&'a self) -> <Pager as ValidateArgs<'a>>::Args {
        (&self.page_size_range, &self.page_no_range)
    }
}

pub async fn pager_from_form_ex(ValidEx(Form(pager), _): ValidEx<Form<Pager>, PagerValidArgs>) {
    assert!((1..=50).contains(&pager.page_size));
    assert!((1..).contains(&pager.page_no));
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let router = Router::new()
        .route("/form", post(pager_from_form_ex))
        .with_state(PagerValidArgs {
            page_size_range: 1..=50,
            page_no_range: 1..,
        });
    // NOTE: The PagerValidArgs can also be stored in a XxxState,
    // make sure it implements FromRef<XxxState>.

    axum::Server::bind(&([0u8, 0, 0, 0], 8080).into())
        .serve(router.into_make_service())
        .await?;
    Ok(())
}

```

### 📦 `Garde<E>`

* Install

```shell
cargo add validator --features derive
cargo add axum-valid
# validator is enabled by default
```

* Example

```rust,ignore
use axum::routing::post;
use axum::{Form, Router};
use axum_valid::Garde;
use garde::Validate;
use serde::Deserialize;
use std::ops::{RangeFrom, RangeInclusive};

#[derive(Debug, Validate, Deserialize)]
#[garde(context(PagerValidContext))]
pub struct Pager {
    #[garde(custom(validate_page_size))]
    pub page_size: usize,
    #[garde(custom(validate_page_no))]
    pub page_no: usize,
}

fn validate_page_size(v: &usize, args: &PagerValidContext) -> garde::Result {
    args.page_size_range
        .contains(&v)
        .then_some(())
        .ok_or_else(|| garde::Error::new("page_size is out of range"))
}

fn validate_page_no(v: &usize, args: &PagerValidContext) -> garde::Result {
    args.page_no_range
        .contains(&v)
        .then_some(())
        .ok_or_else(|| garde::Error::new("page_no is out of range"))
}

#[derive(Debug, Clone)]
pub struct PagerValidContext {
    page_size_range: RangeInclusive<usize>,
    page_no_range: RangeFrom<usize>,
}

pub async fn pager_from_form_garde(Garde(Form(pager)): Garde<Form<Pager>>) {
    assert!((1..=50).contains(&pager.page_size));
    assert!((1..).contains(&pager.page_no));
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let router = Router::new()
        .route("/form", post(pager_from_form_garde))
        .with_state(PagerValidContext {
            page_size_range: 1..=50,
            page_no_range: 1..,
        });
    // NOTE: The PagerValidContext can also be stored in a XxxState,
    // make sure it implements FromRef<XxxState>.
    // Consider using Arc to reduce deep copying costs.
    axum::Server::bind(&([0u8, 0, 0, 0], 8080).into())
        .serve(router.into_make_service())
        .await?;
    Ok(())
}
```

Current module documentation predominantly showcases `Valid` examples, the usage of `ValidEx` is analogous.

## 🗂️ Extractors List


| Extractor             | Backend / Feature | Data's trait bound                                 | Functionality                          | Benefits                                   | Drawbacks                                        |
|-----------------------|-------------------|----------------------------------------------------|----------------------------------------|--------------------------------------------|--------------------------------------------------|
| `Valid<E>`	           | validator	        | `validator::Validate`                              | Validation	                            |                                            |                                                  |                                                 
| `ValidEx<E, A>`	      | validator	        | `validator::ValidateArgs`                          | Validation with arguments              | 		                                         | More complex arguments coding                    |
| `Garde<E>`	           | garde	            | `garde::Validate`                                  | Validation with or without arguments	  |                                            | Require empty tuple as the argument if use state |                                  |
| `Validated<E>`	       | validify	         | `validify::Validate`                               | Validation	                            |                                            |                                                  |
| `Modified<E>`	        | validify	         | `validify::Modify`                                 | Modification / Conversion to response  | 		                                         |                                                  |                                                  
| `Validified<E>`	      | validify	         | `validify::Validify` and `serde::DeserializeOwned` | Construction, modification, validation | Treat missing fields as validation errors	 | Only works with extractors using `serde`         |
| `ValidifiedByRef<E>`	 | validify          | `validify::Validate` and `validify::Modify`        | Modification, validation               |                                            |                                                  |

## ⚙️ Features

| Feature          | Description                                                                                                                              | Module                                       | Default | Example | Tests |
|------------------|------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|---------|---------|-------|
| default          | Enables `validator` and support for `Query`, `Json` and `Form`                                                                           | [`validator`], [`query`], [`json`], [`form`] | ✅       | ✅       | ✅     |
| validator        | Enables `validator` (`Valid`, `ValidEx`)                                                                                                 | [`validator`]                                | ✅       | ✅       | ✅     |
| garde            | Enables `garde` (`Garde`)                                                                                                                | [`garde`]                                    | ❌       | ✅       | ✅     |
| validify         | Enables `validify` (`Validated`, `Modified`, `Validified`, `ValidifedByRef`)                                                             | [`validify`]                                 | ❌       | ✅       | ✅     |
| basic            | Enables support for `Query`, `Json` and `Form`                                                                                           | [`query`], [`json`], [`form`]                | ✅       | ✅       | ✅     |
| json             | Enables support for `Json`                                                                                                               | [`json`]                                     | ✅       | ✅       | ✅     |
| query            | Enables support for `Query`                                                                                                              | [`query`]                                    | ✅       | ✅       | ✅     |
| form             | Enables support for `Form`                                                                                                               | [`form`]                                     | ✅       | ✅       | ✅     |
| typed_header     | Enables support for `TypedHeader`                                                                                                        | [`typed_header`]                             | ❌       | ✅       | ✅     |
| typed_multipart  | Enables support for `TypedMultipart` and `BaseMultipart` from `axum_typed_multipart`                                                     | [`typed_multipart`]                          | ❌       | ✅       | ✅     |
| msgpack          | Enables support for `MsgPack` and `MsgPackRaw` from `axum-msgpack`                                                                       | [`msgpack`]                                  | ❌       | ✅       | ✅     |
| yaml             | Enables support for `Yaml` from `axum-yaml`                                                                                              | [`yaml`]                                     | ❌       | ✅       | ✅     |
| extra            | Enables support for `Cached`, `WithRejection` from `axum-extra`                                                                          | [`extra`]                                    | ❌       | ✅       | ✅     |
| extra_typed_path | Enables support for `T: TypedPath` from `axum-extra`                                                                                     | [`extra::typed_path`]                        | ❌       | ✅       | ✅     |
| extra_query      | Enables support for `Query` from `axum-extra`                                                                                            | [`extra::query`]                             | ❌       | ✅       | ✅     |
| extra_form       | Enables support for `Form` from `axum-extra`                                                                                             | [`extra::form`]                              | ❌       | ✅       | ✅     |
| extra_protobuf   | Enables support for `Protobuf` from `axum-extra`                                                                                         | [`extra::protobuf`]                          | ❌       | ✅       | ✅     |
| all_extra_types  | Enables support for all extractors above from `axum-extra`                                                                               | N/A                                          | ❌       | ✅       | ✅     |
| all_types        | Enables support for all extractors above                                                                                                 | N/A                                          | ❌       | ✅       | ✅     |
| 422              | Use `422 Unprocessable Entity` instead of `400 Bad Request` as the status code when validation fails                                     | [`VALIDATION_ERROR_STATUS`]                  | ❌       | ✅       | ✅     |
| into_json        | Validation errors will be serialized into JSON format and returned as the HTTP body                                                      | N/A                                          | ❌       | ✅       | ✅     |
| full_validator   | Enables `validator`, `all_types`, `422` and `into_json`                                                                                  | N/A                                          | ❌       | ✅       | ✅     |
| full_garde       | Enables `garde`, `all_types`, `422` and `into_json`. Consider using `default-features = false` to exclude default `validator` support    | N/A                                          | ❌       | ✅       | ✅     |
| full_garde       | Enables `validify`, `all_types`, `422` and `into_json`. Consider using `default-features = false` to exclude default `validator` support | N/A                                          | ❌       | ✅       | ✅     |
| full             | Enables all features above                                                                                                               | N/A                                          | ❌       | ✅       | ✅     |

## 🔌 Compatibility

To determine the compatible versions of dependencies that work together, please refer to the dependencies listed in the `Cargo.toml` file. The version numbers listed there will indicate the compatible versions.

If you encounter code compilation problems, it could be attributed to either missing trait bounds, unmet feature requirements, or incorrect dependency version selections.

## 📜 License

This project is licensed under the MIT License.

## 📚 References

* [axum](https://crates.io/crates/axum)
* [validator](https://crates.io/crates/validator)
* [garde](https://crates.io/crates/garde)
* [validify](https://crates.io/crates/validify)
* [serde](https://crates.io/crates/serde)
* [axum-extra](https://crates.io/crates/axum-extra)
* [axum-yaml](https://crates.io/crates/axum-yaml)
* [axum-msgpack](https://crates.io/crates/axum-msgpack)
* [axum_typed_multipart](https://crates.io/crates/axum_typed_multipart)
