
Original container created with:

    podman run -dit --name apache -p 8080:80 docker.io/library/httpd:2.4

Original service created with:

    podman generate systemd --new apache

Current fails with

    Nov 21 09:58:23 sb-podman systemd[1]: Starting apache.service...
    Nov 21 09:58:23 sb-podman podman[22153]: time="2023-11-21T09:58:23Z" level=warning msg="The input device is not a TTY. The --tty and --interactive flags might not work properly"
    Nov 21 09:58:23 sb-podman podman[22153]: Error: creating idfile: open /run/apache.service.ctr-id: permission denied
