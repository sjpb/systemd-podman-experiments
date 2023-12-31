This repo demonstrates running a containerised service using podman, without running
either podman or the service as root.

Unless otherwise stated everything applies to
- RockyLinux 8.9 (Rocky-8-GenericCloud-Base-8.9-20231119.0.x86_64.qcow2) with:
  - systemd 239
  - podman 4.4.1
- RockyLinux 9.3 (Rocky-9-GenericCloud-Base-9.3-20231113.0.x86_64.qcow2) with:
  - systemd 252
  - podman 4.6.1

An initial container was created with:

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
        ...

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

3. For RockyLinux 9.3 (systemd 252):

    [rocky ~]$ systemctl --user -M podman@ status apache

    this requires systemd >= 247 and RockyLinux 8.9 provides systemd 239.

In terms of Ansible control of user units, using:

    systemd:
        ...
        scope: user
    become_user: podman

appears to be sufficient (i.e. setting `XDG_RUNTIME_DIR` is not required). While there 
is an Ansible "become plugin" for machinectl it is not suitable here as it only works for 
root user ([docs](https://docs.ansible.com/ansible/latest/collections/community/general/machinectl_become.html#notes)), 
at least without additional polkit rules.

Seeing logs from the service is also non-obvious. Clearly the syslog (`/var/logs/messages`) will 
not contain logs for user units. They can be accessed by [filtering](https://www.freedesktop.org/software/systemd/man/latest/systemd.journal-fields.html#Trusted%20Journal%20Fields) to the appropriate user unit, 
e.g.:

    [rocky ~]$ sudo journalctl _SYSTEMD_USER_UNIT=apache.service
    ...

However that `journalctl --user` commands as the `podman` user won't work as [by default](https://unix.stackexchange.com/a/486566/365795) journald won't store user unit logs. To fix this `journald` 
configuration in `/etc/systemd/journald.conf` must have `Storage=auto` replaced with 
`Storage=persistent` and journald restarted. Logging can then be accessed using one of 
the following:

1. `su` with the $XDG_RUNTIME_DIR set to the relevant user ID:

        [rocky@sb-podman ~]$ sudo su podman
        [podman@sb-podman rocky]$ XDG_RUNTIME_DIR=/run/user/$UID journalctl --user -u apache.service
        ...

2. Using `machinectl`:

        [rocky ~]$ sudo machinectl shell podman@
        Connected to the local host. Press ^] three times within 1s to exit session.
        [podman ~]$ journalctl --user -u apache.service
        ...

Note that without configuring persistent storage, the above will both return a confusing 
error:

    No journal files were opened due to insufficient permissions.

A futher tweak is to add `--userns=auto` to the podman command line, so that the 
container runs in its own namespace. To be able to access mounted volumes either:
- Use the `U` flag to recursively chown on startup,
- Add a supplementary group to the `podman` user, add that as the group on the host 
  volume directory and leak the group into the container using
  [`--group-add=keep-groups`](https://docs.podman.io/en/latest/markdown/podman-run.1.html#group-add-group-keep-groups) 
  on the podman run command line.

In the latter case all containers run by the `podman` user will have access to each 
other's storage; in the former option they won't but there is the potential for 
performance/timeout problems on startup.

Testing showed that `--userns=auto` worked with RL9.2 but not with RL8.2. It appears 
that may be an apache-container specific problem.

## Useful Links

- https://github.com/containers/podman/discussions/20573
- https://github.com/containers/podman/discussions/15155
- https://github.com/containers/podman/issues/17753
- https://github.com/containers/podman/discussions/9642
- https://github.com/containers/podman/issues/13236
- https://www.redhat.com/sysadmin/rootless-podman-user-namespace-modes
