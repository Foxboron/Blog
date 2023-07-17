---
title: "Store age identities inside the TPM: age-plugin-tpm"
date: "2023-07-17T18:00:00+02:00"
tags:
- english
- tpm
- security
- golang
---

The past year I have been trying to learn more about the Trusted Platform Module
(TPM). This is a small device found on most modern laptops that has several cool
security features like key creation, sealing and attestation, however I have
been struggling to find a small project where I can learn more about it.

To my surprised I learned a couple of months ago that nobody has written a TPM
plugin for age!

[age](https://github.com/FiloSottile/age/) is a file encryption tool written by Filippo Valsorda to replace the file
encryption feature people utilize from GnuPG[^1]. It has [plugin system](https://github.com/C2SP/C2SP/blob/main/age.md) which
allows you to encrypt to whatever key formate and storage you would need.
Currently the most popular plugin seems to be `age-plugin-yubikey` which allows
key storage on yubikeys.

## Hardware tokens

So why do we want to store key material on hardware like Yubikey or a TPM?

I think the most obvious one is to prevent key extraction or the key leaking.
We are all aware of the funny examples when people confuse a SSH private key
with a public key and share the wrong file. If it's stored as a resident key on
the Yubikey, or stored inside the TPM, leaking this file would be harmless as it
just contains some metadata.

It also prevents a compromised system from getting the key extracted. This means
the active exploiter would need continuous presence on the system to abuse
the key file. Disconnecting the machine from the internet would make the key
material inaccessible to the attacker.

Another cool property of the TPM is that it supports defending against
dictionary attacks. After 3 attempts to brute force the password of the key, the
TPM will lock access to it until the TPM has been reset. Which makes brute
forcing not possible.

This feature allows us to effectively move away from a password, or a
passphrase, to a smaller PIN as we don't need to rely on the entropy any more.

A TPM plugin for age would make the above security aspects more easily
accessible for people. Instead of buying an expensive security key, we can just
utilize the TPM we already have.


## age-plugin-tpm

`age-plugin-tpm` is my first attempt at writing something useable that
interfaces with the TPM. It is written in Golang and uses the [`google/go-tpm`](https://github.com/google/go-tpm)
library which is a native API for interacting with the TPM. This is neat as it
allows us to not rely on external C libraries.

https://github.com/Foxboron/age-plugin-tpm

The crypto primitives is a very close reimplementation of what `age` itself is
doing with it's native keys.

It uses NIST P256 keys with a Elliptic-curve Diffie–Hellman (ECDH) shared secret
which is used inside a HMAC based key derivative function (`hkdf`) to protect
the key used for file encryption and decryption. The ECDH shared secret is done
inside the TPM, while rest is done in Go itself.

To use the plugin one can either install it with `go install`, or download the
pre-compiled binaries and throw them into `$PATH`.

{{< highlight shell >}}
λ » age-plugin-tpm --generate -o - | tee age-identity.txt
# Created: 2023-07-17 17:51:01.821730119 +0200 CEST m=+0.473471400
# Recipient: age1tpm1q045hl84w9hhy2kjvs9tyugfqal083qw9nc60m5pk824u8q3manuuy96qpf

AGE-PLUGIN-TPM-1QYQQQKQQYVQQKQQZQPEQQQQQZQQPJQQTQQPSQYQQYR45HL84W9HHY2KJVS9TYUGFQAL083QW9NC60M5PK824U8Q3MANUUQPQ2NEEA5Q3DST4WUWFQLXH67F50DFQZD4WVA704NMM23HXE7Z2JMASQLSQYZ904GFUL3XLSX8HZ0RM6E852JDUAKZAP63ZAPWC9MSR6MP6NTDD5QQSWMLCU8UKWF4ZEP9GH77R6JHQA6H68WSJ3VCJSHALCCWT7NDNMUJNKTYTQ93KQWJ53J03J2W9Z4GXRK0ZERPR3NDAY5GL7NWCAM7YXA3UH7856J780P9R2ER6G57PU9HJRVQC7VAPDKN5N4YM7D5M6F
{{< /highlight >}}

All identities we create with `age-plugin-tpm` are stored outside of the TPM as
the TPM are small devices with limited memory. The private parts of the TPM
object is wrapped by a parent key which never leaves the TPM and can't be
decrypted outside of the TPM.

We can also optionally include a PIN to the key by passing the `--pin` flag or
setting the `AGE_TPM_PIN` environment variable.

To get the recipient from the identity, we can just pass the identity with the
`-y` flag.

{{< highlight shell >}}
λ » age-plugin-tpm -y age-identity.txt
age1tpm1qfnxa6c5q007tt6pn9h4yz6krf04qusdf76ppxnax0na8h9ygwx7zp4dq3l
{{< /highlight >}}

You can also pipe stuff to encrypt and decrypt things.

{{< highlight shell >}}
λ » echo "Hack the planet" | \
    age -r $(age-plugin-tpm -y age-identity.txt) -a | \
    age -d -i age-identity.txt
Hack the planet
{{< /highlight >}}

If you don't feel like trying to use your TPM when trying out the tool, I've
implemented a `--swtpm` and a `AGE_PLUGIN_TPM_SWTPM` environment variable which
is going to start a new instance of the software TPM `swtpm` and allow you to
try out the plugin. This requires `swtpm` installed to work.

This project uses the new `tpmdirect` API which intends to replace the old
`tpm2` library inside `go-tpm`. This allows us to make use of session encryption
when communicating with the TPM. This prevents a common attack vector where
people with physical access to the machine is capable of sniffing secrets part
of the TPM interaction.

In `age-plugin-tpm` the PIN passed to the TPM through an encrypted session, as
well as the output when we generated the shared secret during the ECDH handshake.

[Chris Fenner](https://github.com/Foxboron/age-plugin-tpm/pull/9), who is the author of this library, was also very kind to look over the API usage :)

Generally I think this project should in a useable state for people. The key
format and the crypto should not see any large revisions. However I would like
to try and implement some PCR sealing, but this is a larger thing I need to
learn more about.

The codebase needs some work in some places, and while `go-tpm` supports Windows
I have only been writing this for Linux. Patches and feature requests are always
welcome!


[^1]: GnuPG was invented in 1990. This has made a lot of people very angry and been widely regarded as a bad move.
