===========================================
Creating Customized Ubuntu-14.04 Image
===========================================

This tutorial will give you step-by-step instructions on how to build a Ubuntu-14.04.5-amd64 guest image to run targeted programs that for some reason can only run on recent Linux distributions. The same steps should work also on Ubuntu 16.04 and Ubuntu 18.04 releases. 

.. warning::
    This tutorial includes some nasty hacks of the Makefile, but should work as is, if followed closely. For some steps, you need to have a GUI-enabled system, but after the image is ready, it will be portable, and can be used on any non-GUI server machines. 
.. contents::

Installation ISO Preparation
============================

Download a Ubuntu iso, and patches it with unattended installation scripts. A quick way to do this is to use this git repo. 

.. code-block:: bash

    git clone https://github.com/netson/ubuntu-unattended.git
    cd ubuntu-unattended
    sudo ./ create-unattended-iso.sh

When creating the unattended iso, it is important to create the user with username `s2e` and preferred password `s2e`. It will save you a lot of time in the future. The created images will be in your `$HOME` directory. 


Building Ubuntu images
======================
To build the ubuntu image, we need to add a target image type in the `$HOME/s2e/source/guest-images/images.json`. Append these lines after the debian targets. 

.. code-block:: json

    "ubuntu-14.04.5-x86_64": {
           "name": "Ubuntu x86_64 image",
           "image_group": "linux",
           "iso": {
             "url": "ubuntu-14.04.5-amd64.iso"
           },
           "os": {
             "name": "ubuntu",
             "version": "14.04.5",
             "arch": "x86_64",
             "build": "",
             "binary_formats": ["elf"]
           },
           "hw": {
             "default_disk_size": "40G",
             "default_snapshot_size": "256M",
             "nic": "e1000"
           }
        },

We also need to modify the `$HOME/s2e/source/guest-images/Makefile.linux` to add the ubuntu target. Append the ubuntu target to line `180`:

.. code-block:: bash

    LINUX_IMAGES = debian-9.2.1-i386 debian-9.2.1-x86_64 cgc_debian-9.2.1-i386 ubuntu-14.04.5-x86_64

Add the following after line `203`:

.. code-block:: bash

    $(eval $(call BUILD_S2E_IMAGE,ubuntu-14.04.5-x86_64,,linux-4.9.3-x86_64))
    $(eval $(call TAKE_SNAPSHOT,ubuntu-14.04.5-x86_64,))

Run the image build scripts now, 

.. code-block:: bash

    sudo chmod ugo+r /boot/vmlin*
    s2e image_build ubuntu-14.04.5-x86_64 --iso-dir ~/ --gui

If any error occured during the installation, it's probably because the naming of the iso file, try manually copy the iso file into the image installation directory, and re-run the installation.

.. code-block:: bash

    cp ~/ubuntu-14.04.5-server-amd64-unattended.iso ~/s2e/images/.tmp-output/ubuntu-14.04.5-x86_64/ubuntu-14.04.5-x86_64.iso

If you observe that the QEMU window has been launched, but nothing are displayed, it's because the image customization scripts of `s2e` doesn't work properly for Ubuntu iso. But since we already got the unattended Ubuntu iso, we can manually copy the iso to override the `install_files.iso` in the image installation directory, and re-run the installation:

.. code-block:: bash

    cp ~/ubuntu-14.04.5-server-amd64-unattended.iso  ~/s2e/images/.tmp-output/ubuntu-14.04.5-x86_64/install_files.iso


Installing Kernel and Requirements
==================================
When the unattended installation of Ubuntu finishes, and you see the QEMU window shows the login after reboot, login the `s2e` account and do the following:

.. code-block:: bash

    sudo apt-get -y install gcc-multilib g++-multilib git make gettext libdw-dev
    git clone git://sourceware.org/git/systemtap.git
    cd systemtap
    git checkout release-3.2
    cd ..
    mkdir systemtap-build
    cd systemtap-build
    ../systemtap/configure --disable-docs
    make -j2
    sudo make install
    cd ..

And then, install the s2e Linux kernel:

.. code-block:: bash

    sudo dpkg -i *.deb

    MENU_ENTRY="$(grep menuentry /boot/grub/grub.cfg  | grep s2e | cut -d "'" -f 2 | head -n 1)"
    echo "Default menu entry: $MENU_ENTRY"
    echo "GRUB_DEFAULT=\"1>$MENU_ENTRY\"" | sudo tee -a /etc/default/grub
    sudo update-grub

One last step is to configure the `s2e` user with auto-login, and allow it to run `sudo` without being prompted for password. First, run `sudo visudo`, and add this line to the file: 

.. code-block:: bash

    s2e ALL=(ALL) NOPASSWD: ALL

Second, edit the configure file `/etc/init/tty1.conf`, and append one line at the end:

.. code-block:: bash

    exec /sbin/getty -8 38400 tty1 -a "s2e"

Now, reboot and enjoy the new Ubuntu-14.04 guest image. 


SSH into Guest Image
====================

If you run `s2e` on the server without GUI, you may struggle with not able to open a shell in the guest to install some prerequisite onto the guest VM. You can solve this by installing `openssh-server` on the Ubuntu guest, and then starting the QEMU guest VM by enabling port forwarding. 

.. code-block:: bash

    qemu-system-x86_64 --enable-kvm -m 4096 -smp 4 -drive format=raw,file=$HOME/s2e/images/ubuntu-14.04.5-x86_64/image.raw.s2e --nographic -new user,hostfwd=tcp::10022-:22 -net nic

Now, you can `ssh` into guest with

.. code-block:: bash

    ssh s2e@localhost -p 10022

Enjoy! 