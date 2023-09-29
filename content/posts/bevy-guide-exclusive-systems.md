+++
title = "Exclusive Systems in Bevy"
date = 2023-09-18
description = "How do you use exclusive systems in Bevy?"
tags = ["bevy", "tutorial"]
+++

> Draft

# Bevy 0.11: Exclusive Systems

```rust,hide_lines=1-2 6-8
use bevy::prelude::*;

fn my_exclusive_system(world: &mut World) {
  // do something with the world
}

let mut app = App::new();
app.add_system(my_exclusive_system);
```

This is meant to be an introduction to exclusive systems and using direct world access.

There are two types of systems that you will encounter regularly in Bevy. Normal systems which are what most people are familiar with. They take `SystemParam`'s like `Res` and `Query` and are allowed to run in parallel as long as they don't break rust's aliasing rules. And then exclusive systems, which take `&mut World` in the first paramter and can only be run "exclusively" i.e. no other systems can run at the same time.

There are a couple big advantages to exclusive systems. One is that all modifications to world happen immediately. The other being when it's hard to specify all your system params up front. i.e. user callbacks. Some disadvantages would be having to run single threaded and that it can be hard to get multiple things out of the world do to mutability rules.

## Adding an Exclusive System to your App

```rust
use bevy::prelude::*;

fn main() {
    let mut app = App::new();
    app.add_systems(Update, my_exclusive_system);
}

// An exclusive system is a system that takes `&mut World` as the first parameter.
fn my_exclusive_system(world: &mut World) {
    // do something with the world
} 
```

## ExclusiveSystemParam's

Besides `&mut World` there are 3 other types that are allowed to follow `&mut World` in the function signature.

### `&mut QueryState` and `&mut SystemState`

```rust,hide_lines=1-12
use bevy::prelude::*;
use bevy::ecs::system::SystemState;

#[derive(Component)]
struct Component1;

#[derive(Component)]
struct Component2;

#[derive(Resource)]
struct Resource1;

fn query_state(
  world: &mut World, 
  // we can specify the query params like we do with `Query`
  query: &mut QueryState<(&Component1, &Component2)>) {
  for (c1, c2) in query.iter(world) {
    // ...
  }
}

fn system_state(
  world: &mut World,
  // we can use any params that work normally with systems
  state: &mut SystemState<(
    ResMut<Resource1>,
    Query<(&mut Component1, With<Component2>)>
  )>
) {
  let (res, query) = state.get_mut(world);
  // do something with `res` and `query`
}
```

`QueryState` and `SystemState` are used to get queries and system params from the world respectively. The difference from the normal `Query` or `derive(SystemParam)` is that they need to have the `World` provided to them to get data out of the world. This step is done for you automatically when a normal system runs.

You can also just construct these types for use with the world.

```rust,hide_lines=1
use bevy::prelude::*;

#[derive(Component)]
struct Component1;

#[derive(Component)]
struct Component2;

let mut world = World::new();

let query_state: QueryState<(&Component1, &Component2)> = QueryState::new(&mut world);
```

The disadvantage here is that QueryState and SystemState often are caching data to make things work or make things faster. There are also types like EventReader that don't work correctly when they aren't cached.

### `Local`

Local is a system param that works on both normal systems and exclusive systems. It's useful for caching local state in a system. In fact `&mut SystemState` and `&mut QueryState` is mostly just sugar for `Local<SystemState>` and `Local<QueryState>`.

## Modifying the World Directly

Normal systems aren't allowed to make structural changes to the world directly. Structure changes are things like inserting or removing components or spawning or despawning an entity. These operations require mutable access to the world which is not allowed from normal systems. From those systems these changes are queued through `Commands` and then applied to the `World` when a `apply_deferred` system is run. This makes things like spawning an entity and then acting on that data more complex. But from an exclusive system these changes happen immediately, so it can be easier if you need to do these more complex actions to do them from an exclusive system.

### Spawning and Despawning Entities

In an exclusive system we can observe spawns and despawns immediately.

```rust,hide_lines=1-8 24-40
use bevy::prelude::*;
use bevy::ecs::schedule::ScheduleLabel;

#[derive(Component)]
struct Component1;

let mut world = World::new();

fn spawn_and_despawn(
    world: &mut World, 
    query: &mut QueryState<Entity, With<Component1>>
) {
    // world has no entities
    assert_eq!(query.iter(world).len(), 0);

    // we can get the spawned entity immediately
    world.spawn(Component1);
    let entity = query.single(world);
 
    // the entity is despawned immediately
    world.despawn(entity);
    assert_eq!(query.iter(world).len(), 0);
}

#[derive(ScheduleLabel, Debug, PartialEq, Eq, Hash, Clone)]
struct TestSchedule;

world.add_schedule(Schedule::new(), TestSchedule);
world.schedule_scope(TestSchedule, |world, schedule| {
    schedule.add_system(spawn_and_despawn);

    schedule.run(world);
})
```

### EntityMut and EntityRef

We can get arbitrary data with an `EntityRef` or insert or remove arbitrary data on an entity with `EntityMut`.

```rust
use bevy::prelude::*;

let mut world = World::new();

#[derive(Component)]
struct Name(String);

#[derive(Component, PartialEq, Debug)]
struct Position {
    x: f32,
    y: f32,
}

// world.spawn returns an `EntityMut`
let mut entity_mut = world.spawn(Name("Brian".to_string()));
entity_mut.insert(Position { x: 0.0, y: 0.0 });
let entity = entity_mut.id();

// remove the name component and get ownership
let mut entity_mut = world.entity_mut(entity);
let name = entity_mut.take::<Name>().unwrap();
assert_eq!(name.0, "Brian");

// get some data
let position = entity_mut.get::<Position>().unwrap();
assert_eq!(position, &Position { x: 0.0, y: 0.0 });
```

You might want to use a EntityRef instead of EntityMut when you want to get multiple references.

```rust, hide_lines=1-9
use bevy::prelude::*;

#[derive(Component)]
struct Name(String);

let mut world = World::new();
let entity1 = world.spawn(Name("Andrea".to_string())).id();
let entity2 = world.spawn(Name("Jerry".to_string())).id();

// entity ref has all the same methods that take &self, but none that take &mut
let entity_ref_1 = world.entity(entity1);
let entity_ref_2 = world.entity(entity2);
assert!(entity_ref_1.contains::<Name>() && entity_ref_2.contains::<Name>());
```

If you tried the same with `entity_mut` you would get a compile error.

```rust, compile_fail, hide_lines=1-9
use bevy::prelude::*;

#[derive(Component)]
struct Name(String);

let mut world = World::new();
let entity1 = world.spawn(Name("Andrea".to_string())).id();
let entity2 = world.spawn(Name("Jerry".to_string())).id();

// This will not compile, because rust does not allow 2 mutable references to exist to the world.
let entity_mut_1 = world.entity_mut(entity1);
let entity_mut_2 = world.entity_mut(entity2);
assert!(entity_mut_1.contains::<Name>() && entity_mut_2.contains::<Name>());

// The compile error message:
// error[E0499]: cannot borrow `world` as mutable more than once at a time
//   --> src\lib.rs:203:20
//    |
// 13 | let entity_mut_1 = world.entity_mut(entity1);
//    |                    ------------------------- first mutable borrow occurs here
// 14 | let entity_mut_2 = world.entity_mut(entity2);
//    |                    ^^^^^^^^^^^^^^^^^^^^^^^^^ second mutable borrow occurs here
// 15 | assert!(entity_mut_1.contains::<Name>() && entity_mut_2.contains::<Name>());
//    |         ------------------------------- first borrow later used here
```

There are also failable versions, `get_entity` and `get_entity_mut`, that return an option.

Check out the docs for more methods on `EntityMut` and `EntityRef`.

## Queries

```rust, hide_lines=1-13
use bevy::prelude::*;

#[derive(Component, Deref, DerefMut)]
struct Name(String);

#[derive(Component)]
struct Player;

let mut world = World::new();
let world = &mut world;

world.spawn((Name(("Alberto".to_string())), Player));

let mut query_state = world.query::<&Name>();
for name in query_state.iter(world) {
    println!("{}", name.0);
}

let player_name = world
    .query_filtered::<&Name, With<Player>>()
    .single(world);

println!("{}", player_name.0);

// use &mut on the component to get a mutable reference and _mut version\
// of `single_mut` or `iter_mut`.
let mut player_name = world
    .query_filtered::<&mut Name, With<Player>>()
    .single_mut(world);
player_name.0 = "Matilda".to_string();

println!("{}", player_name.0); // prints "Matilda"
```

## resources 
* resource_scope
* resource
* resource_mut
* init_resource
* insert_resource
* remove_resource

## Miscelaneous

* removed
* send_event
* send_event_batch
* inspect_entity

## Taking ownership of data

* remove_resource
* take

## ParamState

## Common Patterns

### running a closure

### Running Schedules


### Running systems

> Notes to Self: More realistic components and resource.


2 ways to get entity component data on world. `World::get_mut` and `World::entity_mut` + `EntityMut::get`. Do we want both? feels hard to teach when to use one vs the other.
