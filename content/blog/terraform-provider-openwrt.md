---
title: "Easter hack: terraform-provider-openwrt"
date: "2025-04-18T14:00:00+02:00"
tags:
- english
- linux
- openwrt
---
April is usualy tax season for most people in Norway, and as I got some ["money
back on the skætt"](https://www.youtube.com/watch?v=ThRXs74EjeE) I wound up purchasing an [OpenWrt One](https://openwrt.org/toh/openwrt/one) to replace my 13-14 year old Asus router. I've been meaning to learn a bit more
about networking in general and getting an OpenWrt router seemed like a fun
project.

Last year I bought a [Beryl AX](https://www.gl-inet.com/products/gl-mt3000/) from GL-Inet as I was travelling for a few
weeks. It's a qute smol travel router that runs a fork of OpenWrt. But during
a recent conference it was reset and I realized I did not have a backup of any
configuration files for the device. Oops!

So getting the OpenWrt One router it seemed like a nice excuse to figure out if
there was any teraform providers available. And quite disappointingly there was
not any out there actively maintained. So I guess we'll have to write our own!

OpenWrt has a JSON-RPC API that allows you to set configuration files, and
options, through a set of calls. So writing a thin wrapper around this API with
a bit of plumbing does not seem to be too difficult.

See: [HowTo: Using the JSON-RPC API](https://github.com/openwrt/luci/blob/master/docs/JsonRpcHowTo.md)

This RPC API writes files under `/etc/config/`. If you write to the `system`
configuration file with a json document, it will parse this and write out a
corresponding `/etc/config/system` file.

Basic provider setup for my Beryl AX:
{{< highlight hcl >}}
terraform {
  required_providers {
    openwrt = {
      source = "foxboron/openwrt"
    }
  }
}

provider "openwrt" {
  user = "root"
  password = "admin"
  remote = "http://192.168.8.1:8080"
}
{{< /highlight >}}

The API requires a bit of setup, but essentially it uses the `root` (or admin
account?) of the device for authentication. Once that is done we can start
writing some resources.

As this was intended to be a quick hack I've squarely only implemented one
resource. The `openwrt_system` resource.

{{< highlight hcl >}}
resource "openwrt_system" "system" {
  hostname = "OpenWrt"
  timezone = "UTC"
  ttylogin = "0"
  log_size = "64"
  urandom_seed = "0"
}
{{< /highlight >}}

This allows you to do things like setting hostname, timezone and some logging
options. It's not a lot, but it is something!

This is neat, but as complete support would require me to implement something
like ~20 odd resources, it takes a while to get off the ground. After staring at
the API for a little while I did realize you have RPC calls to deal with reading
and writing files. This lead me to realize that you could in theory just wrap
the file creation part of the UCI API, and allow people to just write them out
by hand.

For the UCI documentation and supported configuations: https://openwrt.org/docs/guide-user/base-system/uci

{{< highlight hcl >}}
resource "openwrt_configfile" "system" {
 name    = "system"
 content = <<-EOT
 config timeserver 'ntp'
     option enabled '1'
     option enable_server '0'
     list server '0.openwrt.pool.ntp.org'
     list server '1.openwrt.pool.ntp.org'
     list server '2.openwrt.pool.ntp.org'
     list server '3.openwrt.pool.ntp.org'

 config system
     option hostname 'OpenWrt'
     option timezone 'UTC'
     option ttylogin '0'
     option log_size '64'
     option urandom_seed '0'
  EOT
}
{{< /highlight >}}

This is a `openwrt_configfile` resource that just allows you to wite out UCI
configuration files to the OpenWrt device, and call `uci commit` on the
resource. While this doesn't help you to model everything in HCL, it works as a
nice escape hatch to just do the raw configuration instead of waiting for
support in the provider.

I think this is neat and probably solves quite a bit of problems people have
with the existing providers. They model everything and you do not really have
an escape hatch when the support is not implemented.

The source code is here: https://github.com/Foxboron/terraform-provider-openwrt

I've also published it on the [opentofu registry](https://search.opentofu.org/module/foxboron/openwrt/provider/latest), and the [terraform registry](https://registry.terraform.io/providers/Foxboron/openwrt/latest/docs).

Please note I'm probably not going to care about feature requests beyond
what I'll need to configure my OpenWrt One. So I'll probably just accept
everything that comes my way in the form of pull-requests.
