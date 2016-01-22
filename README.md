# Alpine-based AppVMs for QEMU

This is a playbook that helps you create AppVMs using Alpine Linux. Right now,
only QEMU is supported with SPICE enabled, but support for other environments
may be added in the future.


## Getting Started (QEMU without libvirt)

The following instructions assume your host machine is running Arch Linux and
that you wish to run QEMU without using libvirt. Support for other
distributions may be added in the future.

Download and install the basic prerequisites:
~~~
$ sudo pacman -S ansible spice qemu python2 sshpass
~~~

Add a record for "appvm-browser" to the end of your "~/.ssh/config" file
~~~
...

Host appvm-browser
HostName localhost
User root
Port 10022
~~~

Download the Alpine Linux 64-bit ISO and verify its signature:
~~~
$ wget http://alpinelinux.org/keys/ncopa.asc 
$ gpg --import ncopa.asc 
gpg: key 07D9495A: public key "Natanael Copa <ncopa@alpinelinux.org>" imported
gpg: Total number processed: 1
gpg:               imported: 1

$ wget http://wiki.alpinelinux.org/cgi-bin/dl.cgi/v3.2/releases/x86_64/alpine-3.2.3-x86_64.iso 
$ wget http://wiki.alpinelinux.org/cgi-bin/dl.cgi/v3.2/releases/x86_64/alpine-3.2.3-x86_64.iso.asc
$ gpg --verify alpine-3.2.3-x86_64.iso.asc 
gpg: assuming signed data in 'alpine-3.2.3-x86_64.iso'
gpg: Signature made Thu 13 Aug 2015 03:21:51 PM UTC using RSA key ID 07D9495A
gpg: Good signature from "Natanael Copa <ncopa@alpinelinux.org>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: 0482 D840 22F5 2DF1 C4E7  CD43 293A CD09 07D9 495A
~~~

Clone the alpine-appvm.git repo and navigate to it:
~~~
$ git clone https://github.com/mutantmonkey/alpine-appvm.git
$ cd ./alpine-appvm
~~~

Create QEMU image, boot from the Alpine ISO:
~~~
$ qemu-img create -f raw alpine-appvm.img 4G
Formatting 'alpine-appvm.img', fmt=raw size=4294967296

$ qemu-system-x86_64 -m 1G \
                     -enable-kvm \
                     -cdrom ./alpine-3.2.3-x86_64.iso \
                     -boot order=d \
                     alpine-appvm.img
(a QEMU window will pop up)
~~~

Once the QEMU virtual machine starts, install Alpine Linux to it:
~~~ 
(from the QEMU guest:)

localhost login: root

....

localhost ~# setup-alpine
               (go through installation or use the example 'setup-alpine.conf' with -f)

             ....

             Installation is complete. Please reboot.

hostname  ~# poweroff
~~~

Start the VM again, booting from the 'alpine-appvm.img' disk image:
~~~
$ qemu-system-x86_64 -m 1G \
                     -enable-kvm \
                     -net nic \
                     -net user,hostfwd=tcp::10022-:22 \
                     -vga qxl \
                     -spice port=5900,addr=127.0.0.1,disable-ticketing \
                     alpine-appvm.img

  (this time nothing pops up)
~~~

SSH into the VM and install a few dependencies:
~~~
$ ssh appvm-browser "apk add sudo python"
~~~

Run the Ansible Playbook:
~~~
$ source $ANSIBLE_DIR/hacking/env-setup
$ python2 $ANSIBLE_DIR/bin/ansible-playbook -i hosts --ask-pass browser.yml

    (you should be prompted for the SSH password you set with 'setup-alpine'.)

    ....it runs for a while....

    At the end, you should have a status message like this:

    PLAY RECAP *********************************************************************
    appvm-browser              : ok=52   changed=40   unreachable=0    failed=0
~~~

Power down the VM:
~~~
$ ssh appvm-browser "poweroff"
~~~

Wait for your other QEMU process to exit, start it again with additional 
vdagent parameters, and connect to it with spicec:
~~~
$ qemu-system-x86_64 -m 1G \
                     -enable-kvm \
                     -net nic,model=virtio \
                     -net user \
                     -vga qxl \
                     -spice port=5900,addr=127.0.0.1,disable-ticketing \
                     -device virtio-serial-pci \
                     -device virtserialport,chardev=spicechannel0,name=com.redhat.spice.0 \
                     -chardev spicevmc,id=spicechannel0,name=vdagent \
                     alpine-appvm.img

$ spicec -h 127.0.0.1 -p 5900
(Warning: Shift-F12 releases the cursor in Spice, not ctrl-alt as in QEMU!)

Spice should bring you to the same login screen as before. Login with 
"root" and the password you set during installation and run: "reboot".
~~~

If the installation was successful, upon reboot your spicec 
client should show a Firefox browser window. You're done!


## Final Notes
* Once you configure your browser (add extensions, tweak preferences, etc), 
  if you add an additional "-snapshot" parameter to QEMU's launch command,
  changes made in the QEMU session will not be saved to the image 
  stored on disk!
* When creating the VM, you should add a RNG that passes /dev/random from 
  the host to the guest to improve the available entropy pool.

## Additional Documentation & Resources
* [Ansible documentation](http://docs.ansible.com)
* [Enabling KVM](https://wiki.archlinux.org/index.php/QEMU#Enabling_KVM)
* [Alpine Linux](http://www.alpinelinux.org/)
* [Using SPICE and QXL](http://www.linux-kvm.org/page/SPICE)
