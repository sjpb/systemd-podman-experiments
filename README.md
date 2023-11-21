This repo demonstrates running a containerised service using podman, without running
either podman or the service as root.

The original container created with:

    [rocky]$ podman run -dit --name apache -p 8080:80 docker.io/library/httpd:2.4

Whether the container is running can be tested with:


    [rocky]$ curl localhost:8080
    <html><body><h1>It works!</h1></body></html>

A unit file was then created with:

    [rocky]$ podman generate systemd --new apache

and the aim is to run this service with the user `podman`.

The obvious approach is simply to add `User=podman` to the `Service` section of the 
unit. However service startup fails with:

    Error: creating idfile: open /run/apache.service.ctr-id: permission denied

Therefore the service must be run as a systemd user unit for the `podman` user. To do 
this, user-lingering must be enabled so that there is a user session available for the 
service. Without this, unit startup fails with:

    Failed to connect to bus: No such file or directory

Note that even with user lingering enabled, `su` does not allow interaction with 
systemd user units, failing with the same message. There are two options:

1. `su` with the $XDG_RUNTIME_DIR set to the relevant user ID:

    [rocky ~]$ sudo su podman
    [podman rocky]$ XDG_RUNTIME_DIR=/run/user/$UID systemctl --user status apache
    ● apache.service
    Loaded: loaded (/home/podman/.config/systemd/user/apache.service; enabled; vendor preset: enabled)
    Active: active (running) since Tue 2023-11-21 14:52:00 UTC; 14min ago

2. Use `machinectl` (provided by the `systemd-container` package) to get a proper 
login session:

    [rocky ~]$ sudo machinectl shell podman@.host
    Connected to the local host. Press ^] three times within 1s to exit session.
    [podman ~]$ systemctl --user status apache
    ● apache.service
    Loaded: loaded (/home/podman/.config/systemd/user/apache.service; disabled; vendor preset: enabled)
    Active: active (running) since Tue 2023-11-21 10:42:22 UTC; 7min ago
    Main PID: 1979 (conmon)
    ...

A third should be possible:

    [rocky ~]$ systemctl --user -M podman@ status apache

But this requires systemd >= 247 and RockyLinux 8.8 provides systemd 239.

In terms of Ansible control of user units, using:

    systemd:
        ...
        scope: user
    become_user: podman

appears to be sufficient (i.e. setting `XDG_RUNTIME_DIR` is not required). While there 
is an ansible become plugin for machinectl it is not suitable here as it only works for 
root user ((docs)[
https://docs.ansible.com/ansible/latest/collections/community/general/machinectl_become.html#notes])
 at least without additional polkit rules.

Logging is also slightly problematic. Clearly the syslog (`/var/logs/messages`) will 
not contain logs for user units. However by default journald doesn't store logs either. 
To fix this `journald` configuration in `/etc/systemd/journald.conf` must have 
`Storage=auto` replaced with `Storage=persistent` and journald restarted. Logging can 
then be accessed using one of the following:

1. `su` with the $XDG_RUNTIME_DIR set to the relevant user ID:

    [rocky@sb-podman ~]$ sudo su podman
    [podman@sb-podman rocky]$ XDG_RUNTIME_DIR=/run/user/$UID journalctl --user -u apache.service
    ...

2. Using `machinectl`:

    [rocky ~]$ sudo machinectl shell podman@.host
    Connected to the local host. Press ^] three times within 1s to exit session.
    [podman ~]$ journalctl --user -u apache.service
    ...

3. Running journalctl as root but [filtering](https://www.freedesktop.org/software/systemd/man/latest/systemd.journal-fields.html#Trusted%20Journal%20Fields) to the appropriate user unit:

    [rocky ~]$ sudo journalctl _SYSTEMD_USER_UNIT=apache.service
    ...
