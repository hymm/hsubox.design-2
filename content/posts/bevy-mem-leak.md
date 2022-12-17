+++
title = "Tracking Down a Memory Leak in Bevy"
date = 2022-12-12
description = "Using leak-detector-allocator to diagnose a memory leak"
tags = ["bevy", "memory leak"]
+++

> DRAFT

# Tracking Down a Memory Leak in Bevy

Recently I fixed [a memory leak](https://github.com/bevyengine/bevy/pull/6878) in Bevy.
The issue was reported [here](https://github.com/bevyengine/bevy/issues/6417). The gist
of it was that you could see the memory of Bevy growing with just default plugins plus
a single camera being spawned. This likely meant that the problem was related to rendering
as most of that logic doesn't run unless there is a camera. Because of rust's memory
guarantee's this likely meant something was being added, but not removed. This could
be potentially be an asset or entity being repeated added without being removed or some
collection continually growing. This could be any number of things since rendering is a
complicated area, so I needed more clues about what was happening.

It's about a 2kb/s sized leak from my investigations and so seemed to be something that
was happening during every frame. I knew there were tools for investigating all the
allocations and deallocations happening in a program, so I started to do research on
what was available. I'm on Windows, so a commonly recommended tool like [Valgrind](https://valgrind.org/) 
wasn't an option. While looking for some software that would work on windows, I 
lucked onto [leak-detector-allocator](https://crates.io/crates/leak-detect-allocator)
in crates.io. `leak-detector-allocator` replaces the global allocator and then tracks the 
allocations and deallocations in your rust program and reports any allocations without
a corresponding deallocation. This seemed like it would work reasonably well if I could
get it to work.

Below is code that combines the code from the issue with the example code from `leak-detector-allocator`.
It logs the allocated memory for specific frames, which we will then compare to each other. Note that
using the allocator makes things run around 10x slower and I'm also using debug mode to get the better
symbol names. Hopefully it'll be obvious within the first few hundred frames or so what the memory leak
is or else I'm going to be waiting around a lot. Given the earlier testing, I'm pretty sure
this will work.

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
```
```text
total address:11169, bytes:5827857
```
*// sample output*

The output is a text file with a ton of entries like the above. It has an address, the size 
of the allocation, and a simple stack trace. For the above we see that we allocated 640 bytes 
for a `RawVec<Archetype>`. At the end of the file is the total number of address and bytes that 
are allocated. To find the leak we need to compare the output from different frames. 

VSCode can show the diffs between two different text files. So we use that functionality and then
manually go through the diffs. I'm looking for an allocation that is growing in size or a bunch of somethings
that are not getting deallocated.

![A typical diff between different frames](../deallocate-and-allocate.png "Diff of different frame allocations")
*// diff of allocationed memory from different frames*  

There ends up being a lot of noise in the diff still, since there are things being 
allocated and deallocated with every frame. It fairly easy to ignore most of these 
since our scene is empty and not changing. Most of these diffs line up between frames 
and are of the same size. So these are not contributing to the memory leak and can be safely ignored.

![A suspiciously growing Vec](../growing-vec-entity.png "A suspiciously growing Vec")
// RawVec<Entity> grows from 1024 to 4096

So after sorting through all the allocations there is one that stands out. A `RawVec<Entity>` 
is growing across multiple frames. This is probably a Vec<Entity> as `RawVec` is the 
underlying storage for a Vec. The 30, 180, 360 spacing is so that. A vec is only reallocated 
by doubling it's capacity, so it doesn't grow at a constant rate between individual frames. 
So now we do a search for all the Vec<Entity>'s in the program. Most of them are related to 
rendering and checking them with dbg!(Vec::capacity) shows that most of these are unsurprisingly 
zero since the scene is empty. These are things like VisibleEntities, Bones, etc. Oh, but 
what's this...a Vec<Entity> that stores removed components. RemovedComponents tracks which 
entities have had a component removed in a Vec. It gets reset every frame by calling 
`World::clear_trackers` in the main schedule. But there's more than one World. Checking 
`RemovedComponents`'s capacity on the render world quickly confirms that this is the problem.

An easy fix is to just call clear trackers on the render world, but for consistency and a better fix we call it on all sub apps and 
move the call on the main world out of the schedule and into App::update. That should prevent future issues and prevent memory leaks
for downstream users of sub apps. Running the allocation tracer on 10000 frames confirms that memory us is no longer growing in the example.

I'm not sure how generally useful this technique will be. The scene was fairly static, so
the noisiness of the allocations wasn't too bad. But with a more dynamic scene I imagine
there would be many more allocations to sort through and not all of them will line up
as neatly. But hopefully this can still work to some degree.