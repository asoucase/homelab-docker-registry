# Homelab Docker Registry in Kubernetes + Raspberry Pi 4 
HomeLab Docker Registry in Kubernetes running in a Raspberry Pi 4 cluster.

![](.github/images/diagram.png)



As Raspberry Pi 4 runs on an `arm64` SOC (Broadcom BCM2711), I had to compile and build a docker image of Quiq's docker registry UI for `arm64`. It has been pushed to Docker Hub at  `arturosoucase/quiq-docker-registry-ui`



## Build images for `arm64`

### Install Requirements

For Ubuntu 20.04:

```bash
$ sudo apt install qemu-user-static
```

Ref: [Issue in buildx](https://github.com/docker/buildx/issues/138)

### Buildx in Docker CE

`buildx` comes bundled with Docker CE 19.04+ but requires the experimental mode to be enabled. 

There are two possible options:

- Add `"experimental": "enabled"` to CLI config file `~/.docker/config.json`
- Set environment variable `DOCKER_CLI_EXPERIMENTAL=enabled`

### Verify Buildx 

```bash
$ docker buildx --help

Usage:	docker buildx COMMAND

Build with BuildKit

Management Commands:
  imagetools  Commands to work on images in registry

Commands:
  bake        Build from a file
  build       Start a build
  create      Create a new builder instance
  inspect     Inspect current builder instance
  ls          List builder instances
  rm          Remove a builder instance
  stop        Stop builder instance
  use         Set the current builder instance
  version     Show buildx version information 

Run 'docker buildx COMMAND --help' for more information on a command.
```

### List Builder Instances

```bash
$ docker buildx ls
NAME/NODE      DRIVER/ENDPOINT             STATUS  PLATFORMS
default        docker                              
  default      default                     running linux/amd64, linux/386
```

We are currently using the default builder. Lets create a new builder for the `arm64` platform

### Create Builder and switch to it

Create a new builder with the name `arm-builder` for platform `linux/arm64`

```bash
$ docker buildx create --name testbuilder --platform linux/arm64
arm-builder
```

Switch to the new builder

```bash
$ docker buildx use arm-builder
```

Inspect it

```bash
$ docker buildx inspect --bootstrap
```

By using the `--bootstrap` flag, it ensures that the builder is running before inspecting it.

If we list builder instances, our new builder should appear.

```bash
$ docker buildx ls
NAME/NODE      DRIVER/ENDPOINT             STATUS  PLATFORMS
arm-builder *  docker-container                    
  arm-builder0 unix:///var/run/docker.sock running linux/arm64, linux/amd64, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/386, linux/arm/v7, linux/arm/v6
default        docker                              
  default      default                     running linux/amd64, linux/386
```

### Build!

To build an image, make use of the `buildx` command. You can specify multiple platforms to generate multi-arch manifests.

```bash
$ docker buildx build --platform linux/amd64,linux/arm6 -t demo:latest .
```

You can also add the `--push` flag to push all images to Docker Hub. Make sure you have logged in first!



## Certificates

### Create your own CA

1. Generate RSA key for the CA

```bash
$ openssl genrsa -out ca.key 4096
```

2. Generate a CA certificate using the key (valid for 10 years)

```bash
$ openssl req -new -x509 -key ca.key -subj "/C=AQ/ST=-/O=HomelabCertificateAuthority/CN=homelab" -out ca.crt -days 3650
```

### Generate certificate signed by CA

1. Generate key

```bash
$ openssl genrsa -out master.tld.key 4096
```

2. Generate CSR (certificate signing request

```bash
$ openssl req -new -key master.tld.key -subj "/C=AQ/ST=-/O=HomelabDockerRegistry/CN=master.tld" -out master.tld.csr
```

3. Sign CSR to generate certificate

```bash
$ openssl x509 -req -in master.tld.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 365 -out master.tld.crt
```

To bundle both certs:

```bash
$ cat master.tld.crt ca.crt >> master.tld.bundle.crt
```

> *Note*:
>
> The generate certificate does not include any subject alt names. Firefox is OK with that, but Chrome will complain and mark the connection as not private. 

### Install a CA 

1. Copy certificate (in PEM format) to `/usr/local/share/ca-certificates`
2. Update CA certificates

```bash
$ sudo update-ca-certificates
```



## Push image to your registry

1. Login to private registry

```bash
$ docker login myregistry.local
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /home/demo/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

2. Tag image using private registry name

```bash
$ docker image tag nginx:latest myregistry.local/nginx:latest
```

