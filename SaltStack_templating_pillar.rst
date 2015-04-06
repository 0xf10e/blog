SaltStack: Templating Pillar
============================

As I've mentioned a few times on the `salt users mailing list`_ I
like to template my minion pillar data. When you're using SaltStack
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
.. _nice workaroung: 
    https://github.com/saltstack/salt/issues/11350#issuecomment-38340122

(Sorry for the rambliness, it's 23:45...)

What
----

So, here's an example:
 
 - You have a bunch of servers with a public and a private IP each
 - SSH and some database are supposed to listen only on the private IP,
   the webserver's virtual host for the company website on the public one
 - You don't want to write *all* your states and config templates yourself
   so you're using some formulas_, but they expect the particular IP in
   predefined places in pillar

     * The SSH-formula wants it's ListenAddress under 
       `pillar[sshd_config:ListenAdress]`
     * The database-formula expects 
       `pillar[database:server:daemon:listen_address]`
     * And your webserver's formula goes with a dictionary of domains
       mapped to another dict with settings for this virtual host
       resulting in `pillar[httpserver:vhosts:example.com:address]`
       for the corresponding IP.
     * Oh, yeah, and you want the internal IP on interface `eth0`
       and the external IP on interface `eth1` so you need to add
       them under `pillar[interfaces:eth0:ipv4]` and 
       `pillar[interfaces:eth1:ipv4`.

Luckily all the hosts are in the same subnets so the default gateway is
the same for all of them.

(And just BTW: I'm making those up with some quick looks into some `formulas
on GitHub`_)

The resulting pillar of minion A should look like this::

    sshd_config:
        ListenAddress: 192.168.2.21
    database:
        server:
            daemon:
                listen_address: 192.168.2.21
    httpserver:
        vhosts:
            example.com:
                address: 203.aaa.bbb.137
    interfaces:
        eth0:
            ipv4: 192.168.2.21/24
        eth1:
            ipv4: 203.aaa.bbb.137/26
            gateway: 203.aaa.bbb.129

Now imagine writing this down 10 or 20 or 30 times only changing two 
parameters. Sounds like fun, right?

How (simple)
------------

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

`everything.sls`::
    
    {% set int_ip = int_cidr.split('/')[0] %}
    {% set ext_ip = ext_cidr.split('/')[0] %}
    sshd_config:
        ListenAddress: {{ int_ip }}
    database:
        server:
            daemon:
                listen_address: {{ int_ip }}
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
    {% endif %}
    {% if 'database' in roles %}
        - webserver:
            defaults:
                listen_address: {{ int_ip }}
    {% endif %}

You can probably guess how all those tiny templates we include here will
look like.

But WHY??
---------
So I've showed you a hack to decide about the data to put into pillar
based on pillar before you can access pillar. Not nice, overly complicated
and, guess what, somewhat obsolete [2]_.

But you can keep all of your decisions about which minion sees what
of your data inside pillar and thus on the master.

Coming to the "obsolete" part: There are `External Pillars`_ and the 
option `ext_pillar_first`_. 

.. _external pillars: 
    http://docs.saltstack.com/en/latest/ref/configuration/master.html#ext-pillar
.. _`ext_pillar_first`:
    http://docs.saltstack.com/en/latest/ref/configuration/master.html#ext-pillar-first

.. [2] Which of course does mean I have to quite a bit of cleaning up and
        simplifying things...
