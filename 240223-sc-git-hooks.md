# Git Hooks

There are a number of lifecycle hooks that Git supports and some developers rely on. I thought it might be valuable to go through each of them, what they're for generally, if we should execute them and what to do with the information from them.

## Server Concept

Before I get into those though, I would like to propose something interesting that we could do. A lot of organizations have hooks that they want their employees to use, but it's a pain because the hooks are not in Git transfer protocols, so they have to be managed separately. It could be interesting to have a way for an organization to have managed hooks where all the GitButler users in the org have hooks automatically updated.

It could be interesting to have the client be able to manage it's hooks this way. Note changes in the hooks, version control them, push them to our servers (possibly just in the sessions data).

Even in single player mode, this could be good for having your hooks backed up and transferred between clients. In team mode, this could be pretty cool for a team all using the same hooks.

The other possibility is to just look for hooks in a `.gitbutler/hooks` folder in a repository. This actually might be a better idea. Then the hooks can go through PR review, etc, and can be merged in and distributed automatically and per-repo. We could look there for hooks first and then in the standard `.git/hooks` directory next. 

Actually, looking into this more, this is what [Husky](https://typicode.github.io/husky/) does. It might be cool to also look in the `.husky` directory and automatically run them if we don't see a .gitbutler directory. Here is an [example](https://github.com/tauri-apps/tauri/blob/dev/.husky/pre-commit) from the Tauri project itself.

## Hooks Path

Quick note, there is a newish `core.hooksPath` git config that we should respect when looking for local hook files.

## The Hooks

OK, so what are the hooks?

# The Commit Hooks

## pre-commit

This runs when you start the commit process, before the user enters a commit message. It takes no arguments. If it exits non-zero, the commit cannot proceed. People use this to check for formatting errors and things like that.

This should be a pretty simple one. If we see the file, we run it. If it exits non-zero, we show the user a modal or something with the output and the path to the script we ran.

## prepare-commit-message

This one runs to modify a commit message before it's edited, for example to see if a branch is formatted as an issue and then link that issue in the commit message template.

I'm not sure if we can really do this, since we don't have a two phase commit anymore. We can't really run in constantly, or on commit message textbox focus. I suppose we could have it in the "Generate Message" dropdown as an option ("Run prepare commit message") only if we see the executable file there and have it run when you generate a message. Not sure. Or just have it as an option in the Generate Message thing that overrides the AI fetch, so someone could replace our builtin AI generation with their own. That might be cool.

As arguments, it takes (1) a filepath to a file that contains the default commit message, (2) the type of commit ('message', 'template', 'merge', 'squash', or 'commit'), and (3) the sha of the commit if you're amending a commit. The 2 and 3 arguments are optional and we probably don't need them unless we want to run this on commit squashing and upstream merging, but we could do that too.

If this exits non-zero, we abort the commit and show the output to the user.

## commit-msg

This one runs after the message is edited by the user, before the commit is created. It can either edit the message automatically (to conform to some standards) or exit non-zero to abort the commit.

It takes one argument, which is the path to the file that contains the commit message.

This should be easy to do and potentially useful. We'll just need to not commit and show the output if it exits non-zero.

## post-commit

This one runs after a commit has been written to the odb and the branch updated. Exiting non-zero does nothing. It takes no arguments. Mostly for notifications and stuff.

# Other Local Hooks

There are other hooks too.

## post-checkout

This is run when you checkout a branch. We could probably run this when we apply a new branch, but the arguments will be a little strange. It takes (1) The ref of the previous HEAD, (2) the ref of the new HEAD and (3) a flag indicating if it was a branch checkout or a file checkout. The flag will be 1 and 0, respectively. Since you can't checkout an individual file with our tool, the flag will always be 1 and the head references I suppose would just be the virtual branch name (maybe the `refs/gitbutler` reference would be best, but I'm not sure exactly what to do about the "old" HEAD)

We can do this, but I'm not sure how useful it will be for people exactly, since it's not a branch switch the way Git does it. It's not a full context change.

## pre-rebase

This command runs before `rebase` does anything as a pre-check. If it exits non-zero, the rebase doesn't happen. It's normally used to make sure the rebase doesn't do something stupid. It takes two arguments, the branch being rebased onto and the branch being rebased. If the rebase is on itself, there is no second argument.

We do run rebases, so I suppose we could fire this off when we try to do our rebase series. If it exits non-zero, just treat it like one of the rebase stages failed (which would probably fall back to a merge). If it does this, we should show the user the error message output.

## pre-push

This is called before attempting to push to a remote. We should run this before pushing to upstream (ie GitHub) but not GitButler. If it exits non-zero, we abort the push and show the user the error.

It takes two arguments (1) name of the remote, (2) url of the remote and reads references being pushed from stdin. 

This should be pretty simple, because most (all?) of the time we're just pushing one reference, so we would execute this with input of the type:

```
<local ref> SP <local object name> SP <remote ref> SP <remote object name> LF
```

So an example might be:

```
refs/heads/master fef2f97b20b8610a31f9ca8e3dc50bd01f616205 refs/heads/foreign a953c8829ba01e0eaac0d2a2c6ed7286c0a46496
```

If the branch is being updated, the first sha will be what the remote had and the second is what we're pushing. If the branch is new, the first sha is just 40 0s.

## post-rewrite

The `post-rewrite` hook is run by commands that replace commits, such as `git commit --amend` and `git rebase`, so we could call it after our amending and squashing features.

It's single argument is which command triggered the rewrite (either amend or rebase), and it receives a list of rewrites on stdin of this format:

```
<old-object-name> SP <new-object-name> [ SP <extra-info> ] LF
```

It's exit status is ignored.

## post-merge

This is executed after a merge commit happens. It takes no arguments and it doesn't matter what the exit status is. It's normally used to setup permissions or copying in external files, things like that. Should be simple to just execute it after merging the target branch or applying a stashed/remote branch.

## reference-transaction

This is a very new hook that gets run after any reference is updated to a new SHA. It is passed via stdin a line that looks like this:

```
<old-value> SP <new-value> SP <ref-name> LF
```

The exit status is generally ignored. We could pretty easily try to run this after any ref update.

# Possible Other Hooks

We also could add our own hooks if we can think of some useful things.

* post-virtual-branch-apply
* post-virtual-branch-unapply
* post-target-branch-update

Open to suggestions. :)
