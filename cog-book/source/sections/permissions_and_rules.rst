Permissions and Rules
=====================

The Cog chatbot system comes equipped with a comprehensive and flexible
authorization system which allows operators fine-grained control over
who is able to execute chat commands, extending even to control over
particular *invocations* of chat commands.

In this document, we will discuss the individual pieces of the
authorization system and take a look at how it is used in practice.

Authorization Rules
-------------------

At the core of Cog’s authorization system are **rules**. Each time a
user issues a chat command to Cog, rules governing the execution of that
command are looked up and applied to the current invocation. If a match
is found, the list of permissions the invoking user has is consulted to
see if it includes the permission(s) required in the matching rule. If
it does, the command is executed; if not, command processing immediately
stops.

To make things concrete, we’ll start with a simple authorization rule.
(There is actually a separate rule language that can be used to make
rather complex rules, but for now we’ll start simple. Feel free to read
more :doc:`detailed explanations <command_execution_rules>` of how rules
are formed. In any event, the details of the language are orthogonal to
the authorization system itself.)

::

  when command is operable:bundle must have operable:manage_commands

This is the simplest form of rule, and gives us all we need to discuss
the authorization system. This rule states, in English, that for a user
to execute the ``operable:bundles`` chat command (which allows users to
enable or disable entire bundles of commands at once), they must have
the ``operable:manage_commands`` permission granted to them.

With that rule in place, let’s say I type the following command
invocation in my chat application:

::

  !operable:bundle disable github

I’m telling Cog to disable all the commands in the ``github`` bundle. In order for that command to
be executed, Cog must verify, according to the rule above, that I have
the ``operable:manage_commands`` permission. It just so happens that I
do, so the command succeeds; now nobody can check how many pull requests
are open on their favorite repository.

Perhaps you want to restrict the ability to disable a particularly
important bundle; perhaps you’ve written one called ``prod`` to help
manage your organization’s production environment. We can add this with
a new rule that matches the invocation

::

  !operable:bundle disable prod

That rule might look like this:

::

  when command is operable:bundle
    with arg[0] == "disable"
      and arg[1] == "prod"
  must have site:manage_prod and operable:manage_commands

Here we can see a rule that applies to a very specific invocation of a
command. If you have the ``operable:manage_commands`` permission, you
can manipulate bundles in general, but in order to disable the ``prod``
bundle, you must have the additional ``site:manage_prod`` permission.

As you can imagine, the ability to define rules like this offers a lot
of power. We’ll talk more in depth about rules later; the remainder of
this document will delve into the specifics of the authorization system
itself, explaining the its components and how they all work together.

Rules can be viewed, created, and deleted using the ``operable:rules``
command. In particular, simple rules of the form
``when command is <COMMAND> must have <PERMISSION>`` can be created
thusly:

\`\`\` !operable:rule create <COMMAND> <PERMISSION> \`\`\` Note that
both ``<COMMAND>`` and ``<PERMISSION>`` must exist, and be typed as
fully-qualified names.

.. _components_of_the_authorization_system: 

Components of the Authorization System
--------------------------------------

Permissions
~~~~~~~~~~~

Permissions are at the base of Cog’s authorization system. They act as a
kind of token; you can carry out certain actions if you possess the
correct token(s).

You will notice that permissions have a structure:
``operable:manage_commands``, ``site:manage_prod``, etc. Permissions are
*namespaced*; here we have a ``manage_commands`` permission in the
``operable`` namespace, and a ``manage_prod`` permission in the ``site``
namespace. In general, every bundle of commands defines its own
permission namespace. This allows bundle authors the flexibility to
define permissions that are used by commands in the bundle without
worrying about conflicting with permissions from any other bundles that
happen to be installed on a Cog system. We can have an
``operable:manage_commands`` permission as well as a
``site:manage_commands`` permission without any problems.

There are two ways that permissions can be created. The first is through
bundle installation. All command bundles have the option to define
permissions and authorization rules to help bootstrap the bundle in a
Cog system. The operator is not under any obligation to use these rules
or permissions, and is free to define their own, but they are always
installed with the bundle.

The second way that permissions can be created is directly by the Cog
operator. This is where the special ``site`` namespace comes into play.
``site`` is unique; it is the only permission namespace that is *not*
associated with a corresponding command bundle. All permissions created
by operators are part of the ``site`` namespace. It is the mechanism by
which the permission scheme may be customized to the needs of the
operator’s unique environment and use cases.

Roles
~~~~~

Moving up from permissions, we arrive at roles, which are collections of
associated permissions. While permissions can be created when you
install command bundles, roles are something purely under your control
as a Cog operator; you create them and you manage them.

Users
~~~~~

In order for permissions to be useful, we have to have a way to
associate them with people invoking commands. The Cog **user** is the
unique identity to which **permissions** are ultimately attached.

Each person that can interact with Cog has an associated **user**
account. This is also the identity with which a person will interact
with Cog’s REST API.

It is important to understand that this "Cog User" is *not the same* as
a person’s "handle" in a particular chat system. In fact, a Cog user can
be associated with multiple handles from different chat systems. For
instance, I may be ``@cm`` in HipChat and ``@chris`` in Slack, but
``cmaier`` in Cog. Cog can recognize this and map these various chat
handles back to the same Cog user, allowing authorization to be managed
centrally and independently of which chat system is in use.

Users are scoped to the entire Cog installation; that is, there is no
higher-level namespace (e.g., "organization") into which users are
grouped.

Groups
~~~~~~

Finally, Cog groups collect Cog users together. Any number of users may
be in a group, but *only* users may be members of groups.

Bringing It All Together
------------------------

Now that you know about permissions, roles, users, and groups, how do
you use them?

We know that roles are collections of permissions, and groups are
collections of users, but that ultimately, somehow, permissions become
associated with users. This missing link here is that roles can be
granted to groups.

Thus, a user has all the permissions in all the roles granted to all the
groups of which she is a member.

To grant a permission to a user, then, the user must be placed into a
group that has been granted a role that contains that permission. While
this might seem a bit cumbersome from the perspective of a single user
and a single permission, it makes global management easier; it frees you
to think in terms of the higher-level constructs of roles and groups,
without having to worry about "exceptions to the rule" like individual
users being directly granted a permission, or potentially complicated
group hierarchies.

.. warning:: For those that have used Cog before version 0.4, this document
    describes a departure from the previous permission scheme, where
    users and groups could be granted permissions directly, and groups
    could also contain groups.

As an example, let’s look at how we might set up a Cog system to grant
permissions for the `mist <https://github.com/cog-bundles/mist>`__ EC2
command bundle. For this demonstration, let’s say we have three users:
Alice, Bob, and Charlie. Furthermore, let’s say that Alice is on our
Operations team, while Bob and Charlie are on the Development team.
Let’s also stipulate that everyone on the operations team should be able
to perform any action with Mist, while developers start out with
read-only permissions.

Looking at Mist’s bundle
`configuration <https://github.com/cog-bundles/mist/blob/cf44151121cbe98f9819b6925d9f2246ad422ebd/config.yaml#L58-L64>`__,
we see it declares the following permissions:

-  mist:view

-  mist:change-state

-  mist:destroy

-  mist:create

-  mist:manage-tags

-  mist:change-acl

It looks like we’ll want to give operations folks all of these
permissions, and developers only ``mist:view``. Let’s set up some roles
to express this.

First a ``mist_admin`` role, with all the mist permissions:

.. code:: Shell

    cogctl roles create mist_admin
    cogctl permissions grant mist:view --role=mist_admin
    cogctl permissions grant mist:change_state --role=mist_admin
    cogctl permissions grant mist:destroy --role=mist_admin
    cogctl permissions grant mist:create --role=mist_admin
    cogctl permissions grant mist:manage-tags --role=mist_admin
    cogctl permissions grant mist:change-acl --role=mist_admin

And now, a ``mist_read_only`` role:

.. code:: Shell

    cogctl roles create mist_read_only
    cogctl permissions grant mist:view --role=mist_read_only

Now we have our roles, but we have nothing to grant them to. Let’s
create some groups.

.. code:: Shell

    cogctl groups create operations
    cogctl groups create developers

Now let’s grant the roles to our new groups.

.. code:: Shell

    cogctl roles grant mist_admin --group=operations
    cogctl roles grant mist_read_only --group=developers

We’re almost there. We have the groundwork laid; all that remains is to
add our users.

.. code:: Shell

    cogctl groups add operations --email=alice@mycompany.com
    cogctl groups add developers --email=bob@mycompany.com
    cogctl groups add developers --email=charlie@mycompany.com

Any changes to the permission structure take effect immediately. If the
``mist:view`` permission is removed from the ``mist_read_only`` role,
Bob and Charlie immediately lose the ability to run commands that
require that permission (unless they happen to also be members of
another group that has the permission via some other role). Similarly,
if Danielle is added to the ``operations`` group, she immediately has
all the ``mist`` permissions.

Note also that all authorization rules are written in terms of
*permissions*, and not *roles*,
