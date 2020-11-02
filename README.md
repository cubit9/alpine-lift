[![Dependabot Status](https://api.dependabot.com/badges/status?host=github&repo=bjwschaap/alpine-lift)](https://dependabot.com)
[![Go Report Card](https://goreportcard.com/badge/github.com/bjwschaap/alpine-lift)](https://goreportcard.com/report/github.com/bjwschaap/alpine-lift)
[![Github Action](https://github.com/bjwschaap/alpine-lift/workflows/Go/badge.svg)](https://github.com/bjwschaap/alpine-lift/actions?query=workflow%3AGo)

# Lift

Lift is an Alpine Linux specific light-weight alternative for cloud-init.

<img src="doc/lift.png" width="100px" style="padding: 10px; overflow: auto;" align="left" />

In Alpine environments, one would prefer to take a lift instead of hiking,
running or climbing to the top.

Simply make sure `lift` is run during boot. It will take a url from a passed
in kernel parameter in order to download an `alpine-data` file. This is a
YAML file equivalent to cloud-init's user-data. Lift will download the
alpine-data and perform the initial OS configuration. Lift will run only once,
on first boot of the system, by default.

## Building

To make a statically linked, upx-compressed build suitable for any
recent Alpine version, run:

```shell
make
```

## Usage

In order for `lift` to bootstrap your Alpine node:

* make sure `lift` is in your image (e.g. through `apkovl`), and
* lift is started as a service during boot (provide your own openrc script)
* either pass in a url to the `alpine-data` file with the `-s` parameter to the `lift` binary;
* or pass in a url to the `alpine-data` file trough setting `alpine-data=` kernel boot parameter

During the boot process lift will download the `alpine-data` and configure the instance
accordingly.

## Alpine-data

The downloaded `alpine-data` file can be structured as follows, all keys being optional:

```yaml
password:
timezone:
keymap:
unlift:
motd:
network:
packages:
dr_provision:
sshd:
groups:
users:
runcmd:
write_files:
```

### password

A string with the root password. If not set, the root password will be disabled by default.

### timezone

A string with a valid Linux timezone representation (see: https://wiki.alpinelinux.org/wiki/Setting_the_timezone).
Default: "UTC".

### keymap

A string with the keymap to use. Default: "us us"

### unlift

A boolean indicating if `lift` should delete itself when it's done. Default: `true`.

### motd

A string defining the MOTD/login banner content. If not set or empty, Alpine's default
MOTD will be left in place.

### network

A string used for configuring the network. The contents of this parameter will be
copied into the `/etc/network/interfaces` file. Default:

```text
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
   hostname alpine
```

### packages

A structure containing information about what APK repositories to use, which packages
to install and uninstall, and if `apk update` and/or `apk upgrade` should be executed.

Example:

```yaml
packages:
  repositories:
    - http://dl-cdn.alpinelinux.org/alpine/edge/main
    - http://dl-cdn.alpinelinux.org/alpine/edge/community
  update: true
  upgrade: true
  install:
    - sfdisk
    - linux-utils
  uninstall:
    - lua5.1
```

### dr_provision

A structure containing all information needed to install, and activate, the
Digital Rebar Provision runner process.

Example:

```yaml
dr_provision:
  install_runner: true
  endpoint: {{.ApiURL}}
  assets_url: {{ .ProvisionerURL }}/files
  token: "{{.GenerateInfiniteToken}}"
  uuid: "{{.Machine.UUID}}"
```

This example shows how this block would be added to a Digital Rebar Provision template
for alpine-data (by default on port 8092)

The endpoint must point to the URL of the DRB API. The assets url should point to DRB's
provisioner url where all static files and rendered templates are served (by default on port 8091).

The uuid is the machine uuid, generated by DRB. This uuid is used by the runner process to
'call back' to DRB. This allows for controlling the host from the DRB dashboard/console.

### sshd

A structure containing some basic SSHD configuration settings.

Example:

```yaml
sshd:
  port: 22                         # Port 22
  listen_address: 0.0.0.0          # ListenAddress 0.0.0.0
  authorized_keys:
    - ssh-rsa AAAAB3N...
  permit_root_login: false         # PermitRootLogin no
  permit_empty_passwords: false    # PermitEmptyPasswords no
  password_authentication: false   # PasswordAuthentication no
```

The authorized_keys specified will be appended to the .ssh/authorized_keys file. In essence these
are the keys that will be allowed to login as root through ssh.

### groups

A list of strings with group names that should be created.

Example:

```yaml
groups: [ 'somegroup', 'specialgroup']
```

### users

A list of structures defining users to be created.

Example:
```yaml
users:
  - name: bob
    gecos: a sample user
    password: s3cr3t!
    groups:
      - foo
      - bar
    ssh_authorized_keys:
      - ssh-rsa AAAAB3NzaC1yc2EAAAAD...
  - name: service
    gecos: special service account
    homedir: /opt/service
    shell: /sbin/nologin
    system: true
    primary_group: nobody
```

### write_files

A list of file structures, defining files that should be created by `lift` on first boot. The contents of the file
are either specified in `alpine-data` directly (using `content`), or by specifying a url (using `content-url`).

Example:

```yaml
write_files:
  - path: /usr/local/bin/hello
    content: |+
      #!/bin/sh
      echo "Hello Alpine!"
    permissions: 700
  - path: /etc/license
    content-url: https://www.gnu.org/licenses/lgpl-3.0.txt
    owner: nobody:nobody  # chown format
    permissions: 0644
```

### runcmd
A list of strings with shell commands to be executed just before `lift` exits. The commands will
be executed in the order they are specified. The commands are subshelled through `sh` so interpollation
of variables/subcommands is possible.

Example:
```yaml
runcmd:
  - service docker start
  - sleep 2s
  - docker run -d --rm -p 80:80 nginx
  - docker run -d --rm -p 8080:8080 --name cadvisor -v /:/rootfs:ro -v /var/run:/var/run:ro -v /sys:/sys:ro -v /var/lib/docker/:/var/lib/docker:ro -v /dev/disk/:/dev/disk:ro google/cadvisor:latest
  - echo $(date) > /etc/test
```

Since `runcmd` is the last block to execute, it's possible to combine it with `write_files` to e.g. add scripts
and execute them. This allows for a high level of customization.

## Contributors

* [hblanks](https://github.com/hblanks)
