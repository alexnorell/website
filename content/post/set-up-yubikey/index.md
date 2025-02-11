---
title: "How To Set Up YubiKey On macOS"
date: 2024-07-30T12:00:00-07:00
tags:
  - Git
  - GitHub
  - GPG
  - MacOS
  - SSH
  - YubiKey
categories:
  - Security
draft: false
description: Learn how to transfer GPG keys to a YubiKey, disable YubiKey OTP, and configure WebAuthn for enhanced developer security. This guide also covers removing SMS authentication to achieve a strong security posture with physical and knowledge-based authentication, perfect for developers.
---

This is a follow up to the [How To Set Up GPG on macOS](/post/set-up-gpg) guide We'll build on the GPG key configuration to move the storage off the machine itself to a physical key. We'll also cover setting up the YubiKey for other services and uses, like WebAuthn.

By the end of the guide you will:

- Disable YubiKey OTP
- Move GPG keys to a YubiKey
- Disable the macOS Smart Card prompt
- Add a YubiKey to Web Services
- Disable SMS Authentication

Some requirements:

- GPG set up [following the guide](/post/set-up-gpg)
- A [YubiKey](https://www.yubico.com/product/YubiKey-5-series/)
- `brew` installed

## What is a Hardware Key?

A hardware key, often referred to as a security key, is a physical device used to secure access to computers, networks, and other systems. Think of it as a more advanced, physical version of a password. These keys are used to authenticate a user's identity and provide a higher level of security compared to traditional password methods. They can be plugged into a USB port or connected via NFC, making them versatile for various devices.

### Authentication Types

When it comes to authentication, there are three primary types:

1. **Know-A (Something You Know):** This is the classic password or PIN that users need to remember. It's the most common form of authentication but also the most vulnerable to attacks like phishing or brute force.

2. **Has-A (Something You Have):** This is where hardware keys come into play. The physical key generates or holds authentication tokens that verify the user. It's akin to having a key to your house; without it, access is denied.

3. **Is-A (Something You Are):** This involves biometric authentication, such as fingerprints, facial recognition, or retina scans. It's unique to the individual and provides a high level of security. This one can also be turned into a Has-A with enough force.

### YubiKey Applications

Your YubiKey offers various types of keys and authentication methods to enhance security. Here are the main ones:

1. **FIDO2/WebAuthn:** This is the latest standard for web authentication, providing strong, phishing-resistant two-factor authentication (2FA) and passwordless login.

2. **FIDO U2F:** An older standard but still widely used, especially for 2FA. It offers an extra layer of security for online accounts.

3. **One-Time Passwords (OTP):** YubiKeys can generate OTPs that are valid for a single session. This includes both time-based OTP (TOTP) and HMAC-based OTP (HOTP).

4. **PIV (Personal Identity Verification):** This is used for smart card functionality, often in corporate environments for secure access to systems and buildings.

5. **OpenPGP:** For those who need secure email and document signing, the YubiKey can store OpenPGP keys.

## Install Tools

Everything installed in the [GPG guide](/post/set-up-gpg/#install-tools) is required. Additionally, we'll need the YubiKey management tool.

- `ykman`: YubiKey management software

Install `ykman` using `brew`

```sh
brew install ykman
```

## Disable YubiKey OTP

Have you ever sent a message like this?

{{< slack
    avatar="/images/avatar.small.webp" 
    username="Alex" 
    timestamp="13:69 PM" 
    text="cccccbngctgrnlreuinvkktedheigunthctueiheifjr" 
    reactions=":rofl:|3,:lock:|1"
>}}


When you touch your YubiKey when it is plugged in, it will type out a string and press enter. This string is a [YubiKey One Time Password](https://www.yubico.com/resources/glossary/yubico-otp/)  (YOTP) that is an alternative to the 6 digit [Time-based One Time Passwords](https://en.wikipedia.org/wiki/Time-based_one-time_password) (TOTP) that you might be familiar with in an Authenticator application.

I personally don't use YOTP for anything and would much rather my YubiKey not type out secure strings when touched. So let's disable it.

Run the following command to disable it.

```sh
ykman config usb --disable otp
```

If your key also has NFC available, you'll probably want to disable that too

```sh
ykman config nfc --disable otp
```

## View YubiKey Contents

We can view the GPG configuration of any connected YubiKey using the following command

```sh
gpg --card-status
```

This will output something like this

```txt
Reader ...........: Yubico YubiKey FIDO CCID
Application ID ...: D2760001240100000006286426520000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 28646942
Name of cardholder: [not set]
Language prefs ...: [not set]
Salutation .......:
URL of public key : [not set]
Login data .......: [not set]
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 0 3
Signature counter : 0
KDF setting ......: off
UIF setting ......: Sign=off Decrypt=off Auth=off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```

As we configure the YubiKey, some of these values will change. 

## Reset Factory Defaults

Your YubiKey ships with insecure default credentials for managing the GPG application. The PIN is `123456` and the Admin PIN is `12345678`. We need to reset these to something that only you know. If we don't change these, somebody could take your YubiKey, unlock the key using the Admin PIN, and then export the private keys.

To make sure start with a fresh configuration run the following command to reset the OpenPGP (GPG) application to factory defaults. 

```sh
ykman openpgp reset
```

The defaults will print on the console once completed

```txt
PIN:         123456
Reset code:  NOT SET
Admin PIN:   12345678
```

### Set Admin Pin

We'll be using the `gpg --card-edit` console for configuring our key for use.

```sh
gpg --card-edit
```

This will launch a `gpg/card` prompt. We need to get access to all of the commands, so we will use the `admin` command

```txt
gpg/card> admin
Admin commands are allowed
```

We will then use the `passwd` command to reset the PIN and Admin PIN. The pin can be between 8 and 127 characters long and can be symbols, letters, and numbers.

```txt {hl_lines=[6]}
gpg/card> passwd
gpg: OpenPGP card no. 0630110000040071660000020d204462 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
```

This will launch the pin-entry prompt:

1. Enter the default admin pin: `12345678`
1. Enter your new pin between `8` and `127` characters
1. Repeat your pin

If everything worked, you'll see

```txt {hl_lines=[1]}
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection?
```

### Set Card Pin

This is the pin you will be using every time you want to sign a commit or authenticate using SSH. Make this short and easy to type.

Continue with where we were in setting the admin pin:

```txt
gpg --card-edit
gpg/card> admin
Admin commands are allowed
gpg/card> passwd
gpg: OpenPGP card no. 0630110000040071660000020d204462 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection?
```

We'll select the first option this time, and this will prompt for setting up pins using pin-entry

1. Enter previous default pin: `123456`
1. Enter your short, simple pin
1. Reenter your new pin

If everything worked, you'll see the same output as before when changing the admin pin.

## Update Card Information

Now that we have pins set up, we can also configure the card with additional information like name, language, username, and public key URL.

From the `card-edit` shell, use the `name` command and fill in the prompted information.

```txt
gpg/card> name
Cardholder's surname: Norell
Cardholder's given name: Alex
```

You can set your default language for use in applications that look for this information.

```txt
gpg/card> lang
Language preferences: en
```

You can set your default username that you use for services. This can be accessed programmatically.

```txt
gpg/card> login
Login data (account name): alex
```

You can also have your key point to your Public PGP key, which you can retrieve at your GitHub username.

```txt
gpg/card> url
URL to retrieve public key: https://github.com/alexnorell.gpg
```

And then quit when you're done.

```txt
gpg/card> quit
```

When you check your card status, you'll see your updated values in there.

```txt {hl_lines=[7,8,10,11]}
Reader ...........: Yubico YubiKey FIDO CCID
Application ID ...: D2760001240100000006286426520000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 28646942
Name of cardholder: Alex Norell
Language prefs ...: en
Salutation .......:
URL of public key : https://github.com/alexnorell.gpg
Login data .......: alex
Signature PIN ....: not forced
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
KDF setting ......: off
UIF setting ......: Sign=off Decrypt=off Auth=off
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```

## Migrate GPG Keys

Now that the YubiKey is configured with pins and additional information, we can add our keys to it. To do this, we'll use the `gpg --edit-key` console, using our primary key fingerprint.

```sh
gpg --edit-key $PRIMARY
```

Which will output something like this

```txt {hl_lines=[5]}
gpg (GnuPG) 2.4.5; Copyright (C) 2024 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret subkeys are available.

pub  ed25519/1EFB4E6493D11756
     created: 2024-07-29  expires: 2025-07-29  usage: SC
     trust: ultimate      validity: ultimate
ssb  cv25519/6F5A20E5BB387778
     created: 2024-07-29  expires: 2025-07-29  usage: E
ssb  ed25519/3604FD5A162A94E8
     created: 2024-07-30  expires: 2025-07-30  usage: A
ssb  ed25519/E4380C50A72281A6
     created: 2024-07-30  expires: 2025-07-30  usage: S
[ultimate] (1). Alex Norell (Code Signing and Auth) <alex@norell.co>
```

We can see that our primary private key is not available because it only says that subkeys are available.

To move our subkey private keys to our YubiKey, we need to select them and use the `keytocard` command.

### Encryption Key

We'll work our way top to bottom in this list by starting with the first key, our encryption key. 

```txt {hl_lines=[10,11]}
gpg (GnuPG) 2.4.5; Copyright (C) 2024 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Secret subkeys are available.

pub  ed25519/1EFB4E6493D11756
     created: 2024-07-29  expires: 2025-07-29  usage: SC
     trust: ultimate      validity: ultimate
ssb  cv25519/6F5A20E5BB387778
     created: 2024-07-29  expires: 2025-07-29  usage: E
ssb  ed25519/3604FD5A162A94E8
     created: 2024-07-30  expires: 2025-07-30  usage: A
ssb  ed25519/E4380C50A72281A6
     created: 2024-07-30  expires: 2025-07-30  usage: S
[ultimate] (1). Alex Norell (Code Signing and Auth) <alex@norell.co>
```

We know it is the encryption key because the `usage: E` line. To select the key, we will type `key 1`

```txt {hl_lines=[6]}
gpg> key 1

pub  ed25519/1EFB4E6493D11756
     created: 2024-07-29  expires: 2025-07-29  usage: SC
     trust: ultimate      validity: ultimate
ssb* cv25519/6F5A20E5BB387778
     created: 2024-07-29  expires: 2025-07-29  usage: E
ssb  ed25519/3604FD5A162A94E8
     created: 2024-07-30  expires: 2025-07-30  usage: A
ssb  ed25519/E4380C50A72281A6
     created: 2024-07-30  expires: 2025-07-30  usage: S
[ultimate] (1). Alex Norell (Code Signing and Auth) <alex@norell.co>
```

A `*` has been added next to the `ssb` identifier, showing that this key is the currently selected key. Now that we have it selected, we can run `keytocard`

```txt
gpg> keytocard
Please select where to store the key:
   (2) Encryption key
Your selection? 2
```

You will be prompted to put in your passphrase for the key as well as your YubiKey admin pin.

At this point, the key is on the card, but it is also still available in 

### Authentication Key

Deselect the first key using `key 1` and select the second key using `key 2`. This key is our authentication key, `usage: A`, so we will put it in that slot

```txt
gpg> key 1
gpg> key 2
gpg> keytocard
Please select where to store the key:
   (3) Authentication key
Your selection? 3
```

### Signing Key

```txt
gpg> key 2
gpg> key 3
gpg> keytocard
Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1
```

### Clean up

At this point, the keys are on the YubiKey, but they are also still available locally. We need to `save` the changes in the `gpg` console to delete the keys locally.

```txt
gpg> save
```

### Review

With that all completed, let's take a look at our card now

```sh
gpg --card-status
```

Which shows us that all the keys were added to the card

```txt {hl_lines=["19-32"]}
Reader ...........: Yubico YubiKey FIDO CCID
Application ID ...: D2760001240100000006286426520000
Application type .: OpenPGP
Version ..........: 3.4
Manufacturer .....: Yubico
Serial number ....: 28646942
Name of cardholder: Alex Norell
Language prefs ...: en
Salutation .......:
URL of public key : https://github.com/alexnorell.gpg
Login data .......: alex
Signature PIN ....: not forced
Key attributes ...: ed25519 cv25519 ed25519
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
KDF setting ......: off
UIF setting ......: Sign=off Decrypt=off Auth=off
Signature key ....: F52B A161 D0B0 4D4D 6033  F41B E438 0C50 A722 81A6
      created ....: 2024-07-30 03:33:49
Encryption key....: 4541 CD66 83CE 9B7C C17F  D183 6F5A 20E5 BB38 7778
      created ....: 2024-07-29 21:03:23
Authentication key: 2483 66FB 9415 1E25 1C74  B2B8 3604 FD5A 162A 94E8
      created ....: 2024-07-30 00:27:00
General key info..: sub  ed25519/E4380C50A72281A6 2024-07-30 Alex Norell (Code Signing and Auth) <alex@norell.co>
sec#  ed25519/1EFB4E6493D11756  created: 2024-07-29  expires: 2025-07-29
ssb>  cv25519/6F5A20E5BB387778  created: 2024-07-29  expires: 2025-07-29
                                card-no: 0006 28642652
ssb>  ed25519/3604FD5A162A94E8  created: 2024-07-30  expires: 2025-07-30
                                card-no: 0006 28642652
ssb>  ed25519/E4380C50A72281A6  created: 2024-07-30  expires: 2025-07-30
                                card-no: 0006 28642652
```

We can also see some information about our keys in the `General key info` section.

The `sec` key is our primary key and the `#` next to it indicates only the public key is available. This makes sense since we removed it from our local system in the last guide.

The `ssb` keys are our subkeys and the `>` next to them indicate the private keys are available externally, and we can see they are located on `card-no: 0006 28642652`.

Everything is in order and we have our keys fully moved over to our YubiKey.

## WebAuthn

WebAuthn is a way to enhance your account security by adding a strong second factor to your login process. When you set it up with your YubiKey, you register the YubiKey as an additional login method on the service you want to use. The YubiKey creates a special cryptographic key pair, with the private key staying safe on the device and the public key being registered with the service. This adds a "has-a" factor to your security posture, meaning you need the physical YubiKey in addition to your password, the "knows-a" factor. This combination significantly boosts your account security, protecting you from phishing and other attacks.

### Add YubiKey to GitHub

Use the power of your YubiKey to add WebAuthn 2FA to your GitHub account.

1. Navigate over to https://github.com/settings/security
1. In the **Two-factor methods** section, show the **Security keys** section.
1. Add a nickname for you security key. I named mine my YubiKey model name.
1. Click add
1. Touch the metal part of your YubiKey
1. Enter in a different 2FA method if prompted

Now you should be able to use your YubiKey as a second factor of authentication.

### Other Services

Yubico, the makers of YubiKey, have a [catalog](https://www.yubico.com/works-with-YubiKey/catalog/?series=1&sort=popular-for-individuals) of services that work with hardware keys. Go through your services and make sure that if it has YubiKey

### Backup

Do make sure to store backup codes for your services somewhere. You can print them out and put them in a booklet, add them to your password manager, or set up an entirely separate password manager just for backup codes.

If you ever lose or damage your hardware key, you will need to either have a backup key available or use one of the provided backup codes to gain access to your account.

## Disable SMS 2FA

SMS 2FA is not secure and should not be used. A phone number can be moved to another phone via your provider using social engineering. It has been linked to dozens of large scale breeches and hacks. Just don't use it. Make sure to go through and remove your phone number from any service as a second factor of authentication.

## Troubleshooting

Make sure you only have one hardware token plugged in at a time. While it is possible to do with multiple cards, this guide assumes only a single key is being configured at a time.

### Card isn't detected by GPG

If your card isn't detected by GPG, make sure the OpenPGP application is enabled for your card.

```sh
ykman info
```

Which should print something like this:

```txt {hl_lines=[13]}
Device type: YubiKey 5C NFC
Serial number: 28646942
Firmware version: 5.7.1
Form factor: Keychain (USB-C)
Enabled USB interfaces: FIDO, CCID
NFC transport is enabled

Applications	USB     	NFC
Yubico OTP  	Disabled	Disabled
FIDO U2F    	Enabled 	Enabled
FIDO2       	Enabled 	Enabled
OATH        	Enabled 	Enabled
PIV         	Enabled 	Enabled
OpenPGP     	Enabled 	Enabled
YubiHSM Auth	Enabled 	Enabled
```

If it doesn't say **Enabled**, then you will need to enable it using

```sh
ykman config usb --enable openpgp
```

### Can't Reset Admin Pin

If you can't reset the Admin pin or don't know your Admin pin, you'll need to completely reset the OpenPGP application on the key. Only do this if you can't set the Admin PIN.

```sh
ykman openpgp reset
```

Make sure that the pin you are trying to set it to is at least 8 characters long and no longer that 127 characters.

### Pin is locked

If the wrong pin is entered on your YubiKey 3 times in a row, it will lock the PGP application completely until the Admin PIN is entered and unblocks the PIN. If you lock your YubiKey and you don't remember your Admin PIN, you will need to completely reset the OpenPGP application and you will lose all of your data on you YubiKey. This includes keys stored on the key.

Launch the GPG card console and edit the card

```sh
gpg --card-edit
gpg/card> admin
gpg/card> passwd
```

You will then see the password options

```txt {hl_lines=[2]}
1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 2
```

Select the `unblock PIN` option and enter your Admin pin.

