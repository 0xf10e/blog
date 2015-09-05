Notes on ``iocage``
===================

`iocage website`_ and `iocage on Github`_. 
There's also an interview with it's
creator `Peter Toth`_ on `BSDnow Episode 102`_.

.. _iocage website: https://pannon.github.io/iocage/
.. _iocage on Github: https://github.com/iocage/iocage
.. _Peter Toth: https://twitter.com/pannonp
.. _BSDnow Episode 102:
    http://www.bsdnow.tv/episodes/2015_08_12-may_contain_zfs

  * Salt exec/state module? -> `bougie`_'s `salt-iocage`_
    looks good

  * Nested jails? -> set ``children.max`` > 0 (see
    `jail(8)`_), run iocage inside jail

  * pass dataset to jail with ``jail_zfs=on`` and
    ``jail_zfs_dataset=???`` (see `iocage(8)`_)

.. _bougie: https://github.com/bougie
.. _salt-iocage: https://github.com/bougie/salt-iocage
.. _jail(8):
  https://www.freebsd.org/cgi/man.cgi?query=jail&apropos=0&sektion=8
.. _iocage(8): 
  https://www.freebsd.org/cgi/man.cgi?query=iocage&manpath=FreeBSD+Ports+10.2-RELEASE
