---
layout: post
title:  "Simple Locally Compiled GitHub Pages"
date:   2019-07-21 11:36:43
categories: [gh-pages]
comments: true
---

Sometimes it just takes too long to do something easy. It's been a
couple years since this blog was active, and when I tried to publish a
post I couldn't remember how I did it. It was clear `docs` was the
source branch and `master` was the compiled site, but I couldn't
remember how to easily do that.

Google was filled with unsatisfying solutions. Use Travis CI? A Gulp
task? A 10-15 line shell script with conditionals? Honestly, something
so straightforward shouldn't be so hard.

I finally realized I could accomplish the same thing with a
worktree. I created a worktree called `_site` that checked out master.

```
$ git worktree add $GIT_ROOT/_site/ master
```

Now, when I publish, I just have to push up a separate commit after
building. It's a shell script, but it has no conditionals and I
completely understand what it's doing.

```
#!/bin/sh
jekyll build
pushd _site
git add .
git commit -m 'Deploy'
git push origin master
popd
```
