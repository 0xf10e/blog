SaltStack: Templating Pillar
============================

As I've mentioned a few times on the `salt users mailing list`_ I
like to template my minions' pillar data. When you're using SaltStack
you may already have stumbled over the problem that you can't use
pillar inside pillar, i.e. use pillar based targeting in you pillar's
`top.sls`.

I really don't like having to enter configuration data of my minions
in like four different places. Especially not if it's the same stuff
over and over again because you got 50+ of those hosts and the difference
is two lines in a few pages. 

I fiddled with Jinja and had some rather `strange behavior`_ but `Seth 
House`_ came up with a `nice workaround`_.
    
.. _strange behavior: https://github.com/saltstack/salt/issues/11350
.. _Seth House: https://github.com/whiteinge
.. _nice workaround: 
    https://github.com/saltstack/salt/issues/11350#issuecomment-38340122
.. _salt users mailing list:

(Sorry for the rambliness, I started writing this at 23:45 and still 
don't feel like rewriting everything...)

What
----

So, here's an example:
 
 - You have a bunch of servers with a public and a private IP each
 - SSH and some database are supposed to listen only on the private IP,
   the webserver's virtual host for the company website on the public one
 - The webapp running on the webserver needs the database credentials  of the user
   which needs to be created on the database server
 - You don't want to write *all* your states and config templates yourself
   so you're using some formulas_, but they expect the particular values in
   predefined places in pillar:

     * The SSH-formula wants it's ListenAddress under 
       `pillar[sshd_config:ListenAdress]`
     * The database-formula expects to get the listen address from
       `pillar[database:server:daemon:listen_address]` and users
       to create (and passwords for them) under
       `pillar[database:user:<username>:password:<password>]`
     * Your webserver's formula goes with a dictionary of domains
       mapped to another dict with settings for this virtual host
       resulting in `pillar[httpserver:vhosts:<domain>:<address>]`
       for the corresponding IP.
     * The webapp wants its database credentials in
       `pillar[webapp:db_user]` and `pillar[webapp:db_pass]`
     * Oh, yeah, and you want the internal IP on interface `eth0`
       and the external IP on interface `eth1` so you need to add
       them under `pillar[interfaces:eth0:ipv4]` and 
       `pillar[interfaces:eth1:ipv4]` [0]_.

.. [0] That's actually the style I used in the openvswitch-formula_

Luckily all the hosts are in the same subnets so the default gateway is
the same for all of them.

(And just BTW: I'm making most of those pillar keys up  - with some quick 
looks into some `formulas on GitHub`_)

The resulting pillar of minion A should look like this::

    sshd_config:
        ListenAddress: 192.168.2.21
    database:
        server:
            daemon:
                listen_address: 192.168.2.21
        user:
            webapp-user:
                password: dgaskjgnasgn
    httpserver:
        vhosts:
            example.com:
                address: 203.aaa.bbb.137
    webapp:
        db_user: webapp-user
        db_pass: dgaskjgnasgn
    interfaces:
        eth0:
            ipv4: 192.168.2.21/24
        eth1:
            ipv4: 203.aaa.bbb.137/26
            gateway: 203.aaa.bbb.129

Now imagine writing this down 10 or 20 or 30 times only changing two 
parameters. Sounds like fun, right?

.. _formulas: 
  http://docs.saltstack.com/en/latest/topics/development/conventions/formulas.html
.. _openvswitch-formula: https://github.com/saltstack-formulas/openvswitch-formula
.. _formulas on GitHub:
  https://github.com/saltstack-formulas

How to (simple)
---------------

Those 16 lines? Just use one template for all of them, right?

`top.sls`::

    base:
        '*':
            - hosts.{{ opts.id }}

`hosts/minion_a`::

    include:
        - everything:
            defaults:
                int_cidr: 192.168.2.21/24
                ext_cidr: 203.aaa.bbb.137/26
                db_user: webapp-user
                db_pass: dgaskjgnasgn

`everything.sls`::
    
    {% set int_ip = int_cidr.split('/')[0] %}
    {% set ext_ip = ext_cidr.split('/')[0] %}
    sshd_config:
        ListenAddress: {{ int_ip }}
    database:
        server:
            daemon:
                listen_address: {{ int_ip }}
        user:
            {{ db_user }}:
                password: {{ db_pass }}
    webapp:
        db_user: {{ db_user }}
        db_pass: "{{ db_pass }}"
    httpserver:
        vhosts:
            example.com:
                address: {{ ext_ip }}
    interfaces:
        eth0:
            ipv4: {{ int_cidr }}
        eth1:
            ipv4: {{ ext_cidr }}
            gateway: 203.aaa.bbb.129

Remember the minion sees the *result* of the templating.
So can in fact still target "the minion that get's told to
make its sshd listen on 192.168.2.21" with this::

    salt -I sshd_config:ListenAddress:192.168.2.12 test.ping

How (more complicated than it needs to be)
------------------------------------------
Now I'll make this a little more complicated than it has to be to include the
pillar-based-role-thing. Just ignore the fact that all of our fictional minions
have both the role "webserver" and "database" ;)

So the topfile stays the same. Our minion's `minion_a.sls` only changes slightly::

    include:
        - everything:
            defaults:
                int_cidr: 192.168.2.21/24
                ext_cidr: 203.aaa.bbb.137/26
                db_user: webapp-user
                db_pass: dgaskjgnasgn
                roles:
                    - webserver
                    - database

The `everything.sls` get's a bit more involved as we have to include stuff base
on the elements of the passed list `roles`::

    {% set int_ip = int_cidr.split('/')[0] %}
    {% set ext_ip = ext_cidr.split('/')[0] %}
    include:
        - ssh:
            defaults:
                listen_address: {{ int_ip }}
        - interfaces:
            defaults:
                eth0_cidr: {{ int_cidr }}
                eth1_cidr: {{ ext_cidr }}
    {% if 'webserver' in roles %}
        - webserver:
            defaults:
                vhost_ip: {{ ext_ip }}
                db_user: {{ db_user }}
                db_pass: "{{ db_pass }}"
    {% endif %}
    {% if 'database' in roles %}
        - database: 
            defaults:
                listen_address: {{ int_ip }}
                db_user: {{ db_user }}
                db_pass: {{ db_pass }}
    {% endif %}

You can probably guess how all those tiny templates we include here will
look like.

But WHY??
---------
So I've showed you a hack to decide about the data to put into pillar
based on pillar before you can access pillar. Not nice, overly complicated [1]_
and, guess what, it may become obsolete.

But you can keep all of your decisions about which minion sees what
of your data inside pillar and thus on the master.

.. note:: The following paragraph is based on a misunderstanding on my side.
    `ext_pillar_first`_ causes to external pillars to be *queried* first
    but they're still not available during pillar-rendering.

Coming to the "may become obsolete" part: There are `External Pillars`_ and the 
option `ext_pillar_first`_. If the external pillars would be available
when the master starts parsing the pillar topfile we could define the 
minions' roles in the external pillar *and use those roles in the topfile*.
Then it would just be "ext_pillar says your the webserver, give the webapp
this password for the database" and we wouldn't need all this templating.

Simplest way would be a "cmd_yaml" external pillar grepping the roles
from file with a name equal to the minion's id::

    ext_pillar_first: True
    ext_pillar:
        - cmd_yaml: grep roles /srv/pillar/{minion_id}.sls

As the path suggest this would be like "please, salt-master, read this 
part of the minion's pillar first, you'll need the data in a minute".

To bad this doesn't work [2]_ - yet?

.. _external pillars: 
    http://docs.saltstack.com/en/latest/ref/configuration/master.html#ext-pillar
.. _`ext_pillar_first`:
    http://docs.saltstack.com/en/latest/ref/configuration/master.html#ext-pillar-first

.. [1] Maybe not in this example but any real setup will be.
.. [2] See `pull-request 22461`_ "Use 'minion_id' in cmd_{yaml{,ex},json} 
    ext_pillar functions" on GitHub
.. _pull-request 22461: https://github.com/saltstack/salt/pull/22461

