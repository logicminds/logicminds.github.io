---
layout: post
title: "Adding Puppet Debugger and other gems when using PDK"
date: 2019-04-02T10:05:51-08:00
comments: true
published: true
categories: [debugger, puppet, pdk, devops]
---

The [Puppet Development Kit](https://puppet.com/docs/pdk/1.x/pdk.html) (PDK) makes it really simple and efficient to work on puppet modules. If you have never used the PDK I would encourage you to try it out especially if you use windows.

The [PDK](https://puppet.com/docs/pdk/1.x/pdk.html) wraps all the annoying tasks so that you don't have to endure the pain of modifying Gemfiles, spec files, folder structure and other puppet related module development tasks.

But with any good wrapper code, what you gain in efficiency you do so at the cost of control and customization. And this is certainly true for the PDK. The PDK maintains tight control of certain files like the Gemfile because it was built to meet many use cases and operating systems.  However, Puppet has built some unique methods for losing some of this control.  So if you have become accustomed to the old way developing puppet modules you will now have to unlearn a few things and use the PDK methods instead.

### Adding New Gems

One such action you could do easily before PDK was adding ruby gems to the Gemfile. While you can still do this, you should edit a different file instead.

In this example I want to add the `puppet-debugger` gem to the puppet module's Gemfile. This can be accomplished either by updating the [PDK templates](https://github.com/puppetlabs/pdk-templates) or adding a .sync.yml file that modifies the templates for just that module.

#### Update the template

If you want to apply a Gemfile change to all **new** modules you should fork the [PDK templates](https://github.com/puppetlabs/pdk-templates) and add your gem to the [config defaults.](https://github.com/puppetlabs/pdk-templates/blob/master/config_defaults.yml#L574)

Essentially each section of the config_defaults file is a yamlized version of the associated file's contents. So if you want to add a gem like the puppet-debugger you would simply add it here underneath the Gemfile section. The contents that go here are the same you would find in the Gemfile but in yaml format.

```yaml
Gemfile:
  required:
    ":development":
      - gem: puppet-debugger
        version: "=~ 0.10.1"
      - gem: fast_gettext
        condition: "Gem::Version.new(RUBY_VERSION.dup) >= Gem::Version.new('2.1.0')"
      # json_pure 2.0.2 added a requirement on ruby >= 2. We pin to
      # json_pure <= 2.0.1 if using ruby 1.x
      - gem: json_pure
        version: "<= 2.0.1"
        condition: "Gem::Version.new(RUBY_VERSION.dup) < Gem::Version.new('2.0.0')"
```

You don't need the condition part but if you want to mandate which ruby version the user
should have it would be a good idea to force that.

The condition breaks down like this: is the currently used Ruby version less than 2.0.0?

I have provided some examples for further explanation below.

```
Gem::Version.new(RUBY_VERSION.dup)
=> Gem::Version.new("2.4.4")
[2] pry(main)> Gem::Version.new('2.0.0')
=> Gem::Version.new("2.0.0")
[3] pry(main)> Gem::Version.new(RUBY_VERSION.dup) < Gem::Version.new('2.0.0')
=> false
[4] pry(main)> Gem::Version.new(RUBY_VERSION.dup) < Gem::Version.new('2.5.0')
=> true
```


Once you make the change to this file. Just commit the change to your template repository. 

From then on each new module created from the PDK using your custom template url `--template-url https://github.com/puppetlabs/pdk-templates` 
will add the puppet-debugger gem or whatever changes you made.

#### Updating the .sync file

A more simplified alternative approach is to use the .sync file which doesn't modify the templates. Since the PDK templates are only used on initial creation and conversion of the module. Puppet created a special mechanism that allows for adhoc changes without modifying the pdk-templates.  This mechanism also serves as a way to synchronize changes across all your modules.  

Furthermore, If you can't change the pdk-templates this is the next best thing.  If you have used  [retrospec](https://www.retrospec-puppet.com) the sync feature is similar with the exception of getting way more granular and syncing changes inside the file instead of just the file itself. You can read more about this sync [mechanism](https://puppet.com/docs/pdk/1.x/customizing_module_config.html#customizing-the-default-template) that allows you to "modify the templates" without modifying the template repos. Essentially it is a config file for the templates a la post module creation. By default the PDK doesn't create this file by default.

To add the puppet-debugger gem or any gem for that matter, Create a .sync.yml file with the contents below.

```
# .sync.yml
---
Gemfile:
  required:
    ':development':
      - gem: 'puppet-resource_api'
      - gem: 'puppet-debugger'

```

Once you create this file you need to run `pdk update` and follow the prompts or `pdk update --force` to blindly accept changes.

This will modify the Gemfile like so:

```
group :development do
  # note, I remove contents above for brevity
  gem "puppet-resource_api",                           require: false
  gem "puppet-debugger",                               require: false
end
```

Since the .sync.yml file is now in your module's repo anybody else working on your module will have the same environment. Additionally you can add the .sync.yml file to all your modules and they will have the same set of gems too.

#### Update the Gemfile Directly

You can certainly modify any file in the puppet module's directory just like you did before. However, because some of these files are under PDK's control those changes may get erased if someone runs `pdk update`. So I would recommend updating the .sync.yml file or the template repos first.

## Summary
So there you have it.  PDK allows quite a but of granule control of your module with the .sync file and forking of pdk-templates. 
Now that you know what to modify you can start by adding the puppet-debugger and other gems you want to use.


