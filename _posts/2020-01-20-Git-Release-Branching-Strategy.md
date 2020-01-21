---
layout: post
title: Keeping Git Release Branches Clean
date: 2020-01-20
category: coding
tags: [git]
---

Let's say you have branched off a release of a git repository, and you have a lot of changes that come in after this. Some changes are meant only for the `release`, some are meant only for `master`, and some are meant for both. How should you push these changes without causing merge conflicts in the future, or corrupting any of the branches with unwanted changes?

## Description
Our company does development in long-running git feature branches, and we have release branches that cannot be merged to/from master. We try to avoid `git cherry-pick`, and prefer `git merge`[*](#Note) instead. However, when we have features that need to be committed to both master and release branches, we end up having lots of conflicts because we cannot merge from either branch for fear of corrupting it.

#### Note
* I would also suggest using `git merge --no-ff`, to prevent fast-forward merges.

## Example
For example, let's say that we have three features that we would *like* to get in to Release 2.0:
* Feature `A` *(easy, will probably make it into Release 2.0)*
* Feature `B` *(more difficult, might have to wait until Release 2.1)*

Once we branch the following three features from `master`, our remote branch list looks like this (excluding any previous `release/*` and `tag/*` branches):
* `master`
* `feature/A_easy`
* `feature/B_hard`

When the time for a code freeze of Release 2.0 comes around, we create a new branch for it:
* `release/2.x`

Note that it is called "Release 2.x", so Release 2.0 and any subsequent minor revisions will all be tagged from this branch. At no point will we do a wholesale merge of `release/2.x` branch back into `master`, or vice versa.

## Avoid `git cherry-pick`
We're following Raymond Chen's guidance ([part 1] and [part 2]), and avoiding `git cherry-pick` as much as possible. If we have changes that are destined for Release 2.x, we merge that feature into the `release/2.x` branch. Changes that can wait until Release 3.x are instead merged into `master`. **Changes that are necessary in both Release 2.x and 3.x are merged into both the `release/2.x` and `master` branches.**

That last sentence in the previous paragraph is fundamental to our workflow. We have a lot of long-lived branches, being developed on by multiple people in multiple groups. If a feature is fundamental for a release (e.g. Feature `A` above), we will delay the release until the feature is completely. Other times, we might have a feature that doesn't have to get into the next major release (e.g. Feature `B` above), but we'd still like it in a minor release (Release 2.1). Again, both the `feature/A_easy` and `feature/B_hard` branches were branched off of `master` before the `release/2.x` branch was even created.

## Solution
I *believe* that if we created one additional branch when we branched `release/2.x`, we could have a tidy solution:
* `release_and_master`

Now, we have a feature branch that we want to merge for a release, we still have the same three options:
* `master` *(Release 3.x)*
* `release/2.x` *(Release 2.x)*
* `release_and_master` *(Release 2.x and Release 3.x)*

If we wish to merge a feature to any of the three branches above, we can merge that branch into our feature branch, and then vice versa:
* Merging `feature/A_easy` into `release/2.x`:
~~~ shell
git checkout feature/A_easy
git merge release/2.x
git checkout release/2.x
git merge feature/A_easy
~~~

There are, however, a few extra steps that must be performed after merging to `release_and_master`, namely merging `release_and_master` into the `release/2.x` and `master` branches:
~~~ shell
git checkout master
git merge release_and_master
git push
git checkout release/2.x
git merge release_and_master
git push
~~~
**At no point should `master` or `release/2.x` ever be merged into the `release_and_master` branch. Merges should only flow out from that branch.**

## Conclusion
This method is nice because it avoids the headaches associated with `git cherry-pick`, and preserves the history of a feature as a single merge group. There's no risk of infecting one branch with contents of another. The only downside is that there is one extra step to take every time you want to merge to both `master` and `release`; there is now a push to `release_and_master`, followed by a merge from there to both `master` and `release`. You'd be doing those last two merges in any case, though, so I'd argue you've just swapped the headache of finding a moving common merge point with the extra commit to `release_and_master`.

Will this work? Probably. Are there any issues with it? Not that I have seen, but thus far I have only tested it out on a toy repository. I'd like to try it out more in the future, to see if there are any kinks that need to be worked out.

[comment]: References
[part 1]: https://devblogs.microsoft.com/oldnewthing/?p=98215 "Raymond Chen: Stop cherry-picking (part 1)"
[part 2]: https://devblogs.microsoft.com/oldnewthing/?p=98225 "Raymond Chen: Stop cherry-picking (part 2)"
