# GitButler Op Log Format v2
In the beginning, we had a player.

![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711379701715_CleanShot+2024-03-25+at+16.14.382x.png)


One of our original concepts was that if you had a version control system, it shouldn’t be possible to lose work. Why save only when you commit? Why not just save all the time?

So we made snapshots called “[sessions](https://www.notion.so/gitbutler/Session-Format-236e0251923c45a0814f0de635649c67)” and CRDTs of files in between sessions, so in theory we could get back to any version of any file. However, this proved a little useless. Nobody really used it for much and we haven’t missed it’s functionality much after removing the player.

However, we *still* want to save state and we would like to have an undo type functionality. So, instead of ripping out everything we were doing, let’s simplify the format to basically an operations log. Not unlike what JJ does:

![https://youtu.be/IySXs8i_V8Y?si=gbJXaVGb85NT4Ku1&t=997](https://camo.githubusercontent.com/7ce02fd19ee53f6fc07ff0a5ae74715988d9081dfde0bce3fe2034a0fe516f3a/68747470733a2f2f70617065722d6174746163686d656e74732e64726f70626f7875736572636f6e74656e742e636f6d2f735f413945333843413831424633453738353731454637433041344545344645363434443130454139303041453345414637423145434334363442324438354334375f313731313337393936373638305f436c65616e53686f742b323032342d30332d32352b61742b31362e31362e353332782e706e67)
[my internal jj demo](https://youtu.be/IySXs8i_V8Y?si=gbJXaVGb85NT4Ku1&t=997)


## What we need to save

First, let’s back up a bit. What all does GitButler need to save in addition to the normal Git project data?


1. app and user settings (telemetry bit, ssh key, recent projects, name, email)
2. project specific settings (use ai bit, force push ok, etc)
3. project app state
    - virtual branch data
    - hunk supplementary data
    - conflicts state
    - resolutions state
4. project time series / app state history
    - all app state data
    - working directory state
5. log data

Currently we’re saving `1.` mostly in `user.json` and to some degree, `projects.json` in the App directory.

We’re saving `2.` mostly in the `projects.json` file in the App directory.

We’re saving `3.` and `4.` in the `projects` directory (and `projects/gitbutler` subdirectory) in the App directory.

We’re saving `5.` in the App logs directory.


## Moving Settings

The first suggestion is to move our app and user settings mostly into Git config settings, reusing the rather capable and libgit2 supported Git Config mechanism, rather than rolling our own. We can put things in global or project specific settings (or even global in `/etc`) and overwrite values in a cascading way that is easy to change, etc.

We will probably need to figure out how to synchronize this data with the response from the user API endpoint, as right now we mostly just save that json data directly. For instance, if someone updates their name in our UI, we need to update the Git Config global value *and* update the GitButler server via API to persist. If you connect on a second client, that value should be pulled down and overwrite the global config value on that second client’s device.

We would also like to move the project-specific config values (authentication choice, AI bit, force push option) to the normal `.git/config` file. 


## Updating Log Format

We also discussed updating the log format to be [JSON lines](https://jsonlines.org/) formatted so it can be filtered and processed with tools like `jq` or otherwise programatically.


## Moving Project State to the Project Directory

Currently we’re keeping project state data (such as the virtual branch data) in the App data directory (`~/Library/Application Support/com.gitbutler.app.nightly/projects/`). 

This consists of a sessions based Git repository and the `gitbutler` subdirectory with the virtual branch data:


    ❯ tree -L 3 gitbutler/branches/
    gitbutler/branches/
    ├── 749553ac-462d-4c27-bfc0-d9d6bba9eb2e
    │   ├── id
    │   └── meta
    │       ├── applied
    │       ├── created_timestamp_ms
    │       ├── head
    │       ├── name
    │       ├── notes
    │       ├── order
    │       ├── ownership
    │       ├── tree
    │       └── updated_timestamp_ms
    └── target
        ├── branch_name
        ├── remote_name
        ├── remote_url
        └── sha

Currently this branch data is *also* written to a `virtual_branches.toml` file in the project `.git` directory and we’re suggesting that we move to this being the only place this data is stored.

If we’re going to move all the project state data to the project’s `.git` directory, we might want to put it all in a `.git/gitbutler` subdirectory. So this would store the `virtual_branches.toml` file, the `gitbutler.json` file, and any future data like hunk supplementary data, conflict state or resolutions data. This way if the project directory is moved, everything goes with it.


## Time Series / History Data

For the “time series” data, or in other words, some concept of the history of the app state such that previous states can be reconstructed, we’re currently keeping [“Session” data](https://www.notion.so/gitbutler/Session-Format-236e0251923c45a0814f0de635649c67) as a second git repository that has the main project git repository as a [git alternate](https://git-scm.com/docs/gitrepository-layout#Documentation/gitrepository-layout.txt-objectsinfoalternates).

There are a few issues with this structure. 

One is that there is constantly a chance that the main project will run a pruning garbage collection and will remove objects that we rely on from an alternate standpoint (ie, objects written into the project repo that are referenced from the gb repo but the project repo removes thinking they’re unreferenced).

Other issues are that the same object could be written to both databases unnecessarily, deltification could be sub optimal because only some objects know about others, if the project is moved, the alternate path is out of date, etc.

There is also the issue that our client does not currently use the session data. There is no visualization in the UI anymore, no part of our functionality depends on the session data. It is somewhat complex and almost completely unused (except in [manual data recovery](https://docs.gitbutler.com/troubleshooting/recovering-stuff#gitbutler-sessions) scenarios - but even in those scenarios the CRDT data is normally not used, just the working directory snapshot data).


## Our Time Series / History Data Options

There are several options for how to make this better, but first we should define what our functional objectives are with this data.

Originally, the idea was to record the entire history of the project, at every moment, every buffer flush, forever. A perfect time machine - you could go back to anything your disk looked like from the moment you started tracking the project.

The issue here is that this isn’t very useful in practice. Nobody really wants to do this or gains much value from it. We had this fully built out and none of us really found it valuable, certainly not on a daily basis. It was *interesting* but not valuable.

The other issue is that it’s very much *not* realtime from a syncing / backup perspective. The other usage of this data was the review / sharing aspect on the server side, but the snapshots only happen once every 15-20 minutes on average, so the server never really has the data we want it to have.

However, there is obviously some value to having a history, being able to recover old states or undo things. There is also clearly a need to synchronize your data to a server in a timely manner if we’re going to offer cloud services.

So the question is, how do we record app state in a recoverable way and in a format that can be incrementally transferred to a server?

This is an incredibly interesting question (to nerds). Not of a lot of systems do this - incrementally back up filesystem state in a way where you can go back to any important point in history. Sturdy did something like this, Apple’s Time Machine is interesting to think about.

But for the sake of argument, let’s think about this from the point of view of a set of algorithms that is fairly well known and not massively different from what we’re trying to do, which is video compression.

Videos are not dissimilar from what we’re trying to do. A series of frames of state that generally don’t change a lot from frame to frame, but each frame difference is important. Sometimes it needs to be streamed, sometimes available locally, sometimes rewound or fast forwarded. Sometimes you want to download the whole thing for fast access, sometimes you want to seek into data on a server and just get one frame.

You could store every frame in full, uncompressed. You could store the difference between one frame and the next (or previous). You could store differences within a frame to further compress the data.

If you seek into a streamed video, there are interesting issues. If you’re only storing the difference between each frame and you seek hours into a video, all you have are differences. Then it becomes incredibly expensive, because you need to start with the first frame of the movie and calculate all the differences for hundreds of thousands of frames and applying all of those deltas until you get to your specific frame. 

So, movie formats tend not to do this, instead having keyframes and delta frames. 

However, what is your keyframe interval? If it’s 10 seconds and you’re asking for a frame 9 seconds after the keyframe, there are ~260 deltas that need to be applied in order to get the full frame you’re looking for. If your interval is 2 seconds, there are only ~30 deltas that need to be processed, but more data that needs to be stored.

So, we can think of ourselves as a streaming show. We’re producing frames, we want to get those frames to our server with minimal buffering time, minimal data storage requirements, minimal over-the-wire data size, lossless assurances, and a clear history where we can recreate almost any of our frames locally, clearly and fast.

One thing that does make this slightly more complicated is that we should also consider that there could be multiple clients streaming and we need to be able to synchronize them to some degree. For example, if you work on one computer and then switch to another, it would be nice to pick up where you left off, add to the stream and then switch back.

OK, so, given that frame of mind, what are our main algorithmic options? At the end of the day, we just need to be able to recreate frames (working directory contents and metadata like virtual branch state), so what are the data format options that are best for storing, streaming, multiplexing and recreating those frames?


# Event Log vs Git

A year ago, Nikita suggested something of an operations/event log format instead of using Git. It wasn’t very clearly defined, but my understanding of it would be something of an append only write-ahead log style format.

Here is the [previous Slack discussion](https://gitbutlerworkspace.slack.com/archives/C04M7T1MWN9/p1680773293707539), which sort of led to us [splitting the gb git repo out into a second repository](https://linear.app/gitbutler/issue/GB-185/move-gitbutler-data-to-second-repository) in the first place. Actually, I highly encourage everyone interested in this topic to re-read that conversation because it contains nearly all the arguments between Git vs event log style data storage, compression and transfer.


## Event Log Stuff

The main arguments for an event log format where state could be recreated by aggregating the logs, is that it could be relatively easily streamed. The overhead of sending individual log entries is fairly low, simple and fast.

However, there are also issues. How do you send keyframes, in other words, if you need the entire state of a larger working directory, what does that look like? A tarball? What about larger binary files? Just send them over and over? Delta stuff?

What happens if some of the event logs make it and some fail? This would be like missing interpolated frame data, which would corrupt the application of everything after it. You need all of them or the later ones are no longer applicable until you get another keyframe.

We also need to come up with a network protocol (even if it’s as simple as POSTing to an endpoint), authentication method, storage format, way to see what data is missing, methodology for recreating frames quickly (without using too much storage space), etc.


## Git Data

The other option is similar to what we’re doing now, which is keeping snapshots in Git and using the git protocols to push updates.

What we’re doing now is a rather complicated way to keep CRDT information and do snapshots relatively infrequently so we can interpolate to any point in time in between snapshots. However, this means that we can’t sync this data until a commit is done, so it could be an hour between data.

What we could do is commit much more frequently and attempt a push after every commit. If we identify “operations” as interesting points in time that you might want to rewind to and commit to an operations branch after every interesting point, then you could easily go back to any of those times.

This becomes very similar to what `jj` stores as it’s `jj op log` data:


    ❯ jj op log
    @  9149990fc71e scottchacon@Scotts-MBP-2.fritz.box 2 weeks ago, lasted 2 milliseconds
    │  new empty commit
    │  args: jj new
    ◉  cd0d579bf63f scottchacon@Scotts-MBP-2.fritz.box 2 weeks ago, lasted 3 milliseconds
    │  snapshot working copy
    │  args: jj new
    ◉  2bb4a5bc9586 scottchacon@Scotts-MBP-2.fritz.box 2 weeks ago, lasted 6 milliseconds
    │  new empty commit
    │  args: jj new p n
    ◉  f5b32c93071f scottchacon@Scotts-MBP-2.fritz.box 2 weeks ago, lasted 4 milliseconds
    │  snapshot working copy
    │  args: jj log
    ◉  68bc8de789f3 scottchacon@Scotts-MBP-2.fritz.box 2 weeks ago, lasted 4 milliseconds
    │  new empty commit
    │  args: jj new q
    ◉  b5f739b43683 scottchacon@Scotts-MBP-2.fritz.box 2 weeks ago, lasted 2 milliseconds
    │  create branch new-branch pointing to commit b164e249c9d0ae68a669413ea1e7bb7
    │  args: jj branch create new-branch
    ◉  355bb84940f8 scottchacon@Scotts-MBP-2.fritz.box 2 weeks ago, lasted 2 milliseconds
    │  restore to operation feaa44c65370ed97109778c7b54c51c4ff4b0
    │  args: jj op restore feaa44c65370
    ◉  0db067f8b7b0 scottchacon@Scotts-MBP-2.fritz.box 2 weeks ago, lasted 2 milliseconds
    │  undo operation 4a820c185b4f26ba302f240e72f0
    │  args: jj undo

You’ll also note that `jj` stores “snapshot working copy” changes, so it records occasionally the state of the working directory even if no operations have been done.


## Git over the Wire

Now, the downsides are that in theory, we’re writing a good amount of data. Each of these commits stores a tree that is the entire working directory plus metadata (virtual branch state, etc). 

However, in practice, this is stored and pushed over the wire as heavily delta-compressed data.

I was curious what the actual overhead is, so I did a little test. I committed something similar to what we might want to store - a tree of files (~650 files), a `virtual_branches.toml` file (~100kb) and a 3Mb binary data file (thinking that we might have some sort of binary hunk supplementary data or something).

Assuming that we’re committing this as a virtual tree into a Git branch, with one commit per operation, what does it look like to push each change to a git repository over HTTPS? (We could also do SSH, but HTTPS is both worst case and easier to setup and easier to authenticate. Also in this case, easier to inspect).

There are three calls that occur in order to push data to a Git server over HTTP. 

The first is a GET to `/info/refs`, which does two things. One is that it helps Git determine if it’s talking to a [dumb server](https://git-scm.com/book/en/v2/Git-Internals-Transfer-Protocols#_the_dumb_protocol) or a [smart server](https://git-scm.com/book/en/v2/Git-Internals-Transfer-Protocols#_the_smart_protocol). The other is it gets a list of references on the server so the client can calculate what the server already has and what the difference is that it needs to send over. This is a relatively fast call - on my [Ruby server handler](https://github.com/schacon/grack) (which is the slowest possible implementation of this handler) it completed in 71ms.


![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711458672354_CleanShot+2024-03-26+at+14.10.282x.png)

![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711458827177_CleanShot+2024-03-26+at+14.13.382x.png)


You can see the response from that is fairly simple. A list of where it’s refs are (in this case, `/refs/heads/main` is at `f088…`, and an advertisement of capabilities (`report-status report-status-v2 delete-refs side-band-64k…`). 

Now the client can calculate the difference (in this case, our 3M binary file and our full virtual branch YAML file), create a packfile and POST it:

![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711458994119_CleanShot+2024-03-26+at+14.16.262x.png)


The total POST body was 2.5M, which is pretty good, considering that part of that upload was a previously unseen (by the server) 3M binary file. The whole push took 1.7s. 

This is essentially the commit that I pushed:


![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711459138193_CleanShot+2024-03-26+at+14.18.492x.png)


Finally, there is a `post-receive` POST that tells the server that the reference has been updated and verify that they’re in sync. This call took 16ms.

![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711459726178_CleanShot+2024-03-26+at+14.27.242x.png)


*(For some reason, this call returns JSON, when the other two calls return essentially a binary protocol format, 🤷 )*


## A partial update

OK, not too bad. But now let’s say we change the binary file by adding a few bytes to the end and change one 40 char SHA in the middle of this 1k line (114kb) toml file. What does pushing that update look like?

Let’s commit that. This should create two new blobs for those two new file versions, one new tree (since they’re both in the same directory) and one new commit that references that new tree. Now we push that up. 

First, let’s see how much we added to our object database, raw-data wise. (A little confusing, but my commit message was literally “57 bytes?” because that’s roughly how much data I added).


    ❯ git cat-file -p HEAD
    tree a663f52a0e57808b16ccb792ceb5d2f2e02e7385
    parent 111d785c794503461531028bb18e172b43ed0d59
    author Scott Chacon <schacon@gmail.com> 1711457928 +0100
    committer Scott Chacon <schacon@gmail.com> 1711457928 +0100
    
    57 bytes?
    
    ❯ git cat-file -p HEAD | wc -c
         222
    
    ❯ git cat-file -p HEAD^{tree}
    100644 blob 11d3c818b0ac9ce41fe3e55ae2e7b215ca0f5ee3        binary.data
    040000 tree f06e47a2fffaa1f975fefaf8677a20656bf64948        gitbutler-app
    100644 blob e3565aebc5e49a7d9d5d2f38b9b9bf6b5a0788da        virtual_branches.toml
    
    ❯ git cat-file -p HEAD^{tree} | wc -c
         207
    
    ❯ git cat-file -p HEAD^{tree}:binary.data | wc -c
     3230986
    
    ❯ git cat-file -p HEAD^{tree}:virtual_branches.toml | wc -c
      116668

So the new commit object is 222 bytes. The new tree object is 207 bytes. The new, slightly modified binary file is a brand new 3.1M and the new virtual branch toml file is 116k (the files are stored as new content blobs with new shas).

Now, we can see in our local objects directory that Git has gzipped them, but the new objects are still not tiny (2.4M instead of 3.1, and 46k instead of 116k respectively). This will eventually be delta-fied into a packfile locally when we run `git gc` at some point, but for now, every small change to any file saves a new gzipped version on disk.


    ❯ ls -lah .git/objects/11/d3c818b0ac9ce41fe3e55ae2e7b215ca0f5ee3 
    -r--r--r--  1 scottchacon  staff   2.4M Mar 26 13:58 .git/objects/11/d3c818b0ac9ce41fe3e55ae2e7b215ca0f5ee3
    
    ❯ ls -lah .git/objects/e3/565aebc5e49a7d9d5d2f38b9b9bf6b5a0788da 
    -r--r--r--  1 scottchacon  staff    46K Mar 26 13:58 .git/objects/e3/565aebc5e49a7d9d5d2f38b9b9bf6b5a0788da

So what happens when we push them to the server?

There are again the three calls. The GET for `info/refs` that takes 65ms. The packfile (data) POST that takes 466ms and the `post_receive` POST that takes 20ms.


![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711459904130_CleanShot+2024-03-26+at+14.31.352x.png)


What is very interesting here is that the data push of the new 3M file and new 100k file was done in a 681 byte POST body (including headers and everything) and took less than half a second.


![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711460101569_CleanShot+2024-03-26+at+14.34.542x.png)


And this works with multiple commits too. If I add a little bit to my toml file and binary file several times and push three commits at one time, I get a similar result - 1515 bytes pushed in 800ms, from which I can recreate any of 3 slightly different versions of a 3Mb binary file or a 100k toml file.


![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711460470715_CleanShot+2024-03-26+at+14.40.572x.png)

## So what?

So I guess what I’m trying to determine is if there is a good point coming up with a rather more custom and complicated (and thus, error prone) event log format when Git calculates the interpolated frame data for us rather well already. It essentially creates on the fly a pretty good dynamic deltified difference and checksummed transfer format.

Certainly, if we know what our binary format is, we can probably do slightly better than generic xdelta for binary delta compression, but at what cost in terms of time and effort? 

If we’re creating an operations log entry every few minutes and pushing it whenever we create new data (perhaps with some sort of backoff so it’s at most once every X minutes so as not to DDoS ourselves), we’re talking about a bunch of very small pushes, generally a few hundred bytes per operation saved - probably not massively worse than whatever a custom logging format streamed over an HTTP based API would look like.


## My Suggestions

I’m open to counterarguments on the event log concept, but if we decide to stick with storing our operational state in Git as one commit per interesting operation, like `jj` does, I would suggest that we change a few things from our current approach:

**Suggestion One**
One, instead of “sessions”, we store “operations”. Any change to virtual branch data (renames, file dragging, commits, undos, pushes, file discards, apply/unapplys). Any perceived filesystem changes *when the app is focused,* rather than constant CRDTs*.* The commit message would be a description of the operation, such that we could just show a list of them and essentially get a free and simple undo log.

The tree that we record in the commit would look something like:


    ❯ git cat-file -p HEAD^{tree}
    040000 tree f06e47a2fffaa1f975fefaf8677a20656bf64948        working-directory
    100644 blob d544c909d06a8673fc62a25734599b0cac7ed75c        hunk-data
    040000 tree 550275bbecab9db17cf476af30a11420722ed84d        conflicts
    040000 tree 11d3c818b0ac9ce41fe3e55ae2e7b215ca0f5ee3        resolutions
    100644 blob e3565aebc5e49a7d9d5d2f38b9b9bf6b5a0788da        virtual_branches.toml

Or something to that effect. The virtual branch data as a toml file, the working directory as a tree snapshot, then any supplementary data that we can’t or don’t want to recalculate (hunk data) or would like to share (conflict resolution data). 

This is cool too, because with this, an “undo” implementation becomes trivially easy. We show a list of the operations as a simple revwalk through the log. User clicks on something in the history and says “revert to this” and we just checkout the `working-directory` tree into the working directory and dump the `virtual_branches.toml` file into the `.git/gitbutler/virtual_branches.toml` file, add this as a new op at the top of the log, reload the UI and *boom*, we’re reverted.

So, real back of the napkin math here, just to get an idea of size. We could have hundreds of operations per day, let’s say 200 per day. Let’s say the average delta size is 1kb, which is probably twice as high as it actually would be. Let’s say the user works 250 days a year. We’re talking about something on the order of 50Mb/year, probably worst case, on top of actual project data, for full operational backup. I don’t think that’s ridiculous, plus we can always do partial clones (shallow or blobless) should the history get very large. Or we could do some automatic graftable rewrite to shallow it out every year. Lots of ways around this if it becomes a problem.

**Suggestion Two**
Two, we keep all the git data in the main project again, not a secondary shadow repo.

There are two main issues in this regard that we need to make sure we address. One is that we need to have Git reference the operations log head so that the objects within it are not `gc`’d. We also need to do this in a way that it doesn’t mess with `git log` `--``all` if at all possible. The other is that if a user has two clients, there is a way to see and reference data from the other client.

For the first issue, I have one rather stupid idea. The reflog is checked by `gc` for object reachability. However, we can’t just put a fake branch there, because if the branch that the log is referencing has been deleted, it looks like gc cleans up the reflog too. 

However, if we make up a fake branch (`refs/heads/gitbutler/target` or something) and point it to whatever `origin/master` is currently pointed at (or maybe we keep it pointing to the SHA that our base branch is, which might actually be useful), but in the reflog we pretend that *originally* it was pointed at our op log branch and we just keep overwriting that file, Git will both see it as reachable and will not list it in `git log` `--``all` or any of the other tooling.

The format of the reflog is something like this (but with full 40-char shas):


    ❯ cat .git/logs/refs/heads/gitbutler/integration 
    00000000 7a1c9eeb3 Scott Chacon <schacon@gmail.com> 1710174928 +0100  update target
    7a1c9eeb 9a21ca95e GitButler <gb@gitbutler.com> 1710174933 +0100  commit: GB Commit

So if we keep overwriting the first line with `00000` and `[current op log head sha]` and then the second line with `\[op log head sha\] [base branch sha]` then this could do what we’re looking for.

The second issue was how to fetch and retain op log data from multiple clients, which could be done in this same manner. We could essentially keep our own refs under `.git/gitbutler/refs` that have a ref per client and keep one line in the fake reflog per head. 

Or, alternately, and probably better, we could handle this by having the client only do fast-forward pushes and if there is something on the server, we fetch, do a dumb merge (no content merging, just create a new commit with the parents being the local head and the remote head) and push. 

For example, if we have some commits on client A and client B. 

![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711522474624_CleanShot+2024-03-27+at+07.54.282x.png)


Client B pushes to the server. Then client A syncs.

![](https://paper-attachments.dropboxusercontent.com/s_A9E38CA81BF3E78571EF7C0A4EE4FE644D10EA900AE3EAF7B1ECC464B2D85C47_1711522609063_CleanShot+2024-03-27+at+07.56.432x.png)


Now we create a dumb commit and push it up. It’s actually interesting, because that is also an operation, so it makes sense for it to be in the log. Then when client B pushes, it sees there is a new head and it merges (or fast-forwards), etc.

It should work fine and it should also be nice to just see the list of operations come into our undo log from other connected clients automatically.

