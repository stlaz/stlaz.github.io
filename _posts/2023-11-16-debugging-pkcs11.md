---
layout: post
title:  "Debugging and developing with PKCS#11"
date:   2023-11-16 04:27:37 -0600
categories: PKCS11 crypto Golang SoftHSM HSM
---

## Introduction

I am not an expert on PKCS#11 and HSM tokens. These are mostly my notes from when
I was attempting to write a PKCS#11 crypto library for the lolz.

## The buildup

In my company, we've got this concept of "Hack Days" a "Hack week" that happen
every now and then - study what you will, grow your talent, etc.

Some time ago I decided that it would be a good idea to try to build an OIDC server
of my own, because why not - it's a useful exercise to deeply understand the struggle
of those who came up with the protocol spec.

Eventually, I felt that the project was ready to start serving HTTPS. This is when
I remembered an idea that there could be a world where certificates don't live
in filesystem, but instead are hidden behind a PKCS11 interface and could just
as well live in a Hardware Security Module (HSM), where you'll never be able
to get your hands on the physical private key. Wouldn't that be great, eh?

## The specs

Googling PKCS11 RFC you'll find that the actual specification does not live in
an RFC but on the OASIS Open website -
[PKCS #11 Cryptographic Token Interface Base Specification](https://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/pkcs11-base-v2.40.html).

Those of us who are used to reading RFCs that first describe the problem and then
go on to the details of the proposed solution might be a bit surprised by the structure
of the document that literally only just describes the API.

It takes a while to get used to the doc, and for starters I found it to be enough to just
run through it to understand the base concepts.

There is a [PKCS#11 User Guide](https://docs.oasis-open.org/pkcs11/pkcs11-ug/v2.40/pkcs11-ug-v2.40.html)
available to consume from the same website which might actually be a bit more useful to
understand why the API is designed the way it is.

## Golang

Luckily for me, some good people already invested their time and there is a
golang [PKCS#11 library](https://github.com/miekg/pkcs11) that provides C-bindings.
The repo may seem unmaintained but from my experience, the maintainer actually
appears to be watching over it quite vigilantly. That's great!

Now, a good start for serving HTTPS content (or just being a TLS server, really)
would be to be able to:

1. create private keys
2. generate certificates

I know there's much more to it, but hey, let's start small.

There actually is a library that provides everything you may need: [thalesignite/crypto11](https://github.com/thalesignite/crypto11).
In the beginning I did not understand PKCS#11 too well and I missed that this
library actually generates the keys the way you would expect it to, and so I
went and tried implementing it [on my own](https://github.com/stlaz/pkcs11-crypto).
This is what I learnt.

## PKCS#11 application structure

Applications that make use of PKCS11 usually all require the same set of
parameters, reflecting what PKCS11 needs in order to function properly;

1. A path to a PKCS11 module (a dynamic library) that implements the PKCS11 API,
   e.g. `/usr/lib/opensc-pkcs11.so`. These libraries are often vendor-specific,
   although more generic implementations like [OpenSC](https://github.com/OpenSC/OpenSC)
   exist.
1. An identifier of the slot with a token you want to be working with - there can be more than
   one present at a time but you usually only need to pick one (as defined by the
   `C_OpenSession()` [interface](https://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/errata01/os/pkcs11-base-v2.40-errata01-os-complete.html#_Toc441755805).
1. User pin to authenticate the user for the operations they are about to do.

An example with OpenSC's `pkcs11-tool`:
```
pkcs11-tool --modul /usr/lib/softhsm/libsofthsm2.so \
    --login --login-type user \
    --keypairgen --id 1 --key-type rsa:2048 \
    --slot 1035341506
```

This command uses the `libsofthsm2.so` module that's often used for testing as it
emulates a real HSM in software.

A key pair getting generated, so authentication is required (`--login --login-type user`).
There are three types of users defined by PKCS11, those are

- `CKU_SO` - "Security Officer" - a user that's usually responsible for initializing a token
- `CKU_USER` - a user of a token
- `CKU_CONTEXT_SPECIFIC` - the specification seems to imply the user type would be dynamically resolved based on the actions that are being performed.

Generation of cryptomaterial typically requires to be of type "user". Interestingly,
when testing with my YubiKey 4 nano, "Security Officer" authentication was required there.

The above command then specifies that a token in the slot 1035341506 `--slot 1035341506`
should be used for the keygen operation.

## PKCS#11 tooling

TODO:
- pkcs11-tool (openSC)
- libsofthsm

## PKCS#11 debugging

TODO:
- PKCS11SPY
