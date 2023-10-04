---
title: "Store ssh keys inside the TPM: ssh-tpm-agent"
date: "2023-10-04T19:00:00+02:00"
tags:
- english
- tpm
- security
- golang
---

After writing [age-plugin-tpm](https://github.com/Foxboron/age-plugin-tpm) a friend of mine at the [hackerspace](https://hackeriet.no) was super
excited to *finally* have easy file encryption with TPM sealed keys, all without
having to rely on `gnupg`. "This is great!" he said.

"I wish I could have my SSH keys sealed in a TPM just as easily".

We should have left it at that.

I shouldn't have replied with a random assortment
of facts like "I know [`google/go-tpm`](https://github.com/google/go-tpm) now", or "but Go has a [`ssh-agent` protocol](https://pkg.go.dev/golang.org/x/crypto/ssh/agent) implementation" followed-up with "Filippo has already implemented [`yubikey-agent`](https://github.com/FiloSottile/yubikey-agent/), it can't be that hard". So I wound up writing a new ssh agent.

https://github.com/Foxboron/ssh-tpm-agent

## Client Keys 
The usual way people seal SSH keys with the TPM is with the PKCS #11[^1] support
in `openssh`, which is... not great? From what I can tell based off on [a blog
post](https://jade.fyi/blog/tpm-ssh/) you will largely encounter a bunch of
commands that only really makes sense if you are familiar with TPMs, which is a
poor basis for useful security tools.

TPM keys allows us to prevent SSH keys being stolen or brute forced and strongly
tie them to the client. This is useful for enterprise settings where you want to
provision a known key to the hardware.

Clearly we can do better.

{{< highlight shell >}}
λ ~ » go install github.com/foxboron/ssh-tpm-agent/cmd/...@latest
go: downloading github.com/foxboron/ssh-tpm-agent v0.1.0

λ ~ » ssh-tpm-keygen
Generating a sealed public/private ecdsa key pair.
Enter file in which to save the key (/home/fox/.ssh/id_ecdsa):
Enter pin (empty for no pin):
Confirm pin:
Your identification has been saved in /home/fox/.ssh/id_ecdsa.tpm
Your public key has been saved in /home/fox/.ssh/id_ecdsa.pub
The key fingerprint is:
SHA256:q2DERbkPtV4ZLj4/QGgxpzC6TZtScUw/cXzO8xsLhDw
The key's randomart image is the color of television, tuned to a dead channel.
{{< /highlight >}}

Done!

Now we just need an agent that will serve our key, this is provided by
the `ssh-tpm-agent`[^2] binary.

{{< highlight sh >}}
λ ~ » ssh-tpm-agent --install-user-units
Installed /home/fox/.config/systemd/user/ssh-tpm-agent.service
Installed /home/fox/.config/systemd/user/ssh-tpm-agent.socket
Enable with: systemctl --user enable --now ssh-tpm-agent.socket

λ ~ » systemctl --user enable --now ssh-tpm-agent.socket
λ ~ » export SSH_AUTH_SOCK="$(ssh-tpm-agent --print-socket)"
λ ~ » ssh-add -L
ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIwh5H2ICVRI1dt4PdusX2E2lRErFyNXFBsWPRCIR9isktm05s43E4uBrpXgQB4+G/F348Xi2hSeJQt3E+vmLTU= fox@framework
{{< /highlight >}}

Lets see if it works!

{{< highlight shell >}}
λ ~ » sudo systemctl start sshd
λ ~ » cat .ssh/id_ecdsa.pub > .ssh/authorized_keys
λ ~ » ssh localhost
Last login: Mon Oct  2 20:45:06 2023 from ::1
λ ~ »
{{< /highlight >}}

It should be noted that only RSA 2048 and ECDSA P256 keys are supported.
Depending on your needs this might not be ideal, however the TPM spec doesn't
give us other key types to implement.

The goal of this tooling is to be fairly simple. It implements it's own
`ssh-tpm-keygen`, `ssh-tpm-agent` and `ssh-tpm-add`. The intention is to closely
mirror the existing openssh tools and refrain from reinventing the wheel when it
comes to the basic usage.

However, it does implement some quality of life improvements.

`ssh-tpm-agent` is going to scan `$HOME/.ssh` for TPM sealed keys (`.tpm`
suffixed files with the `TPM PRIVATE KEY` PEM block). If this feature is not
something you would enjoy, you can use your `ssh_config` to define the
`ssh-tpm-agent` socket as the `IdentityAgent` and pass the *public key* as the
`IdentityFile`. `openssh` won't recognize these weird new `.tpm` keys, however
the public keys are perfectly valid SSH keys.

{{< highlight ssh_config >}}
Host localhost
    IdentityAgent /run/user/1000/ssh-tpm-agent.sock
    IdentityFile ~/.ssh/id_ecdsa.pub
{{< /highlight >}}

You can also use `ssh-tpm-agent` as an ssh agent proxier and collect all your
ssh-agents through one socket. In this example we will point the agent towards
our GnuPG ssh agent, which has a valid SSH key. Any keys not found in our
`ssh-tpm-agent` will be proxied to the next agent until we find a matching key.

{{< highlight sh >}}
λ ~ » ssh-tpm-agent -A "/run/user/9001/gnupg/d.9mnncqwudgr47xbf97yfancp/S.gpg-agent.ssh"
λ ~ » export SSH_AUTH_SOCK="$(ssh-tpm-agent --print-socket)"
λ ~ » ssh-add -L
ssh-rsa d2h5IGFyZSB5b3UgbG9va2luZyBhdCBteSBSU0Ega2V5Cg== cardno:7 800 346
ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBIwh5H2ICVRI1dt4PdusX2E2lRErFyNXFBsWPRCIR9isktm05s43E4uBrpXgQB4+G/F348Xi2hSeJQt3E+vmLTU= fox@framework
{{< /highlight >}}


## Host Keys

Someone on some forum commented it would be cool if `ssh-keysign` could be used
with `ssh-tpm-agent`. `ssh-keysign` is what `sshd` uses to sign authentication
requests with host keys, and at some point I learned that ssh actually defer
private key look ups to the configured Agent if it encounters a public key.

That means we just need to specify `HostKeyAgent` and `HostKey` entries for the
two supported key types in `ssh-tpm-agent`.

{{< highlight shell >}}
λ ~ » cat /etc/ssh/sshd_config.d/10-ssh-tpm-agent.conf
# This enables TPM sealed host keys

HostKeyAgent /var/tmp/ssh-tpm-agent.sock
HostKey /etc/ssh/ssh_tpm_host_ecdsa_key.pub
HostKey /etc/ssh/ssh_tpm_host_rsa_key.pub
{{< /highlight >}}

There is also a `ssh-tpm-hostkeys` binary available to show the current host
keys on the host system, and install user units. Please note this probably needs
to be packaged correctly as it expects binaries under `/usr/bin` and `go
install` does not actually do that.

{{< highlight shell >}}
λ ~ » sudo ssh-tpm-hostkeys --install-system-units
Installed /usr/lib/systemd/system/ssh-tpm-agent.service
Installed /usr/lib/systemd/system/ssh-tpm-agent.socket
Installed /usr/lib/systemd/system/ssh-tpm-genkeys.service
Enable with: systemctl enable --now ssh-tpm-agent.socket

λ ~ » sudo systemctl enable --now ssh-tpm-agent.socket
λ ~ » sudo systemctl restart sshd

λ ~ » sudo ssh-tpm-hostkeys
ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBCLDH2xMDIGb26Q3Fa/kZDuPvzLzfAH6CkNs0wlaY2AaiZT2qJkWI05lMDm+mf+wmDhhgQlkJAHmyqgzYNwqWY0= root@framework
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDAoMPsv5tEpTDFw34ltkF45dTHAPl4aLu6HigBkNnIzsuWqJxhjN6JK3vaV3eXBzy8/UJxo/R0Ml9/DRzFK8cccdIRT1KQtg8xIikRReZ0usdeqTC+wLpW/KQqgBLZ1PphRINxABWReqlnbtPVBfj6wKlCVNLEuTfzi1oAMj3KXOBDcTTB2UBLcwvTFg6YnbTjrpxY83Y+3QIZNPwYqd7r6k+e/ncUl4zgCvvxhoojGxEM3pjQIaZ0Him0yT6OGmCGFa7XIRKxwBSv9HtyHf5psgI+X5A2NV2JW2xeLhV2K1+UXmKW4aXjBWKSO08lPSWZ6/5jQTGN1Jg3fLQKSe7f root@framework
{{< /highlight >}}

`ssh-tpm-keygen -A` will also generated host keys into `/etc/ssh`, which is
similar to what `ssh-keygen` currently supports as well.

There is a slight issue where sshd will use `rsa-sha2-256` or `rsa-sha2-512`
when it *feels* like it. Currently only `rsa-sha2-256` is supported as one TPM
key can only support one hashing algorithm, `ssh-tpm-agent` is currently
not downgrading your crypto from `sha512` to `sha256`.

Amazingly this just works. Now we can strongly tie TPM sealed keys to the host
it self. Either by provisioning ssh keys on the client, or host keys. This makes
a lot of sense as these keys can not be stolen from the host itself and prevents
re-use attacks.

I have also been wondering if there is a way to make some extensions to the
login protocol which would allow ssh host attestation towards clients. This
would be be useful for cases where you are remote unlocking a host and would
like some assurances from the host itself before typing a sensitive password. 

Hopefully this project is useful for more people! There is some release
candidates published and I intend to fix a package for Arch Linux soon'ish,
there is also a Nix package in the works.

Now, I have been wondering if I can hook up `go-tpm` with `uhid`...


[^1]: PKCS #11 was invented in 1994. This has made a lot of people very angry and been widely regarded as a bad move.
[^2]: I'm almost certain I will regret implementing these switches because of packaging complexities.
