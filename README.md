# Finding "Deleted" Commits - a demo

Below is a demonstration of how "deleted" commits can be recovered, both locally and from the remote repository.

NOTE: this was done in a public repository.

## Local Demonstration

First, let's see if we can recover a secret from a "deleted" commit that was overwritten by a `git rebase --interactive` that used `fixup` to meld a commit with a secret into a previous commit.

### Setup

```sh
$ git checkout -b demo/add-secret
Switched to a new branch 'demo/add-secret'

# create a file with a secret in it

$ touch secret.txt

$ echo 'SOME_SECRET=abcxyz' >> secret.txt

# make another file without a secret

$ echo 'This file does not contain any secrets' > not-secret.txt

$ git add not-secret.txt

# commit the non-secret file

$ git commit -m "chore(wip): add not-secret.txt"
[demo/add-secret 4f7baa3] chore(wip): add not-secret.txt
 1 file changed, 1 insertion(+)
 create mode 100644 not-secret.txt

# commit the secret file

$ git add -A

$ git status
On branch demo/add-secret
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   secret.txt

$ git commit -m "chore(wip2): add file with a secret"
[demo/add-secret b310c4a] chore(wip2): add file with a secret
 1 file changed, 1 insertion(+)
 create mode 100644 secret.txt

# update the non-secret file and commit it again

$ echo "\nThis is an update to not-secret.txt" >> not-secret.txt

$ git add -A

$ git commit -m "chore(wip3): update not-secret.txt"
[demo/add-secret 68161a9] chore(wip3): update not-secret.txt
 1 file changed, 2 insertions(+)
```

### Modifying the Tree with `git rebase --interactive` to "delete" the commit with a secret

Initial git log of commits with a secret:

```sh
git log --graph --all --reflog --pretty='%Cred%h%Creset -%C(auto)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset'
```
Output:

```
* abb7f35 - fix(wip4): remove secret from file (28 seconds ago) <Sinan Bolel>
* 68161a9 - chore(wip3): update not-secret.txt (5 minutes ago) <Sinan Bolel>
* b310c4a - chore(wip2): add file with a secret (6 minutes ago) <Sinan Bolel>
* 4f7baa3 - chore(wip): add not-secret.txt (7 minutes ago) <Sinan Bolel>
* 438b648 - (origin/main, main) chore: add gitignore and readme (9 minutes ago) <Sinan Bolel>
```

Now, let's rebase and fixup that commit:

```sh
git rebase -i HEAD~4

# and `fixup` all the commits down into `4f7baa3`.
```

Then, let's run `git log --graph --pretty='%h %d %s (%cr) <%an>'` to see the log:

```
* bc2d7d6 - (HEAD -> demo/add-secret) chore(wip2): add file with a secret (5 minutes ago) <Sinan Bolel>
* 4f7baa3 - chore(wip): add not-secret.txt (12 minutes ago) <Sinan Bolel>
* 438b648 - (origin/main, main) chore: add gitignore and readme (14 minutes ago) <Sinan Bolel>
```

However, if we check `git reflog`:

```
bc2d7d6 (HEAD -> demo/add-secret) HEAD@{0}: rebase (finish): returning to refs/heads/demo/add-secret
bc2d7d6 (HEAD -> demo/add-secret) HEAD@{1}: rebase (fixup): chore(wip2): add file with a secret
c46ff43 HEAD@{2}: rebase (fixup): # This is a combination of 2 commits.
b310c4a HEAD@{3}: rebase (start): checkout HEAD~4
abb7f35 HEAD@{4}: commit: fix(wip4): remove secret from file
68161a9 HEAD@{5}: commit: chore(wip3): update not-secret.txt
b310c4a HEAD@{6}: commit: chore(wip2): add file with a secret
4f7baa3 HEAD@{7}: commit: chore(wip): add not-secret.txt
438b648 (origin/main, main) HEAD@{8}: checkout: moving from main to demo/add-secret
438b648 (origin/main, main) HEAD@{9}: commit (initial): chore: add gitignore and readme
```

...we see that the commits that were "erased" by the `fixup` are still actually there! Then, using the commit hashes, we can get exactly what was in that secret commit:

```
$ git show b310c4a


commit b310c4a0ff84b7da3ee2c3bb42e4bed3ddf06f8e
Author: Sinan Bolel <1915925+sbolel@users.noreply.github.com>
Date:   Wed May 22 15:06:48 2024 -0400

    chore(wip2): add file with a secret

diff --git a/secret.txt b/secret.txt
new file mode 100644
index 0000000..587c58d
--- /dev/null
+++ b/secret.txt
@@ -0,0 +1 @@
+SOME_SECRET=abcxyz
```
There is our secret!!!

## Remote Repo Demo

Now that we've confirmed the commit wasn't actually deleted locally, let's see if we can also recover it from the remote repository on GitHub.

Same setup:

`git log --graph`

```
* a45ba63  (HEAD -> demo/remote) chore(wip3): update not-secret.txt (14 seconds ago) <Sinan Bolel>
* 091babf  chore(wip2): add file with a secret (14 seconds ago) <Sinan Bolel>
* 4b2a762  chore(wip): add not-secret.txt (15 seconds ago) <Sinan Bolel>
* 438b648  (origin/main, main) chore: add gitignore and readme (22 minutes ago) <Sinan Bolel>
```

```
$ git push
```

Update secret.txt:

```
$ git diff

...skipping...
diff --git a/secret.txt b/secret.txt
index 587c58d..715b60f 100644
--- a/secret.txt
+++ b/secret.txt
@@ -1 +1 @@
-SOME_SECRET=abcxyz
+SOME_SECRET=$REDACTED
```

Commit:

```
$ git add secret.txt

$ git commit -m "fix(wip4): remove secret from file"

[demo/remote e06cb4a] fix(wip4): remove secret from file
 1 file changed, 1 insertion(+), 1 deletion(-)
```

Then rebase:

```
$ git rebase -i HEAD~3

pick 3a433da chore(wip): add not-secret.txt
fixup d028043 chore(wip2): add file with a secret
fixup 7857d6a chore(wip3): update not-secret.txt

# Rebase 438b648..7857d6a onto 438b648 (3 commands)
```

Verify the log doesn't show the fixed-up commit:

```
$ git log --graph

* add76df  (HEAD -> demo/remote) chore(wip2): add file with a secret (2 minutes ago) <Sinan Bolel>
* 4b2a762  chore(wip): add not-secret.txt (5 minutes ago) <Sinan Bolel>
* 438b648  (origin/main, main) chore: add gitignore and readme (27 minutes ago) <Sinan Bolel>
```

Force push:

```
$ git push -f

Enumerating objects: 8, done.
Counting objects: 100% (8/8), done.
Delta compression using up to 10 threads
Compressing objects: 100% (5/5), done.
Writing objects: 100% (6/6), 1.28 KiB | 119.00 KiB/s, done.
Total 6 (delta 1), reused 0 (delta 0), pack-reused 0 (from 0)
remote: Resolving deltas: 100% (1/1), done.
To github.com:sbolel/hidden-commits-demo.git
 + a45ba63...add76df demo/remote -> demo/remote (forced update)
```

Check the remote reflog:

```
$ git log --graph --reflog --remotes --pretty='%h %d %s (%cr) <%an>%Creset'

* 716ae1a  # This is a combination of 2 commits. # This is the 1st commit message: (3 minutes ago) <Sinan Bolel>
| * b7c1cbe  (HEAD -> demo/add-secret, origin/demo/add-secret) chore(wip2): add file with a secret (3 minutes ago) <Sinan Bolel>
|/
| * b5ed205  fix(wip4): remove secret from file (3 minutes ago) <Sinan Bolel>
| * 3675848  chore(wip3): update not-secret.txt (4 minutes ago) <Sinan Bolel>
| * 2549e89  chore(wip2): add file with a secret (4 minutes ago) <Sinan Bolel>
|/
* a6cee72  chore(wip): add not-secret.txt (4 minutes ago) <Sinan Bolel>
| * 7857d6a  (demo/2) chore(wip3): update not-secret.txt (16 minutes ago) <Sinan Bolel>
| * d028043  chore(wip2): add file with a secret (16 minutes ago) <Sinan Bolel>
| * 3a433da  chore(wip): add not-secret.txt (16 minutes ago) <Sinan Bolel>
|/
| * add76df  (origin/demo/remote, demo/remote) chore(wip2): add file with a secret (16 minutes ago) <Sinan Bolel>
| | * 3f3a00f  # This is a combination of 2 commits. # This is the 1st commit message: (16 minutes ago) <Sinan Bolel>
| |/
| | * e06cb4a  fix(wip4): remove secret from file (17 minutes ago) <Sinan Bolel>
| | * a45ba63  chore(wip3): update not-secret.txt (20 minutes ago) <Sinan Bolel>
| | * 091babf  chore(wip2): add file with a secret (20 minutes ago) <Sinan Bolel>   <------- HERE IS THE ISSUE
| |/
| * 4b2a762  chore(wip): add not-secret.txt (20 minutes ago) <Sinan Bolel>
|/
| * bc2d7d6  chore(wip2): add file with a secret (33 minutes ago) <Sinan Bolel>
| | * c46ff43  # This is a combination of 2 commits. # This is the 1st commit message: (33 minutes ago) <Sinan Bolel>
| |/
| | * abb7f35  fix(wip4): remove secret from file (33 minutes ago) <Sinan Bolel>
| | * 68161a9  chore(wip3): update not-secret.txt (38 minutes ago) <Sinan Bolel>
| | * b310c4a  chore(wip2): add file with a secret (39 minutes ago) <Sinan Bolel>
| |/
| * 4f7baa3  chore(wip): add not-secret.txt (39 minutes ago) <Sinan Bolel>
|/
* 438b648  (origin/main, main) chore: add gitignore and readme (41 minutes ago) <Sinan Bolel>
```

Show the commit:

```
$ git show 091babf

diff --git a/secret.txt b/secret.txt
new file mode 100644
index 0000000..587c58d
--- /dev/null
+++ b/secret.txt
@@ -0,0 +1 @@
+SOME_SECRET=abcxyz
```

<span style="font-size: 36px;">⚠️ WE'VE FOUND THE SECRET!!!</span>
