---
title: "Getting started with podman on an enterprise network"
authors: ["Geert Van Wauwe"]
draft: true
---


Discuss basic setup podman, logging into fedora vm and configuring certs.

# Podman on an enterprise network
## Podman
Podman is a container engine that can be used for running and managing containers. It is similar to docker.
Transitioning from the paid docker
## Certificates
Most enterprise network issue self-signed certificates for their applications and sometimes even clients.
# download Podman
`brew install podman`
`podman machine init -v $HOME/certs:/etc/containers/certs.d:ro` downloads the linux virtual machine that runs the linux kernel. The `-v` option makes sure the root certificate of ZScaler and some other certificates are added.
Though they might be located at another directory, the gist of it is clear. However, this does not work because it requires servername and port (see https://access.redhat.com/documentation/en-us/red_hat_quay/3/html/manage_red_hat_quay/using-ssl-to-protect-quay#:~:text=Configuring%20podman%20to%20trust%20the%20Certificate%20Authority,%2Fetc%2Fdocker%2Fcerts)

`podman machine init -v $HOME/certs:/etc/pki/ca-trust/source/anchors:ro` to update the system-wide trust store.

`podman machine start` starts default virtual machine. This is necessary because every container requires the linux kernel, and that linux kernel is in this fedora distribution.
`podman machine ssh` to connect to the default virtual machine
necessary run `update-ca-trust extract` : https://man.archlinux.org/man/update-ca-trust.8#:~:text=update%2Dca%2Dtrust(8)%20is%20used%20to%20manage,CA)%20certificates%20and%20associated%20trust.
verify that the wanted CA certificate is now part of the extracted certificates using:
`grep -rni yournameofcertificate /etc/pki/ca-trust/extracted`
this should show at least one occurrence of the certificated you added.

## building using podman build

Podman can build containers based on a Containerfile. Under the hood Podman uses Buildah.

### containers on a corporate network with a Custom CA.
A container run on your device on a corporate network should trust the custom CA.
In our case this custom CA is Zscaler.
For containers based on the alpine image, this can be achieved by appending the root certificate of the CA to `/etc/ssl/certs/ca-certificates.crt`.
This should be sufficient for 99 percent of the use cases.

#### update-ca-certificates
It is possible that you will run `update-ca-certificates` at some point in the future because you want to manage your certificates this way.
The `update-ca-certificates` library is not present by default on alpine and should be downloaded if you want to use it.
The command `update-ca-certificates` will create a new `ca-certificates.crt` file and will thus overwrite the appended custom CA certificate.
Place your custom ca certificate in `/usr/local/share/ca-certificates/` as well to make sure `update-ca-certificates` places the custom CA certificate in the ca-certificates.crt file.


#### container file

    FROM alpine:3.14

    COPY ca-bundle.pem /usr/local/share/ca-certificates/ca-bundle.pem
    RUN cat /usr/local/share/ca-certificates/ca-bundle.pem >> /etc/ssl/certs/ca-certificates.crt


if you want to change directories, don't just blindly use the linux command to change directory:
`cd directory/you/want/to/go/to`
since docker/podman commands in a container file are wrapped by the podman run and podman commit commands, the execution of the command has no effect.

However you can use the WORKDIR command to accomplish this in the dockerfile, so in the dockerfile just add:
`WORKDIR directory/you/want/to/go/to`

##### clarifications
the first argument of the copy command is a relative location in the containercontext i.e. the folder of containerfile.

#### building the container
run `podman build -t nameofyourcontainer .` in the folder where the container file is located.

run `podman images` to see whether your built container is present in the list.

# running a container
podman run -it nameofcontainer 

## finding imagemagick delegates
### container file

    FROM alpine:3.14

    COPY ca-bundle.pem /usr/local/share/ca-certificates/ca-bundle.pem
    RUN cat /usr/local/share/ca-certificates/ca-bundle.pem >> /etc/ssl/certs/ca-certificates.crt
    