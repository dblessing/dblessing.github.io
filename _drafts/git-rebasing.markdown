---
layout: post
title:  "Git Rebasing: Essential for contributing"
date:   2015-08-17 18:00:00
categories: git git-rebase open-source
---

Years ago when I first started contributing to open-source projects git was new to me. I quickly
grasped committing files and pushing to a remote repository. Other than the initial uneasiness with
contributing code there was one thing that scared me. I was absolutely terrified of seeing the 
message on GitHub saying my code couldn't be merged. Project maintainers would review my code 
and then ask me to rebase so they could merge. Wiping away the sweat I would go to work rebasing. 
Many times I would fail and completely mess up my work. It was easier to copy/paste and redo the 
work than to figure out how to rebase properly and get a clean set of commits pushed. 

Over time I learned some tricks and I now rebase with ease. I'm sharing these tips in hopes that
this will take away another hurdle many new contributor face. After following these tips it will
probably still take some practice. Be patient and most of all don't be afraid to ask for help.
Project maintainers *should* be understanding and willing to point you in the right direction.

There are two types of rebasing I will cover. The first is just a standard rebase that will solve
the merge conflict issues. The second is an interactive rebase, which I will cover later.

## Rebasing to resolve merge conflicts

These tips assume you have created a 'feature' branch off of master for a project. Additionally,
`origin` refers to your fork while `upstream` refers to the original/upstream project. Both can (should)
be configured as remotes for your git repository. See [Configuring upstreams](#configuring-upstreams)
below for a tip on how to configure this, if you're not familiar with the concept.

First, ensure you have a clean state. That means you have either committed, stashed, or reverted any untracked
changes. When you run `git status` you should see `nothing to commit, working directory clean`.
Next, switch to the master branch and pull upstream changes by running `git pull upstream master`.
Switch back to your feature branch and complete the rebase by running `git rebase master --preserve-merges`.
Assuming all is well you will see `Successfully rebased and updated refs/heads/<branch>`. If you see
anything else you will need to resolve the conflicts, commit, and continue the rebase. 

Before examining how to resolve conflicts I want to revisit the rebase command for a moment. What does
`--preserve-merges` do? There are lots of details but one main reason I settled upon using
this flag is that a merge preserving rebase considers a smaller set of commits for replay. Over time
I had the most success (least nasty conflicts) by using this flag. This is certainly not required to
do a normal rebase but I recommend trying it. If you're interested in the details this SO article titled
[What exactly does Git's "rebase --preserve-merges" do (and why?)](http://stackoverflow.com/questions/15915430/what-exactly-does-gits-rebase-preserve-merges-do-and-why)
has a very in-depth explanation.

Now, to resolve any conflicts that occur during a rebase start by running `git status` to see what the 
conflicts are. You will see `error: could not apply f78d8ae... <commit message>`. Another `git status`
will reveal the conflicting files under `unmerged paths:`

```
$ git status
rebase in progress; onto 519c739
You are currently rebasing branch 'changes' on '519c739'.
  (fix conflicts and then run "git rebase --continue")
  (use "git rebase --skip" to skip this patch)
  (use "git rebase --abort" to check out the original branch)

Unmerged paths:
  (use "git reset HEAD <file>..." to unstage)
  (use "git add <file>..." to mark resolution)

        both modified:   conflicting-file

no changes added to commit (use "git add" and/or "git commit -a")
```

By examining the `conflicting-file` contents we see a set of weird characters:

```
$ cat conflicting-file 
foo
<<<<<<< HEAD
qux
quux
=======
bar
baz
>>>>>>> f78d8ae... Conflicting change
```

The changes between `<<<<<<< HEAD` and `=======` represent changes made in upstream master since your branch
was checked out. Changes between `=======` and `>>>>>>> f78d8ae... <commit message>` represent
changes in your branch. Git does not know which order these changes should go in. At this point I cannot
give you perfect advice on how to resolve the conflict. In some cases you will only want the old changes 
or the new changes but not both. Over time you will gain a better understanding for how to resolve this 
problem. In this case we want both changes but need to adjust ordering slightly. We remove the markers
(the weird <<<, >>> and ==== lines) and make the 'code' look like something we want:

```
foo
bar
baz
qux
quux
```

Commit this file as you normally would and then run `git rebase --continue`. This will continue replaying
commits until either another conflict is encountered or all commits have been replayed. Sometimes there
are many conflicts that must be resolved but hopefully the rebase is smooth. Once complete, you will need
to force push your feature branch to the `origin` (your fork) to update the merge request. Do this by 
adding a `-f` flag to your push - `git push origin <feature-branch> -f`. Always examine the merge request
after this operation to ensure the commits and changes are what you expect.

* * *

Unfortunately I don't have comments enabled on my blog yet. If you have comments, please send them to
[@drewblessing](https://twitter.com/drewblessing) on Twitter.
