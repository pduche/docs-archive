---
layout: default
title: "PE 2015.2 » Quick Start » Firewall"
subtitle: "Firewall Quick Start Guide"
canonical: "/pe/latest/quick_start_firewall.html"
---

[downloads]: http://info.puppetlabs.com/download-pe.html
[sys_req]: ./install_system_requirements.html
[agent_install]: ./install_agents.html
[install_overview]: ./install_basic.html

Welcome to the Puppet Enterprise Firewall Quick Start Guide. This document provides instructions for getting started managing firewall rules with PE.

With a firewall, admins define a set of policies (firewall rules) that usually consist of things like application ports (TCP/UDP), node interfaces (which network port), IP addresses, and an accept/deny statement. These rules are applied from a “top-to-bottom” approach. For example, when a service, say SSH, attempts to access resources on the other side of a firewall, the firewall applies a list of rules to determine if or how SSH communications are handled. If a rule allowing SSH access can’t be found, the firewall will deny access to that SSH attempt.

To best manage firewall rules with PE, separate these rules into `pre` and `post` groups.

In this exercise, you'll learn to:

* [Write a simple module that contains a class called `my_firewall` to define the firewall rules for your PE-managed infrastructure](#write-the-my-firewall-class).
* [Use the PE console to add the `my_firewall` class to your agent nodes](#use-the-pe-console-to-add-the-my-firewall-class).
* [Write an additional class to open ports for the Puppet master](#open-ports-for-the-puppet-master)
* [Enforce the desired state of the `my_firewall` class](#enforce-the-desired-state-of-the-my-firewall-class).

## Install Puppet Enterprise and the Puppet Enterprise Agent

If you haven't already done so, install PE. See the [system requirements][sys_req] for supported platforms.

1. [Download and verify the appropriate tarball][downloads].
2. Refer to the [installation overview][install_overview] to determine how you want to install PE, and then follow the instructions provided.
3. Refer to the [agent installation instructions][agent_install] to determine how you want to install your PE agents, and then follow the instructions provided.

>**Tip**: Follow the instructions in the [NTP Quick Start Guide](./quick_start_ntp.html) to have PE ensure time is in sync across your deployment.

## Install the puppet-firewall Module

The firewall module, available on the Puppet Forge, introduces the firewall resource, which is used to manage and configure firewall rules from within the Puppet DSL.  You can learn more about the module by visiting [http://forge.puppetlabs.com/puppetlabs/firewall](http://forge.puppetlabs.com/puppetlabs/firewall).

**To install the firewall module**:

From the PE master, run `puppet module install puppetlabs-firewall`.

You should see output similar to the following:

        Preparing to install into /etc/puppetlabs/puppet/environments/production/modules ...
        Notice: Downloading from https://forgeapi.puppetlabs.com ...
        Notice: Installing -- do not interrupt ...
        /etc/puppetlabs/puppet/environments/production/modules
        └── puppetlabs-firewall (v1.6.0)

> That's it! You've just installed the firewall module. You'll need to wait a short time for the Puppet server to refresh before the classes are available to add to your agent nodes.


## Write the `my_firewall` Module

Some modules can be large, complex, and require a significant amount of trial and error. This module, however, will be a very simple module to write: it contains just three classes.

> ### A Quick Note about Module Directories
>
>By default, the modules you use to manage nodes are located in `/etc/puppetlabs/code/environments/production/modules`. This includes modules installed by PE, those that you download from the Forge, and those that you write yourself.
>
>**Note**: PE also installs modules in `/opt/puppetlabs/puppet/modules`, but don’t modify anything in this directory or add modules of your own to it.
>
>There are plenty of resources about modules and the creation of modules that you can reference. Check out [Modules and Manifests](./puppet_modules_manifests.html), the [Beginner's Guide to Modules](/guides/module_guides/bgtm.html), and the [Puppet Forge](https://forge.puppetlabs.com/).

Modules are directory trees. For this task, you'll create the following files:

 - `my_firewall/` (the module name)
   - `manifests/`
     - `pre.pp`
     - `post.pp`
      - `init.pp`

**To write the `my_firewall` module**:

1. From the command line on the Puppet master, navigate to the modules directory: `cd /etc/puppetlabs/code/environments/production/modules`).
2. Run `mkdir -p my_fw/manifests` to create the new module directory and its manifests directory.
3. From the `manifests` directory, use your text editor to create `pre.pp`.
4. Edit `pre.pp` so it contains the following Puppet code:

        class my_firewall::pre {

         # Default firewall rules
         firewall { '000 accept all icmp':
           proto   => 'icmp',
           action  => 'accept',
         }

         firewall { '001 accept all to lo interface':
           proto   => 'all',
           iniface => 'lo',
           action  => 'accept',
         }

         firewall { '002 accept related established rules':
           proto   => 'all',
           state   => ['RELATED', 'ESTABLISHED'],
           action  => 'accept',
         }

         # Allow SSH
         firewall { '100 allow ssh access':
           port   => '22',
           proto  => tcp,
           action => accept,
         }

       }

5. Save and exit the file.
6. From the `manifests` directory, use your text editor to create `post.pp`.
7. Edit `post.pp` so it contains the following Puppet code:

        class my_firewall::post {

          firewall { "999 drop all other requests":
            action => "drop",
          }

        }

8. Save and exit the file.
9. From the `manifests` directory, use your text editor to create `init.pp`.
10. Edit `init.pp` so it contains the following Puppet code:

         class my_firewall {

           stage { 'fw_pre':  before  => Stage['main']; }
           stage { 'fw_post': require => Stage['main']; }

           class { 'my_fw::pre':
             stage => 'fw_pre',
           }

           class { 'my_fw::post':
             stage => 'fw_post',
           }

          resources { "firewall":
             purge => true
          }

        }

11. Save and exit the file.

> That's it! You've written a module that contains a class that, once applied, ensures your firewall has rules in it that will be managed by PE. You'll need to wait a short time for the Puppet server to refresh before the classes are available to add to your agent nodes.
>
> Note the following about your new class:
>
> * `pre.pp` defines the “pre” group rules the firewall applies when a service requests access.
> * `post.pp` defines the rule for the firewall to drop any requests that haven’t met the rules defined by `pre.pp`.
> * `init.pp` applies the previous two classes, as well as telling Puppet when to apply the classes in relation to the `main` stage (which ensures the classes are applied in the correct order).

## Create the firewall_example Group

Groups let you assign classes and variables to many nodes at once. Nodes can belong to many groups and inherit classes and variables from all of them. Groups can also be members of other groups and inherit configuration information from their parent group the same way nodes do. PE automatically creates several groups in the console, which you can read more about in the [PE docs](https://docs.puppetlabs.com/pe/latest/console_classes_groups_preconfigured_groups.html).

In this procedure, you’ll create a simple group called, __firewall_example__, which will contain all of your nodes. Depending on your needs or infrastructure, you may have a different group that you'll assign your firewall class to.

[firewall_add_node]: ./images/quick/firewall_add_node.png

**To create the firewall_example group**:

1. In the console, click __Nodes__ > __Classification__.
2. In the __Node group name__ field, name your group **firewall_example**.
3. Click __Add group__.
4. Select the __firewall_example__ group.
5. From the __Rules__ tab, in the __Node name__ field, enter the name of the PE-managed node you'd like to add to this group.

   >**Warning**: Do not add the Puppet master to this group. Your firewall class does not yet contain rules to allow access to the Puppet master. You will add these rules later when you write a class to [open ports for the Puppet master](#open-ports-for-the-puppet-master).

6. Click __Pin node__.
7. Click __Commit 1 change__.
8. Repeat steps 5-7 for any additional nodes you want to add.

## Use the PE Console to Add the `my_firewall` Class

[classbutton]: ./images/quick/add_class_button.png
[add_firewall]: ./images/quick/firewall_add_firewall.png
[assign_firewall_group]: ./images/quick/assign_my_fw_group.png


**To add the** `my_firewall` **class to the firewall_example group**:

1. In the console, click __Nodes__ > __Classification__.

2. Select the __firewall_example__ group.

3. Click the __Classes__ tab.

4. In the __Class name__ field, begin typing `my_firewall`, and select it from the autocomplete list.

   **Tip**: You only need to add the main `my_firewall` class. It contains the other classes from the module.

5. Click __Add class__.

6. Click __Commit 1 change__.

   **Note**: The `my_firewall` class now appears in the list of classes for the __firewall_example__ group, but it has not yet been configured on your nodes. For that to happen, you need to kick off a Puppet run.

7. When Puppet runs, it will configure the nodes using the newly-assigned classes. Wait one or two minutes.

> Congratulations! You’ve just created a firewall class that you can use to define and enforce firewall rules across your PE-managed infrastructure.

## Open Ports for the Puppet Master

[master_run_report]: ./images/quick/master_log.png
[select_master_log]: ./images/quick/select_master_log.png

The Puppet master is just like any other application you run from your infrastructure: you need to allow special firewall access to ensure you can access the Puppet master correctly.

1. From the command line on the Puppet master, navigate to the modules directory: `cd /etc/puppetlabs/code/environments/production/modules`.
2. Run `mkdir -p my_master/manifests` to create the new module directory and its manifests directory.
3. From the `manifests` directory, use your text editor to create `init.pp`.
4. Edit `init.pp` so it contains the following Puppet code:

        class my_master {
          include my_firewall

         firewall { '100 allow PE Console access':
           port   => '443',
           proto  => tcp,
           action => accept,
         }

         firewall { '100 allow Puppet master access':
           port   => '8140',
           proto  => tcp,
           action => accept,
         }

         firewall { '100 allow ActiveMQ MCollective access':
           port   => '61613',
           proto  => tcp,
           action => accept,
         }

       }

5. Using the console, add `my_master` class to the Puppet master.
6. Add the Puppet master to the __firewall_example__ group.
7. After Puppet runs, navigate to the **Reports** page, and click the latest run report for your Puppet master.

8. Click the __Events__ tab.

   You will see three events on this node, indicating that your firewall now allows access to the MCollective, the PE console, and the Puppet master.

   >**Tip**: For all firewall configuration needs for your PE installation, refer to the [system requirements documentation](./install_system_requirements.html#firewall-configuration).

> Next, let’s take a look at how PE will manage any configuration drift for these rules by enforcing the desired state of the `my_firewall` class.

## Enforce the Desired State of the `my_firewall` Class

Finally, let's take a look at how PE ensures the desired state of the `my_firewall` class on your agent nodes. In the previous task, you applied your firewall class. Now imagine a scenario where a member of your team changes the contents of the `iptables` to allow connections on a random port that was not specified in `my_firewall`.

1. Select an agent node on which you applied the `my_firewall` class, run `iptables --list`.
2. Note that the rules from the `my_firewall` class have been applied.
3. From the command line, insert a new rule to allow connections to port **8449** by running `iptables -I INPUT -m state --state NEW -m tcp -p tcp --dport 8449 -j ACCEPT`.
4. Run `iptables --list` again and note this new rule is now listed.
5. Puppet runs on the agent node.
6. Run `iptables --list` on that node once more, and notice that PE has enforced the desired state you specified for the firewall rules.

> That's it---PE has enforced the desired state of your agent node!

## Other Resources

The Puppet Labs Firewall module (`puppetlabs-firewall`), is part of the PE [supported modules](http://forge.puppetlabs.com/supported) program; these modules are supported, tested, and maintained by Puppet Labs. You can learn more about the Puppet Labs Firewall module by visiting [the Puppet Forge](http://forge.puppetlabs.com/puppetlabs/firewall).

Check out the other quick start guides in our PE QSG series:

- [NTP Quick Start Guide](./quick_start_ntp.html)
- [SSH Quick Start Guide](./quick_start_ssh.html)
- [DNS Quick Start Guide](./quick_start_dns.html)
- [Sudo Users Quick Start Guide](./quick_start_firewall.html)

Puppet Labs offers many opportunities for learning and training, from formal certification courses to guided online lessons. We've noted a few below; head over to the [learning Puppet page](https://puppetlabs.com/learn) to discover more.

* [Learning Puppet](http://docs.puppetlabs.com/learning/) is a series of exercises on various core topics about deploying and using PE.
* The Puppet Labs workshop contains a series of self-paced, online lessons that cover a variety of topics on Puppet basics. You can sign up at the [learning page](https://puppetlabs.com/learn).






