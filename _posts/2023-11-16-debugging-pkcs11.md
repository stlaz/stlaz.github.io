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
   `C_OpenSession()` [interface](https://docs.oasis-open.org/pkcs11/pkcs11-base/v2.40/errata01/os/pkcs11-base-v2.40-errata01-os-complete.html#_Toc441755805)).
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

It's always useful to be looking around tools that deal in the same problem domain
when dealing with standards.

### OpenSC

During my own attempts on PKCS11, I found `pkcs11-tool` to be most helpful when
figuring out that certain operations are even possible, and thus if it's just me
reading the specification wrong or if there's something wonky in a given PKCS11
module implementation.

The `pkcs11-tool` is distributed along with OpenSC packages, most likely on any Linux
distro, or you can get the sources at [the OpenSC GitHub page](https://github.com/OpenSC/OpenSC).

The `man pkcs11-tool` explains the usage pretty well, you can see an example
in the previous section.

### SoftHSMv2

Another thing that one needs when dealing with PKCS11 is the actual tokens that
you can test with.

I originally started with YubiKey 4 nano that I've been keeping unused for a long
time. After a couple of unsuccessful tries, even though I managed to generate
a key pair on it in the end, I decided to rather look elsewhere. The device does
not seem to be fit for use with PKCS11, and many operations appear unsupported.

To be fair, I don't think the YubiKey device I was using was ever supposed to be
used the way I was trying to expose.

I ended up with SoftHSMv2 which is a project I knew from times when I was learning
about DNSSEC in FreeIPA where it used to be leveraged for testing.

SoftHSMv2 allows you to simulate a HSM from software, so that you don't have to
invest into a real token that is still quite expensive if you just want to fool
around.

The usage is quite simple. First you list all slots where you can initialize tokens:
```sh
$ softhsm2-util --show-slots
Available slots:
Slot 0
    Slot info:
        Description:      SoftHSM slot ID 0x0
        Manufacturer ID:  SoftHSM project
        Hardware version: 2.6
        Firmware version: 2.6
        Token present:    yes
    Token info:
        Manufacturer ID:  SoftHSM project
        Model:            SoftHSM v2
        Hardware version: 2.6
        Firmware version: 2.6
        Serial number:
        Initialized:      no
        User PIN init.:   no
        Label:
```

You then use the slot to initialize a token:
```sh
$ softhsm2-util --slot 0 --init-token --so-pin 12345678 --pin 123456 --label "test"
The token has been initialized and is reassigned to slot 607694088
```

This creates a token at slot `607694088` (randomly assigned) that you can then
use as you please. I chose passwords `123456` for normal user and `12345678` for
the privileged Security Officer.

The following example uses the command described in the "PKCS#11 application structure" section,
and then shows that objects were generated on the software token. It also shows
that different objects are visible based on user authentication.

```sh
$ pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so \
    --login --login-type user \
    --keypairgen --id 1 --key-type rsa:2048 \
    --slot 607694088
Logging in to "test".
Please enter User PIN:
Key pair generated:
Private Key Object; RSA
  label:
  ID:         01
  Usage:      decrypt, sign, signRecover, unwrap
  Access:     sensitive, always sensitive, never extractable, local
Public Key Object; RSA 2048 bits
  label:
  ID:         01
  Usage:      encrypt, verify, verifyRecover, wrap
  Access:     local
$ pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so -O --slot 607694088
Public Key Object; RSA 2048 bits
  label:
  ID:         01
  Usage:      encrypt, verify, verifyRecover, wrap
  Access:     local
$ pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so -O \
    --login --login-type user --slot 607694088
Logging in to "test".
Please enter User PIN:
Public Key Object; RSA 2048 bits
  label:
  ID:         01
  Usage:      encrypt, verify, verifyRecover, wrap
  Access:     local
Private Key Object; RSA
  label:
  ID:         01
  Usage:      decrypt, sign, signRecover, unwrap
  Access:     sensitive, always sensitive, never extractable, local
```


## PKCS#11 debugging

TODO:
- PKCS11SPY
