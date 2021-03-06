{% include '/version.md' %}

# MySQL

## Quest Objectives

- Install and configure a MySQL server.
- Add a MySQL user, add a database, and grant permissions.

## Getting started

In this quest, we'll continue to explore how existing modules from the Puppet
forge can simplify otherwise complex configuration tasks. You will use Puppet
Labs' MySQL module to install and configure a server, then explore the custom
resource types included with the module. To get started, enter the following
command.

    quest begin mysql

## WhySQL?

The Puppet Labs MySQL module is a great example of how a well-written module can
build on Puppet's foundation to simplify a complex configuration task without
sacrificing robustness and control.

The module lets you install and configure both server and client MySQL
instances, and extends Puppet's standard resource types to let you manage MySQL
*users*, *grants*, and *databases* with Puppet's standard resource syntax.

## Server install

<div class = "lvm-task-number"><p>Task 1:</p></div>

Before getting started configuring your MySQL server installation, fetch the
`puppetlabs-mysql` module from the Puppet Forge with the `puppet module` tool.
(If you've already installed the modules from the cache, this task should
already be marked as complete and you can skip this step.)

    puppet module install puppetlabs-mysql
	
With this module installed in the Puppet master's module path, all the included
classes are available to classify nodes.

<div class = "lvm-task-number"><p>Task 2:</p></div>

Now we'll edit the `site.pp` to classify the Learning VM with the MySQL server class.

    vim /etc/puppetlabs/code/environments/production/manifests/site.pp

If you completed the NTP quest, you will already have a node declaration for the
`learning.puppetlabs.vm` certname. If not, create it now:

```puppet
node 'learning.puppetlabs.vm' {

}
```

Within that node block, you can declare your `::mysql::server` class and set its
parameters. For this example, we'll specify a root password and set the
server's max connections to '1024'. (You may notice that the formatting in vim is a
bit funky when typing or pasting nested hashes. You can disable this formatting with
the `:set paste` command in vim.)

```puppet
node 'learning.puppetlabs.vm' {
  class { '::mysql::server':
    root_password    => 'strongpassword',
    override_options => {
      'mysqld' => { 'max_connections' => '1024' }
    },
  }
}
```
	
Notice that in addition to standard parameters like the `root_password`, the class
takes a `override_options` as a hash, which you can use to address any
configuration options you would normally set in the `/etc/my.cnf` file. Using a
hash lets you manage these settings without requiring each to be written
into the class as a separate parameter. The structure of the `override_options`
hash is analogous to the `[section]`, `var_name = value` syntax of a `my.cnf` file.

<div class = "lvm-task-number"><p>Task 3:</p></div>

Use the `puppet parser validate` tool to check your syntax, then trigger a
puppet run:

    puppet agent -t
	
If you want to check out your new database, you can connect to the MySQL monitor
with the `mysql` command, and exit with the `\q` command.

To see the result of the 'max_connections' override option you set, use `less` to
view the `/etc/my.cnf.d/server.cnf` file:

    less /etc/my.cnf.d/server.cnf

And you'll see that Puppet translated the hash into appropriate syntax for the
MySQL configuration file:

    [mysqld]
    ...
    max_connections = 1024

## Scope

It was easy enough to use Puppet to install and manage a MySQL server. The
`puppetlabs-mysql` module also includes a bunch of classes that help you manage
other aspects of your MySQL deployment. 

These classes are organized within the module directory structure in a way that
matches Puppet's scope syntax. Scope helps to organize classes, telling Puppet
where to look within the module directory structure to find each class. It also
separates namespaces within the module and your Puppet manifests, preventing
conflicts between variables or classes with the same name.

Take a look at the directories and manifests in the MySQL module. Use the `tree`
command with a filter to include only `.pp` manifest files:

    tree -P "*.pp" /etc/puppetlabs/code/environments/production/modules/mysql/manifests/ 

You'll see something like the following:

    /etc/puppetlabs/code/environments/production/modules/mysql/manifests/
    ├── backup.pp
    ├── bindings
    │   ├── java.pp
    │   ├── perl.pp
    │   ├── php.pp
    │   ├── python.pp
    │   └── ruby.pp
    ├── bindings.pp
    ├── client
    │   └── install.pp
    ├── client.pp
    ├── db.pp
    ├── init.pp
    ├── params.pp
    ├── server
    │   ├── account_security.pp
    │   ├── backup.pp
    │   ├── config.pp
    │   ├── install.pp
    │   ├── monitor.pp
    │   ├── mysqltuner.pp
    │   ├── providers.pp
    │   ├── root_password.pp
    │   └── service.pp
    └── server.pp
	
Notice the `server.pp` manifest in the top level of the `mysql/manifests`
directory. 

You were able to declare this class as `mysql::server`. Based on this scoped
class name, Puppet knows to find the class definition in a manifest called
`server.pp` in the manifest directory of the MySQL module.

So `mysql::server` corresponds to:

    /etc/puppetlabs/code/environments/production/modules/mysql/manifests/server.pp

To take an example one level deeper, the `mysql::server::account_security` class
corresponds to:

    /etc/puppetlabs/code/environments/production/modules/mysql/manifests/server/account_security.pp

## Account security

For security reasons, you will generally want to remove the default users and
the 'test' database from a new MySQL installation. The `account_security` class
mentioned above does just this.

<div class = "lvm-task-number"><p>Task 4:</p></div>

Go back to your `site.pp` manifest and include the `mysql::server::account_security`
class in the `learning.puppetlabs.vm` node after the class declaration for `::mysql::server`. 
Remember, you don't need to pass any
parameters to this class, so a simple `include` statement will work in place
of a parameterized class declaration.

```puppet
node 'learning.puppetlabs.vm' {
  ...
  include ::mysql::server::account_security
  ...
}
```

Validate your site.pp

    puppet parser validate /etc/puppetlabs/code/environments/production/manifests/site.pp

When it checks out, trigger a puppet run:

    puppet agent -t

You will see notices indicating that the test database
and two users have been removed:

    Notice:
    /Stage[main]/Mysql::Server::Account_security/Mysql_database[test]/ensure:
    removed
    Notice:
    /Stage[main]/Mysql::Server::Account_security/Mysql_user[@localhost]/ensure:
    removed
    Notice:
    /Stage[main]/Mysql::Server::Account_security/Mysql_user[root@127.0.0.1]/ensure:
    removed

(Remember, though, that no automation tool is a substitute for a full understanding your
system's security requirements!)

## Types and providers

The MySQL module includes some custom *types and providers* that let you manage
some critical bits of MySQL as resources with the Puppet DSL just like you would
with a system user or service.

A **type** defines the interface for a resource: the set of *properties* you can
use to define a desired state for the resource, and the *parameters* that don't
directly map to things on the system, but tell Puppet how to manage the
resource. Both properties and parameters appear in the resource declaration
syntax as attribute value pairs.

A **provider** is what does the heavy lifting to bring the system into line with
the state defined by a resource declaration. Providers are implemented for a
wide variety of supported operating systems. They are a key component of the
Resource Abstraction Layer (RAL), translating the universal interface defined by
the **type** into system-specific implementations.

The MySQL module includes custom types and providers that make `mysql_user`,
`mysql_database`, and `mysql_grant` available as resources.

## Database, user, grant:

<div class = "lvm-task-number"><p>Task 5:</p></div>

These custom resource types make it possible to manage a new database with a
few lines of puppet code. 

Add the following resource declaration to the `learning.puppetlabs.vm` node
definition in your your `site.pp` manifest.  (Remember the `:set paste` command
if you need it.)

```puppet
node 'learning.puppetlabs.vm' {
  mysql_database { 'lvm':
      ensure  => present,
      charset => 'utf8',
  }
  ...
}
```

Similarly, with a user, all you have to do is specify the name and host as the
resource title, and set the ensure attribute to present. Enter the following
in your node definition as well.

```puppet
node 'learning.puppetlabs.vm' {
  mysql_user { 'lvm_user@localhost':
    ensure => present,
  }
  ...
}
```

Now that you have a user and database, you can use a grant to define the
privileges for that user. 

Note that the `*` character will match any table. Thus, `table => 'lvm.*'`
below means that the `lvm_user` has `ALL` permissions to all tables in
the `lvm` database. 

```puppet
node 'learning.puppetlabs.vm' {
  mysql_grant { 'lvm_user@localhost/lvm.*':
    ensure     => present,
    options    => ['GRANT'],
    privileges => ['ALL'],
    table      => 'lvm.*',
    user       => 'lvm_user@localhost',
  }
  ...
}
```

Once you've added declarations for these three custom resources, use the `puppet
parser validate` command on the `site.pp` manifest to check your syntax

    puppet parser validate /etc/puppetlabs/code/environments/production/manifests/site.pp

When your code looks good, trigger a puppet run:
	
    puppet agent -t
	
## Review

In this quest, you learned how to install and make configuration changes to a
MySQL server. You also got an overview of how classes are organized within the
module structure and how their names within your Puppet manifests reflect this
organization.

The MySQL module we used for this quest provides a nice example of how custom
types and providers can extend Puppet's available resources to make service or
application specific elements easily configurable through Puppet's resource
declaration syntax.
