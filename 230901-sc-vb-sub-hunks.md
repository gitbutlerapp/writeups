# Virtual Branch Sub-Hunk

So, when we run the `get_applied_status` function from the `get_status_by_branch`
function that takes the diff from the working directory to the base branch, we
essentially take all the hunks and try to assign ownership of them to some
applied virtual branch.

For each hunk we see in a file, we look for previously specified owned hunks in
all virtual branches and see if they match up. This could be either an exact
match, or it could be an intersection.

Note: it would be interesting to also look for similar hunks in other files
which have dissapeared from those files in order to keep them in the correct
branches in case of code movement. But that's another project.

For sub-hunks, we will need to take a hunk we get from the git diff output and
be able to split it into multiple smaller hunks and assign ownership of those to
various branches. Let's take a simple example of the diff output for a Cargo.toml
file change:

```diff
diff --git a/Cargo.toml b/Cargo.toml
index 6c7bc31..7e5307a 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -57,6 +57,8 @@ rand = "0.8.5"
 tracing = "0.1.37"
 tracing-subscriber = "0.3.17"
 tracing-appender = "0.2.2"
+sentry-anyhow = "0.31.0"
+tokio-util = "0.7.8"

 [features]
 # by default Tauri runs in production mode
```

Here we have a change at line 59 that adds two lines (lines 60 and 61). We want
one of these lines to be assigned to branch A and the other to branch B, such
that the diff for branch A would be:

```diff
diff --git a/Cargo.toml b/Cargo.toml
index 6c7bc31..95631a1 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -57,6 +57,7 @@ rand = "0.8.5"
 tracing = "0.1.37"
 tracing-subscriber = "0.3.17"
 tracing-appender = "0.2.2"
+tokio-util = "0.7.8"

 [features]
 # by default Tauri runs in production mode
```

and the diff for branch B would be:

```diff
diff --git a/Cargo.toml b/Cargo.toml
index 6c7bc31..3b61a7c 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -57,6 +57,7 @@ rand = "0.8.5"
 tracing = "0.1.37"
 tracing-subscriber = "0.3.17"
 tracing-appender = "0.2.2"
+sentry-anyhow = "0.31.0"

 [features]
 # by default Tauri runs in production mode
```

And the `ownership` entry for the branches would be:

```
branchA: Cargo.toml:61
branchB: Cargo.toml:60
```

The trick here is to be able to unapply and re-apply the branches in a way that
works as expected. I.e. unapplying branch A should remove just line 61, and
reapplying it should add just that line back in the same order, since
technically those two patches would result in a conflict as both introduce
a change to line 60 of the original base.

Since we are actually starting with the merge conflict resolution in our
directory and creating the conflicting versions manually, we should be able
to record what the resolution is, detect it and apply the recorded resolution
automatically.

## Determining Sub-Hunks

Given a hunk and a set of selections (line numbers or line ranges), we should
be able to return a new set of hunks that can be displayed in the UI and moved
around. This would allow us to create a UI where you can click on lines in a
displayed hunk and then drag them out into a new sub hunk that can be moved to a
different branch.

In other words, given the `diff::workdir` output and the `ownership` entries, if
we see ownership ranges that split a diff in a way that matches entries in a sub
hunk database we keep, we should be able to split that `diff::workdir` output
into smaller hunks in some cases.

If a vbranch contains sub hunks, we can use that to update the ownership
properly by doing the reverse operation - taking the working directory diff hunk,
figuring out if it has been split into subhunks and determining what line numbers
are associated with each subhunk in order to update the vbranch ownership
properly.

## Updating Subhunk Ownership When Files Change

When we see a file change and it updates a hunk that has been previously split,
we will have to identify the original split data and attempt to determine which
subhunk the new changes should update. For example, in the original Cargo diffs,
if we add a new line in the middle, does it go into subhunk A or B? It could
then later be split into a third subhunk in yet another branch.

This really just updates the ownership entries (ie, which branch ownership do we
put new changes into), and then the `get_applied_status` call will use that data
to create the subhunks properly.

## Unapplying a Split Branch

Unapplying a split branch would be fairly straightforward if we can take the
diff and ownership data and create a subhunk specifically for this branch. Then
we just do the same thing we're doing now, taking the base tree and applying the
hunks that belong to it, except in the case of a split hunk, we only apply the
subhunks owned by that branch.

## Applying a Split Branch

Re-applying a split branch is a little more complicated, because if part of the
split hunk is still applied, the other part will almost certainly result in a
merge conflict.

For this reason, when unapplying a split branch, we should record the state of
the hunks before unapplying, given the original hunk as the "resolution" and
the subhunks as the left and right sides. We could do this by recording the SHA
of the contents of each side, similar to the way `git rerere` does it and save
those on disk on in our sqlite database.

Then when re-applying, we would SHA the left and right sides when we see a merge
conflict, lookup if we have a recorded resolution, and if so, use that to resolve
the conflict automatically and updating the ownership fields accordingly.

## Example

Ok, let's walk through an example of how this might work:

We edit the Cargo.toml file. This introduces two new lines (60 and 61).

```diff
diff --git a/Cargo.toml b/Cargo.toml
index 6c7bc31..7e5307a 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -57,6 +57,8 @@ rand = "0.8.5"
 tracing = "0.1.37"
 tracing-subscriber = "0.3.17"
 tracing-appender = "0.2.2"
+sentry-anyhow = "0.31.0"
+tokio-util = "0.7.8"

 [features]
 # by default Tauri runs in production mode
```

Our ownership entries would now look like:

```
branchA: Cargo.toml:60-61
```

We decide to move the `sentry-anyhow` line into a new `branchB`. We create the
new branch in the UI, select our line and drag it into the new branch. The call
is something like:

```js
update_virtual_branch(projectId: branchBId, {ownership: 'Cargo.toml:60'})
```

This just updates the ownership entry for branchB to contain just line 60 and
splits the ownership of branchA into two ranges, so our ownerships are now like
this:

```
branchA: Cargo.toml:61
branchB: Cargo.toml:60
```

Now we request the virtual branch data via a new call to `list_virtual_branches`
to update the UI and this calls `get_status_by_branch` which calls
`get_applied_status` on both of our branches (since they're both applied).

The `get_applied_status` call runs `diff::workdir` which returns our hunk, which
it can see adds a line 60 and a line 61. Here we would need to notice that this
particular hunk is split, since 60 is in `branchB` but 61 is in `branchA` and
would need to split the hunk into two subhunks - one containing just line 60 and
one containing just line 61.

The GitHub Desktop project does something very similar to this with it's version
of patch application. It will iterate through line hunks, determine which to keep,
update the diff header properly and create a patch that is applicable with the
`git apply` command:

https://github.com/desktop/desktop/blob/0d03faca5ee683972575d3e55419251e9ec2c2c7/app/src/lib/patch-formatter.ts#L129

We would return these two subhunks in the proper branches as `VirtualBranchHunk`
objects instead of the hunk that was originally returned from the `diff::workdir`
call.

Note: One issue at this stage would be if we paste some function up higher and it
shifts all the lines and then we run the status call again, will we match the
hunk ranges properly? I don't really know enough about exactly how we handle
this scenario now, but if it's difficult, we could keep a cache of what "splits"
we have by content and try to match based on that. But let's tackle this later,
it should be doable one way or another.

Ok, so now we unapply `branchA`, which does a couple of things. It runs `write_tree`
on all of the applied branches. This would go through the above process again,
giving us the following subhunk for `branchA`:

```diff
diff --git a/Cargo.toml b/Cargo.toml
index 6c7bc31..95631a1 100644
--- a/Cargo.toml
+++ b/Cargo.toml
@@ -57,6 +57,7 @@ rand = "0.8.5"
 tracing = "0.1.37"
 tracing-subscriber = "0.3.17"
 tracing-appender = "0.2.2"
+tokio-util = "0.7.8"

 [features]
 # by default Tauri runs in production mode
```

The `write_tree` call will apply this patch and write a tree out that contains
this change. It will also do the same for the `branchB` side, and then take that
tree and check it out into the working directory, effectively removing the `branchA`
"tokio-util" line.

What it will _also_ now do, if it notices that it's stashing a branch that has
subhunks in it, is update our resolution database. It will take the contents
of the two subhunks and force order them so that the same two subhunks will
always come in the same order, no matter which side they're on. Then it will
concatenate those contents with a NUL byte and SHA it to get a "conflict ID".

We can then store that somewhere. The `git rerere` mechanism won't _quite_ work
for this exactly, because it caches the resolution of entire files, I believe.
Whereas we want to cache the resolution of hunks. We could do something simple
and have a directory under `.git/gitbutler` that has one file per conflict ID
and the contents are simply the resolution.

So, our `write-tree` call would then do something like this. We would take the
contents of both sides and order them (so `sentry-anyhow` would come before
`tokio-util`), create a buffer of:

```
sentry-anyhow = "0.31.0"\n\NULtokio-util = "0.7.8"\n
```

We would SHA this content to get, say `45f317e0d38dd5354fc1cbdefa077d5f6c49fb89`,
then write the resolution to `.git/gitbutler/resolutions/45f317e0d38dd5354fc1cbdefa077d5f6c49fb89`
as the content:

```
sentry-anyhow = "0.31.0"
tokio-util = "0.7.8"
```

Finally, the unapply operation would rewrite the ownership of the applied
branches to be:

```
branchB: Cargo.toml:60
```

Now if we want to re-apply the `branchA`, we would try to apply the branch and
we would see that there is a merge conflict in our Cargo.toml file. If we inspect
the conflict, we would see it look like this:

```
<<<<<<< ours
tokio-util = "0.7.8"
=======
sentry-anyhow = "0.31.0"
>>>>>>> theirs
```

Now we can go into this block, take the contents of each side, order them,
NUL concat them, SHA it and we get `45f317e0d38dd5354fc1cbdefa077d5f6c49fb89`.
We look that up in `.git/gitbutler/resolutions` and find our entry. Then we just
replace the conflict block with that content and mark it as resolved. Since there
are no other conflicts, we can move forward.

When we noticed that we had to apply a resolution, we can then look at the
resolution and see if we can identify sides that each line came from based on
matching content from the conflict sides and apply that to the branch ownership
of each side, so that the ownerships go back to:

```
branchA: Cargo.toml:61
branchB: Cargo.toml:60
```
