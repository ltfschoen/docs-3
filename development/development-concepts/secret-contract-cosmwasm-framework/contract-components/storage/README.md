---
description: An explainer on the varying storage frameworks for Secret contracts
---

# Storage

## How Storage Works

[CosmWasm storage](https://github.com/scrtlabs/cosmwasm/tree/55ed7ffc4110124c8bf24fc59915cc209e558e38/packages/storage) uses a key-value storage design. Smart contracts can store data in binary, access it through a storage key, edit it, and save it. Similar to a HashMap, each storage key is associated with a specific piece of data stored in binary. The storage keys are formatted as references to byte arrays (`&[u8]`).

One advantage of the key-value design is that a particular data value is only loaded when the user explicitly loads it using its storage key. This prevents any unnecessary data from being processed, saving resources.

Any type of data may be stored this way as long as the user can serialize/deserialize (serde) the data to/from binary. Doing this manually every single time is cumbersome and repetitive, this is why we have wrapper functions that does this serde process for us.

All the data is actually stored in `deps.storage` , and the examples below show how to save/load data to/from there with a storage key.

## Storage Keys

Creating a storage key is simple, any way of generating a constant `&[u8]` suffices. Developers often prefer generating these keys from strings as shown in the example below.

```rust
pub const CONFIG_KEY: &[u8] = b"config";
```

For example, the above key is likely used to store some data related to core configuration values of the contract. The convention is that storage keys are often all created in `state.rs`, and then imported to `contract.rs`. However, since storage keys are just constants, they could be declared anywhere in the contract.

The example above also highlights that storage keys are not meant to be secret nor hard to guess. Anyone who has the open source code can see what the storage keys are (and of course this is not enough for a user to load any data from the smart contract).

#### Prefixed Storage

One common technique in smart contracts, especially when multiple types of data are being stored, is to create separate sub-stores with unique prefixes. Thus instead of directly dealing with storage, we wrap it and put all `Foo` in a Storage with key `"foo" + id`, and all `Bar` in a Storage with key `"bar" + id`. This lets us add multiple types of objects without too much cognitive overhead. Similar separation like Mongo collections or SQL tables.

Since we have different types for `Storage` and `ReadonlyStorage`, we use two different constructors:

```rust
use cosmwasm_std::testing::MockStorage;
use cosmwasm_storage::{prefixed, prefixed_read};

let mut store = MockStorage::new();

let mut foos = prefixed(b"foo", &mut store);
foos.set(b"one", b"foo");

let mut bars = prefixed(b"bar", &mut store);
bars.set(b"one", b"bar");

let read_foo = prefixed_read(b"foo", &store);
assert_eq!(b"foo".to_vec(), read_foo.get(b"one").unwrap());

let read_bar = prefixed_read(b"bar", &store);
assert_eq!(b"bar".to_vec(), read_bar.get(b"one").unwrap());
```

Please note that only one mutable reference to the underlying store may be valid at one point. The compiler sees we do not ever use `foos` after constructing `bars`, so this example is valid. However, if we did use `foos` again at the bottom, it would properly complain about violating unique mutable reference.

The takeaway is to create the `PrefixedStorage` objects when needed and not to hang around to them too long.

#### Typed Storage

As we divide our storage space into different subspaces or "buckets", we will quickly notice that each "bucket" works on a unique type. This leads to a lot of repeated serialization and deserialization boilerplate that can be removed. We do this by wrapping a `Storage` with a type-aware `TypedStorage` struct that provides us a higher-level access to the data.

Note that `TypedStorage` itself does not implement the `Storage` interface, so when combining with `PrefixStorage`, make sure to wrap the prefix first.

```rust
use cosmwasm_std::testing::MockStorage;
use cosmwasm_storage::{prefixed, typed};

let mut store = MockStorage::new();
let mut space = prefixed(b"data", &mut store);
let mut bucket = typed::<_, Data>(&mut space);

// save data
let data = Data {
    name: "Maria".to_string(),
    age: 42,
};
bucket.save(b"maria", &data).unwrap();

// load it properly
let loaded = bucket.load(b"maria").unwrap();
assert_eq!(data, loaded);

// loading empty can return Ok(None) or Err depending on the chosen method:
assert!(bucket.load(b"john").is_err());
assert_eq!(bucket.may_load(b"john"), Ok(None));
```

Beyond the basic `save`, `load`, and `may_load`, there is a higher-level API exposed, `update`. `Update` will load the data, apply an operation and save it again (if the operation was successful). It will also return any error that occurred, or the final state that was written if successful.

```rust
let on_birthday = |mut m: Option<Data>| match m {
    Some(mut d) => {
        d.age += 1;
        Ok(d)
    },
    None => NotFound { kind: "Data" }.fail(),
};
let output = bucket.update(b"maria", &on_birthday).unwrap();
let expected = Data {
    name: "Maria".to_string(),
    age: 43,
};
assert_eq!(output, expected);
```

#### Bucket

Since the above idiom (a subspace for a class of items) is so common and useful, and there is no easy way to return this from a function (bucket holds a reference to space, and cannot live longer than the local variable), the two are often combined into a `Bucket`. A Bucket works just like the example above, except the creation can be in another function:

```rust
use cosmwasm_std::StdResult;
use cosmwasm_std::testing::MockStorage;
use cosmwasm_storage::{bucket, Bucket};

fn people<'a, S: Storage>(storage: &'a mut S) -> Bucket<'a, S, Data> {
    bucket(b"people", storage)
}

fn do_stuff() -> StdResult<()> {
    let mut store = MockStorage::new();
    people(&mut store).save(b"john", &Data{
        name: "John",
        age: 314,
    })?;
    OK(())
}
```

#### Singleton

Singleton is another wrapper around the `TypedStorage` API. There are cases when we don't need a whole subspace to hold arbitrary key-value lookup for typed data, but rather a single storage key. The simplest example is some _configuration_ information for a contract. For example, in the n[ame service example](https://github.com/deus-labs/cw-contracts/tree/main/contracts/nameservice), there is a `Bucket` to look up name to name data, but we also have a `Singleton` to store global configuration - namely the price of buying a name.

Please note that in this context, the term "singleton" does not refer to [the singleton pattern](https://en.wikipedia.org/wiki/Singleton\_pattern) but a container for a single element.

```rust
use cosmwasm_std::{Coin, coin, StdResult};
use cosmwasm_std::testing::MockStorage;

use cosmwasm_storage::{singleton};

#[derive(Serialize, Deserialize, Clone, Debug, PartialEq, JsonSchema)]
pub struct Config {
    pub purchase_price: Option<Coin>,
    pub transfer_price: Option<Coin>,
}

fn initialize() -> StdResult<()> {
    let mut store = MockStorage::new();
    let config = singleton(&mut store, b"config");
    config.save(&Config{
        purchase_price: Some(coin("5", "FEE")),
        transfer_price: None,
    })?;
    config.update(|mut cfg| {
        cfg.transfer_price = Some(coin(2, "FEE"));
        Ok(cfg)
    })?;
    let loaded = config.load()?;
    OK(())
}
```

`Singleton` works just like `Bucket`, except the `save`, `load`, `update` methods don't take a key, and `update` requires the object to already exist, so the closure takes type `T`, rather than `Option<T>`. (Use `save` to create the object the first time). For `Buckets`, we often don't know which keys exist, but `Singleton`s should be initialized when the contract is instantiated.

Since the heart of much of the smart contract code is simply transformations upon some stored state, we may be able to just code the state transitions and let the `TypedStorage` APIs take care of all the boilerplate.

### Summary

CosmWasm storage is built on a key-value storage design, similar to a HashMap, where data is stored in binary format and accessed using storage keys represented as byte arrays (`&[u8]`). Developers can serialize and deserialize data to/from binary, simplifying this process through wrapper functions. Storage keys are typically constants declared in state files and are not meant to be secret.&#x20;

Prefixed storage allows for creating separate sub-stores with unique prefixes for organizing different data types, while Typed Storage provides a type-aware interface to reduce serialization boilerplate. CosmWasm also supports advanced abstractions like Buckets and Singletons. Buckets create subspaces for storing collections of items, while Singletons manage single elements, typically used for global contract configuration data. These APIs streamline storage handling, enabling efficient data management in smart contracts.

Now, we will explore some of these storage methods, such as `PrefixedStorage`, `Singleton`, and `Keymap`, in more depth to see how they interact with on-chain data.
