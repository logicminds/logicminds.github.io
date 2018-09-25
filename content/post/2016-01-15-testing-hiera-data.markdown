---
layout: post
title: "Testing Hiera Data"
date: 2016-01-16 18:49:38 -0800
comments: true
published: true
categories: [devops, puppet, testing, hiera validation]
---
As a puppet consultant I often run into the same problems with multiple clients. Many times I can reuse a magical script and my value instantly becomes obvious. I like to think I have things figured out, but sometimes there are just problems we as a community have not solved yet.  

The problem I am talking about is hiera validation. Most of us are too busy learning puppet, ruby, markdown, and git that testing is not a priority until your puppet code blows up [in your face](https://youtu.be/BkkUvMGz4A0?t=42). But for those who know hiera well, and understand what bad data means, than read on.

```
Error: Could not retrieve catalog from remote server: Error 400 on SERVER: Could not find data item lvm::volumes in any Hiera data file and no default supplied on node puppetmaster3278.nwops.io

```

## What do we test
Traditionally, the answer to testing hiera has been to just check for correct YAML syntax. But the problem with this is that YAML is very accepting of various inputs. We need to validate hiera data not YAML.

### What we care about
  - How the hell do we test the data going into hiera?
  - How do we test that the keys we type, match up with the puppet parameters in the modules?  
  - How do we test that the values in the keys are in the same format that the puppet code is expecting?  
  - How do you test that you added all the necessary keys?
  - How do you test your ability to create correct YAML syntax?
  - How do you do all this is under 1 second?

## How do we test
Up until now most of use used our eyeballs to test hirea data. But my client won't keep me around forever and I can't just say look for these errors. Using `YAML.load_file('common.yaml')` [does not do a damn thing](https://github.com/gds-operations/puppet-syntax/issues/39)! After all we are validating hiera data not YAML.

At a previous client I used rspec-puppet and the hiera-puppet-helper to not only mock hiera but to also test real hiera data with my unit tests. But anytime someone changed the data the tests broke.

Below is an example of using rspec-puppet with hiera.

```
require 'hiera'
hiera_file = File.expand_path(File.join(__FILE__,  '..', '..', '..', '..', 'hieradata', 'spec.yaml'))
shared_context :hiera do
  let(:hiera_config) do
    hiera_file
  end
  hiera = Hiera.new(:config => hiera_file)
end

it { is_expected.to contain_file('/tmp/test').with_content("hello")}
```

But none of the above solutions work reliably. And the reason behind our inability to test hiera data is we don't know what we are testing.  We don't know what kind of data needs to go in those values.  There is no definition or map that magically tells us what should and should not put in our hiera data.  Or is there?

This is where a module schema becomes invaluable. The module schema details the exact definition of all the parameters for that module.  So all we need to do is extract the schemas from every module being used into a giant master schema.

Creating a master schema might seem impossible because every single implementation of hiera data is unique. But I assure you its not impossible, just incredibly tedious. But we are devops dammit!  Lets automate that shit!

## Building a master schema
I have previously explained how to [build a module schema](http://logicminds.github.io/blog/2016/01/15/how-to-build-a-module-schema). And I have added support for auto generating schemas with [retrospec puppet tool](https://github.com/nwops/puppet-retrospec). But we need to build something slightly different for validating hiera data. We need a master schema that contains all the schemas from all the modules we are using.

There are a few ways we can build up a master schema.

### 1. Use existing hiera data
The hiera data you have contains all the keys and values that are currently being used.  Ruby makes it easy to turn
YAML files into native ruby objects.  So you can read your hiera data files and map all the keys and values into a schema pretty quickly. The only downside is your schema won't be very specific until you take some time to define the schema with complex data types that might be lurking in your puppet code.

I have written such a script for my client, and it works pretty awesome.  My CI job fails when someone inserts hiera data without an associated mapping for it. It will even suggest the mapping to use in the master schema.  Basically your just working backwards to create a schema when given data.

One additional trick is to ensure that all the hiera data keys are set to `required: true` in your schema. Because the key already exists in your hiera data it will help enforce spelling mistakes in key names.

### 2. Use Puppetfile to dynamically generate schemas on the fly
For now this method is more of a pipe dream.  But in a perfect world where every module contains a well defined schema. We could easily read the contents of the module's schema for each module defined in the Puppetfile and merge together the contents into a single master schema that would be unique for each permutation of the Puppetfile.

That script might look something like this:

```ruby
#!/usr/bin/env ruby
@master_schema = {}

# returns schema as a string
def get_remote_schema(url)
  uri = URI.parse(url)
  http = Net::HTTP.new(uri.host, uri.port)
  request = Net::HTTP::Get.new(uri.request_uri)
  response = http.request(request)
  response.body
end

# implement the mod method that Puppetfile uses
def mod(name, opts={})
  schema = YAML.load(get_remote_schema(opts[:url])
  @master_schema.merge!(schema['hostclass']) # host class parameters only
end  

eval(File.read('Puppetfile') # read and eval the puppetfile

# write the master schema out
File.open('master_schema.yaml', 'w') {|file| file.write(@master_schema.to_yaml)}

```

With this method we are relying solely on the developers schema to validate our hiera.  Of course you could always
maintain a better more static master schema. Additionally, you could make some pull requests to update the developer's schema which in turn benefits everyone.  

And this is where it becomes tedious. Since nobody has ever thought about creating
schemas for their puppet code it might take some time to build up your master schema.

Your schema can be as little or big as you want. The better it is the more errors that will be caught. So lets move on and assume you have a well defined master schema.

## Building a script to validate your data
Now that you have a master schema and lots of hiera data to validate, how do you validate all the keys across all the files? Basically you need to build a script that uses the kwalify parser to validate hiera files against your master schema.  Below are some snippets from a much bigger script that does this validation.

```ruby
# round up all the hiera files
@hiera_files = Dir.glob(File.join('data','**', "*yaml"))
@hiera_files.each do |file|
  validate_file(file)
end

# an instance of the kwalify parser
# since we don't need to create a new instance each time
# we cache the object here
# returns a kwalify validator parser instance
def parser
  unless @parser
    ## load schema data
    schema = Kwalify::Yaml.load_file(@schema_file)
    ## create validator
    validator = Kwalify::Validator.new(schema)
    @parser = Kwalify::Yaml::Parser.new(validator)
  end
  @parser
end

# use the kwalify validator to validate each file
# returns an array of errors if any.
def validate_file(file)
  logger.debug "Validating file: #{file}"
  ## load document and parse
  document = parser.parse_file(file)
  begin
    errors = parser.errors || []
  rescue Kwalify::SyntaxError => e
     return [e]
  end
end
```

There is actually a lot more to this script. One example is that hiera allows us to define a key in any file.  But this validation doesn't know that because it works with one file at a time.  So if a schema requires a key and that hiera data file doesn't contain the key, validation fails.  So we have to treat all the files as one big file. We can either concat all the files together or load every file into a giant hash and use the hash to remember which keys have been validated already when they are required.  Below is an example of building a giant hash.


```ruby
# load em up!  
@referenced_keys = {}
@hiera_files.each do |file|
  @referenced_keys.merge!(YAML.load_file(file))
end
```

So validation fails because it cannot find a required key but we can use some logic to determine it a thats really a problem by using our giant hash.

```ruby
  def validate_file(file)
    logger.debug "Validating file: #{file}"
    ## load document and parse
    document = parser.parse_file(file)
    begin
      errors = parser.errors || []
    rescue Kwalify::SyntaxError => e
       return [e]
    end   
    # if a given key is already defined somewhere else we don't
    # want to mark the key as required
    errors.find_all do |error|
      if error.error_symbol == :required_nokey
        # since this is a required no key error there is no path
        # so we must find the key we are missing
        key = error.to_s.match(/key\s'(.*):'/)[1]
        logger.debug "Looking up key #{key}"
        !@referenced_keys.has_key?(key) # if key is found this is not an error
      else
        # all other errors should be returned as normal errors
        true
      end
    end
  end
```

And really thats all there is to it. These are just snippets from the entire script. The other part of my script deals with creating maps given the data and also some pretty output.  But hopefully it should give you an idea how to build your own validator using the kwalify parser.  I would share the rest of my script but its not ready for prime time and I don't have the capacity to maintain another gem for public consumption.  You could also go the other route by concatenating all the files together and using `kwalify -lf master_schema.yaml giant_hiera_data_file.yaml` but that might have some drawbacks.

## Summary
I have shown you how to [create module schemas](http://logicminds.github.io/blog/2016/01/15/how-to-build-a-module-schema). And this article shows you how to create a master schema in order to validate your data. So once you build up the master schema and create a validation script, the payoff is huge!  Everyone wins. So go forth and give this a try.
