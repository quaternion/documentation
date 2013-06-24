---
title: Preparing Your Application
layout: default
---

<div class="alert-box radius">
  This will focus on preparing a Rails application, but most ideas expressed
  here have parallels in Python, or PHP applications
</div>

### 1. Commit your application to some externally available source control hosting provider.

If you are not doing already, you should host your code somewhere with a
provuder such as Github, BitBucket, Codeplane, or repositoryhosting.com.

<div class="alert-box radius">
At present Capistrano v3.0.x only supports Git. It's just a matter of time
until we support Subversion, Mecurial, Darcs and friends again. Please
contribute if you know these tools well, we don't and don't want to force our
miscomprehended notions upon anyone.
</div>

### 2. Move secrets out of the repository.

<div class="alert-box alert">
If you've accidentally committed state secrets to the repository, you might
want to take <a
href="https://help.github.com/articles/remove-sensitive-data">special
steps</a> to erase them from the repository history for all time.
</div>

Ideally one should remove `config/database.yml` to something like
`config/database.yml.example`, you and your team should copy the example file
into place on their development machines, under Capistrano this leaves the
`database.yml` filename unused so that we can symlink the production database
configuration into place at deploy time.

The original `database.yml` should be added to the `.gitignore` (or your SCM's
parallel concept of ignored files)

{% prism bash %}
    $ cp config/database.yml{,.example}
    $ echo config/database.yml >> .gitignore
{% endprism %}

This should be done for any other secret files, we'll create the production
version of the file when we deploy, and symlink it into place.

### 3. Initialize Capistrano in your application.

{% prism bash %}
    $ cd my-project
    $ cap install
{% endprism %}

This will create a bunch of files, the important ones are:

{% prism bash %}
  ├── Capfile
  ├── config
  │   ├── deploy
  │   │   ├── production.rb
  │   │   └── staging.rb
  │   └── deploy.rb
  └── lib
      └── capistrano
              └── tasks
{% endprism %}

### 4. Configure your server addresses in the generated files.

We'll just work with the staging environment here, so you can pretend that
`config/deploy/production.rb` doesn't exist, for the most part that's yoru
business.

Capistrano breaks down common tasks into a notion of *roles*, that is, taking
a typical Rails application that we have roughly speaking three roles, `web`,
`app`, and `db`.

It can be confusing, as the boundary of web and app servers is a bit blurry if
using [Passenger]() with Apache, which in effect embeds your app server in the
web server (embeds Passenger in the Apache process itself), confusingly
Passenger can also be used in modes where this isn't true, so we'll ignore
that for the time being, and if you know the difference (i.e you are using
nginx as your web server, and puma/unicorn, or similar for your app server,
that should be fine) we can assume that they're the same, which is pretty
common.

The example file generated will look something like this:

{% prism ruby %}
    set :stage, :staging

    # Simple Role Syntax
    # ==================
    # Supports bulk-adding hosts to roles, the primary
    # server in each group is considered to be the first
    # unless any hosts have the primary property set.
    role :app, %w{example.com}
    role :web, %w{example.com}
    role :db,  %w{example.com}

    # Extended Server Syntax
    # ======================
    # This can be used to drop a more detailed server
    # definition into the server list. The second argument
    # something that quacks like a has can be used to set
    # extended properties on the server.
    server 'example.com', roles: %w{web app}, my_property: :my_value

    # set :rails_env, :staging
{% endprism %}

Both the simple role, and extended server syntaxes result in one or more
servers for each role being defined. The `app` and `db` roles are just
placeholders, if you are using the `capistrano/rails-*` addons (more on
that later) then they have a meaning, but if you are deploying something
simpler, feel free to delete them if they're meaningless to you.

The extended server syntax exists to allow the definition of arbitrary server
properties; it's there incase people want to build the server list more
comprehensively from something like the *EC2* command line tools, and want to
use the extended properties for something that makes sense in their
environment.

Servers can be defined in a bunch of ways, the following shows defining two
servers, one where we set the username, and another where we set the port.
These host strings are parsed and expanded out in to the equivilent of the
server line after the comment:

{% prism ruby %}
  role :all, %w{hello@world.com example.com:1234}
  # ...is the same as doing...
  server 'world.com' roles: [:web], user: 'hello'
  server 'example.com', roles: [:web], port: 1234
{% endprism %}

### 5. Set the shared information in `deploy.rb`.

The `deploy.rb` is a place where the configuration common to each environment
can be specified, normally the *repository URL* and the *user as whom to
deploy* are specified here.

The generated sample file starts with the following, and is followed by a few
self-documenting, commented-out configuration options, feel free to play with
them a little:

{% prism ruby %}
    set :application, 'my app name'
    set :repo, 'git@example.com:me/my_repo.git'
    ask :branch, proc { `git rev-parse --abbrev-ref HEAD`.chomp }
{% endprism %}

Here we'd set the name of the application, ideally in a way that's safe for
filenames on your target operating system.

Second we set the repository URL, this *MUST* be somewhere that the server we
are deploying to can reach.

Here's how this might look in a typical example, note that we'll cover
authentication in the next chapter, for now we'll assume this repository is
open source, we'll take an example application from the [Rails Examples and
Tutorials](http://railsapps.github.io/) site; there we'll find maintained a
handful of typical Rails apps with typical dependencies.

The Rails application they host, which uses Devise (for authentication) and
Cancan (for authorization) along side Twitter Bootstrap for assets has been
forked to the Capistrano repository, but you can find the (unchanged) original
[here](https://github.com/RailsApps/rails3-bootstrap-devise-cancan).

{% prism ruby %}
    set :application, 'rails3-bootstrap-devise-cancan-demo'
    set :repo, 'https://github.com/capistrano/rails3-bootstrap-devise-cancan'
    set :branch, 'master'
{% endprism %}

I've simplified the `:branch` varaible to simply be a `set` varaible, not a
question prompt, as this repository only has a master branch.

## Roundup

**At this point Capistrano knows where to find our servers, and where to find
our code.**

We've not covered how we authorise our servers to check out our code (there
are three pretty good ways of doing that with Git), nor have we determined how
to authorise Capistrano on our servers yet.