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

<!-- TODO -->
## PKCS#11 application structure
## PKCS#11 tooling
## PKCS#11 debugging
