# NSpawn

Short for `namespaces spawn`. Create system-level containers (as opposed to application-level containers like Docker does).

The `machinectl` utility uses the `/var/lib/machines` directory by default to identify containers, though it's not required.

```txt
"systemd-nspawn is like the chroot command, but it is a chroot on steroids."
```

* https://blog.selectel.com/systemd-containers-introduction-systemd-nspawn/
* https://patrickskiba.com/sysytemd-nspawn/2019/02/08/introduction-to-systemd-nspawn.html
* https://wiki.archlinux.org/index.php/Systemd-nspawn

## Basics

### Running a container

To run a container, all that is needed beforehand is a directory containing an OS filesystem, which can be created using tools like `debootstrap` or `pacstrap`.

1. Run `systemd-nspawn` and point it to the filesystem directory to use
    * The `-U` flag tells systemd to use user namespaces to separate the user IDs within the container from those on the host system

```sh
# systemd-nspawn -D <filesystem_directory>
user@host$ sudo systemd-nspawn -D /var/lib/machines/debian -U
Spawning container debian on /var/lib/machines/debian.
Press ^] three times within 1s to kill container.
Selected user namespace base 1766326272 and range 65536.
root@debian:~#
```

2. Change the root password if this is the first time

```sh
root@debian:~# passwd
New password:
Retype new password:
passwd: password updated successfully
```

2. Log out and back in

```sh
root@debian:~# logout
Container debian exited successfully.
```

3. Boot the container with the `-b` flag to bring up it's init system (generally also systemd - depends on how the filesystem was created)

```sh
user@host$ sudo systemd-nspawn -U -b -M debian
...<truncated>...
Debian GNU/Linux 10 debian console

debian login: root
Password:
...<truncated>...
root@debian:~# id
uid=0(root) gid=0(root) groups=0(root)
```

### Running a simple command in the container

The `systemd-nspawn` utility can run commands directly in a container from the host:

```sh
$ sudo systemd-nspawn -M jessie echo hello
Spawning container jessie on /var/lib/machines/jessie.
Press ^] three times within 1s to kill container.
hello
Container jessie exited successfully.
```

## Additional Notes

```txt
user        4613  0.0  0.0  20772  5308 pts/0    Ss   09:30   0:00  |       |   \_ bash                                 --> user-level terminal
root        8079  0.0  0.0  23908  5600 pts/0    S    09:51   0:00  |       |   |   \_ sudo                             --> escalation to root to use machinectl
root        8080  0.0  0.0  20868  5540 pts/0    S    09:51   0:00  |       |   |       \_ bash                         --> root shell
root       29025  0.0  0.0  16692  6180 pts/0    S+   10:04   0:00  |       |   |           \_ systemd-nspawn           --> container instantiation
root       29027  0.1  0.0  28152  4236 ?        Ss   10:04   0:00  |       |   |               \_ systemd              --> container's init system instantiation
root       29049  0.0  0.0  32964  6500 ?        Ss   10:04   0:00  |       |   |                   \_ systemd-journal
root       29089  0.0  0.0  17492  1788 ?        Ss   10:04   0:00  |       |   |                   \_ cron
root       29091  0.0  0.0 262884  3384 ?        Ssl  10:04   0:00  |       |   |                   \_ rsyslogd
root       29094  0.0  0.0  63316  2940 pts/0    Ss   10:04   0:00  |       |   |                   \_ login
user       29110  0.0  0.0  20232  3140 pts/0    S    10:04   0:00  |       |   |                       \_ bash         --> user-level terminal inside container
user       29131  0.0  0.0   4236   688 pts/0    S+   10:05   0:00  |       |   |                           \_ sleep    --> command
```

Containers can be downloaded, exported, or imported as RAW/TAR files (or newer versions of `machinectl` can do Docker directly):

```sh
# machinectl export-tar <imagename> <filepath>
$ sudo machinectl export-tar debianjessie /tmp/jessie.tar
Enqueued transfer job 1. Press C-c to continue download in background.
Exporting '/var/lib/machines/debianjessie', saving to '/tmp/jessie.tar' with compression 'uncompressed'.
Operation completed successfully.
Exiting.
$ file /tmp/jessie.tar
jessie.tar: POSIX tar archive
$ ls -lh /tmp/jessie.tar
-rw-r--r-- 1 root root 382M Aug  3 10:13 jessie.tar
# machinectl import-tar <filepath> <imagename>
$ machinectl import-tar /tmp/jessie.tar debianjessie
Enqueued transfer job 1. Press C-c to continue download in background.
Importing '/tmp/jessie.tar', saving as 'debianjessie'.
Imported 0%.
Imported 8%.
Imported 19%.
Imported 22%.
Imported 28%.
Imported 36%.
Imported 42%.
Imported 57%.
Imported 70%.
Imported 84%.
Imported 95%.
Operation completed successfully.
Exiting.
```

Filesystems can be contained inside .tar files and retrieved remotely via:

```sh
$ sudo machinectl pull-tar <URL> <imagename>
```

Machinectl will also look for a `<imagename>.nspawn` file, which is [a container settings file](https://www.freedesktop.org/software/systemd/man/systemd.nspawn.html).
Interestingly, when retrieving remote .nspawn files, systemd will place them in the `/var/lib/machines` folder by default, which will also disallow any possibly-privileged settings from being read:

```txt
"automatically downloaded (and thus potentially untrusted) settings files are placed in /var/lib/machines/ instead (next to the container images), where their security impact is limited."
```

## Ideas

* Is it possible for an image to specify that EDR kernel modules cannot be loaded into a container?

    ```txt
    "The host system cannot be rebooted and kernel modules may not be loaded from within the container." (Arch Wiki)
    ```

* Set up some TAR container images remotely and run commands through them instead of the host system using `pull-tar`
* If virtual ethernet devices are used (e.g. using `-n` with `systemd-nspawn`), can we avoid restrictions placed on specific network interfaces?
* How can an attacker create containers without requiring root access?

  ```txt
  "machined is Polkit enabled, this allows the creation of polkit rules to allow certain actions to be performed without becoming the root user." (Arch Wiki)
  ```

  * /usr/share/polkit-1/actions/org.freedesktop.machine1.policy

* Can I point `systemd-nspawn` to the local root filesystem (`/` or a softlink to it), specify my own UID to run in the namespace, and take control of all files?
