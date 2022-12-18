+++
title = "Tracking Down a Memory Leak in Bevy"
date = 2022-12-12
description = "Using leak-detector-allocator to diagnose a memory leak"
tags = ["bevy", "memory leak"]
+++

> DRAFT

# Tracking Down a Memory Leak in Bevy

Recently I fixed [a memory leak](https://github.com/bevyengine/bevy/pull/6878) in Bevy.
The issue was reported [here](https://github.com/bevyengine/bevy/issues/6417). The example
code just uses the default plugins and spawns a single camera. 

```rust
#![windows_subsystem = "windows"]

use bevy::prelude::{App, Camera2dBundle, Commands, DefaultPlugins};

fn main() {
    App::new()
        .add_plugins(DefaultPlugins)
        .add_startup_system(setup_camera)
        .run();
}

fn setup_camera(mut commands: Commands) {
    commands.spawn().insert_bundle(Camera2dBundle::default());
}
```

So with some simple theorizing, we can guess that the problem is related to rendering.
Most of that logic doesn't run when there isn't a camera. Unfortunately this doesn't
narrow the problem much as rendering is a fairly large portion of the code base. We also
probably know that this is probably caused by some thing that keeps growing in size or 
a something like a circular reference. This is because rust is pretty good at deallocating
things automatically when they are no longer used. This theorizing doesn't really narrow things
down too much, so we're going to need some more information to find the leak.

Using some basic memory monitoring tools, we can see that it's about a 2kb/s sized 
leak. So it seems to be something that is happening during every frame. There are tools 
for investigating all the allocations and deallocations happening in a program. So hopefully we
can use one of those. Unfortunately I'm on Windows, so [Valgrind](https://valgrind.org/) 
wasn't an option. It was something that commonly came up when googling for tools.
I eventually found [leak-detector-allocator](https://crates.io/crates/leak-detect-allocator)
in crates.io. It seemed pretty promising. `leak-detector-allocator` replaces the global 
allocator in rust and then tracks the allocations and deallocations in your rust program.
It only reports any allocations without a corresponding deallocation. This seemed like 
it would work reasonably well if I could get it to work. 

After a bit of weirdness with the program crashing due to the stack getting too big. I
was able to cobble together some code that did work. Below is code that combines the code 
from the issue with the example code from `leak-detector-allocator`. It logs the allocated 
memory for specific frames, which we can then compare to each other. Note that
using the allocator makes things run around 10x slower and I'm also using debug mode to get the better
symbol names. So hopefully it'll be obvious within the first few hundred frames or so what the memory leak
is or else I'm going to be waiting around a lot. Given the earlier investigations with just watching the
total memory use, I'm pretty sure this will work.

```rust
use bevy::{
    app::AppExit,
    prelude::{App, Camera2dBundle, Commands, DefaultPlugins, EventWriter, Local},
};
use leak_detect_allocator::LeakTracer;

#[global_allocator]
static LEAK_TRACER: LeakTracer<10> = LeakTracer::new();

fn main() {
    dbg!("start");
    LEAK_TRACER.init();
    let mut app = App::new();
    app.add_plugins(DefaultPlugins)
        .add_startup_system(setup_camera)
        .add_system(log_leaks);
    app.run();
}

fn setup_camera(mut commands: Commands) {
    commands.spawn(Camera2dBundle::default());
}

fn log_leaks(mut c: Local<i32>, mut exit: EventWriter<AppExit>) {
    // count the frames
    *c += 1;
    // log to file when we get to certain frame counts
    if *c == 2 || *c == 60 || *c == 120 || *c == 180 || *c == 360 {
        let mut out = String::new();
        let mut count = 0;
        let mut count_size = 0;
        LEAK_TRACER.now_leaks(|address: usize, size: usize, stack: &[usize]| {
            count += 1;
            count_size += size;
            out += &format!("leak memory address: {:#x}, size: {}\r\n", address, size);
            for f in stack {
                // Resolve this instruction pointer to a symbol name
                out += &format!(
                    "\t{:#x}, {}\r\n",
                    *f,
                    LEAK_TRACER.get_symbol_name(*f).unwrap_or("".to_owned())
                );
            }
            true // continue until end
        });
        out += &format!("\r\ntotal address:{}, bytes:{}\r\n", count, count_size);
        let path = format!("foo-{}.txt", *c);
        std::fs::write(path, out.as_str().as_bytes()).ok();
    }

    // exit app when we're done logging.
    if *c > 360 {
        exit.send(AppExit);
    }
}
```

*// sample output*
```text
leak memory address: 0x1a112603bf0, size: 640
	0x7ff773919fcd, backtrace::backtrace::trace_unsynchronized<leak_detect_allocator::impl$0::alloc_accounting::closure_env$0<10> >
	0x7ff773919fcd, backtrace::backtrace::trace_unsynchronized<leak_detect_allocator::impl$0::alloc_accounting::closure_env$0<10> >
	0x7ff7739116a6, leak_detect_allocator::LeakTracer<10>::alloc_accounting<10>
	0x7ff773911a2f, leak_detect_allocator::impl$1::alloc<10>
	0x7ff77391822c, ecs_guide::_::__rg_alloc
	0x7ff775a2024f, alloc::alloc::Global::alloc_impl
	0x7ff775a20b7d, alloc::alloc::impl$1::allocate
	0x7ff775a1d148, alloc::raw_vec::finish_grow<alloc::alloc::Global>
	0x7ff775998741, alloc::raw_vec::RawVec<bevy_ecs::archetype::Archetype,alloc::alloc::Global>::grow_amortized<bevy_ecs::archetype::Archetype,alloc::alloc::Global>
	0x7ff77599c0d9, alloc::raw_vec::RawVec<bevy_ecs::archetype::Archetype,alloc::alloc::Global>::reserve_for_push<bevy_ecs::archetype::Archetype,alloc::alloc::Global>
    
total address:11169, bytes:5827857
```

The output is a text file with a ton of entries like the above. It has an address, the size 
of the allocation, and a simple stack trace. For the above we see that we allocated 640 bytes 
for a `RawVec<Archetype>`. At the end of the file is the total number of address and bytes that 
are allocated. To find the leak we will need to compare the output from different frames as bevy
is a program and uses memory so there will be many allocations that show. // awkward phrasing 

VSCode can show the diffs between two different text files. So we use that functionality and then
manually go through the diffs. We're looking for an allocation that is growing in size or a bunch 
of somethings that are never getting deallocated.

![A typical diff between different frames](../deallocate-and-allocate.png "Diff of different frame allocations")
*// diff of allocationed memory from different frames*  

Even just looking at the diffs there ends up being a lot of things that are just
normal operation of the program. It's pretty normal for things to be allocated for
a bit, but then get dealllocated later. A lot of these are happening every frame
and it's fairly easy to ignore most of these. We can identify these since they are
the same size and even line up in the diff frame to frame. This assumption would 
have been harder to make if the scene was more dynamic.

Scanning through the diffs that aren't so obvious, I end up seeing this suspicious entry.

![A suspiciously growing Vec](../growing-vec-entity.png "A suspiciously growing Vec")
// RawVec<Entity> grows from 1024 to 4096 from frame 120 to 360

The first thing to notice that the allocation on the right is fairly large compared 
to others at 4096 bytes. And we can also see a similar deallocation on the left that is
of the same `RawVec<Entity>` type that is only 1024 bytes.

So it seems a [`RawVec<Entity>`](https://doc.rust-lang.org/src/alloc/raw_vec.rs.html#31) is growing 
across multiple frames. This is a probably actually a `Vec<Entity>` 
as `RawVec` is the underlying storage for a `Vec`. One thing to note about a vec is 
typically reallocated by doubling it's capacity, so we don't always see it grow 
between frames. In this case it takes 240 frames to go to 4x the capacity. This pattern
of growth also seemed to be happening when watching the total memory used value in something
like the windows task manager. As the memory leak grew larger the time it would take for
the amount of memory commited to change would take longer and longer. So this seems like a
likely suspect.

Now we just need to figure out which `Vec<Entity>` is growing. Doing a text search shows some
candidates. 

![Search for Vec<Entity>](../vec-entity-search.png "Search for Vec<Entity>")
// Small clip of VSCode search for Vec<Entity>

Most of the things found are related to rendering. These are things like 
caching the VisibleEntities, Bones, etc. Checking them by running a `dbg!(Vec::capacity)` 
shows that most of these are zero. This is unsurprising since the scene is empty, but
still worth ruling out.

Oh, but what's this...a Vec<Entity> is used to store which entities have removed a component
the previous frame. It normally gets reset by calling `World::clear_trackers` in the main 
schedule, so is probably not leaking there. But besides the main world there
is a separate render world. The render world exists to completely separate the data
needed for simulation vs the data needed for rendering, so that the two schedules can
run in parallel. It's possible that `clear_trackers` is not being called on the render
world. Checking `RemovedComponents`'s capacity on the render world quickly confirms that this 
is the problem. And from a quick search it is confirmed that `clear_trackers` is only being
called on the main world. Horay! We've found at least one memory leak.

An easy fix is to just call `clear_trackers` in the render world's schedule. But rendering
is using an abstraction called `SubApp`'s and users are able to add their own sub apps. So
it'd be best to fix this in a way that works both for all aub apps. So instead we run it in
`App::update` so it can't be forgotten to be called as long as we're using `bevy_app`. We 
also move the call for the main world into this function for consistency's sake.

```rust
pub fn update(&mut self) {
    self.schedule.run(&mut self.world);

    for sub_app in self.sub_apps.values_mut() {
        (sub_app.runner)(&mut self.world, &mut sub_app.app);
        sub_app.app.world.clear_trackers();
    }

    self.world.clear_trackers();
}
```

Running the allocation tracer with this fix for 10000 frames confirms that memory use is 
no longer growing in the example. So it's confirmed this at least fixes the memmory leaks
in the simple example.

I'm not sure how generally useful this technique will be. The scene was fairly static, so
the noisiness of the allocations wasn't too bad. But with a more dynamic scene I imagine
there would be many more allocations to sort through and not all of them will line up
as neatly. But hopefully this can still work to some degree.