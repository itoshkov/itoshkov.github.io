# Overview

> A conceptual model is an explanation, usually highly simplified, of how
> something works. It doesn’t have to be complete or even accurate as long as it
> is useful.
>
> Don Norman - The Design of Everyday Things

In this tutorial I'll try to describe how git works, without using git. Instead,
we'll create a simple, git-like system using just zip files `diff` and `patch`.

The idea is to build a good model of how git works _conceptually_.

You can [go here](https://youtu.be/dQw4w9WgXcQ) if you prefer to watch this
tutorial instead of reading it.

I'll be using some simple Linux commands like `mkdir`, `cd`, `ls`, `cp` and of
course `zip`.

We'll be developing our system by solving the problems we face one at a time.
There are many solutions to each of them as you can see by the plethora of
Version Control Systems (VCS) developed since
[1972](https://en.wikipedia.org/wiki/Version_control#History). Our own choices
will be informed by git's choices, because we are trying to build a conceptual
model of how git works. For example it tracks changes between snapshots - the
current state of the project as a whole, whereas some other VCSes track changes
of individual files.

## Why Do We Need Version Control Systems?

Software development is hard. Developing anything but the most trivial programs
takes a lot of time and effort. Often we need to change things to:
* fix bugs
* implement new features
* accommodate new requirements
* make architectural changes as the program grows
* etc.

It is not uncommon to need to maintain several versions of our software
simultaneously. This is especially true for applications that are installed on
premise, like desktop and mobile apps, but even if we are developing an internal
web application we need to be able publish bug-fixes and develop new features at
the same time.

Moreover, many projects are developed by teams of people. Collaborating without
_accidentally_ deleting other people's code is essential.

Accomplishing these tasks gets increasingly harder, with the size of the project
and the team. It is very easy to mess things up if you don't have a good system
to help you.

## Why Git?

*TODO*

# Version Control Without Git

## The Humble Beginning

Let's start by creating our new project:

```bash
mkdir ProjectX
cd ProjectX
```

And let's put our top-secret program in `main.c`:

```c
#include <stdio.h>

void main() {
	printf("ALL YOUR BASE ARE BELONG TO US.");
}
```

This is our starting point. This is our first version and we want to save it and
be able to come back to it. One way we can do that is to copy the whole
`ProjectX` directory, like this:

```bash
cd ..
cp --recusive ProjectX ProjectX-v0
cd ProjectX
```

Nice! Well, sort of.

When we run our program we see that we forgot to print new line at the end.
Let's fix this.

```c
#include <stdio.h>

void main() {
	printf("ALL YOUR BASE ARE BELONG TO US.\n");
}
```

We can see the differences between this version and `v0` like this:

```bash
diff --recusive --unified ../ProjectX-v0 .
```

The `--recusive` option tells `diff` go compare the directories recursively, and
the `--unified` -- to show the differences in the so-called unified format.

```diff
--- ../ProjectX-v0/main.c	2021-09-06 13:58:33.718024787 +0300
+++ ./main.c	2021-09-06 13:58:46.690096675 +0300
@@ -1,5 +1,5 @@
 #include <stdio.h>

 void main() {
-	printf("ALL YOUR BASE ARE BELONG TO US.");
+	printf("ALL YOUR BASE ARE BELONG TO US.\n");
 }
```

As you can see, the file `main.c` has one line removed and one line added.

OK, it's time to save this new version of our software:

```bash
cd ..
cp --recusive ProjectX ProjectX-v1
cd ProjectX
```

## Archives

One of the problems with copying folders like that is, that they tend to take a
lot of space. Also, it's easy to accidentally change things in the "archive"
folder, instead of in the working one.

We can fix these by using zip files instead of folders. Zip files still can be
changed, but at least it's harder to do so by mistake.

Let's also start calling these archives commits. For now a commit is just a
snapshot of the current sources, stored in a zip file.

Here's how to "convert" the old directories to the new format:

```bash
cd ..
cd ProjectX-v0
zip -r ../ProjectX-v0.zip .
cd ..
cd ProjectX-v1
zip -r ../ProjectX-v1.zip .
cd ..
rm -rf ProjectX-v0 ProjectX-v1 # remove the old directories
```

One thing we can't do directly anymore is to compare versions. But we can just
unzip the version(s) that we need in some temporary folders and then compare.
It's a bit of a hassle, but we can automate it with a small script if we want
to.

Another thing I don't like is, that the version archives are just lying around
and littering the parent directory. Let's move them in a new folder inside the
project:

```bash
cd ProjectX
mkdir -p .repo/commits
mv ../ProjectX-v0.zip repo/commits/c0.zip
mv ../ProjectX-v1.zip repo/commits/c1.zip
```

In Linux, UNIX and macOS files and folders starting with dot `.` are "hidden".
If you type `ls` you won't see them, but if you type `ls -a` you will. Besides
that, they are just normal files and folders.

We created a `commits` folder inside, because we'll proably want to put other
things in the `.repo` in the future.

To see how our current system works, let's modify the `main.c` file again and
make a new version:

```c
#include <stdio.h>

void main() {
	printf("CATS: ALL YOUR BASE ARE BELONG TO US.\n");
}
```

```bash
zip -r .repo/commits/c2.zip .
```

```
  adding: main.c (stored 0%)
  adding: .repo/ (stored 0%)
  adding: .repo/commits/ (stored 0%)
  adding: .repo/commits/c1.zip (stored 0%)
  adding: .repo/commits/c0.zip (stored 0%)
  adding: main (deflated 85%)
  adding: main.o (deflated 67%)
```

Oops! We added the previous commits in the zip file too. Notice that we are also
storing the `main.o` and `main` files. These are not needed as they are
generated by the compiler.

To fix this, let's create a list of files that we want to track. We'll put it in
another hidden folder, called `.commit`.

```bash
mkdir .commit
```

Put the following in a `.commit/track` file:

```
.commit/*
main.c
```

```bash
rm .repo/commits/c2.zip
zip -r -i@.commit/track .repo/commits/c2.zip .
```

The `-i@.commit/track` option tells `zip` to include only the files mentioned in
the `.commit/track` file.

## Branches

So far our versions are named sequentially `c0`, `c1`, `c2` and so on. And this
order is the only thing that tells us that `c2` was created from `c1`, and that
`c1` was created from `c0`. That's good enough for linear development, but is
very insufficient for branching development.

But what are branches and why do we need them? Suppose that we have released
`c2` in the world, and are working on new features. We have created new commits
`c3` and `c4`, but overall the feature is not ready yet.

While we are working on it, we receive complaints about some major problem in
`c2`. And our customers can't wait for us to finish with our new feature. They
need a fixed `c2` now!

Luckily, we do have our `c2` source code. We can "switch" to it, fix the bug,
and release the good version. But where should we put the fix itself?

We might be tempted to temporary save the changes somewhere, switch back to our
`c4` version, then apply the fix and make a `c5` commit. After all, we do want
this fix to be part of the next version that we are going to release, right?

But what if there's yet another problem with the "fixed" `c2`? We wouldn't be
able to go back to exactly this code, as we didn't save it anywhere. No, we need
a better solution.

We can modify the commits, so they "know" which their parrent commit is. Let's
add this in another file, called `.commit/info`. For `c2` it will look like
this:

```
parent: c1
```

Since it is in the `.commit` folder, it will be tracked automatically.

The first commit is a bit special as it doesn't have a parent. Its
`.commit/info` will look like this:

```
parent:
```

We can also add more information to this file like:
* author of the commit
* date and time of the commit
* one line summary
* bigger, multi-line description of the changes

But we won't do that in this tutorial, to keep things simple.

We went back and fixed all our commits to have this file. Now we can switch back
to `c2`, implement the fix and create a new commit, let's say `c5`. The commit
history will now look like this:

```
c0 -- c1 -- c2 -- c3 -- c4
             \
              c5
```

Good! Now we can go back to `c5` when we need to.

Like when we needed to properly fix the broken fix from before. Let's assume we
added one more commit to `c4`, we then went back to `c5` and implemented the new
fix on top of it. The picture now will look like this:

```
c0 -- c1 -- c2 -- c3 -- c4 -- c6
             \
              c5 -- c7
```

## Names

So far so good, but we have just 2 branches and it's getting a bit tedious to
remember the top commit for each of them. Let's name them!

We can have a file for each branch, containing the name of the most recent
commit of this branch. For the `main` branch this will be `c6` and for
`release-1` -- `c7`:

```bash
mkdir .repo/branches
echo 'c6' > .repo/branches/main
echo 'c7' > .repo/branches/release-1
```

The `echo 'c6' > .repo/branches/main` command will just create a new file called
`.repo/branches/main` and make its content be `c6`. If the file already exists,
it will overwrite it.

Let's create another commit in our `main` branch to see how it works:

```bash
# Set the parent info:
echo 'parent: c6' > .commit/info

# Make the commit:
zip -r -i@.commit/track .repo/commits/c8.zip .

# Update the branch pointer:
echo 'c8' > .repo/branches/main
```

This is tedious and errorprone. I hope somebody will create a program to
automate it ...

## Switching

Above I was saying things like "let's switch to `c2`", or "let's switch to
`main`", but I never explained how. Here is how:

```bash
# Remove all the files and subfolders except for `.repo`:
rm -rf * .commit

# Unzip the relevant commit
unzip .repo/commits/c2.zip
```

It's not that bad, if we ignore the ugly way we clean up the working folder,
right?

But what if we now go to lunch, and when we come back next day we forget which
commit we switched to? Yes, sometimes lunches are that long. We can then write
some code and then commit:

```bash
# Set the parent info:
echo 'parent: c8' > .repo

# Make the commit:
zip -r -i@.track .repo/commits/c9.zip .

# Update the branch pointer:
echo 'c9' > .repo/branches/main
```

Oops! We committed in the wrong branch! The commit was based off of `c2` but we
forgot that and treated it as if it was based on `c8` instead.

It's time to add another level of indirection. We can create a file
`.repo/HEAD`, which will contain the name of the current branch or commit.

Switching to a branch will look like this:

```bash
# Remove all the files and subfolders except for `.repo`:
rm -rf * .track

cat .repo/branches/main
# This will tell us which commit the main brach points to

# Unzip the relevant commit
unzip .repo/commits/c8.zip

# Update HEAD to point to  a branch
echo 'branches/main` > .repo/HEAD
```

Switching to a commit -- like this:

```bash
# Remove all the files and subfolders except for `.repo`:
rm -rf * .track

# Unzip the relevant commit
unzip .repo/commits/c2.zip

# Update HEAD to point to specific commit
echo 'commits/c2` > .repo/HEAD
```

You might be wondering why would you want to switch to a specific commit instead
to a branch? A common case is if you want to go back a couple of commits to see
if a bug was present there or not.

In git, this situation when the HEAD points to a specific commit instead of a
branch, is known as a "detached HEAD state". I mention it here, because it
sounds scary (and a bit gruesome), but it's really nothing to worry about.

Let's switch back to `main` as shown above and make a commit there:

```bash
cat .repo/HEAD
# --> branches/main (ok, we're on the main branch)

cat .branches/main
# --> c8 (the top commit of main is c8)

# Set the parent info:
echo 'parent: c8' > .commit/info

# Make the commit:
zip -r -i@.commit/track .repo/commits/c9.zip .

# Update the branch pointer:
echo 'c9' > .repo/branches/main
```

Notice that, even though we made a new commit, we didn't have to change
`.repo/HEAD` as we are still on the same branch.

## Merging

Here is the current situation:

```
                                 HEAD -> main
                                          |
                                          v
c0 -- c1 -- c2 -- c3 -- c4 -- c6 -- c8 -- c9
             \
              c5 -- c7
                    ^
                    |
                release-1
```

We want to create a new release, but first we want to grab the fixes from
`release-1` and apply them to our `main` branch.

One way to do that, is the following:
1. Find the most recent commit, which is parent to both `c9` and `c7`. This is
   `c2`.
2. Find the changes between `c2` and `c7` (the `release-1` branch). We can unzip
   both the commits in 2 temporary folders and then `diff -r -u c2-temp c7-temp >
   changes.diff` them.
3. Patch the code in the working folder (`c9`): `patch < changes.diff`
4. Manually fix any conflicts
5. Commit

There's one more thing. This new commit should have two parents. So let's make
its `.commit/info` file read

```
parent: c9 c7
```

And the full picture now looks this way:

```
                                       HEAD -> main
                                                |
                                                v
c0 -- c1 -- c2 -- c3 -- c4 -- c6 -- c8 -- c9 -- c10
             \                                 /
              c5 -- c7 ------------------------
                    ^
                    |
                release-1
```

As you can see, we moved only `main`, but left `release-1` unchanged. That is
because we merged the latter into the former.

## Collaboration and remote repositories

*TODO*

# Git

I will now make what is known as a pro move, and direct you to the excellent
[Pro Git](https://git-scm.com/book/en/v2) book! It is free and easy to read, and
now, hopefully, even easier to understand!

It might look like a cop-out and it is. But also, there are already quite good
tutorials, which describe how to use git. My main objective here was to describe
how git works and to some extent to answer the question "why is it like that".
