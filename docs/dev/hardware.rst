Adding support for new hardware
===============================
This page will give a short overview on how to add support
for new hardware to Gluon.

Hardware requirements
---------------------
Having an ath9k (or ath10k) based WLAN adapter is highly recommended,
although other chipsets may also work. VAP (multiple SSID) support
is a requirement. At the moment, Gluon's scripts can't handle devices
without WLAN adapters (although such environments may also be interesting,
e.g. for automated testing in virtual machines).

.. _hardware-adding-profiles:

Adding profiles
---------------
The vast majority of devices with ath9k WLAN uses the ar71xx target of OpenWrt.
If the hardware you want to add support for is also ar71xx, adding a new profile
is enough.

Profiles are defined in ``targets/<target>-<subtarget>/profiles.mk``. There are two macros
used to define which images are generated: ``GluonProfile`` and ``GluonModel``. The following examples
are taken from ``profiles.mk`` of the ``ar71xx-generic`` target::

    $(eval $(call GluonProfile,TLWR1043))
    $(eval $(call GluonModel,TLWR1043,tl-wr1043nd-v1-squashfs,tp-link-tl-wr1043n-nd-v1))
    $(eval $(call GluonModel,TLWR1043,tl-wr1043nd-v2-squashfs,tp-link-tl-wr1043n-nd-v2))

The ``GluonProfile`` macro takes at least one parameter, the profile name as it is
defined in the Makefiles of OpenWrt (``openwrt/target/linux/<target>/<subtarget>/profiles/*``
and ``openwrt/target/linux/<target>/image/Makefile``). If the target you are on doesn't define
profiles (e.g. on x86), just add a single profile called ``Generic`` or similar.

It may optionally take a second parameter which defines additional packages to include for the profile
(e.g. ath10k). The additional packages defined in ``openwrt/target/linux/<target>/<subtarget>/profiles/*``
aren't used.

The ``GluonModel`` macro takes three parameters: The profile name, the suffix of the image file
generated by OpenWrt (without the file extension), and the final image name of the Gluon image.
The final image name must be the same that is returned by the following command.

::

    lua -e 'print(require("platform_info").get_image_name())'


This is just so the autoupdater can work. The command has to be executed _on_ the target (eg. the hardware router with a flashed image). So you'll first have to build an image with a guessed name, and afterwards build a new, correctly named image. On targets which aren't supported by the autoupdater,
``require("platform_info").get_image_name()`` will just return ``nil`` and the final image name
may be defined arbitrarily.

On devices with multiple WLAN adapters, care must also be taken that the primary MAC address is
configured correctly. ``/lib/gluon/core/sysconfig/primary_mac`` should contain the MAC address which
can be found on a label on most hardware; if it does not, ``/lib/gluon/upgrade/010-primary-mac``
in ``gluon-core`` might need a fix. (There have also been cases in which the address was incorrect
even on devices with only one WLAN adapter, in these cases an OpenWrt bug was the cause).

Adding support for new hardware targets
---------------------------------------
Adding a new target is much more complex than adding a new profile. There are two basic steps
required for adding a new target:

Adjust packages
'''''''''''''''
One package that definitely needs adjustments for every new target added is ``libplatforminfo`` (to be found in
`packages/gluon/libs/libplatforminfo <https://github.com/freifunk-gluon/packages/tree/master/libs/libplatforminfo>`_).
Start with a copy of an existing platform info source file and adjust it for the new target (or just add a symlink if
there is nothing to adjust). Then add the new target to the DEPENDS variable in the ``libplatforminfo`` package Makefile.

On many targets, Gluon's network setup scripts (mainly in the packages ``gluon-core`` and ``gluon-mesh-batman-adv-core``)
won't run correctly without some adjustments, so better double check that everything is fine there (and the files
``primary_mac``, ``lan_ifname`` and ``wan_ifname`` in ``/lib/gluon/core/sysconfig/`` contain sensible values).

Add support to the build system
'''''''''''''''''''''''''''''''
A directory for the new target must be created under ``targets``, and it must be added
to ``targets/targets.mk``. In the new target directory, the following files must be created:

* profiles.mk
* config (optional)

For ``profiles.mk``, see :ref:`hardware-adding-profiles`.
The file ``config`` can be used to add additional, target-specific options to the OpenWrt config.

After this, is should be sufficient to call ``make GLUON_TARGET=<target>`` to build the images for the new target.
