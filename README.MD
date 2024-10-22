# Install VirtualBox guest additions onto Ubuntu server guests

> [!NOTE]
> The following steps have been successfully tested with Ubuntu guests of:
>
> - `24.04LTS`
> - `22.04LTS`
> - `18.04LTS`
> - `16.04LTS`
> 
> using VirtualBox editions of `7.0`, `6.1`, `6.0`, `5.2` and `5.1` as host.

- [Install steps](#install-steps)
- [Mount with guest/host permissions matched](#mount-with-guesthost-permissions-matched)
- [Allow symlink creation for shared folders](#allow-symlink-creation-for-shared-folders)
- [Reference](#reference)

## Install steps

- Create Ubuntu server instance under VirtualBox (obviously).
- Start VM, goto **Devices - Insert Guest Additions CD image** to mount the ISO image.
- From the terminal, run the following commands:

	```sh
	$ sudo su
	$ apt install gcc make
	$ mkdir --parents /media/cdrom
	$ mount /dev/cdrom /media/cdrom
	$ /media/cdrom/VBoxLinuxAdditions.run
	$ reboot
	```

- After reboot:

	```sh
	$ modinfo vboxguest
	$ sudo usermod --append --groups vboxsf -- "$USER"
	$ cat /etc/group | grep "$USER"
	```

- Host shares should now be mounted in Ubuntu guest under `/media` via the installed `VBoxService` service, set to start on system boot-up.
- All done.

**Note:** the above steps can be repeated on an existing VM image for guest addition upgrades, `VBoxLinuxAdditions.run` will handle the uninstall and reinstall process automatically.

## Mount with guest/host permissions matched

By default with auto mounted shares, VirtualBox will set an interesting set of ownership and file/directory permissions for the guest which (for me at least) is sub-optimal.

To correct this behavior and correctly mirror permissions between host and guest we can do the following:

- Shut down the guest (if not already) and ensure "Auto-mount" is _disabled_.
- Restart the guest.
- From the shell, determine the `umask` and `uid/gid` of the host user:

	```sh
	$ umask
	0022
	$ id
	uid=1000(USERNAME) gid=1000(USERNAME) groups=1000(USERNAME),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
	```

- Now mount the share as follows, inserting `UMASK`, `UID`, `GID` and `SHARE_NAME`:

	```sh
	$ mkdir --parents /path/to/share
	$ sudo mount.vboxsf -o umask=UMASK,uid=UID,gid=GID SHARE_NAME /path/to/share
	```

- If this works, we can now add a permanent entry to `/etc/fstab`, again inserting `UMASK`, `UID`, `GID` and `SHARE_NAME`:

	```
	SHARE_NAME /path/to/share vboxsf umask=UMASK,uid=UID,gid=GID 0 0
	```

- Remount all file systems

	```sh
	$ sudo mount -a
	```

- All done.

## Allow symlink creation for shared folders

By default symlink creation by a VirtualBox guest against a shared/host filesystem is not allowed. To unlock run the following:

```sh
$ vboxmanage setextradata VBOX_GUID VBoxInternal2/SharedFoldersEnableSymlinksCreate/SHARED_FOLDER_NAME 1
```

A list of active/valid `VBOX_GUID`'s can be returned via:

```sh
$ vboxmanage list vms
```

## Reference

- https://askubuntu.com/questions/123025/what-is-the-correct-way-to-share-directories-in-mac-and-ubuntu-with-correct-perm/123156#123156
- https://askubuntu.com/questions/252853/how-to-mount-a-virtualbox-shared-folder-at-startup/252865#252865
