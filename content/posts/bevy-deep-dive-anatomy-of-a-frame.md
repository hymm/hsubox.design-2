+++
title = "Bevy Deep Dive: Anatomy of a Frame"
date = 2026-01-30
description = "How to read a bevy frame captured in tracy"
tags = ["bevy", "bevy-deep-dive"]
+++

> Draft


![Overview](../tracy-capture-overview.png "Tracy capture of a whole frame")

Looking at a bevy trace from tracy can be a bit confusing. This is a walkthrough of how bevy code translates to what you see from tracy.

## Plugins

## Apps and Sub Apps

* App::run
    * App Runner (ex. Winit, Run Once, Simple)
        * App::update
            * SubApp::update
            * SubApp::extract

### `App::run`

`Plugin::build` gets called when calling `App::add_plugins`. When `App::run` gets run this calls the app runner. Without `DefaultPlugins` this runs the app once. With `MinimalPlugins` this calls `App::update` in a loop. With `DefaultPlugins`, the app::update is called by the winit runner which coordinates with the os to get the swap chain.

### `App Runner`

Responsible for calling Plugin::ready, Plugin::finish, Plugin::clean in the correct way.

and Calls App::update. 

[configure](https://docs.rs/bevy/0.18.0/bevy/app/struct.App.html)

### Winit Runner

With `WinitPlugin`, `App::update` is called on the `` windowing event.

### `App::update`

Consider it being one frame's worth of updates.

Called by the runner. Runs the `update_schedule` in the sub app.  Users may want to [override](https://docs.rs/bevy/0.18.0/bevy/app/struct.SubApp.html) the default update schedule for more control. After the main app schedule is run, then the sub app main schedules are run. By default there are two sub apps. The RenderExtract app and the Render app. When pipelined rendering is enabled only the extract app is directly run from `App::update`. The render app is controlled separately.

Sub apps run extract and then update [`SubApps::update`](https://github.com/bevyengine/bevy/blob/v0.18.0/crates/bevy_app/src/sub_app.rs#L511-L527)


### Pipelined Rendering

![App and Render Thread](../tracy-capture-pipelined-rendering.png "Tracy capture of app and render thread")
*App and Render Thread*

The goal of pipelined rendering is to have 2 frames in flight at the same time. We separate this into the simulation that is run on the app thread and things that are done for rendering that are done on the rendering thread. There is an extract phase where both the render app and the main app are on the same thread.

Render app is removed from `App` and sent to render thread

## Schedules

A schedule is responsible for running systems and sync points. There are a number of worker threads that can be used to run systems. Schedules can use either a single threaded executor or a multithreaded executor

## Main App

Order of Schedules

Run once on the first call to App::update:
Configurable with `MainStartupOrder` resource.

MainStartup
    1. StateTransition
    * PreStartup
    * Startup
    * PostStartup

Run on every call to App::update:

* MainScheduleOrder
    1. First
    2. PreUpdate
    * StateTransition
        * DependentTransitions
        * ExitSchedules
        * TransitionSchedules
        * EnterSchedules
    * RunFixedMainLoop
        * FixedMain
    * Update
    * SpawnScene
    * PostUpdate
    * Last

## Render App

RenderStartup Schedule

Render Schedule
Render Sets:
* ExtractCommands
* Queue
* Prepare
* Render
* Cleanup
* PostCleanup

## Rendering `Extract` Schedule

runs in SubApp::extract. Main world is injected into the render app `MainWorld(World)` resource to allow extracting data from the main world to the render world.

## `RenderExtract` App

When pipelined rendering is enabled, responsible for waiting for the render world to be sent back. Runs the `Extract` Schedule when it gets the render world and then sends the render app back to the render thread.

## Other Schedules


## Sync Points and Commands (Deferred)

## `Query::par_iter` and other parallelism

