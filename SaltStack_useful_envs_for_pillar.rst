DRAFT - SaltStack: Useful Environments for Pillar
=================================================

Currently (2015-05) environments for pillar are pretty useless.
You can specify them in your master's config under `pillar_roots`
but they're all thrown together anyway.

Idea #1: Restricting States to Pillar-Envs
------------------------------------------
An idea I just had: What if we could restrict the pillar-
environment a state sees?

If in your states topfile you could add an option 
"pillar_env: foobar"?

Not that I can think of an actual usecase for this but
maybe it's a start of turning the pillar envirnments
into something useful.

Idea #2: Limiting the Keys a Pillar-Env can set
-----------------------------------------------

Giving someone access to your pillar is dangerous. Kinda obvious.
But imagine a well meaning web designer discovering that he can
not only enter data for the webapp and his own ssh-key to log 
into the production server - say for looking at logfiles - but
he can also add his ssh-key for root and make a quick "fix" to 
the webserver's config.
Why bother those grumpy sysadmins when you can fix it yourself, 
right?

No let's say we could do something like this::

    pillar_roots:
        base:
            - /srv/salt/pillar
        webteam:
            - limit_keys:
                - fancyWebApp
                - users:joe:ssh-keys
                - users:jane:ssh-keys
            - /srv/salt/webteam_pillar
            - /srv/salt/webapp_defaults

Jane and Joe could enter whatever they want but if they're data 
isn't below the specified keys it's not included in the minion's 
pillar.

I think I've proposed something with a similar effect to someone 
on salt-users or a colleague: Let some less privileged (in terms 
of permissions on your systems) user edit some yaml file. The 
content is then loaded under a certain pillar key so whatever
they enter doesn't mess up stuff under, say, `pillar[ssh-server]`.
But the needed templating isn't trivial for someone still new to
templating pillar.

Still there are cases like syntax errors [1]_ that may break
a minion's pillar completly.

.. [1] Try entering a date like 2014-04-01 without quoting into
    pillar - Msgpack exception, "Can't serialize datetime object"
    or something like that. Causes the whole pillar of the 
    affected minion to turn into a string and thus become useless
    because the minion expects a dictionary.
