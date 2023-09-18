+++
title = "Exclusive Systems in Bevy"
date = 2023-09-18
description = "How do you use exclusive systems in Bevy?"
tags = ["bevy", "tutorial"]
draft = true
+++

# Bevy 0.11: Exclusive Systems

```rust,hide_lines=1-2 6-8
use bevy::prelude::*;

fn my_exclusive_system(world: &mut World) {
  // do something with the world
}

let mut app = App::new();
app.add_system(my_exclusive_system);
```

There are two types of systems that you will encounter regularly in Bevy. Normal systems which are what most people are familiar with. They take `SystemParam`'s like `Res` and `Query` and are allowed to run in parallel as long as they don't break rust's aliasing rules. And then exclusive systems, which take `&mut World` in the first paramter and can only be run "exclusively" i.e. no other systems can run at the same time.

There are a couple big advantages to exclusive systems. One is that all modifications to world happen immediately. The other being when it's hard to specify all your system params up front. i.e. user callbacks. Some disadvantages would be having to run single threaded and that it can be hard to get multiple things out of the world do to mutability rules.

## Modifying the World

### EntityMut

### EntityRef

## ExclusiveSystemParam's

Besides `&mut World` there are 3 other types that are allowed to follow `&mut World` in the function signature.

### `Local`


### `&mut SystemState`


### `&mut QueryState`

## Taking ownership of data




## Getting multiple things out of world

### `resource_scope`

### Resource Hokey-Pokey


### ParamState

## Running Schedules