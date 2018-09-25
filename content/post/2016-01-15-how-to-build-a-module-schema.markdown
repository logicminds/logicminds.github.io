---
layout: post
title: "How to build a module schema"
date: 2016-01-15 18:16:16 -0800
comments: true
published: true
categories: [puppet, testing, validation]
---
As a long time puppet module developer and puppet consultant I have noticed some trends over the years. Puppet modules are becoming increasingly sophisticated and complex.  When consuming forge modules I often spend time figuring out how it works and what the developer requires for parameters.  Especially when it comes to inserting data into hiera.

Schemas help validate the user is doing what the developer intended.

Creating a schema might not seem necessary but anybody using your module will greatly appreciate it. So if your a module developer here is how you can create a schema.

## Install Kwalify
Kwalfy is a project that allows you to easily write a schema and parse the yaml file against the schema you created.

To install: `gem install kwalify`

## Create a Schema Yaml file
The schema details all the inputs for your module and there are a two ways to create one.  Manual or Automatic. If doing manually head over to the [kwalify](http://www.kuwata-lab.com/kwalify/ruby/users-guide.01.html#schema) docs to see what options you can specify.  For automatic generation use the [retrospec puppet tool](https://github.com/nwops/puppet-retrospec) to auto generate a schema.  But retrospec doesn't do enough.  The schema definition does not accurately reflect parameter types because puppet is a loosely typed language, so even retrospec cannot determine with the resource exactly needs.  So you will need to further define your schema if you have complex data. However, retrospec will get you started and you can update the schema as needed.

The module schema should be named modulename_schema.yaml and placed in the root of your module.

```
cd /Users/user1/github/puppetlabs-apache
retrospec puppet
+ /Users/user1/github/puppetlabs-apache/puppetlabs-apache_schema.yaml
```

## Map Your parameters
Your schema file should detail exactly what your puppet class or definition is
expecting. Be extremely detailed.  Your users will love you.  

### Simple example
```yaml
---
  type: map
  mapping:
    hostclass:
      type: map
      mapping:
        "tomcat::catalina_home":
          type: str
          required: false
        "tomcat::user":
          type: str
          required: false
```
### Complex Example
Example of a more detailed schema, this is an example using the puppet lvm module.
Not everyone will have a complex schema, but some of us will.

```yaml
---
type: map
mapping:
  hostclass:
    type: map
    mapping:
      "lvm::volume_groups":
        type: map
        required: false
        mapping:
          =:
            type: map
            required: true
            mapping:
              'physical_volumes':
                type: seq
                desc: 'A list of physical volumes to use for the volume group'
                sequence:
                  - type: str
                required: true
              'logical_volumes':
                type: map
                required: true
                mapping:
                  =:
                    type: map
                    desc: 'The logical volume name'
                    required: true
                    mapping:
                      'size':
                        type: str
                        pattern: /[1-9]\d?[kKmMgGtT]{1}/
                        required: true
                        desc: 'The size of the logical volume'
                      'mountpath':
                        type: str
                        required: true
                        desc: "Where to mount the logical volume"
                      'fs_type':
                        default: ext4
                        desc: "The filesystem type to use when formating. "
                        type: str
                        required: false
                        enum: ['swap', 'ext4', 'ext3']

```
### Use a data type
Just like in puppet 4 kwalify allows the use of data types. I suspect str, any,
bool, number, map, and seq will be the most used.
Below are all the types we can use in our schema:

   * str
   * int
   - float
   - number (== int or float)
   - text (== str or number)
   - bool
   - date
   - time
   - timestamp
   - seq
   - map
   - scalar (all but seq and map)
   - any (means any data)

### Use a constraint
You can require and item, and require a specific type, and you can also require
the value to a specific set. This is very similar to puppet 4 data types.

  - regex patterns
  - enums
  - ranges
  - length

### Document the default values and description
Now this might seem redundant because you already documented these in puppet.  
But if the schema contains the description and default values, the user only has
to view a single file instead of many files. Furthermore, with proper tooling
we should be able to either pull this data out of puppet or use the schema to
populate the puppet code with default values and comments that are defined in the schema.    

```yaml
'fs_type':
  default: ext4
  desc: "The filesystem type to use when formating. "
  type: str
  required: false
  enum: ['swap', 'ext4', 'ext3']
```

See [Rules and Constraints](http://www.kuwata-lab.com/kwalify/ruby/users-guide.01.html#schema) for more info.

### Make items required
You can require parameters if your puppet code doesn't supply a default.
If the user doesn't supply the fs_type, validation will fail.

```yaml
'fs_type':
  type: str
  required: true
  enum: ['swap', 'ext4', 'ext3']
```

## Validate Your schema with Kwalify
To ensure your schema is a valid schema use the following command.  This simply
validates your schema against the kwalify tool.

`kwalify -m your_schema.yaml`

## Create a sample data set for testing purposes
Below is an example of data set that would be used with the puppet lvm module.
This would most likely be in hiera.

```yaml
hostclass:
  lvm::volume_groups:
    vg00:
      physical_volumes:
        - /dev/sda2
        - /dev/sdd
        - /dev/sdc
      logical_volumes:
        audit:
          size: 160M
          mountpath: /var/audit
        lv04:
          size: 4G
          mountpath: /tmp
        lv_swap:
          size: 0G
          fs_type: swapp
          mountpath: swap
    vg01:
      physical_volumes:
        - /dev/sdb
      logical_volumes:
        lv01:
          size: 32M
          mountpath: /system/app
        lv02:
          size: 16M
          mountpath: /system/oracle
        lv03:
          size: 40M
          mountpath: /var/opt/oracle
        backup:
          size: 0K
          mountpath: /backup
```

## Validate your data against your schema
Now we will validate the data against our schema. Notice that my sample data set contains a few errors.  Did you even notice?
This is intentional, because this shit won't work if you accidentally had these values in hiera.  Because that would never happen ...

```
kwalify -lf test_schema.yaml /tmp/test_schema_data.yaml
/tmp/test_schema_data.yaml#0: INVALID
- (line 16) [/hostclass/lvm::volume_groups/vg00/logical_volumes/lv_swap/size] '0G': not matched to pattern /[1-9]\d?[kKmMgGtT]{1}/.
- (line 17) [/hostclass/lvm::volume_groups/vg00/logical_volumes/lv_swap/fs_type] 'swapp': invalid fs_type value.
- (line 33) [/hostclass/lvm::volume_groups/vg01/logical_volumes/backup/size] '0K': not matched to pattern /[1-9]\d?[kKmMgGtT]{1}/.
```

Now the cool about this type of validation is that it is immediate and extremely specific.
If every module developer had a well defined schema, there would be no errors in our hiera data.  

## Summary
There is still lots to learn but the process for creating a schema is the same across every module. So hopefully this article is enough to get you started. Having a schema for your module allows the end user to easily assemble a master schema in order to validate all of their hiera data.  It could also be used to create a README file that details all the parameters as well. I suspect that if schema files take off we can see some further tooling around generation and possibly more use cases for schemas.

So stay tuned as my next article talks about just how to [validate hiera](http://logicminds.github.io/blog/2016/01/16/testing-hiera-data) using schemas.
