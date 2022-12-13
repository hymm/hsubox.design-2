+++
title = "Bevy Mem Leak"
date = 2022-12-12
+++
# Tracking Down a Memory Leak in Bevy

A memory leak was reported in [a bevy issue](https://github.com/bevyengine/bevy/issues/6417). 
Where a bevy app leaked memory with only a camera added. Since this is rust
the memory leak was likely from something that was growing like an asset or entity that
was being repeatedly added. While this was a fairly stripped down app, bevy still has a 
fair amount of things going on in it's default configuration that figuring out what was 
causing the memory leak was going to be a bit of a pain if there wasn't more information 
to go on. It's about a 2kb sized leak from my investigations.

I'm on Windows, so [Valgrind](https://valgrind.org/) wasn't an option. While looking for
an option that would work, I lucked onto [leak-detector-allocator](https://crates.io/crates/leak-detect-allocator)
in crates.io. `leak-detector-allocator` replaces the global allocator and then tracks the 
allocations and deallocations in your rust program and reports any allocations without
a corresponding deallocation. 

I combined the example from the issue and the example code from the crate to log the allocations
at specific frames. Using this allocator makes things run about 10x slower and I'm using debug mode
too, so hopefully I can get the data I need from the first few 100 frames or so. Given the size of the
leak I think this will probably work.

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

I used the VSCode compare files functionality to find the diffs between different frames. And then
manually go through the diffs to see if I can find an allocation that is growing in size or a lot of small
things not being deallocated.

// picture of one frame compared to another

So just comparing two frames is not enough, because some things are being allocated and deallocated every frame.
But by comparing three frames to each other we can ignore a lot of the noise, because even though there are these things.
The actual things being rendering is very static. i.e. an empty scene.

So after sorting through all the allocations there is one that stands out. A `RawVec<Entity>` is growing across multiple frames.
This is probably a Vec<Entity> as `RawVec` is the underlying storage for a Vec.
The 30, 180, 360 spacing is so that. A vec is only reallocated by doubling it's capacity, so it doesn't grow at a constant rate
between individual frames. So now we do a search for all the Vec<Entity>'s in the program. Most of them are related to rendering
and checking them with dbg!(Vec::capacity) shows that most of these are unsurprisingly zero since the scene is empty. These are
things like VisibleEntities, Bones, etc. Oh, but what's this...a Vec<Entity> that stores removed components. RemovedComponents
tracks which entities have had a component removed in a Vec. It gets reset every frame by calling World::clear_trackers in the main
schedule. But there's more than one World. Checking RemovedComponents's capacity on the
render world quickly confirms that this is the problem.

An easy fix is to just call clear trackers on the render world, but for consistency and a better fix we call it on all sub apps and 
move the call on the main world out of the schedule and into App::update. That should prevent future issues and prevent memory leaks
for downstream users of sub apps.