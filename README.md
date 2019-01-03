<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Dev Docs](#dev-docs)
  - [Theme](#theme)
  - [Getting started with doc development](#getting-started-with-doc-development)
  - [Update the theme](#update-the-theme)
  - [Layout](#layout)
    - [Global Menu](#global-menu)
    - [Page Menu](#page-menu)
  - [Generating new doc sections](#generating-new-doc-sections)
  - [Viewing the docs](#viewing-the-docs)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Dev Docs

The dev docs are managed by a static site generator called [Hugo](https://gohugo.io/).

Working with hugo requires you to learn the hugo generator system.
When creating more docs you can follow the [hugo docs](https://gohugo.io/documentation//) for more information.

## Theme

In order to maintain the Decred brand we have forked the the hugo docs theme and replaced with
the Decred brand colors, fonts and images. You can work directly with this theme [here](https://gitlab.com/coreynwops/decred-hugo-theme).

The usage of this theme with regards to these docs will be as a git submodule. Please be aware of this. If you are unfamilar with git submodules I would recommend spending the time to read [this article](https://medium.com/@porteneuve/mastering-git-submodules-34c65e940407) about its usage.

## Getting started with doc development

In order to get started with writing docs or updating the site navigation you first need to clone this repo and perform the following steps.

0. [install hugo](https://gohugo.io/getting-started/installing/)
1. `git clone https://gitlab.com/coreynwops/decred-dev-docs`
1. `git submodule update --init`
1. `cd decred-dev-docs`
1. `hugo`

If successful you should see:

```
 ~/decred/decred-dev-docs$ hugo

                   | EN
+------------------+----+
  Pages            | 76
  Paginator pages  |  0
  Non-page files   |  0
  Static files     | 93
  Processed images |  0
  Aliases          | 34
  Sitemaps         |  1
  Cleaned          |  0

Total in 86 ms
```

## Update the theme

There will be a time when the decred hugo theme requires an update. In order to update you
will need to run the following command:

`git submodule update`

You can perform this at any time.

Make sure you read [this article](https://medium.com/@porteneuve/mastering-git-submodules-34c65e940407) for further information.

## Layout

### Global Menu

The docs are layed out with a few different menus. First, there is the top nav bar menu. Some of the links that exist there are just placeholders so if something needs to be added or removed. This menu is considered the global menu, as noted in the `config.toml` file with `[[menu.global]]`

### Page Menu

Each main page of the global menu has its own menu system as well. Btw, this is often referred as the taxonomy. As an example you can see the main menu system here:

```
[[menu.docs]]
  name = "Getting Started"
  weight = 1
  identifier = "getting-started"
  url = "/getting-started/"
```

In this example the menu.docs site menu has a link to the getting-started page.

These docs are all stored under the content directory.

## Generating new doc sections

I have created am archetype for generating a new doc. Essentially, this is just a template so we can easily create or modify archetypes.

`hugo new -k doc_section en/<topic>/<subtopic>/_index.md`

ie. `hugo new -k doc_section en/communication/irc/_index.md`

So when you want to create a new topic this is all that is required. After you create this file, you just need to add the markdown content in the `_index.md` file.

Also keep in mind there are some cool snippets you can use in your content. Please see
the [shortcodes](https://gohugo.io/content-management/shortcodes/) for more info

## Viewing the docs

You can run the hugo site generator at any time to ensure your docs do not contain errors.
`hugo`

To see the docs website you can run `docs serve` which generates the docs and starts a webserver. This is also a good way to develop too since hugo adds file watchers and re-generates the static content as you modify it. This is all done live so no need to restart `hugo serve`.

```
  ~/decred/decred-dev-docs   master ●  hugo serve

                   | EN
+------------------+----+
  Pages            | 76
  Paginator pages  |  0
  Non-page files   |  0
  Static files     | 93
  Processed images |  0
  Aliases          | 34
  Sitemaps         |  1
  Cleaned          |  0

Total in 44 ms
Watching for changes in /decred/decred-dev-docs/{content,data,themes}
Watching for config changes in /decred/decred-dev-docs/config.toml
Serving pages from memory
Running in Fast Render Mode. For full rebuilds on change: hugo server --disableFastRender
Web Server is available at http://localhost:1313/decred-dev-docs/ (bind address 127.0.0.1)
Press Ctrl+C to stop
```

## Spell Check
`docker run -ti --rm -v ${PWD}:/app --workdir=/app tmaier/markdown-spellcheck:latest -a -n --en-us content/**/*.md`

`--report "content/**/*.md" --ignore-acronyms --ignore-numbers --en-us`