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
pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so \
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


## PKCS#11 introspection

When writing an application that's using PKCS11 directly, you may end up in
a situation where you think you're following the specification properly but
your token is still returning errors.

In those cases, I found it useful to use the `pkcs11-tool` as a reference implementation.
OpenSC aslo ships with a `pkcs11-spy.so` module. Using these two helps you not only to
make sure that what you're trying to do is possible with the token, but you can actually
see how your PKCS11 is interacting with the token.

In order to work with the `pkcs11-spy.so` module, you need to set the `PKCS11SPY` environment
to the PKCS11 module that you would otherwise use to communicate with your token.
In your application, you then use the `pkcs11-spy.so` module instead of the one you'd normally
use.

The examle below shows extracting a pub key with the SoftHSM2 PKCS11 module. The pub key is then
parsed by `openssl` to show its attributes.

<details>
<summary><b>Click here</b> to view the example. It's quite long so I hid it under this expander.</summary>
{% highlight sh %}
$ PKCS11SPY=/usr/lib/softhsm/libsofthsm2.so pkcs11-tool --module /usr/lib/pkcs11/pkcs11-spy.so \
    --login --login-type user \
    --slot 607694088 --read-object \
    --id 01 --type pubkey \
    -o pubkey.der


*************** OpenSC PKCS#11 spy *****************
Loaded: "/usr/lib/softhsm/libsofthsm2.so"

0: C_GetInterface
P:21960; T:0x126605819501440 2024-02-29 12:33:33.286
[compat]
[in] pInterfaceName 000056238d873ca9 / 7
    00000000  50 4B 43 53 20 31 31                             PKCS 11
[in] pVersion = NULL
[in] flags =
Returned:  0 CKR_OK

1: C_Initialize
P:21960; T:0x126605819501440 2024-02-29 12:33:33.286
[in] pInitArgs = (nil)
Returned:  0 CKR_OK

2: C_GetSlotList
P:21960; T:0x126605819501440 2024-02-29 12:33:33.287
[in] tokenPresent = 0x0
[out] pSlotList:
Count is 2
[out] *pulCount = 0x2
Returned:  0 CKR_OK

3: C_GetSlotList
P:21960; T:0x126605819501440 2024-02-29 12:33:33.287
[in] tokenPresent = 0x0
[out] pSlotList:
Slot 607694088
Slot 1
[out] *pulCount = 0x2
Returned:  0 CKR_OK

4: C_OpenSession
P:21960; T:0x126605819501440 2024-02-29 12:33:33.287
[in] slotID = 0x2438ad08
[in] flags = 0x4
[in] pApplication = (nil)
[in] Notify = (nil)
[out] *phSession = 0x1
Returned:  0 CKR_OK

5: C_GetTokenInfo
P:21960; T:0x126605819501440 2024-02-29 12:33:33.287
[in] slotID = 0x2438ad08
[out] pInfo:
      label:                  'test                            '
      manufacturerID:         'SoftHSM project                 '
      model:                  'SoftHSM v2      '
      serialNumber:           'd623f2572438ad08'
      ulMaxSessionCount:       0
      ulSessionCount:          -1
      ulMaxRwSessionCount:     0
      ulRwSessionCount:        -1
      ulMaxPinLen:             255
      ulMinPinLen:             4
      ulTotalPublicMemory:     -1
      ulFreePublicMemory:      -1
      ulTotalPrivateMemory:    -1
      ulFreePrivateMemory:     -1
      hardwareVersion:         2.6
      firmwareVersion:         2.6
      time:                   '2024022911333300'
      flags:                   42d
        CKF_RNG
        CKF_LOGIN_REQUIRED
        CKF_USER_PIN_INITIALIZED
        CKF_RESTORE_KEY_NOT_NEEDED
        CKF_TOKEN_INITIALIZED
Returned:  0 CKR_OK
Logging in to "test".
Please enter User PIN:

6: C_Login
P:21960; T:0x126605819501440 2024-02-29 12:33:35.119
[in] hSession = 0x1
[in] userType = CKU_USER
[in] pPin[ulPinLen] 000056238e35b760 / 6
    00000000  31 32 33 34 35 36                                123456
Returned:  0 CKR_OK

7: C_FindObjectsInit
P:21960; T:0x126605819501440 2024-02-29 12:33:35.127
[in] hSession = 0x1
[in] pTemplate[2]:
    CKA_CLASS             CKO_PUBLIC_KEY
    CKA_ID                000056238d880340 / 1
    00000000  01                                               .
Returned:  0 CKR_OK

8: C_FindObjects
P:21960; T:0x126605819501440 2024-02-29 12:33:35.127
[in] hSession = 0x1
[in] ulMaxObjectCount = 0x1
[out] ulObjectCount = 0x1
Object 0x2 matches
Returned:  0 CKR_OK

9: C_FindObjectsFinal
P:21960; T:0x126605819501440 2024-02-29 12:33:35.128
[in] hSession = 0x1
Returned:  0 CKR_OK

10: C_GetAttributeValue
P:21960; T:0x126605819501440 2024-02-29 12:33:35.128
[in] hSession = 0x1
[in] hObject = 0x2
[in] pTemplate[1]:
    CKA_KEY_TYPE          00007ffe63e29878 / 8
[out] pTemplate[1]:
    CKA_KEY_TYPE          CKK_RSA
Returned:  0 CKR_OK

11: C_GetAttributeValue
P:21960; T:0x126605819501440 2024-02-29 12:33:35.128
[in] hSession = 0x1
[in] hObject = 0x2
[in] pTemplate[1]:
    CKA_MODULUS           0000000000000000 / 0
[out] pTemplate[1]:
    CKA_MODULUS           0000000000000000 / 256
Returned:  0 CKR_OK

12: C_GetAttributeValue
P:21960; T:0x126605819501440 2024-02-29 12:33:35.128
[in] hSession = 0x1
[in] hObject = 0x2
[in] pTemplate[1]:
    CKA_MODULUS           000056238e396400 / 256
[out] pTemplate[1]:
    CKA_MODULUS           000056238e396400 / 256
    00000000  A1 B8 EA AE F9 2A 2D 6E F3 D6 A3 20 29 F9 9D 5D  .....*-n... )..]
    00000010  EA BB A6 0B 2B 1A 49 CA 5C 54 45 08 78 03 4F D4  ....+.I.\TE.x.O.
    00000020  7F 54 5F 9A EB 5B D6 54 27 A0 BD 74 49 5B 34 2F  T_..[.T'..tI[4/
    00000030  E2 52 C4 31 78 73 1B DB 2F 91 CD 49 E2 E7 3B 97  .R.1xs../..I..;.
    00000040  9E 3F DE 82 25 FC E2 B0 21 C7 43 D6 8F 5D 4B EB  .?..%...!.C..]K.
    00000050  87 BC 79 1A 31 40 6E 52 16 6C FA 0E 89 01 DC 09  ..y.1@nR.l......
    00000060  C7 D3 62 81 30 DC 98 30 18 86 ED 09 D7 9A 05 C1  ..b.0..0........
    00000070  7F BB 8F 62 39 EC 06 C1 46 82 CB 1B E2 50 6E 83  ..b9...F....Pn.
    00000080  22 37 76 48 36 BA 08 D2 C8 84 96 6F 9A 08 AC B8  "7vH6......o....
    00000090  B0 68 90 1D 53 42 A7 05 D4 F0 31 61 98 67 D1 3B  .h..SB....1a.g.;
    000000A0  C3 27 8C 60 27 59 DD 47 0D 09 EF 21 08 07 30 53  .'.`'Y.G...!..0S
    000000B0  5F 56 52 F8 AC 43 08 FF 65 87 BC 14 E4 5B 3E 49  _VR..C..e....[>I
    000000C0  29 7D 67 1E 56 DE A5 64 06 C6 E8 DB 33 87 57 41  )}g.V..d....3.WA
    000000D0  A4 AF F2 D0 F1 36 7B B5 D8 73 3E 70 DD F2 26 63  .....6{..s>p..&c
    000000E0  90 4B 3E FB 37 6B 30 40 61 D4 65 22 8D 5A B2 C0  .K>.7k0@a.e".Z..
    000000F0  46 27 43 AD 99 50 D8 9A 0F 0D 20 EB 29 8A F6 1F  F'C..P.... .)...
Returned:  0 CKR_OK

13: C_GetAttributeValue
P:21960; T:0x126605819501440 2024-02-29 12:33:35.128
[in] hSession = 0x1
[in] hObject = 0x2
[in] pTemplate[1]:
    CKA_PUBLIC_EXPONENT   0000000000000000 / 0
[out] pTemplate[1]:
    CKA_PUBLIC_EXPONENT   0000000000000000 / 3
Returned:  0 CKR_OK

14: C_GetAttributeValue
P:21960; T:0x126605819501440 2024-02-29 12:33:35.129
[in] hSession = 0x1
[in] hObject = 0x2
[in] pTemplate[1]:
    CKA_PUBLIC_EXPONENT   000056238e3965d0 / 3
[out] pTemplate[1]:
    CKA_PUBLIC_EXPONENT   000056238e3965d0 / 3
    00000000  01 00 01                                         ...
Returned:  0 CKR_OK

15: C_CloseSession
P:21960; T:0x126605819501440 2024-02-29 12:33:35.132
[in] hSession = 0x1
Returned:  0 CKR_OK

16: C_Finalize
P:21960; T:0x126605819501440 2024-02-29 12:33:35.132
Returned:  0 CKR_OK

$ openssl asn1parse -in ./pubkey.der -inform der
    0:d=0  hl=4 l= 290 cons: SEQUENCE
    4:d=1  hl=2 l=  13 cons: SEQUENCE
    6:d=2  hl=2 l=   9 prim: OBJECT            :rsaEncryption
   17:d=2  hl=2 l=   0 prim: NULL
   19:d=1  hl=4 l= 271 prim: BIT STRING
{% endhighlight %}

</details>
