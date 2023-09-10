+++
title = "Looking Back at Stageless"
date = 2023-09-10
description = "Reflections on Bevy's Scheduling Rewrite"
tags = ["bevy", "stageless"]
+++

> Draft

# Looking Back at Stageless

[Bevy's 3rd birthday](https://bevyengine.org/news/bevys-third-birthday/) happened recently. In the past year I've participated in game jams with Bevy and contributed in various ways. But for this post I'd like to put a spotlight on the stageless scheduling rewrite that was finished back in March. Much of the heavy lifting for it was done by @alice-i-cecile and @maniwani (with honorable mention to @cart), but I'd like to think I contributed significantly by providing ideas, reviewing things, and fixing bugs in the final implementation. First and foremost I'd like to thank everyone who I don't mention in this post that contributed ideas and reviews. Without you stageless wouldn't have happened.

## Intro

Officially known as "Scheduling V3", the stageless work was a rewrite to scheduling that aimed to solve a bunch of problems with scheduling that had come to be obvious  with numerous issues and help threads on discord. The main issues being:

1. Run criteria were hard to compose due to them having 4 states, so you couldn't use simple boolean logic on them
2. States used looping criteria which could lead to deadlocks if the state was repeatedly changed and could have other surprising behavior.
3. Applying the command buffers to the world required creating a new stage, but stages were hard barriers between systems, so would require moving the systems to the new stage.

Stages were the container for systems and these three problems were intertwined with how stages behaved. Thus "removing" stages and making Bevy "stageless" was how the work was referred to. What actually happened was a reframing and extending of the existing APIs rather than fully deleting stages, but the term stuck around. Through many discussions on discord and github, the major fixes for these problems that we landed on was to:

* Remove looping run criteria and just make run criteria return a simple bool
* Replace looping run criteria with looping inside an exclusive system instead.
* Allow ordering exclusive systems with normal systems, which allows manually adding a sync point for commands to be applied anywhere. Before, stages could only be scheduled at the beginning or end of a stage.

## Timeline

What follows is a rough timeline of the year and a half journey to bring stageless to fruition.

* **Sept 9, 2021** [ECS Mad Science Lab, stageless-ish R&D](https://github.com/bevyengine/bevy/discussions/2801) started by @Ratysz and @alice-i-cecile. This dicussion and #ecs-dev on discord are the central places discussions happen before the RFC is created. Alice notibly created [this graph](https://github.com/bevyengine/bevy/discussions/2801#discussioncomment-1352070) about the complexity of how the problems are tied together.
* **Oct 7, 2021** [Greedy Stageless RFC](https://github.com/bevyengine/rfcs/pull/34). This is my public entrance into working on stageless. The RFC has the concept of ordering exclusive systems and simplifying run criteria to a bool. But has automatic sync points instead of manual, and weird looping criteria as an analog to run criteria. I don't think any of these ideas were particularly novel as they're mentioned in other issues. My motivation in writing it was bring together these ideas and making it clear that the features were possible to support in the system executor.
* **Oct 12, 2021** @maniwani publishes [Hierarchical Stage Labels](https://github.com/bevyengine/rfcs/pull/35). Addressing points 1 and 2 above, but doesn't address 3. It's more focused on making more ergonomic states and tries to elevate states as a more central concept.
* **Jan 14, 2022** [Stageless RFC](https://github.com/bevyengine/rfcs/pull/45). Discussions continue on discord and github between Alice, Maniwani, Ratysz, and myself about how to solve the problems. Eventually using looping in exclusive systems is brought up on discord. This isn't an original idea as Alice had been recommending this to users for complex control flow for while. But realizing we could use this for fixed timesteps and states was. This was the last major piece and Alice proceeds to write the first draft of the Stageless RFC.
* **Mar 20, 2022** [iyes_loopless](https://github.com/IyesGames/iyes_loopless) is published with some of the ideas from the stageless RFC. There are some limitations due to being external to bevy, but this helps prove out some of the ideas in stageless and brings some much needed API improvements to users.
* **Apr 1, 2022** Maniwani publishes the [Scheduling v3 PR](https://github.com/bevyengine/bevy/pull/4391) Draft. At the time, the RFC had yet to be merged. But creating this draft helped clarify and solidify concepts that the rfc was proposing.
* **Sept 19, 2022** Stageless RFC is merged. There are 23 reviewers on the PR and 620 comments. Some of the gap in time here is just things lying dormant due to other project priorities.
* **Nov 13, 2022** [Schedule V3 PR](https://github.com/bevyengine/bevy/pull/6587) is opened by maniwani. This is a cleanup of the original draft PR with things implemented in a different module from the original scheduling code and not integrated into the rest of the engine yet.
* **Jan 16, 2023** Schedule V3 PR is merged with 11 reviewers and 332 comments.
* **Feb 1, 2023** [Base Sets](https://github.com/bevyengine/bevy/pull/7466) is merged. This tries to fix default sets, but unfortunately lost in the discussion is that it doesn't actually fix the [original issue](https://github.com/bevyengine/bevy/issues/7258).
* **Feb 6, 2023** [Engine is migrated to schedule v3](https://github.com/bevyengine/bevy/pull/7267) by Alice and others is merged. This gets rid of the old scheduling code and migrates the rest of the engine.
* **Mar 6, 2023** Bevy 0.10 is released with Schedule V3. In the month between the previous and the release there was a lot of reviewing and fixing of bugs due to the new code.
* **Mar 13, 2023** [Schedule First and Removing Base Sets](https://github.com/bevyengine/bevy/pull/8079). Base sets ended up being a confusing concept for users to learn. This pr removes base sets and makes which schedule a system belongs to an explicit rather than implicit API. This was actually an option considered during the discussions that led to base sets, but base sets were chosen due to being a more succinct api.

## Post Mortem

First of all I'll say that from a user perspective Scheduling V3 is very successful. It reduced the concepts and thinking needed around scheduling and generally made things way more ergonomic. The process to get there had it's ups and downs though.

### Are RFCs useful?
The RFC process was a mix of good and bad. 

### RFCs What went right

* pared down the implementation to a MVP. A lot of feature got cut that were discussed like auto sync points, atomic system sets, and if-needed ordering constraints.
* got to really understand how the pieces of stageless fit together

### RFCs What went wrong

* The final RFC was a little too much for one RFC. Probably should have been split up.
* A lot of the early review was over minor stuff like wording.
* too many API details in early versions which had to be continuously revised as implementations were created.
* Github PR format not the best for editting and commenting on prose.

### Implementation What Went Right

* The API users got was in the end was much more ergonomic and intuitive than what existed before
* Most of what was changed was a reframing and extending the capabilities of what already existed. Doing this led to a simpler mental model needed around scheduling

### Implementation What Went Wrong

* We should have broken up the initial PR. Alice and I had a plan for this, but got pushback. I should have fought a lot harder than I did to actually split things up. Having such a large PR led to a lot of bugs and contributed to Alice and I burning out trying to get things ready for 0.10.
* There's still confusion among users of when to use Schedules vs SystemSets vs States. Some of this is a lack of teaching tools, but some of it suggests that we should try and simplify and combine the concepts further.
* Users still want nested states. Multiple states can work for most scenarios, but cleanup which needs a specific ordering for entering/exiting states can be a problem.
* Calling the whole thing "Stageless". What we ended up with looked a lot like stages at a macro level. Just that the stages allow very different things to happen inside of them.

## So what can we learn for future RFCs, PRs and rewrites?

### RFCs have a weird place in current Bevy

The RFC process is a bit flawed for Bevy currently. A lot of this is due to Github's PR interface which encourages nit picking and is very unideal for editting prose. There should have been a pre-RFC discussion where the actual proposal was available to comment on for the large scale ideas. The forum style discussions would have avoided a lot of nit picking. Cart has been doing this with the assets and scene rewrites. One issue with this seems to be that these never get promoted to being actual rfcs. That's probably ok, but then where do rfcs actually fit into the proces? Maybe for people other than Cart this is needed as a accepted rfc means that controversial prs can be merge without his approval.

### Break up large PRs

Always encourage breaking up large PRs. Large PRs are hard to merge because the most limiting factor with Bevy velocity right now is reviewer time. Large PRs do not respect reviewer time. With large rewrites this probably means figuring out a way for both the original code and the rewrite to live in the repository at the same time. This is not ideal as fixes may need to be applied to both sets of code. But this seems to be the least worst solution.


## Future Scheduling Stuff I may work on
* **Auto Sync Points** I'm currently trying to pick this work back up. Users are currently encouraged to write `(my_command_system, apply_deffered, needs_to_observe_first_systems_commands).chain()`, when there's a problem with a systems not seeing another system's comamnds. The problem with this is that it could lead to an overly linear schedule with too many `apply_deferred` systems. The scheduler can be made to insert sync points automatically when there's a before/after relation between two systems with commands.
* Scheduling still isn't as easy or understandable as writing imperative code. This is partially just a consequence of having multithreaded scheduling, but I'd like to experiment and see if there's a way of making it easier to understand. It might be possible with something like allowing writing the schedule using async/await.
* Data privacy. A lot of my [thinking](https://github.com/hymm/rfcs/blob/side-effect-lifecycle-scheduling/rfcs/36-side-effect-lifecycle-scheduling.md) in the intial period of thinking about stageless was around dirty state and how to prevent the wrong systems from seeing that dirty state. Stageless didn't add anything to help solve these problems. I'd like to continue my thinking around it and see if there's a way to solve the problem in ECS. 