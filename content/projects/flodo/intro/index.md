---
title: "intro"
date: 2024-02-04
author: "Farzan"
tags: ["flodo"]
topics: ["flodo"]
categories: ["flodo"]
# layout: "simple"
showTableOfContents: false
summary: "an introduction to what flodo is and isn't."
draft: false
---

_flodo_ is todo app with a new approach, one where there are no due dates. 

in flodo, we have '_do_'s and '_bucket_'s. 

'_do_'s are simply that, things to do. they are categorized as _need to do_, _want to do_, _should do_, _would be cool_.

a _do_ can be _done_, _doing_, _did_, _will do_, _might do_.


'_buckets_'s are an opinionated collections of '_do_'s. a _bucket_ is akin to a _project_, or an _idea_. for instance, a git repo could be a bucket, and `git log` would represent '_done do_'s.

a _bucket_ can be analyzed via plugins. for instance, a _git bucket_ can be analyzed by a _git plugin_, this plugin would read the git logs of the current directory. 

because there are no due dates in flodo, we still need a way to create a sense of commitment. for this we would use a minimal NLP to periodically remind user of their past _do_ and _bucket_ entries. 

_buckets_ are described, and description is used to find similarities between them. so if user was working on a _bucket_ a few weeks back and stopped, and this week they are working on a _similar bucket_, we would take this opporutnity to ask them if they _might want to do_.


in the background, flodo will use [fiber]() to stitch together tasks such as analyzing similar buckets, or summarizing what's been done. 


### cli examples

```
flo 

flo init # start a bucket here - add .flodo file

flo here # show what was done here if .flodo exists

flo summarize # summarize what's been done since last time and add it to .flodo

# below are all added to global database
flo should do x 
flo might do x
flo need do x
flo maybe do x

# same can be done for local .flodo via:
flo . should do x
# OR
flo here should do x


# updating dos 

flo do x did y
flo do x done


# for local
flo . do x did y

# for buckets 
flo bucket x should do y

flo bucket x do y did z



# pick a bucket and work in it 
flo bucket x
## '-' denotes last context
flo - do y did z

# can also set context for do 

flo do x
flo - did y

# or both 

flo bucket x do y
flo - did z


-----

## showing overal
flo show

flo show yesterday 

flo start today 

flo recommend 


## see in browser

flo dashbaord

```