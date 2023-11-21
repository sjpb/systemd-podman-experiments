
Original container created with:

    podman run -dit --name apache -p 8080:80 docker.io/library/httpd:2.4

Original service created with:

    podman generate systemd --new apache
