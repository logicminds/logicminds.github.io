---
title: "Decouple Your Control Repo Branches"
date: 2021-03-13T09:52:43-08:00
published: false
layout: post
comments: true
categories: [puppet, devops, r10k]
---

Since the inception of R10k there has been a recipe of 1 part branch to 1 part puppet environment for puppet control repos.  This concoction has allowed us to create one-off ephemeral test environments that we all love simply by creating a new git branch. 

![](/images/beaker_on_fire.gif) 

We as a community have also incorporated the [gitflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) model into puppet development best practices.


Many environments use this r10k workflow where code is created in feature branches then merged to dev, then merged up to QA then acceptance and finally the coveted PRODUCTION branch.  The r10k gitflow model is present at most puppet installations.  This model has proven to reduce bugs, minimize downtime and improve confidence. 

1. merge feature_branch into dev
2. merge dev into QA
3. merge QA into acceptance (pre-prod)
4. merge acceptance into production

![](/images/07_control_repo_update.png)

If there is a code issue anywhere in this cycle, you simply create a new branch and merge that fix into the branch and voila.  

1. Create bug_fix branch based off of branch where the fix will be applied (destination)
2. Make changes
3. Create commit and pull request
4. Have team review and merge to destination branch


But what about the other branches / environments?  If you want the fix to also be applied to the other environments you will also need to merge that fix to all the other branches. This can be done with a [cherry-pick](https://git-scm.com/docs/git-cherry-pick) pretty easily. However, cherry picks will create unique commit checksums when applied and sometimes cause code conflicts that have to be resolved.  Additionally, you have to do this for EVERY BRANCH! 

  1. Quick and dirty
  2. Can take a long time to patch other branches
  3. Code conflicts might occur
  4. Branches will never be in sync again
  5. Cherry picked commit will be unique for each branch

Another way to push a bugfix is to create a bugfix branch from your development branch.  Merge the bugfix branch into development where all feature branches are merged into.  Once merged into development you then merge upwards into QA, Acceptance and finally PRODUCTION!  This is the cleanest way to patch a bug in your puppet code. But it has a few issues.

  1. Takes multiple merge requests
  2. Can introduce new features or changes into other branches other than the bug fix
  3. Can take a long time, sometimes weeks based on culture and policies

I would be willing to bet most of you have a a control repo with a messy history of commits, tons of implicit merge commits and branches that are 1000s of commits behind each other.  This is unfortunately perfectly normal due to having a puppet environment tightly coupled to a git branch and promoting code in a progressive manner.  

There are [other strategies](https://sairamkrish.medium.com/git-branching-strategy-for-true-continuous-delivery-eade4435b57e) that work differently but also have some drawbacks.  But ultimately the problem lies mostly with having branches as the article suggests.

https://www.wearefine.com/news/insights/env-branching-with-git/


## Removing branches
If we want to simplify our control-repo we simply remove branches.  Mo branches, mo problems.  No Code is the best code. Less branches == less environments.  But what if we need environment to test our code with? Can we test code in a environment without having to create a branch?

## Introducing YAML based environments
A YAML based environment is one that is created via a manually assigned map of code revision to puppet environments.  In essence, you can have 100 puppet environments while only maintaining a single git branch.
Furthermore, for the first time ever you can assign a puppet environment to a git tag and version your control repository like a puppet module.  A few advantages:

  1. Low number of branches, 2-3 max (not including feature branches)
  2. Create versioned releases from the control repo
  3. Deploy releases to any environment
  4. No more code promotion, deployments only



## The term Environment is overused
The term environment has many meanings which complicates code and data separation and just talking in general.
  1. puppet code environment
  2. network environment
  3. support level environment
   




Branches?  Wait.  What?



1. Create a branch based off of the destination of where the fix should be applied
2. Commit that fix to all the other branches 
 based on the where the fix needs to be applproduction branch, or dev, or QA, or acceptance.  Then merge that fix into all the other branches.   Wait, what? 



## Disadvantages
### Network and Datacenter environments
These environments while mostly tied to puppet code can sometimes also be tied to network environments or other physical boundaries in data centers thus complicating branch strategies.

When a branch is tied to a network environment you are now testing the code against different inputs that you might have forgotten about or didn't have access to.  

### Hotfixes and bugs
Another disadvantage here is that moving minor code changes like a tiny hiera change might take days or weeks to move through an entire pipeline in large environments.

This specific problem of minor code changes is called a hotfix and must be applied immediately. It is a one-off change that must be applied to almost all branches otherwise a subsequent merges will not contain the hotfix and regressions will occur.

This is by far the number one issue with using R10k and multiple git branches IMHO.

### Merge conflicts
Another downside to R10k is managing merge conflicts between branches.  Merging branches into each other in succession is great for reducing risk but can often cause merge conflicts that is time consuming to resolve. One method to is to force push the source into the destination.  `git push origin dev:qa -f`.  This will overwrite anything in QA and the same must be done for each upward branch. This means that any differences between the branches will be wiped out and could cause code regressions as a result. This is why any hotfixes must always be merged into dev.

## Advantages
R10k is awesome and the advantages outweigh the disadvantages. This article won't discuss the awesomeness R10k provides. 

## Fixing the R10k disadvantages by decoupling the branches from environments
Previously when puppet environments were coupled with a git branch you were not able to simply associate new code into that environment without merging code into that branch. 

With the new YAML based environments you can simply assign any code to any branch without merging. This simply means that 
if you want untested code in production is just a matter of reassignment.

With YAML based environments you just a single `main` git branch with short lived test/feature branches always being merged back into main and removed.  One branch, that is it.  

Puppet environments are now controlled with the following yaml data not presence of git branches.  In the sample file below each environment points to a git ref only. 
There is no branch in git named qa, development, test, or production. Instead the yaml data is referenced and the git ref is used to 
populate the environment code.

In this scenario we now get to use version tags!  These version tags stem from the main branch and should represent well tested code.  This scenario will open doors for many folks trying to version there code now. 

```
# /etc/puppetlabs/r10k/environments.yaml
---
production:
  type: git
  remote: git@github.com/puppetlabs/puppet-control-repo.git
  ref: v0.2.0

qa:
  type: git
  remote: git@github.com/puppetlabs/puppet-control-repo.git
  ref: 3cb49372

test:
  type: git
  remote: git@github.com/puppetlabs/puppet-control-repo.git
  ref: 89daf111

development:
  type: git
  remote: git@github.com/puppetlabs/puppet-control-repo.git
  ref: main


```

## Populating the YAML environment data source
There are several ways to provide this data
* Use a environments.yaml file and manually update this file
* Use bolt tasks to update
* Use a script to fetch the data
* Call a remote API to provide the data

Some of you may have noticed that updating a file on the puppet server is tedious and annoying.  Don't worry you can automated this.

## Additional notes
The astute reader might have noticed that you don't have the ability to create ephemeral environments on the fly git branch with yaml environments.  
## Tightly Coupled Branches Cons
* Tied to a specific network environment which is not tested first. 



This workflow, when implemented forces code to move a slower through the paces because of all the checks and balances.  Once in production
risk is low and confidence is high.  However, as you know all software sucks. There will be bugs

In a production environment we 

Current R10k workflow
Problem
Problem 2
Work arounds
Solution
Example
How to setup
Walkthrough why this is better
Some cons
Some pros

Decouple your environments from git branches

https://github.com/puppetlabs/r10k/blob/master/doc/dynamic-environments/configuration.mkd#experimental-features

#  Disproving our infallibility was never been easier. 
  We as sysadmins have taken on the developer workflow.  In essence, we are now, no different than developers. Better developers I might add becase we also control the hardware ;)

