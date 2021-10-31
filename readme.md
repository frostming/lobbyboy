# Lobbyboy

> What is a lobby boy? A lobby boy is completely invisible, yet always in sight.
> A lobby boy remembers what people hate. A lobby boy anticipates the client's
> needs before the needs are needed. A lobby boy is, above all, discreet to a
> fault.
>
> <p align='right'>--The Grand Budapest Hotel</p>

[![Test](https://github.com/laixintao/lobbyboy/actions/workflows/unittest.yaml/badge.svg)](https://github.com/laixintao/lobbyboy/actions/workflows/unittest.yaml)

**This project is still under testing, it worked but may have bugs.**

<!-- vim-markdown-toc GFM -->

* [What is lobbyboy?](#what-is-lobbyboy)
* [Key Features](#key-features)
* [Installation](#installation)
  * [Systemd Example](#systemd-example)
  * [Run in Docker](#run-in-docker)
* [Providers](#providers)
  * [Builtin Providers](#builtin-providers)
    * [DigitalOcean Provider](#digitalocean-provider)
  * [Write Your Own Providers](#write-your-own-providers)
  * [Publish Your Own Providers](#publish-your-own-providers)
* [FAQ](#faq)
* [I Want to Know More!](#i-want-to-know-more)

<!-- vim-markdown-toc -->

## What is lobbyboy?

Well, lobbyboy is a ssh server. Yes, like `sshd`. But instead of spawn a new
shell on the server like sshd, when you ssh to lobbyboy, lobbyboy will create a
new server(VPS) from available providers(meaning to say, DigitalOcean, AWS, GCP,
Vultr, etc), then redirect you to the newly created servers. Of course, if
lobbyboy finds any servers available already, he will just ask if you want to
enter the existing server, or still want to create a new one.

```
                                                       create
 +------------------+          +--------------------+  new server  +--------------------------+
 |                  |          |                    |------------->|                          |
 User(You!)         |--ssh----->    lobbyboy        |              |  DigitalOcean            |
 |                  |          |                    |------------->|  (or any other providers |
 +------------------+          +--------------------+     ssh      +--------------------------+
```

## Key Features

- talks in SSH2 protocol, no need to install any software of configs for
  client-side, just ssh to lobbyboy!
- extendable provider: just implement 3 methods, then lobbyboy can work with any
  provider!
- destroy the server when you no longer needed.
- manage ssh keys for you

## Installation

Install libkrb5-dev first, this is a dependency for gssapi support.

```bash
apt install libkrb5-dev
```

Install via pip:

```bash
pip install lobbyboy
```

Then generate config file:

```bash
lobbyboy-config-example > config.toml
# Edit your config before running!
```

Run server

```bash
lobbyboy-server -c config.toml
```

You can ssh to Lobbyboy now, if you keep the default user `Gustave` in default
config. You can ssh to Lobbyboy via:

```bash
ssh Gustave@127.0.0.1 -p 12200 -i dev_datadir/test_id_rsa
Welcome to Lobbyboy 0.2.2!
There are 1 available servers:
  0 - Create a new server...
  1 - Enter vagrant lobbyboy-41 127.0.0.1 (0 active sessions)
Please input your choice (number):
```

Lobbyboy is supposed to be a server daemon, so you can manage it by
systemd/[supervisord](http://supervisord.org/) or put it into a docker.

### Systemd Example

### Run in Docker

// TBD

## Providers

// TBD

### Builtin Providers

Lobbyboy current support two Providers:

- DigitalOcean
- Vagrant (Need vagrant and virtualbox to be installed)

Different Providers support different configs, please see the
[example config](https://github.com/laixintao/lobbyboy/blob/main/lobbyboy/conf/lobbyboy_config.toml)
for more detail.

#### DigitalOcean Provider

This Provider will create Droplet from DigitalOcean.

Supported Features:

- Create a new ssh key every time create a droplet.
- Ask user to input region/droplet size/image when creating.
- User can save favorite Droplet region/size/image in configs to quick create.
- Destroy droplet when it is not in use.

Please see
[configs](https://github.com/laixintao/lobbyboy/blob/main/lobbyboy/conf/lobbyboy_config.toml)
to check available options.

```shell
$ ssh lobbyboy.kawabangga.com
Welcome to Lobbyboy 0.2.2!
There is no available servers, provision a new server...
Available VPS providers:
  0 - digitalocean
  1 - vagrant
Please choose a provider to create a new server: 0
Please choose new droplet to create:
  0 - Manually choose a new droplet to create...
  1 - sgp1:s-1vcpu-1gb:ubuntu-21-04-x64
  2 - sgp1:s-1vcpu-1gb:freebsd-12-x64-zfs
  3 - sfo1:s-1vcpu-1gb:fedora-33-x64
Please enter the number of choice: 0
Fetching metadata from digitalocean...
Please choose region:
  0 - New York 1 (nyc1)
  1 - San Francisco 1 (sfo1)
  2 - New York 2 (nyc2)
  3 - Amsterdam 2 (ams2)
  4 - Singapore 1 (sgp1)
  5 - London 1 (lon1)
  6 - New York 3 (nyc3)
  7 - Amsterdam 3 (ams3)
  8 - Frankfurt 1 (fra1)
  9 - Toronto 1 (tor1)
 10 - San Francisco 2 (sfo2)
 11 - Bangalore 1 (blr1)
 12 - San Francisco 3 (sfo3)
Please enter the number of choice: 0
Please choose droplet size:
  0 - s-1vcpu-1gb
  1 - s-1vcpu-1gb-amd
  2 - s-1vcpu-1gb-intel
  3 - s-1vcpu-2gb
  4 - s-1vcpu-2gb-amd
  5 - s-1vcpu-2gb-intel
  6 - s-2vcpu-2gb
  7 - s-2vcpu-2gb-amd
  8 - s-2vcpu-2gb-intel
```

### Write Your Own Providers

Providers are VPS vendors, by writing new providers, lobbyboy can work with any
VPS vendors.

To make a new Provider work, you need to extend base class
`lobbyboy.provider.BaseProvider``, implement 3 methods:

```
  def new_server(self, channel):
      """
      Args:
          channel: paramiko channel

      Returns:
          created_server_id: unique id from provision
          created_server_host: server's ip or domain address
      """
      pass

  def destroy_server(self, server_id, server_ip, channel):
      """
      Args:
          channel: Note that the channel can be None.
                    If called from server_killer, channel will be None.
                    if called when user logout from server, channel is active.
      """
      pass

  def ssh_server_command(self, server_id, server_ip):
      """
      Args:
          server_id: the server ssh to, which is returned by you from ``new_server``
          server_ip: ip or domain name.
      Returns:
          list: a command in list format, for later to run exec.
      """
      pass
```

Then add your Provider to your config file.

Those 3 configs are obligatory, as lobbyboy has to know when should he destroy
your spare servers. You can add more configs, and read them from
`self.provider_config` from code, just remember to add docs about it :)

```
[provider.<your provider name>]
loadmodule = "lobbyboy.contrib.provider.vagrant::VagrantProvider"
min_life_to_live = "1h"
bill_time_unit = "1h"
```

### Publish Your Own Providers

// TBD

## FAQ

Q: Can I use lobbyboy as a proxy, like adding it to my `ProxyCommand` in ssh
config?

A: No. Lobbyboy works like a reverse proxy, meaning to say, for ssh client, it
just like a ssh server(sshd maybe), ssh client get a shell from lobbyboy, and
doesn't know if it is local shell or it is a nested shell which runs another
ssh. (but you know it, right? :D )

## I Want to Know More!

- [介绍 Lobbyboy 项目](https://www.kawabangga.com/posts/4576) (in Chinese)
