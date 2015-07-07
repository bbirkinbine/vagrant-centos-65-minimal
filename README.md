
Vagrant compatible build of:

	CentOS 6.5 (i386 or x86_64) minimal + changes below to be vagrant compatible

		No chef/ruby installed on this base box, assumes using chef omnibus or other way to get ruby/chef/puppet on the system

		These instructions were created using VirtualBox 4.3.4 r91027 on MacOSX 10.8.4


	CentOS 7 - https://github.com/bbirkinbine/packer-vagrant-centos-7-1503-01-x86_64-minimal


Contact:

	http://www.brianbirkinbine.com/contact.html


.box location:

	http://files.brianbirkinbine.com/vagrant-centos-65-i386-minimal.box

	http://files.brianbirkinbine.com/vagrant-centos-65-x86_64-minimal.box


Vagrantfile:

		Make sure you have a valid Vagrantfile you are working on (did you run 'vagrant init')?

		Make sure the following is configured to point to the newly created .box in your Vagrantfile

		# 32 bit (i386)
		config.vm.box = "vagrant-centos-65-i386-minimal"
		config.vm.box_url = "http://files.brianbirkinbine.com/vagrant-centos-65-i386-minimal.box"

		or (not both)

		# 64 bit (x86_64)
		config.vm.box = "vagrant-centos-65-x86_64-minimal"
		config.vm.box_url = "http://files.brianbirkinbine.com/vagrant-centos-65-x86_64-minimal.box"


Notes:

	Started with CentOS 6.5 (i386 or x86_64) minimal ISO created in Virtual Box
		CentOS-6.5-i386-minimal.iso	a4f27ab51d0d2c9d36b68c56b39f752b (md5)
		CentOS-6.5-x86_64-minimal.iso	0d9dc37b5dd4befa1c440d2174e88a87 (md5)

	VirtualBox settings changed from default
		Type: Linux
		Version: Red Hat (32 or 64 bit depending on which CentOS ISO you are using for minimal installation)
		Motherboard->Base Memory: 768 # so we could run graphical installer to not have LVM created by default
		Disabled Audio
		Network->Adapter1->NAT
		Hard drive size: 40 GB dynamic

	First boot Kernel panic
		If you get a Kernel panic on first boot with a new CentOS ISO, there was a bug that would cause
		selinux policy from loading properly the first time under certain conditions

			https://bugzilla.redhat.com/show_bug.cgi?id=856332

		The error message would look similar to:

			Kernel panic - not syncing: Attempted to kill init!
			Pid: 1, comm: init Not tainted 2.6.32-358.el6.x86_64 #1
			Call Trace:
		 	 [<ffffffff8150cfc8>] ? panic+0xa7/0x16f
		 	 <more lines similar to line above here>

		The workaround for this requires you to interrupt the grub boot and tell selinux to not enforce on boot.  You can then
		either yum update to get latest packages and/or change /etc/selinux/config to permissive (instead of enforcing).

		During the boot screen where it says to 'Press any key to enter the menu', press any key to get to the grub boot menu.

		At the GNU GRUB boot menu, press the 'e' key to edit the boot parameters.
		On the second line where it has kernel /boot/vmlinuz-2.6.xxxx, append to the line enforcing=0 and hit enter.
		You can then boot the modified change by hitting the 'b' key.

		You can then continue with the steps below (the /etc/selinux/config change below should fix this issue).
		

	Installed onto a single partition (ext4 no LVM on /, no swap partition) 40gb dynamically allocated
		root password: vagrant
 
	CentOS 6.5 changes from minimal:  All modified files have the original file appended with .vagrantbackup on the filesystem

		selinux set to permissive
			/etc/selinux/config
				SELINUX=permissive

		Disable iptables and ip6tables
			# chkconfig iptables off
			# chkconfig ipt6ables off

		Adjust networking to start
			/etc/sysconfig/network-scripts/ifcfg-eth0
				changed ONBOOT=yes instead of ONBOOT=no
				changed NM_CONTROLLED=no instead of NM_CONTROLLED=yes
				removed the HWADDR line
				removed the UUID line

		Remove udev file to be re-generated on boot
			# shred -uv /etc/udev/rules.d/70-persistent-net.rules

		NOTE: When creating initial VM, have to restart VM after changes above for networking to work and continue
			Once you verify dhcp is working with eth0, re-do the 'Adjust networking to start' and
			'Remove udev file to be re-generated on boot' so that packaged vagrant .box will be correct

		Clear out any existing dhcp leases
			# shred -uv /var/lib/dhclient/*

		Configure SSHD to not use DNS on connect
				# vi /etc/ssh/sshd_config
					Change 'UseDNS no' instead of 'UseDNS yes' 

		Installed pre-requisites so that virtualbox additions/extensions will be successful
			yum install -y gcc make perl kernel-devel-`uname -r` kernel-headers-`uname -r`

		Install virtualbox additions/extensions
			create /media/cdrom mount point
				# mkdir /media/cdrom
			mount /media/cdrom for virtualbox Guest additions
				(make sure to install/mount in virtual box for correct device to be presented to the OS)
				# mount -o ro /dev/cdrom /media/cdrom
			install virtualbox additions/extensions
				# cd /media/cdrom
				# export KERN_DIR="/usr/src/kernels/`uname -r`"
				# ./VBoxLinuxAdditions.run --nox11
					Ignore any errors regarding OpenGL or Windows System drivers, we dont have X11 installed on minimal build
			verify that virtual box services are working
				# VBoxControl --version
				# VBoxService --version

		Setup vagrant user/group/sudo/ssh
			Create vagrant user
				# useradd -m -G wheel vagrant
			Set vagrant user password to vagrant
				# echo vagrant | passwd vagrant --stdin
			Uncomment the wheel group in sudoers to use sudo without re-entering a password
				# visudo
					Find the line that has '%wheel ALL=(ALL) NOPASSWD: ALL' and make sure it is uncommented
			Disable the requirement to have a tty, this is so that vagrant user can sudo remotely over ssh
				# visudo
					Find the line that has 'Defaults requiretty' and change to 'Defaults !requiretty'
			Add a line to keep the SSH_AUTH_SOCK environment variable in sudoers
				# visudo
					Add 'Defaults	env_keep += "SSH_AUTH_SOCK"
			Configure vagrant user to have the vagrant ssh public key
				# mkdir -m 0700 /home/vagrant/.ssh
				# curl -s https://raw.github.com/mitchellh/vagrant/master/keys/vagrant.pub > /home/vagrant/.ssh/authorized_keys
				# chown -R vagrant:vagrant /home/vagrant/.ssh
				# chmod 0600 /home/vagrant/.ssh/authorized_keys

		Clean up any package caches
			# yum clean all

		Clean up any logs, bash history, keys, etc

			# the following section needs to be run as root
			
			# zero out /etc/resolv.conf (re-created on boot)
			cd /etc
			> ./resolv.conf

			# clear out auto-generated ssh keys on system start
			cd /etc/ssh
			shred -uv *key*

			# clear out log files
			cd /var/log
			> wtmp
			> cron
			> dmesg
			> dmesg.old
			> lastlog
			> secure
			> messages
			> maillog

			# force re-creation of random-seed
			cd /var/lib
			shred -uv ./random-seed

			# clear out files in /root (if any)
			cd /root
			shred -uv .ssh/*
			shred -uv .lesshst .bash_history 
			exit

			# re-login as root directly
			shred -uv .bash_history
			shutdown -h now


	Vagrant packaging

		Go to the directory containing your virtualbox disk image, in my case 'VirtualBox VMs'

		cd 'VirtualBox VMs'
		
		# for i386 (32 bit)
		# vagrant package --base vagrant-centos-65-i386-minimal --output vagrant-centos-65-i386-minimal.box

		# for x86_64 (64 bit)
		# vagrant package --base vagrant-centos-65-x86_64-minimal --output vagrant-centos-65-x86_64-minimal.box

		
	Vagrant box

		from the same directory where the .box image was created

		# for i386 (32 bit)
		# vagrant box add vagrant-centos-65-i386-minimal ./vagrant-centos-65-i386-minimal.box

		# for x86_64 (64 bit)
		# vagrant box add vagrant-centos-65-x86_64-minimal ./vagrant-centos-65-x86_64-minimal.box


Verify:

	md5:	vagrant-centos-65-i386-minimal.box	0033447e785ce51db089abc91b7cdfcc
	md5:	vagrant-centos-65-x86_64-minimal.box	f06620ec3e1ac50a084bd5e3aed8472e


	# 32 bit (i386)
	[vagrant@localhost ~]$ sudo rpm -Va
	....L....  c /etc/pam.d/fingerprint-auth
	....L....  c /etc/pam.d/password-auth
	....L....  c /etc/pam.d/smartcard-auth
	....L....  c /etc/pam.d/system-auth
	S.5....T.  c /etc/ssh/sshd_config
	.......T.  c /etc/inittab
	S.5....T.  c /etc/sudoers
	[vagrant@localhost ~]$ sudo find / -name "*.vagrantbackup" -ls
	2490378    4 -rw-------   1 root     root         3879 Nov 22 17:37 /etc/ssh/sshd_config.vagrantbackup
	2491393    4 -r--r-----   1 root     root         4002 Mar  1  2012 /etc/sudoers.vagrantbackup
	2491372    4 -rw-r--r--   1 root     root          458 Dec  1 09:10 /etc/selinux/config.vagrantbackup


	# 64 bit (x86_64)
	[vagrant@localhost ~]$ sudo rpm -Va
	....L....  c /etc/pam.d/fingerprint-auth
	....L....  c /etc/pam.d/password-auth
	....L....  c /etc/pam.d/smartcard-auth
	....L....  c /etc/pam.d/system-auth
	.......T.  c /etc/inittab
	S.5....T.  c /etc/sudoers
	S.5....T.  c /etc/ssh/sshd_config
	[vagrant@localhost ~]$ sudo find / -name "*.vagrantbackup" -ls
	1704949    4 -r--r-----   1 root     root         4002 Mar  1  2012 /etc/sudoers.vagrantbackup
	1704936    4 -rw-r--r--   1 root     root          458 Dec  3 09:44 /etc/selinux/config.vagrantbackup
	1703946    4 -rw-------   1 root     root         3879 Nov 22 17:40 /etc/ssh/sshd_config.vagrantbackup


Security:

	Remember that this is designed for use with Vagrant, vagrant by default has a root password of vagrant.
	The default username/password for vagrant is vagrant/vagrant and the public/private keys are widely available.

	Do not use this for production use or on the Internet without removing the vagrant user, changing the root password,
		and consider enabling the firewall (iptables) and if you are familiar with selinux, enable it /etc/selinux/config

	You have been warned.


Credits:

	* http://docs-v1.vagrantup.com/v1/docs/base_boxes.html
	* http://cbednarski.com/articles/creating-vagrant-base-box-for-centos-62/
	* https://github.com/mitchellh/vagrant/issues/997
