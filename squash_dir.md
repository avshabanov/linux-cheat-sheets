# squash_dir
squash_dir is an init-script for OpenRC (used e.g. by Gentoo) which allows to keep a directory compressed by the in-kernel squashfs but simultaneously allows to write on it. Every time you boot your computer your squashed directory is mounted writable and if you made any changes to it it's recompressed when you shutdown your computer.


## Required kernel modules

	Device Drivers --->
	  Block Devices --->
	    <M> Loopback device support

	File systems --->
	  Miscellaneous Filesystems --->
	    <M> SquashFS
	     (include also LZO/XZ support if you plan to use any of them to compress the dir)


In order to get `loop` kernel module automatically loaded at boot edit `/etc/conf.d/module` and set

	modules="loop"

## Unionfs implementations

Together with squashfs, squash_dir needs to use a unionfs implementation. You have to choose one of:

 - [aufs](http://aufs.sourceforge.net).
 - [overlayfs](http://git.kernel.org/?p=linux/kernel/git/mszeredi/vfs.git) 
 - [unionfs-fuse](http://podgorny.cz/moin/UnionFsFuse) (unionfs-fuse-0.25 or newer is required).
 - [unionfs](http://www.fsl.cs.sunysb.edu/project-unionfs.html).
 - [funionfs](http://bugs.gentoo.org/show_bug.cgi?id=151673)
 - [dtrx](http://brettcsmith.org/2007/dtrx/)


Some of them are available in Gentoo in the official Portage tree or via overlays. Search for

 - sys-fs/aufs2
 - sys-fs/aufs3
 - sys-fs/aufs (from 'mv' overlay)
 - sys-fs/unionfs-fuse
 - sys-fs/funionfs
 - sys-fs/unionfs

After you install one of them, make sure to apply the kernel patch they include and recompile your kernel. If the choosed ebuild has the USE="kernel-patch" and you enable it, the patch will be automatically applied to your kernel sources at /usr/src/linux.


## Installation

The easiest way of installing squash_dir is using the mv overlay via layman

	emerge -n layman
	layman -f
	layman -a mv
	emerge --sync

If you use eix, you can alternatively run `eix-sync`. In order to automatically update mv overlay with eix-sync create the file `/etc/eix-sync.conf` with the next content:

	mv

Now the mv overlay is ready. To install squash_dir (make sure you enable the suitable USE according to the Unionfs implementation you installed in the previous step)

	emerge -av sys-fs/squash_dir

TIP: If you want to see a progress bar while squash_dir is (de)compressing:

	echo "sys-fs/squashfs-tools progress-redirect" >> /etc/portage
	emerge -1 squashfs-tools


## Usage

To compress a dir you need to:

 - Create a symlink in /etc/init.d/squash_NAME pointing to /etc/init.d/squash_dir
 - Create a config file in /etc/conf.d/squash_NAME with at least 3 variables:
  - DIRECTORY: Directory to compress and also where the squashed filesystem should finally be mounted
  - DIR_SQUASH: Directory to store the unchanged and compressed data
  - DIR_CHANGE: Directory to store changes made to DIR_SQUASH
 - Add the service to the default runlevel and start it

	rc-update add squash_NAME default
	/etc/init.d/squash_NAME start

An optional variabel you can use in the config file is `COMPRESSION` for specify the compression method used by mksquashfs. If empty, the default mksquashfs algorithm (currently: "gzip") is used. Other possible values are "gzip", "xz", "lzo". If this variable is not specified, it defaults to that algorithm which is presumably best compressing (which is currently "xz"). "lzo" uses to be the fastest one. If you chose "xz" or "lzo" make sure you also enable corresponding kernel option for squashfs.

There are plenty more available configuration variables. Check the README file included in squash_dir package for more information.

# Examples

__NOTE:__ I use squash_dir to make room in my SSD drive and avoid extra writes. All my squashed files and dirs are stored in a rotational disks raid. I've plenty room in the raid so I've used lzo compression for all the examples because it uses to be the fastest one and I don't want to wait too much in every shutdown in case there are changes in my squashed dirs. If you are running low on disk space consider using other options such "xz". Whatever compression method you use, make sure `sys-fs/squashfs-tools` and kernel are compiled with corresponding options.


## Compress Portage tree

The Portage tree is the perfect candidate to be compressed with squash_dir because it has thousand of small text files that can be easily recovered in case something bad happens.

First you need to make sure both DISTDIR (by default /usr/portage/distfiles) and PKGDIR (by default /usr/portage/packages) are set to a place other than PORTDIR (by default /usr/portage/), because it makes no sense to keep them compressed by squash_dir. Both variables can be set in `/etc/make.conf`.

Create the symlink

	ln -s /etc/init.d/squash_dir /etc/init.d/squash_portage

Create the corresponding config file `/etc/conf.d/squash_portage` with the content:

	DIRECTORY="/usr/portage"
	DIR_SQUASH="/mnt/raid/0/squash/portage.readonly"
	DIR_CHANGE="/mnt/raid/0/squash/portage.changes"
	FILE_SQFS="/mnt/raid/0/squash/portage.sqfs"
	COMPRESSION="lzo"

Start the service

	rc-update add squash_portage default
	/etc/init.d/squash_portage start

## Compress kernel sources

The kernel sources dir is also a good candidate to be compressed with squash_dir. It also has thousand of small files that can be easily recovered.

Create the symlink

	ln -s /etc/init.d/squash_dir /etc/init.d/squash_kernel

Create the corresponding config file `/etc/conf.d/squash_kernel` with the content:

	DIRECTORY="/usr/src"
	DIR_SQUASH="/mnt/raid/0/squash/kernel.readonly"
	DIR_CHANGE="/mnt/raid/0/squash/kernel.changes"
	FILE_SQFS="/mnt/raid/0/squash/kernel.sqfs"
	COMPRESSION="lzo"

Start the service

	rc-update add squash_kernel default
	/etc/init.d/squash_kernel start


## Compress /var/db

The directory `/var/db` could be a perfect candidate to be squashed but it's rather vital to your system. If it is lost you have practically completely messed up your system. It's up to you to take the risk of using squash_dir with it. If you decide to go on, at least a backup of the previous compressed data should always be kept using the `FILE_SQFS_OLD` config variable. I don't recomend to squash it but if you decide to do so, here is a possible configuration to use:


Create the symlink

	ln -s /etc/init.d/squash_dir /etc/init.d/squash_db

Create the corresponding config file `/etc/conf.d/squash_db` with the content (Note here I'm using my raid1, not my raid0'):

	DIRECTORY="/var/db"
	DIR_SQUASH="/mnt/raid/1/squash/db.readonly"
	DIR_CHANGE="/mnt/raid/1/squash/db.changes"
	FILE_SQFS="/mnt/raid/1/squash/db.sqfs"
	FILE_SQFS_OLD="/mnt/raid/1/squash/db.sqfs.bak"
	COMPRESSION="lzo"

Start the service

	rc-update add squash_db default
	/etc/init.d/squash_db started

## Compress docs

Create the symlink

	ln -s /etc/init.d/squash_dir /etc/init.d/squash_doc

Create the corresponding config file `/etc/conf.d/squash_doc` with the content:

	DIRECTORY="/usr/share/doc"
	DIR_SQUASH="/mnt/raid/0/squash/doc.readonly"
	DIR_CHANGE="/mnt/raid/0/squash/doc.changes"
	FILE_SQFS="/mnt/raid/0/squash/doc.sqfs"
	COMPRESSION="lzo"

Start the service

	rc-update add squash_doc default
	/etc/init.d/squash_doc start

## And more ...

Same for:

	/usr/share/gtk-doc/
	/usr/share/man/
	/usr/share/mime/
	/usr/share/sgml/
	/usr/share/fonts
	/usr/share/locale
	/usr/share/myspell
