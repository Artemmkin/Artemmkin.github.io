---
layout: post
title: >
  Master Git (part V). Change commits in your history. Interactive rebase.
tags: [git,vercont]
---
When you work with with version control, you sometimes find yourself in situations when you would like to edit or delete the saved changes in your project's history. Well, in this case, you should know that rewriting your project's history could be _dangerous_.

Unless you work on a project alone, you will have to share your changes to the project (commits) with other people. In most cases, you share commits via a central server like Bitbucket or Github other than directly with other developers. I'm sure you already know that when we make a new commit it's based on some other commit(s) which is considered to be the new commit's parent(s). So once you shared your commits, they will constitute the basis for the work of other people.  Rewriting or deleting commits that you already shared with others would mean a whole lot of trouble for those who based their work on those commits.
So unless you want your colleagues to hate you, follow this simple rule:
~~~yml
Never change commits which you already shared with others.
~~~

If you want to undo the changes in your project's history, just create a new commit that corrects or completely removes the changes introduced by previous commits. There is the [revert](http://artemstar.com/2017/03/27/git/) command which allows you to do just that.

If you understood that rewriting your public history is bad, then you're safe to keep reading :) Today I will show you how changing commits could be useful.
<!--break-->

I already talked about commands like [git commit --amend](http://artemstar.com:4000/2017/03/27/git) which helps you to fix up the last commit and [git reset](http://artemstar.com:4000/2017/03/27/git/) that allows you to restore to the previous commit removing all the commits following it.


Again, those are very useful, but should be used on local commits only.


Today we will talk about a more powerful command, that is **`git rebase -i`**, which gives you full control on how you want to reshape your project's history.

### Rebase

Remember what rebasing is? It simply allows you to change the base of your current branch. Basically this means taking the changes introduced by commits and creating new commits with the same changes but with a different base.


![200x200](/public/img/git/rebase3.png)

 `-i` option starts interactive mode which gives you full control on how this process goes.

### Case 1: Two different branches
So let's imagine we have the same situation as depicted on the picture


![200x200](/public/img/git/rebase01.png)

and from the feature branch let's run interactive rebase.
~~~yml
$ git checkout feature
$ git rebase -i master
~~~
This will open up a text editor and offer you different options on how to create new commits. In the comments, described all the things we can do.

![200x200](/public/img/git/rebaseint1.png)

As we can see, we can edit existing commits (`edit`) including their messages (`reword`), merge several commits into one (`squash`, `fixup`), run commands as we create new commits (`exec`), and remove commits (`drop`).

Let's imagine that my two commits added new functions which are closely related and work in couple. In this case, I may want to squash the second commit into the first and leave in project's history a single commit instead of two.  I will use the `squash` option which, unlike `fixup`, allows me to change the message for the resulting commit.


So I change `pick` to `s`(`squash`) for the second commit.


![200x200](/public/img/git/rebaseint2.png)

After saving and exiting, it will open up a text editor again, asking me to edit the commit message.
It shows all the messages for the commits. If we leave it as is, all of them will be used for the the new commit's message.

![200x200](/public/img/git/rebaseint3.png)

So I'll just edit the first one and delete the rest which in my case is just the second commit message:


![200x200](/public/img/git/rebaseint4.png)


In the end, we get one new commit which applies to a new base (`master`) the same changes as the two commits we just squashed.


![200x200](/public/img/git/rebase04.png)

### Case 2: the same branch

Interactive rebase can be useful not only when we rebase one branch onto another. It is often used to _rewrite the history of a branch_ - the thing we talked about in the beginning when we mentioned commands like [amend](http://artemstar.com:4000/2017/03/27/git/) and [reset](http://artemstar.com:4000/2017/03/27/git/).

Let's imagine I have a local branch with a series of commits which I haven't shared with anyone yet. And before I do share it with anyone, I would like to edit some of them, probably squash a few as I made a bunch of small commits which better look together, maybe correct some typos in the commit messages, etc.

![200x200](/public/img/git/rebaseint5.png)


First, I choose the range of commits which I want to change. I will change commits that follow the commit `bdc72fe` - this will be the base for all new commits which will be created - remember _rebasing creates new commits_ at the specified base.

~~~yml
$ git rebase -i bdc72fe
~~~
This will open up a text editor as we've seen before and list the commits coming after commit we specified which is `bdc72fe`.

![200x200](/public/img/git/rebaseint6.png)

My commits regarding the timeout served the same purpose of introducing a new parameter to the function and removing a temporary workaround. So I'll squash (`s`) all those commits in one. I also noticed that I made a typo in the word _controller_, so I'll reword that message (`r`). And the last commit introduced a new function which I decided to be unnecessary, so I'll drop that commit (`d`).  

![200x200](/public/img/git/rebaseint7.png)

By changing commit messages as described earlier, I achieve the state of the project's history I want:

![200x200](/public/img/git/rebaseint8.png)


### How to undo rebase O_o ?

Git allows you to restore history even when you rewrite it. I have already touched on using [reflog](http://artemstar.com/2017/03/28/git/) to restore changed history to the previous state in one of my posts.

So I'll simply show you how I restore my project's history to the state before we started rebasing.

We use command `git reflog` to find the commit to which HEAD pointed to before we started the rebase. It's usually the last commit that you've made.

![200x200](/public/img/git/rebaseint9.png)

the I'll use [hard reset](http://artemstar.com/2017/03/27/git/) in order to restore to the state when the last commit was added to my branch and before we started rebasing.
~~~yml
$ git reset --hard HEAD@{17}
~~~
![200x200](/public/img/git/rebaseint5.png)
