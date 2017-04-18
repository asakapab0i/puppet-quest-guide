{% include '/version.md' %}

# Roles and Profiles

## Quest objectives

- Understand the roles and profiles pattern.
- Create **profile** classes that wrap component classes and set parameters
  specific to your site infrastructure.
- Create **role** classes to define the full configuration for a system through
  a combination of profile classes.
- Learn how to use regular expressions to create more flexible node
  definitions.

## Getting started

In the last quest, we shifted from managing the Pasture application on a single
system to distributing its components across the `pasture-prod.puppet.vm` and
`pasture-db.puppet.vm` hosts. The application server and database server you
configured each play a different role in your infrastructure and have a
different classification in Puppet.

As your Puppetized infrastructure grows in scale and complexity, you'll need to
manage more and more kinds of systems.  Defining all the classes and parameters
for these systems directly in your `site.pp` manifest doesn't scale well. The
**roles and profiles** pattern gives you a consistent and modular way to define
how the components provided your Puppet modules come together to define each
different kind of system you need to manage.

When you're ready to get started, enter the following command:

    quest begin roles_and_profiles

## What are roles and profiles?

The **roles and profiles** pattern we cover in this quest isn't a new tool or
feature in the Puppet ecosystem. Rather, it's a way of using the tools we've
already introduced to create something you can maintain and expand as your
Puppetized infrastructure evolves.

The explanation of roles and profiles begins with what we call **component
modules**. Component modules—like your Pasture module and the PostgreSQL module
you downloaded from the Forge—are designed to configure a specific piece of
technology on a system. The classes these modules provide are written to be
flexible. Their parameters provide an API you can use to specify precisely how
you need the component technology to be configured.

The roles and profiles pattern gives you a consistent way to organize these
component modules according to the specific applications in your
infrastructure.

A **profile** is a class that calls declares one or more related component
modules and sets their paramaters as needed. The set of profiles on a system
defines and configures the the technology stack it needs to fulfull its
business role.

A **role** is a class that combines one or more profiles to define the desired
state for a whole system. A role should correspond to the business purpose of a
server. If your CTO asks what a system is for, the role should fit that
high-level answer: something like "a database server for the Pasture
application." A role itself should **only** compose profiles and set their
parameters—it should not have any parameters itself.

## Writing profiles

Using roles and profiles is a design pattern, not something written into the
Puppet source code. As far as the Puppet parser is concerned, the classes that
define your roles and profiles are no different than any other class.

To get started setting up your profiles, create a new `profile` module
directory in your modulepath:

    mkdir -p profile/manifests

We'll begin by creating a pair of profiles related to the Pasture application.
The profile for the application server will use a conditional statement manage
two different configurations: a 'large' deployment that connects to an external
database, and a 'small' deployment that makes use of the default SQLite
database. A second profile will manage the PostgreSQL database that backs the
'large' instance of the app server.

To make it clear that all of these profiles relate to the Pasture application,
we'll place them in a `pasture` subdirectory within `profile/manifests`.

Create that subdirectory:

    mkdir profile/manifests/pasture

Next, create a profile for the Pasture application.

    vim profile/manifests/pasture/app.pp

Here, you'll define the `profile::pasture::app` class.


The quest tool created a `pasture-app-small.puppet.vm` and
`pasture-app-large.puppet,vm` node for this quest, so we can determine the
appropriate profile based on the node name. We'll use a conditional statement
to set the `$default_character` and `$db_uri` parameters based on whether the
node's `name` fact contains the string 'large' or 'small'. In the 'small' case,
we'll use the special `undef` value to leave these parameters unset and use the
defaults set in the `pasture` component class. We'll also add an `else` block
to fail with an appropriate error message if the `name` variable doesn't match
'small' or 'large'.

```puppet
class profile::pasture::app {
  if 'large' in $facts['name'] {
    $default_character = 'elephant'
    $db_uri            = 'postgres://pasture_user:m00!@pasture_db.puppet.vm/pasture'
  } elsif 'small' in $facts['name'] {
    $default_character = undef
    $db_uri            = undef
  } else {
    fail("The #{$facts['name'] node name must match 'large' or 'small'.}")
  }
  class { 'pasture':
    default_message   => 'Hello Puppet!',
    sinatra_server    => 'thin',
    default_character => $default_character,
    db_uri            => $db_uri,
  }
}
```

Next, create a profile for the Pasture database using the `pasture::pasture_db`
component class:

    vim profile/manifests/pasture/db.pp

You don't need to customize any parameters here, so you can use the `include`
syntax to declare the `pasture::pasture_db` class.

```puppet
class profile::pasture::db
  include pasture::pasture_db
}
```

While these profiles define the configuration of components directly related to
the Pasture application, you'll typically need to manage aspects of these
systems that aren't directly related to their business role. A profile module
may include profile classes to manage things like user accounts, DHCP
configuration, firewall rules, and NTP. Because these classes are applied
across many or all of the systems in your infrastructure, the convention is to
keep them in a `base` subdirectory. To give an example of a base profile, we'll
create a `profile::base::motd` profile class to wrap the `motd` component class
you created earlier.

Create a `base` subdirectory in your `profile` module's `manifests` directory.

    mkdir profile/manifests/base

Next, create a manifest to define your `profile::base::motd` profile.

    vim profile/manifests/base/motd.pp

Like the `profile::pasture::db` profile class, the `profile::base::motd` class
is a wrapper class with an `include` statement for the `motd` class.

```puppet
class profile::motd {
  include 'motd'
}
```

Writing these wrapper classes may initially seem like unnecessary complexity,
especially in the case of simple component classes like `motd` and
`pasture::pasture_db`. The value of consistency, however, outweighs the effort
of creating these wrapper classes.

A profile class is the single source of truth to define how a component
is configured in your site infrastructure. This ensures that you have a single
clear place to make any changes to how that component is configured. If you
ever decide to add parameters to your MOTD module, for example, you will be
able to easily set those parameters in a single profile class and see the
changes take effect across any nodes whose role includes that profile. If you
had set parameters specifically on a role or node level, on the other hand,
any changes would have to be duplicated across every system.

Now you have a few profiles available. Each of these profiles defines how a
component on a system should be configured. The next step is to combine them
into roles.

## Writing roles

A role combines profiles to define the full set of components you want Puppet
to manage on a system. A role should consist of only `include` statements to
pull in the list of profile classes that make up the role. A role should not
directly declare non-profile classes or individual resources.

A role's name should be a simple description of the business purpose of the
system it describes. Specific implementation details related to the technology
stack are left to the profiles. For example, `role::myapp_webserver` and
`role::myapp_database` are appropriate names for role classes, while
`role::postgres_db` or `role::apache_server` are not.

In this case, we need two roles to define the systems involved in the Pasture
application: `role::pasture_app` and `role::pasture_database`. First, create
the directory structure for your `role` module.

    mkdir -p role/manifests

Create a manifest to define your `role::pasture_app` role.

    vim role/manifests/pasture_app.pp

```puppet
class role::pasture_app {
  include profile::pasture::app
  include profile::motd
}
```

Next, create a role for your database server:

    vim role/manifests/database.pp

```puppet
class role::pasture_db {
  include profile::pasture::db
  include profile::motd
}
```

## Classification

With your roles clearly defined, classification is very simple. Each node in
your infrastructure can be classified with a single role class.

We can make use of another feature of Puppet's node definition syntax to make
our classification model even more concise. So far, you've used simple strings
as titles for the node definition blocks in your `site.pp` manifest. Instead of
a string, you can use a [regular expression](http://www.regular-expressions.info/)
as a node definition's title. This lets you apply the node definition to any
node whose title matches the pattern the regular expression specifies.

We'll use the following regular expression to match the two nodes we want to
classify as Pasture application servers: `/^pasture-app/`. The `/` characters
here are delimiters that indicate that we're using a regular expression rather
than an ordinary string. The `^` character marks the beginning of the string.
The rest of the characters are the ones we want to match in our node name.

If you would like to learn more about regular expressions, you can use
[rubular.com](http://rubular.com/) as a quick reference and testing tool. You
might test the regular expression given above against your node names to
validate the match. You can also refer to
[regular-expressions.info](http://www.regular-expressions.info/) for a more
in-depth guide on regular expressions. Generally, the kinds of regular expressions
needed for node definitions are quite simple.

Be aware that there are some special characters in regular expressions that you
will need to escape if you want to use them as literal characters to match in
your node name. For example, the dot (`.`) is a special character that will
match any single character.  If you want to match a name that includes a dot or
any other special character, you will need to escape it with a backslash
(`\.`). If the regular expressions in your node definitions start getting
overly complex, it may be a sign that you need to revisit your node naming
scheme or look into an alternate classification method such as the PE console
[node
classifier](https://docs.puppet.com/pe/latest/console_classes_groups.html) or
an [external node
classifier](https://docs.puppet.com/puppet/4.10/nodes_external.html).

To get started creating your node definitions, open your `site.pp` manifest.

    vim /etc/puppetlabs/code/environments/production/manifests/init.pp

Create a node definition block for your application server nodes.  Use the
`/^pasture-app/` regular expression as the title of your node definition and
include the `role::pasture_app` role class.

```puppet
node /^pasture-app/ {
  include role::pasture_app
}
```

Add a second node definition block for your database role. We only have one
database node, but using a regular expression here as well will make it easy
to scale in the future.

```puppet
node /^pasture-db/ {
  include role::pasture_db
}
```

Apply your changes.

## Review

TBD
More resources!