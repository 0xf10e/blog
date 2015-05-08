DRAFT - SaltStack: Aggregated Pillar
====================================

As I mentioned earlier I first thought the "ext_pillar_first"
option would make the `salt-master` parse the external pillars
first. Thus  they would be available during parsing of normal
pillar (file based, from `pillar_roots`) and could be used for
assignment in the topfile and templating.

Make ext_pillar available in pillar-topfile
-------------------------------------------
Like I wrote above: External pillars would be parsed first
and one could use the minion-specific data during redering
of the minions pillar.

Aggregate pillar during parsing
-------------------------------
Like in a script. Assign value, use later.

Aggregate pillar-data while going throught the topfile.

Why not a runner?
-----------------

Runners are modules on the master.

Pillars are rendered on the master.

Why not use a runner to "seed" your pillar with roles and stuff?
