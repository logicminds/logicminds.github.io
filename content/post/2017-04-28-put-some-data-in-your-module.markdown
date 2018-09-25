---
layout: post
title: "Put some data in your module"
date: 2017-06-01 08:31:20 -0700
comments: true
sharing: true
categories: [devops, puppet, retrospec, generator, hiera]
---
Puppet 4.9 cemented the final touches to putting data in puppet modules with the release of hiera 5.
Which by the way hiera is now part of Puppet project instead of a separate gem as with previous versions.

So now that that whole params.pp mess is over we can finally move forward and create modules with rich sets
of data without resorting to the params.pp hack.

So lets get started. In order to put data in your module you need to have a few things.

## Requirements
Ideally, you should be running puppet 4.9 to take advantage of hiera 5 and data in modules.
But if you happen to be running puppet 4.7-4.8 you can still utilize hiera 4 which contains similar features.  

Just note that when upgrading to puppet 4.9+ hiera 4 implementations will need to eventually be onverted to hiera 5.
This article will only focus on hiera 5 but the methods below are extremely similar for hiera 4.

 * Puppet 4.9+  

## Data in module the manual way
  1. Create a hiera.yaml file inside your module with at a minimum the following contents.  Also figure out which backend type to use.
  2. Create a data directory `mkdir data`. This should be in your module's root directory.
  3. Create a common.yaml file
  4. Optionally, create a `data/os` directory
  5. Add hiera data to common.yaml

```yaml
  ---
  version: 5
  defaults:
    datadir: data
    data_hash: yaml_data
  hierarchy:
      - name: "OS family"
        path: "os/%{facts.os.family}.yaml"
      - name: "common"
        path: "common.yaml"
```

  Note: only keys that start with your module's class name will be referenced, unless you are doing hiera interpolation.

## Automatic way
Retrospec automates all the steps above. So to get started install retrospec.

`gem install puppet-retrospec facter` and perform the following.

1. cd your_module_path
2. retrospec puppet module_data

Retrospec will clone down it's templates and create all the necessary files to get module data up and running. Yup! A single command.  Pretty easy right?  Best of all you can do this over and over on all your modules!

If you doing advanced lookup backends retrospec also allows additional options should you want to create a custom function for lookup.  See `retrospec puppet module_data -h` for more options.

`retrospec puppet module_data -b data_hash -n my_custom_hash -t native`


```
Options:
  -b, --backend-type=<s>     Which hiera backend type to use (hiera, data_hash, lookup_key, data_dig) (default: hiera)
  -n, --backend-name=<s>     The name of the custom backend (default: custom)
  -t, --function-type=<s>    What type of function to create the backend type with (native or v4) (default: native)
  -h, --help                 Show this message
```

Most of the time you will just want to use the defaults but some will be cooking up some fancy backends and need a way to easily generate the lookup backend.

## Hierarchy Setup
Unfortunately, Retrospec does not put the data in your hiera files. It does dump all the class parameters into common.yaml but you still need to sort out the values and hierarchy configuration for each parameter key. This might take some time to figure out.

If your module currently uses params.pp I would start there.
