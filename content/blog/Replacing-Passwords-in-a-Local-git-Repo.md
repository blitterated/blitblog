---
date: 2021-03-17T00:09:26Z
title: "Replacing Passwords in a Local Git Repo"
draft: true
image:
   hero: audio-vaporwave-fence.png
   thumb:
author: blitterated
tags: ["git", "bfg", "secrets"]
categories: ["Software"]
---
## Background

I'd created a simple python script to pull down a user's tweets and filter them by a hashtag. I kept playing around with it, generalizing it, and refactoring it. Naturally, I used a local git repository to track changes and prevent myself from losing work. But in the process, I'd committed the keys and secrets for my Twitter developer account into the repo. [BFG](https://rtyley.github.io/bfg-repo-cleaner/) to the rescue! I used BFG to replace the offending text of my keys and secrets throughout all the commits in the repository. It's not too straight forward, but not to bad either, kind of like 3.6 roentgen. This is a log of my journey down that particular rabbit hole.

## A couple of things to know up front

#### First, I needed a file showing BFG how to remap the secrets and keys in my credentials file

In this project I had some Twitter API credentials in a file called `twcred.py`. Its contents looked like this:

```python
CONSUMER_KEY = "dW5Cwh5uYQ8QTl6OiZhYrIg"
CONSUMER_SECRET = "Ew61yLgmmNY3EZHIrouDRAgnPaNwiDhSDUaVFT2sUtQYAzK8"
ACCESS_TOKEN_KEY = "BACZzHjsS82nAFCX49q8xj3dbsjvLDMecAsvx1YfE2xHNFKG"
ACCESS_TOKEN_SECRET = "aWZe9qyUZxCm2i6BJ4ygKFLdxKlZJ6WlxZw1rpEm3X"
```

These are, of course, [totally fake](#a-quick-and-dirty-fake-twitter-secret-generator)

I made a copy of `twcred.py` and then ran a quick `ex` command to reformat it into an acceptable expression-file for BFG

What is the expression-file? Here's a relevant exceprt from BFG's `--help`:

    -rt, --replace-text <expressions-file>
        filter content of files, replacing matched text. Match expressions should be listed in the file, one expression per line - by default, each expression is treated as a literal, but 'regex:' & 'glob:' prefixes are supported, with '==>' to specify a replacement string other than the default of '***REMOVED***'.

It's a good explanation for jogging your memory, but the author of BFG made a better post explaning it on [Stack Overflow](https://stackoverflow.com/a/15730571). It basically breaks down to this

|   Text in expression-file   |   Action   |
| ---------------------------- | --------------- |
| `PASSWORD` | Replace `PASSWORD` with `***REMOVED***` (default) |
| `PASSWORD==>examplePass` | Replace `PASSWORD` with `examplePass` |
| `PASSWORD==>`    | Replace `PASSWORD` with the empty string |

There's some additional options using regexes, but you can read up on those separately.

#### Second, BFG will rewrite your all your commits _except for the last one_

This is important to know if you don't want to repeat the process after the first time.

The reasoning for doing this is sound. The author of BFG doesn't want to screw up the commit that is potentially going into production.

When I completed the process, I was surprised to see that `git status` didn't see any changes between the last commit and my current credentials file which still had the original credentials in them. When I found out why, I had to:

* Make the same changes to credentials file that BFG had done for all previous commits.
* Commit those changes


* Redo the BFG replace-text process to get that one second-to-last commit changed.

In the following steps, I'll be sure that the changes are made to the credential's file and committed first. That way the changes are already in the last commit, and all previous commits will be rewritten.

## Here's the steps

#### Create a working directory

```bash
mkdir pytwit_cleanup
cd pytwit_cleanup/
```

#### Grab BFG jar from Maven repo (get the latest version)

You can [follow this link](https://rtyley.github.io/bfg-repo-cleaner/#download) to get the download link to the latest version. Just copy it from the link on that page.

```bash
curl -O https://repo1.maven.org/maven2/com/madgag/bfg/1.14.0/bfg-1.14.0.jar
```

#### I made a temporary alias for convenience

There was a lot of typing and retyping while trying things out. This helped save my median nerves.

```bash
alias bfg="java -jar bfg-1.14.0.jar"
bfg
```

#### Copy the file with your keys and secrets (or passwords, salts, what-have-you) from the original repo

```bash
cp pytwit/twcred.py ./twcred_replace.txt
```

#### Run a simple `ex` command to reformat the file for BFG

Okay, this really wasn't all that simple. But I had fun doing it, and I really wanted to dig into using `ex` instead of `sed` or `perl` for this. I've been more interested in `ex` ever since reading [Tom Ryder's great article](https://blog.sanctum.geek.nz/actually-using-ed/) on `ed`

```bash
ex -s -c '%s:\(\w\+\) = "\([-A-Za-z0-9]\+\)":\2==><YOUR_\1>:g|x' twcred_replace.txt
```

There's a little more info on using `ex` for search and replace [down below](#a-little-more-info-on-using-ex-for-search-and-replace)

#### Create a commit in the original repository that removes the secrets

I wish I'd paid closer attention to the website. It says right on the front page,

> Your current files are sacred. By default the BFG doesn't modify the contents of your latest commit on your master (or 'HEAD') branch, even though it will clean all the commits before it."

The "by default" statement is interesting. The only reference to editing the latest commit in the `--help` was for a switch related to taking care of blob files in the repo


    --no-blob-protection
        allow the BFG to modify even your *latest* commit.
        Not recommended: you should have already ensured your latest commit is clean.

I haven't tested this switch out. Even the author recommends cleaning the last commit manually.

The following was done in another shell outside of the working directory. I went ahead and pulled the `<YOUR_WHATEVER_SECRET>` values [created above](#run-a-simple-ex-command-to-reformat-the-file-for-bfg), put them in the `twcred.py` file in place of the real secret values, and committed it back to the original repository. It looks like this now:

```bash
λ (master) ~/src/pytwit: cat twcred.py
CONSUMER_KEY = "<YOUR_CONSUMER_KEY>"
CONSUMER_SECRET = "<YOUR_CONSUMER_SECRET>"
ACCESS_TOKEN_KEY = "<YOUR_ACCESS_TOKEN_KEY>"
ACCESS_TOKEN_SECRET = "<YOUR_ACCESS_TOKEN_SECRET>"
```

#### Clone a bare repository for BFG to do its magic in

Ok, back to the working directory [created earlier](#create-a-working-directory). This is done with the `--mirror` switch for `git clone`. It's basically just the contents of the `.git` repo and nothing else.

```bash
git clone --mirror ../pytwit
```

I like to use `tig` to poke around in repo histories. The `-C` allows me to do it from the working directory since the clone repo will be a child directory.

```bash
tig -C pytwit.git/
```

#### Go for broke

Now it's time to rewrite history! Don't forget `bfg` is an [alias created above](#i-made-a-temporary-alias-for-convenience). Also, we're doing this on a clone of the original.

```bash
bfg --replace-text twcred_replace.txt pytwit.git/
```

I like to poke around the commits where the passwords once existed.

```bash
tig -C pytwit.git/
```

When the `bfg` command completed, it output the following:

```text
Using repo : /Users/b1tb4ng/src/pytwit_cleanup/pytwit.git

Found 11 objects to protect
Found 2 commit-pointing refs : HEAD, refs/heads/master

Protected commits
-----------------

These are your protected commits, and so their contents will NOT be altered:

 * commit 7c4ec6db (protected by 'HEAD')

Cleaning
--------

Found 18 commits
Cleaning commits:       100% (18/18)
Cleaning commits completed in 82 ms.

Updating 1 Ref
--------------

  Ref                 Before     After
  ---------------------------------------
  refs/heads/master | 7c4ec6db | b03615c5

Updating references:    100% (1/1)
...Ref update completed in 13 ms.

Commit Tree-Dirt History
------------------------

  Earliest      Latest
  |                  |
  ..... ......... ..Dm

  D = dirty commits (file tree fixed)
  m = modified commits (commit message or parents changed)
  . = clean commits (no changes to file tree)

                        Before     After
-------------------------------------------
First modified commit | d9daa7be | b15a188d
Last dirty commit     | d9daa7be | b15a188d

Changed files
-------------

  Filename    Before & After
  -------------------------------
  notes.md  | 591ce98d ⇒ ec33a900
  twcred.py | 5d24eba6 ⇒ 4dd97f9e


In total, 3 object ids were changed. Full details are logged here:

  /Users/b1tb4ng/src/pytwit_cleanup/pytwit.git.bfg-report/2021-03-15/15-47-04

BFG run is complete! When ready, run: git reflog expire --expire=now --all && git gc --prune=now --aggressive
```

So the next step is to dip into the bare repo and run those git commands at the very end of the output

```bash
cd pytwit
git reflog expire --expire=now --all && git gc --prune=now --aggressive
```


#### Pushing back to your original repo

Pushing back to the original local repo is where things got a little weird. Mainly because I didn't understand pushing to a non bare repository with a checked out working copy. I just wanted to scrub my local repo before I pushed it to a brand new repo on Github.

Just to be extra safe, since the original repo was the *only* full copy of the code I went ahead and tar balled the original repo. This was done in another shell outside of the working directory.

```bash
tar -czvf pytwit.tgz pytwit/
```

Here's the abridged output from the first attempt:

```
λ (BARE:master) ~/src/pytwit_cleanup/pytwit.git: git push
...
To /Users/b1tb4ng/src/pytwit_cleanup/../pytwit
 ! [remote rejected] master -> master (branch is currently checked out)
error: failed to push some refs to '/Users/b1tb4ng/src/pytwit_cleanup/../pytwit'
```

Ok, so I checked out another branch called `list` in the original repo and tried again.

```
λ (BARE:master) ~/src/pytwit_cleanup/pytwit.git: git push
...
To /Users/b1tb4ng/src/pytwit_cleanup/../pytwit
 + 95357ae...147f023 master -> master (forced update)
 ! [remote rejected] list (branch is currently checked out)
error: failed to push some refs to '/Users/b1tb4ng/src/pytwit_cleanup/../pytwit'
```

Then I tried to `--force` it to no avail.

```
λ (BARE:master) ~/src/pytwit_cleanup/pytwit.git: git push --force
...
To /Users/b1tb4ng/src/pytwit_cleanup/../pytwit
 ! [remote rejected] list (branch is currently checked out)
error: failed to push some refs to '/Users/b1tb4ng/src/pytwit_cleanup/../pytwit'
```

I thought I'd be smart and just check out `master` again and delete the `list` branch. It didn't work.

```
λ (BARE:master) ~/src/pytwit_cleanup/pytwit.git: git push
...
To /Users/b1tb4ng/src/pytwit_cleanup/../pytwit
 ! [remote rejected] list (deletion of the current branch prohibited)
error: failed to push some refs to '/Users/b1tb4ng/src/pytwit_cleanup/../pytwit'
```

Eventually I just wound up cheating and faking the original repo as bare and leaving the working copy files in place.

I opened another terminal in the original repo and ran this:

```bash
git config --bool core.bare true
```

Did a push from the original terminal in the working directory's bare repo with no issues:

```bash
λ (BARE:master) ~/src/pytwit_cleanup/pytwit.git: git push
Everything up-to-date
```

And then put the clothes back on the original repo:

```bash
git config --bool core.bare false
```

After verifying with `tig` for a bit, Bob was my uncle!

The End.


## Tangents, Diversions, and a Shave of the Yak

#### A little more info on using `ex` for search and replace

Here's a great little summary of how to use `ex` for a basic search and replace from the command line. I found the meat of it [here on Stack Overflow](https://askubuntu.com/a/758107), and added the switches for future reference.

```text
ex -s -c '%s/OLD/NEW/g|x' file

    % select all lines
    s substitute
    g replace all instances in each line
    x write if changes have been made and exit

    -s silent mode
    -c command
```

Beware the `|x` on the end of the `ex` command above. "Write if changes have been made and exit." See the test loop in the [section below](#a-test-loop-to-see-if-basic-search-and-replace-working).

[Here's more info on ex's and vim's substitutions](http://vimregex.com/#substitute).

#### A test loop to see if basic search and replace working

This was used to suss out some issues with `ex` just hanging. It turns out if no changes are made, the `x` command after the pipe doesn't try to save the file and exit. You're forced to exit with a `q` keypress

It simply tries to replace a `=` char with a `~`.

```bash
ex -s -c '%s:=:\~:g|x' twcred_replace.txt
rm twcred_replace.txt && cp pytwit/twcred.py ./twcred_replace.txt
ls -la twcred_replace.txt
cat twcred_replace.txt
```

#### A quick and dirty fake twitter secret generator

I used this at the Python REPL to gin up some fake credentials. I spent way too much time on this, but it was fun. I may have been golfing a little. I didn't say I was good at it.

```python
import random, string
def ersatz(length): return ''.join(random.choices(string.ascii_letters + string.digits, k=length))
def gen(name, length): print('{} = "{}"'.format(name, ersatz(length)))
fake_it = [('CONSUMER_KEY', 23), ('CONSUMER_SECRET', 48), ('ACCESS_TOKEN_KEY', 48), ('ACCESS_TOKEN_SECRET', 42)]
[gen(nm, ln) for (nm, ln) in fake_it]
```

Yes, it's gross in Python to use a comprehension to generate side effects (`gen()` prints). I know of a few FP languages that are perfectly fine with this approach. Besides, I said it was quick and dirty. Well, the part about it being quick was a lie. I spent way too much time on this, but it's quick enough for you, old man. At least it wasn't as dirty as this:

```python
list(map(gen, *zip(*fake_me)))
```

Blame [this post](https://stackoverflow.com/questions/47045669/python-map-function-with-multiple-arguments-to-a-list-of-tuples) on Stack Overflow.

#### I needed some fake git refs, too

```python
def hx(): return ''.join(random.choices(string.hexdigits, k=8)).lower()
```

## References

* Using BFG to `--replace-text`
  * [Removing sensitive data from a repository](https://docs.github.com/en/github/authenticating-to-github/removing-sensitive-data-from-a-repository)
  * [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/)
  * [How to substitute text from files in git history?](https://stackoverflow.com/questions/4110652/how-to-substitute-text-from-files-in-git-history)
* Using `ex` to create an expressions file for BFG
  * [Find and replace text within a file using commands](https://askubuntu.com/a/758107)
  * [Editing with ex](https://www.cs.ait.ac.th/~on/O/oreilly/unix/vi/ch05_02.htm)
  * [Substitute Command](http://vimregex.com/#substitute)
* Quick and dirty fake generator
  * [Efficiently generate a 16-character, alphanumeric string (Stack Overflow)](https://stackoverflow.com/a/30779367)
  * [Assigning a lambda expression to a variable (pep8)](https://docs.quantifiedcode.com/python-anti-patterns/correctness/assigning_a_lambda_to_a_variable.html)
  * [Python: Iterate over list of tuples](https://code-maven.com/python-iterate-list-of-tuples)
  * [Python map function with multiple arguments to a list of tuples (Stack Overflow)](https://stackoverflow.com/questions/47045669/python-map-function-with-multiple-arguments-to-a-list-of-tuples)
  * [List Comprehension](https://www.w3schools.com/python/python_lists_comprehension.asp)
  * [multiple variables in list comprehension? (Stack Overflow)](https://stackoverflow.com/a/40351298)
  * [Asterisks in Python: what they are and how to use them](https://treyhunner.com/2018/10/asterisks-in-python-what-they-are-and-how-to-use-them/)
* git
  * [bare repository](http://git-scm.com/docs/gitglossary.html#def_bare_repository)
  * [Is there a git uncheckout? (Stack Overflow)](https://stackoverflow.com/questions/2311958/is-there-a-git-uncheckout)
  * [Git push error '[remote rejected] master -> master (branch is currently checked out)' (Stack Overflow)](https://stackoverflow.com/questions/2816369/git-push-error-remote-rejected-master-master-branch-is-currently-checked)
  * [What is this Git warning message when pushing changes to a remote repository?](https://stackoverflow.com/questions/804545/what-is-this-git-warning-message-when-pushing-changes-to-a-remote-repository/28262104#28262104)
* Other
  * [How to show hidden files on a Mac](https://www.macworld.co.uk/how-to/show-hidden-files-mac-3520878/)
  * [How to link to part of the same document in Markdown? (Stack Overflow)](https://stackoverflow.com/questions/2822089/how-to-link-to-part-of-the-same-document-in-markdown)
  * [Anchors in Markdown](https://gist.github.com/asabaylus/3071099)
  * [Markdown tables](https://www.markdownguide.org/extended-syntax/#tables)
  * [Markdown Quick Reference Cheat Sheet](https://wordpress.com/support/markdown-quick-reference/)

