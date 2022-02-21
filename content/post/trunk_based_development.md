---
title: "Trunk based development for your control repo"
date: 2021-03-13T09:52:43-08:00
published: false
layout: post
comments: true
categories: [puppet, devops, r10k]
---
Since the inception of R10k there has been a recipe of 1 part branch to 1 part puppet environment for puppet control repos.  This concoction has allowed us to create one-off ephemeral test environments that we all love simply by creating a new git branch. 

Since R10k was first introduced we have always used the promotion workflow to usher features into environments.  While this workflow served us well there is a different kind of code setup that is now possible with R10k that improves upon what we already know.  

Read below to see how r10k can be switched from a promotion based workflow to a trunk based development workflow.   But first, some background info.

## Promotion workflow
Many environments use the r10k workflow where code is created in feature branches then merged to dev, then merged up to QA then acceptance and finally the coveted PRODUCTION branch.  Additionally, there is usually an agreement within the team around which branch is the source of truth, but many times the answer is "it depends". 

This workflow is present at most puppet installations and has proven to reduce bugs, minimize downtime and improve confidence. I refer to this as the promotion workflow. This is a good strategy and has worked well over the years.  Although it does have some issues which I will detail below.

A typical scenario:

1. merge feature_branch into dev
2. merge dev into QA
3. merge QA into acceptance (pre-prod)
4. merge acceptance into production

![](/static/images/07_control_repo_update.png)

### Bug Fixes
If there is a code issue anywhere in this cycle, you simply create a new branch and merge that fix into the branch and voila. 

A typical bugfix scenario:

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

I would be willing to bet most of you have a a control repo with a messy history of commits, tons of implicit merge commits and branches that are 1000s of commits behind each other.  This is unfortunately perfectly normal due to having a puppet environment tightly coupled to a git branch and promoting code to each environment.  

There are [other strategies](https://sairamkrish.medium.com/git-branching-strategy-for-true-continuous-delivery-eade4435b57e) that work differently but also have some drawbacks.  But ultimately the problem lies mostly with having long lived branches as the [article suggests](https://www.wearefine.com/news/insights/env-branching-with-git/).

## Whats the solution?
If we want to simplify our control-repo we simply remove branches.  Mo branches, mo problems.  No Code is the best code.  Yes I am serious here.  

But if we remove branches how do we test code?  Can we test code in a environment without having to create a branch?  YES.  Read below.

## Introducing YAML based environments
A YAML based environment is one that is created via a manually or dynamically assigned map of a code revision to environment name.  In essence, you can have 100 puppet environments while only maintaining a single git branch.

Furthermore, for the first time ever you can assign a puppet environment to a git tag and version your control repository like a puppet module without any extra work. 

With an implementation of [YAML based environments](https://github.com/puppetlabs/r10k/blob/master/doc/dynamic-environments/configuration.mkd#experimental-features) you can fully utilize [TRUNK BASED DEVELOPMENT](https://trunkbaseddevelopment.com/) for your puppet codebase.

#### YAML based environments bring a few extra advantages
  1. Low number of branches, 1-3 max (not including ephemeral feature branches)
  2. Create versioned releases from the control repo
  3. Deploy code/releases to any environment
  4. No more environment based code promotion

### How is this possible?
R10k by default maps the branch name to a environment name with the same name.  If you want to manually control this behavior all you need to supply is a map for r10k to use.  In the **simplest form**  a YAML file similar to below is all that is needed to manually assign code to an environment.  What this file details is the environment name, location of source code and the revision of code to use.

This map file can be updated at any time and would live outside of the control repo. Furthermore, R10k can even call an http API or any script to produce the same content the below file contains.


```yaml
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

### Eating cake
Now, some may have noticed that a manually assigned map of code to environments doesn't allow for the dynamic environment creation we are accustomed to.  While this is true, you should know that R10k allows you to have multiple sources of which can be configured to use YAML based and dynamic (traditional) thus bringing both worlds together into a true TRUNK based development workflow with automatic environment creation for auto branch creation. 

Below is a snippet of a basic setup for puppet enterprise.  Non PE users can use the r10k puppet module with a similar hiera key

```yaml
# control_repo/data/common.yaml
puppet_enterprise::master::code_manager::sources:
 puppet:
   remote: 'N/A'
   type: yaml
   config: '/etc/puppetlabs/r10k/environments.yaml' 
 
 git:
 # 1/1 environment/branch
   remote: 'https://github.com/nwops/kontrol-repo.git'
   type: git
   prefix: false
   # Just in case someone pushes similar branch names, ignore to prevent collision
   ignore_branch_prefixes:
     - main
     - production
     - qa
     - test
     - development
```
