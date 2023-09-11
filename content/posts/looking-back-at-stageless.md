+++
title = "Looking Back at Stageless"
date = 2023-09-10
description = "Reflections on Bevy's Scheduling Rewrite"
tags = ["bevy", "stageless"]
+++

# Looking Back at Stageless

[Bevy's 3rd birthday](https://bevyengine.org/news/bevys-third-birthday/) happened recently, so I'll take this as an opportunity to reflect. In the past year, I've had some significant contributions to Bevy including finishing [pipelined rendering](https://github.com/bevyengine/bevy/pull/6503), but for this post I'd like to focus on the year-and-a-half journey of the Stageless scheduling rewrite. Much of the heavy lifting for it was done by [@alice-i-cecile](https://github.com/alice-i-cecile)(RFC) and [@maniwani](https://github.com/maniwani)(Implementation), but I'd like to think I contributed significantly by providing ideas, reviewing things, and fixing bugs in the final implementation. From gathering ideas to the implementation PR and numerous follow-up PRs, it was quite the project. This post is mostly about thinking about what went right and what went wrong in the process.

## Intro

Officially known as "Scheduling V3", the stageless work was a rewrite of system scheduling that aimed to solve a bunch of problems that had come to be obvious with numerous issues and help threads on Discord. The main issues being:

1. `RunCriteria` were hard to compose due to them having 4 states. This prevented using simple boolean logic on them when trying to combine them and potentially forcing users to deal with all 16 combinations of state.
2. `State` used looping criteria which could lead to deadlocks if the state was repeatedly changed and could have other surprising behavior related to the looping.
3. Applying the command buffers to the world required creating a new stage, but stages were hard barriers between systems, so would require moving the systems to the new stage.

`Stages` were the container for systems and these three problems were intertwined with how stages behaved. Thus "removing" stages and making Bevy "stageless" was how the work was referred to. Rather than completely removing stages, what happened was a reframing and extending of the existing APIs. But the term stuck around. Through many discussions on Discord and GitHub, the major fixes for these problems that we landed on were to:

* Remove looping run criteria and just make run criteria return a simple bool
* Replace looping run criteria with looping inside an exclusive system instead. No engine changes was necessary to enable this. But `States` and `FixedTimeSteps` needed to be ported to use this method.
* Allow ordering exclusive systems with normal systems, which would allow manually adding a sync point for commands to be applied anywhere. Before, exclusive sytems could only be scheduled at the beginning or end of a stage.

## Timeline

What follows is a rough timeline of the year-and-a-half journey to bring Stageless to fruition.

### Pre-RFC Period

* **Sept 9, 2021** [ECS Mad Science Lab, stageless-ish R&D](https://github.com/bevyengine/bevy/discussions/2801) started by @Ratysz and @alice-i-cecile. This discussion and #ecs-dev on Discord are the central places where the brainstorming happens before the RFC is created. Alice notably created [this graph](https://github.com/bevyengine/bevy/discussions/2801#discussioncomment-1352070) about the complexity of how the problems are tied together.
* **Oct 7, 2021** [Greedy Stageless RFC](https://github.com/bevyengine/rfcs/pull/34). This is my public entrance into working on Stageless. The RFC has the concept of ordering exclusive systems and simplifying run criteria to a bool. But has automatic sync points instead of manual and weird looping criteria as an analog to run criteria. I don't think any of these ideas were particularly novel as they're mentioned in other issues. My motivation in writing it was to bring together these ideas and make it clear that the features were possible to support in the system executor.
* **Oct 12, 2021** Maniwani publishes [Hierarchical Stage Labels](https://github.com/bevyengine/rfcs/pull/35). This was focused on addressing points 1 and 2 above, but doesn't address 3. It tried to make more ergonomic states and tries to elevate states as a central concept.
* Discussions continue on Discord and GitHub between Alice, Maniwani, Ratysz, and myself about how to solve the problems. Eventually using looping in exclusive systems is brought up on Discord. This wasn't a fully original idea as Alice had been recommending this to users for complex control flow for a while. But realizing we could use this for fixed timesteps and states was.

### RFC Period

* **Jan 14, 2022** [Stageless RFC](https://github.com/bevyengine/rfcs/pull/45). Moving looping to inside exclusive systems was the last major piece and Alice proceeded to write the first draft of the Stageless RFC.
* **Mar 20, 2022** [iyes_loopless](https://github.com/IyesGames/iyes_loopless) is published with some of the ideas from the stageless RFC. There are some limitations due to being an external plugin, but this helps prove out some of the ideas in Stageless and brought some much needed API improvements to users.
* **Apr 1, 2022** maniwani publishes the [Scheduling v3 PR](https://github.com/bevyengine/bevy/pull/4391) Draft. At this time the RFC had yet to be merged. However, creating this draft helped clarify and solidify the concepts that the RFC was proposing.
* **Sept 19, 2022** Stageless RFC is merged. There are 23 reviewers on the PR and 620 comments. 

### Post RFC Merged

* **Nov 13, 2022** [Schedule V3 PR](https://github.com/bevyengine/bevy/pull/6587) is opened by maniwani. This is a cleanup of the original draft PR with things implemented in a different module from the original scheduling code and not integrated into the rest of the engine yet.
* **Jan 16, 2023** Schedule V3 PR is merged with 11 reviewers and 332 comments.
* **Feb 1, 2023** [Base Sets](https://github.com/bevyengine/bevy/pull/7466) by cart is merged. This tries to fix default sets, but unfortunately lost in the discussion is that it doesn't fix the [original issue](https://github.com/bevyengine/bevy/issues/7258).
* **Feb 6, 2023** [Engine is migrated to schedule v3](https://github.com/bevyengine/bevy/pull/7267) by Alice and others is merged. This gets rid of the old scheduling code and migrates the rest of the engine.
* **Mar 6, 2023** Bevy 0.10 is released with Schedule V3. In the month between the previous and the release, there was a lot of reviewing and fixing of bugs due to the new code.
* **Mar 13, 2023** [Schedule First and Removing Base Sets](https://github.com/bevyengine/bevy/pull/8079) by cart. Base sets ended up being a confusing concept for users to learn. This pr removes base sets and makes which schedule a system belongs to an explicit rather than implicit API. This was an option considered during the discussions that led to base sets, but base sets were chosen due to being a more succinct API.

Note that this was not a continuous process. In some of the major gaps between dates no work was being done on Stageless. This is just how open source work goes sometimes.

## Post Mortem on the RFC process
### What Went Right

* We were able to pare down all the ideas from the earlier discussion down to an MVP. Some things got cut like auto sync points, atomic system sets, and if needed ordering constraints.
* Seeing these things on paper let us understand how all the pieces would fit together. And gave us confidence that the fix was the right fix.

### What Went Wrong

* The GitHub interface for reviewing PRs did not work well. 
    * It was easy for comments to get lost and buried as the text was changed. 
    * Line-by-line editing encouraged nitpicking on phrasing when the larger concepts had not been nailed down yet.
* The final RFC was probably too big. Once the big concepts had been nailed down, it probably should have been broken up. The size of the RFC made it hard to review and hard to revise. Personally each review took 4-8 hours and I did multiple full reviews of the RFC and many partial reviews too. Smaller RFCs would have given better focus to each feature, "Run Conditions", "States and Fixed Timesteps", and "Ordering Exclusive Systems" and it would have been easier to carve out time to review them. 

### How do RFCs fit into current Bevy?

I think that having seen the work that went into the Stageless RFC, people are now reluctant to open their own RFCs. They might think it'll be easier to just open a PR and hash things out there. In a lot of cases they're probably right, but I suspect that there are a handfull of current PRs that would have been better served going through the RFC process as they get hung up on their more controversial aspects until the author gets too busy to continue or loses interest. I personally avoid reviewing some PRs because I don't understand the problem space well enough to weigh-in on the controversial aspects of those PRs.

Stageless itself should probably have had a more formal Pre-RFC period where what features needed to be included and which features should be cut was decided. A lot of the early reviews of Stageless focused too much on the details and we ended up with a decent amount of churn on the prose for features that were eventually cut. 

Cart has been doing something like Pre-RFC's with his recent rewrites like with [assets](https://github.com/bevyengine/bevy/discussions/3972) and [scenes](https://github.com/bevyengine/bevy/discussions/9538). However, these never get promoted to full RFCs. Cart gets away with this because of his BDFL status and his PRs always get priority. This isn't a problem, but it ends up devalueing RFCs more than it should for the average contributor. If the more controversial details have already been hashed out in an RFC. Reviewers can then focus on implementation details and code quality, which more reviewers feel comfortable with. This is all assuming that our experts who review RFCs are able to review RFCs faster than PRs. I'm assuming this is true, but it might be a bad assumption.

Overall, we need more features to go through the RFC process and see if we can iron out the flaws that currently exist and give a better path for controversial changes to be implemented. Especially now that we have SME's that can merge RFCs. Then we can better understand how RFCs fit into the Bevy contribution picture.

## Post Mortem on the Implementation
### What Went Right

* From a user perspective the scheduling rewrite is very successful. After a few follow-up PRs, I've only really seen good things said about the new APIs.
* Most of what was changed was a reframing and extending the capabilities of what already existed. Doing this led to a simpler mental model needed around scheduling and a mostly smooth trasition for existing users.

### What Went Wrong

* Calling the whole thing "Stageless". It gave too much focus to actually removing stages completely, but what we ended up with looked a lot like stages at a macro level. We might have ended up at the macro structure quicker if the name was different.
* We should have broken up the initial PR. Alice and I had a plan for breaking things up but got pushback. I should have fought a lot harder than I did to actually split things up. Having such a large PR led to a lot of bugs and contributed to Alice and I burning out trying to get things ready for 0.10.

### New Rule: Always encourage breaking up large PRs

My new personal rule for my own bevy work and reviewing other PRs is to "always encourage breaking up large PRs". The most limiting factor with Bevy velocity is reviewer time. The larger the PR the larger contiguous block of time a reviewer has to set aside. Large PRs do not respect the limited time that our reviewers have. 

Sometimes breaking things up might not be possible, but I suspect the amount of times that happens is smaller than we think. Ideally we're able to break large work into multiple refactors that keeps the code working. This can be more work for the author, but I think it'll lead to better code and faster velocity. If this isn't an option, we can have the original code and the rewrite to living in the repository at the same time. This is not ideal as fixes may need to be applied to both sets of code. But this sometimes will be the least worst solution if we need multiple PRs to reach feature parity with an existing solution.

## Future Scheduling Stuff I may work on

* **Auto Sync Points** I'm currently trying to pick this work back up. Users are currently encouraged to write `(my_command_system, apply_deffered, needs_to_observe_first_systems_commands).chain()`, when there's a problem with a system not seeing another system's commands. The problem with this is that it could lead to an overly linear schedule with too many `apply_deferred` systems. The scheduler can be made to insert sync points automatically when there's a before/after relation between two systems with commands.
* Scheduling still isn't as easy or understandable as writing imperative code. This is partially just a consequence of having multithreaded scheduling, but I'd like to experiment and see if there's a way of making it easier to understand. It might be possible with something like allowing writing the schedule using async/await.
* Data privacy. A lot of my [thinking](https://github.com/hymm/rfcs/blob/side-effect-lifecycle-scheduling/rfcs/36-side-effect-lifecycle-scheduling.md) in the initial period of thinking about stageless was around dirty state and how to prevent the wrong systems from seeing that dirty state. Stageless didn't add anything to help solve these problems. I'd like to continue my thinking around it and see if there's a way to solve the problem in ECS. 

## Final Thoughts

I'm really proud to have worked on Stageless. It really feels like it brought a lot of value to Bevy users. For myself, I learned a lot about Bevy and got much better at writing and understanding Rust. I look forward to continuing to contribute to Bevy and participate in this community in the future.
