---
title: "How To Set Up GPG On macOS"
date: 2024-07-29T12:00:00-07:00
tags:
  - Git
  - GitHub
  - GPG
  - MacOS
  - SSH
categories:
  - Security
draft: false
description: This comprehensive guide provides step-by-step instructions to set up GPG on macOS for software development, covering key generation, SSH authentication, and git commit signing. By following this tutorial, developers can create and manage GPG keys, configure their system for secure coding practices, and deploy keys to GitHub while adhering to best security practices, including removing the primary private key. Essential tools like brew, gnupg, and pinentry-mac are required, ensuring a streamlined setup process for enhanced security in software development.
---

This guide covers everything you need for setting up GPG on macOS for software development. By the end of this guide, you will accomplish the following:

- Create a primary GPG key
- Create an authentication key for use with SSH
- Create a signing key for git commit signatures
- Deploy these keys to GitHub
- Remove the primary private key to follow best practices

Some requirements: 

- You have `brew` installed
- You have a password manager set up

## What is GPG

[GNU Privacy Guard](https://gnupg.org/) (GPG) is an encryption software that enables secure communication and data storage. It is an implementation of the [OpenPGP]() standard, but has become the defacto standard implementation. GPG uses a hybrid approach combining symmetric-key cryptography for speed and public-key cryptography for secure key exchange.  can utilize GPG for signing and verifying documents, encrypting emails, and safeguarding files. Secure email providers like [Proton Mail](https://proton.me) use it for encryption of emails and files. 

It can also be used for signing git commit messages with a cryptographic signature using a private key only known by the developed. This ensures that the commits are coming from the developer, as long as the developer is the only one with the private key used to sign the commit message. A developer's public key can be added to their GitHub and GitLab accounts to verify the validity of the signatures directly in those platforms, usually displaying them in the commit logs with a badge on each signed commit.

{{< img src="images/sign_commit.svg" title="Sign Git Commit" >}}

## Install Tools

Things we need:

- [`gnupg`](https://www.gnupg.org/download/): The GPG Encryption Software
- [`pinentry-mac`](https://github.com/GPGTools/pinentry): Allow for dialog boxes to appear to input your pin for GPG signing

To install using `brew`:

```sh
brew install gnupg pinentry-mac
```

## Set up pinentry

Pin entry needs to be configured for your local machine to prompt for dialog boxes as necessary. To do this, we need to tell the GPG agent which software to use. 

This command will set the necessary `pinentry-program` configuration and then tell the agent to reload to pick up the changes.

```sh
echo "pinentry-program $(which pinentry-mac)" >>  ~/.gnupg/gpg-agent.conf
gpg-connect-agent reloadagent /bye
```

## Create GPG Key

We are now ready to start to create a key. Run the following command to launch the GPG generation wizard.

```sh
gpg --full-generate-key
```

This will then show you a response that looks like this:

```txt
gpg (GnuPG) 2.4.5; Copyright (C) 2024 g10 Code GmbH
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (9) ECC (sign and encrypt) *default*
  (10) ECC (sign only)
  (14) Existing key from card
Your selection? 9
```

We'll use the `(9) ECC (sign and encrypt)` option to use [elliptic-curve cryptography](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography) keys.

```txt
Please select which elliptic curve you want:
   (1) Curve 25519 *default*
   (4) NIST P-384
   (6) Brainpool P-256
Your selection? 1
```

ECC allows for different curves to be used for the generation of the keys. We will be using the default curve, `(1) Curve 25519`.

```txt
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Tue Jul 29 13:58:13 2025 PDT
Is this correct? (y/N) y
```

We now need to set how long the key will be valid for. We will be setting it to 1 year. The key can be updated in the future to extend this duration without needing to generate a new key. Set the valid time to `1y` and select `y`.

Now fill out your personal information about the key. This needs to be a valid email address and one that will need to be configured for your git commits.

```txt
GnuPG needs to construct a user ID to identify your key.

Real name: Alex Norell
Email address: alex@norell.co
Comment: Code Signing and Auth
You selected this USER-ID:
    "Alex Norell (Code Signing and Auth) <alex@norell.co>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
```

Once this is all filled out, you will be prompted to enter a passphrase for your key. **Enter a strong passphrase. This should be a generated string that you will store in your password manager.**

{{< img src="images/pinentry.webp" title="Pinentry Prompt" >}}

After the passphrase has been set, the output will look similar to this.

```txt
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: revocation certificate stored as '/Users/alex/.gnupg/openpgp-revocs.d/CFCF97E03D157D4497D1E9361EFB4E6493D11756.rev'
public and secret key created and signed.

pub   ed25519 2024-07-29 [SC] [expires: 2025-07-29]
      CFCF97E03D157D4497D1E9361EFB4E6493D11756
uid                      Alex Norell (Code Signing and Auth) <alex@norell.co>
sub   cv25519 2024-07-29 [E] [expires: 2025-07-29]
```

Some things to take note of from this output:

1. The fingerprint of the key is located in the `pub` section. In my case, it is `CFCF97E03D157D4497D1E9361EFB4E6493D11756`.
1. A revocation key has been generated and put in the `~/.gnupg/openpgp-revocs.d` folder. The file name is the fingerprint of the key.

You have now created your primary GPG key. With this we will be able to generate additional subkeys that we can scope to different behaviors like signing and authentication.

### Primary key fingerprint

If you don't know what your primary key fingerprint is, run the following command: 

```sh
gpg --list-secret-keys --with-subkey-fingerprints
```

This will output something like this:

```txt {hl_lines=[4]}
~/.gnupg/pubring.kbx
------------------------------
sec   ed25519 2024-07-29 [SC] [expires: 2025-07-29]
      CFCF97E03D157D4497D1E9361EFB4E6493D11756
uid           [ultimate] Alex Norell (Code Signing and Auth) <alex@norell.co>
ssb   cv25519 2024-07-29 [E] [expires: 2025-07-29]
      4541CD6683CE9B7CC17FD1836F5A20E5BB387778
```

The primary key fingerprint is the 40 character hex string below the `sec` key. We'll need this a lot, so lets export it as a variable in our shell environment for ease of use.

```sh
export PRIMARY="CFCF97E03D157D4497D1E9361EFB4E6493D11756"
```

## Primary Keys, Subkeys, and Revocation Keys

A **primary key** is your main cryptographic key that represents your digital identity. It is used to create and manage subkeys, sign other keys, and establish trust in the web of trust model. The primary key is essential for the overall security of your GPG setup, and its loss or compromise can affect the entire key structure. As a developer, you often use the primary key to sign your own public key, ensuring its authenticity and allowing others to verify your ownership of the key.

**Subkeys** are cryptographic keys derived from your primary key, designated for specific tasks like signing commits, encrypting data, or authenticating via SSH. They provide additional security by compartmentalizing different functions, reducing the exposure of your primary key. For instance, you can create a subkey specifically for signing your Git commits, which helps maintain the integrity of your codebase by ensuring that commits are authenticated and traceable. Another common use of subkeys is for SSH authentication, allowing you to securely access remote servers without exposing your primary key.

A **revocation key** is a special key associated with your primary key that can be used to revoke it if it is compromised or lost. This is a critical component of key management, as it ensures that a compromised key cannot be misused indefinitely. For example, if your laptop is stolen and your private keys are at risk, you can use the revocation key to revoke the compromised keys and inform others in your network to stop trusting them. This helps maintain the security and trust within the system, allowing you to generate a new key pair and continue working securely.

## Create SSH Key

We need to create an authentication subkey to use for SSH. If you don't know your primary key fingerprint, refer to [this section](#primary-key-fingerprint).

To create an authentication key, we'll use the `gpg --quick-add-key` command to create an authentication key using the `ed25519` cipher and a 1 year expiration.

```sh
gpg --quick-add-key $PRIMARY ed25519 auth 1y
```

In order to use this key for SSH, we need to allow the key to be used in SSH. This requires putting the keygrip of the auth key into the `sshcontrol` configuration for `gnupg`. A keygrip is a unique identifier for the key that is only used internally for `gnupg`. 

To retrieve the keygrip of the authentication key we need to list out the keys along with their keygrips:

```txt {hl_lines=[8,9]}
gpg --list-secret-keys --with-keygrip $PRIMARY
```

This will then output the keys and the keygrips. Grab the keygrip associated with the `[A]` auth key. 

```txt {hl_lines=[7,8]}
sec   ed25519 2024-07-29 [SC] [expires: 2025-07-29]
      CFCF97E03D157D4497D1E9361EFB4E6493D11756
      Keygrip = 4A762297DD4C564CF3E3721353D83E7F3A60A88D
uid           [ultimate] Alex Norell (Code Signing and Auth) <alex@norell.co>
ssb   cv25519 2024-07-29 [E] [expires: 2025-07-29]
      Keygrip = C33722CC06CDE71A27647C43A4087CB2AF007FD4
ssb   ed25519 2024-07-30 [A] [expires: 2025-07-29]
      Keygrip = BF3D19AE8AA118DA4362BDDC7752810D297F9AD3
```

In my case, my keygrip for the authentication key is `BF3D19AE8AA118DA4362BDDC7752810D297F9AD3`.

We'll use `sed` and `awk` to extract this keygrip and put it into the `~/.gnupg/sshcontrol` configuration file.

```sh
gpg --list-secret-keys --with-keygrip $PRIMARY | \
sed -n '/\[A\]/,/Keygrip/p' | \
awk '/Keygrip/ {print $3}' \
>> ~/.gnupg/sshcontrol
```

Enable SSH support using standard sockets by updating the `~/.gnupg/gpg-agent.conf` file:

```sh
echo "enable-ssh-support\nuse-standard-socket" >> ~/.gnupg/gpg-agent.conf
```

A couple of shell configurations need to be made to allow for GPG to act as an SSH agent. Run the following commands configure your shell:

```sh
echo 'export GPG_TTY="$(tty)"' >> ~/.zshrc
echo 'export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)' >> ~/.zshrc
echo 'gpgconf --launch gpg-agent' >> ~/.zshrc
```

Start a new shell and run the following command to export your public key:

```sh
ssh-add -L
```

The output should look something like this:

```sh
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFQKwKvCIgEYuplYXoEzT7jBGaTwiZ9P5thkvzthCait (none)
```

You now have an SSH key that is generated as a subkey from your primary GPG key.

## Configure Git

We need a signing subkey for git to sign your commits. To do this, run the following command to create the key:

```sh
gpg --quick-add-key $PRIMARY ed25519 sign 1y
```

Now that we have a signing subkey, we need to configure `git` to use the key to sign commits. To do this, we first need to know the fingerprint of the signing key. Retrieve it using the following command:

```sh
gpg --list-keys --with-subkey-fingerprints $PRIMARY
```

The fingerprint is locating under the key with the `[S]`.

```txt {hl_lines=[8,9]}
pub   ed25519 2024-07-29 [SC] [expires: 2025-07-29]
      CFCF97E03D157D4497D1E9361EFB4E6493D11756
uid           [ultimate] Alex Norell (Code Signing and Auth) <alex@norell.co>
sub   cv25519 2024-07-29 [E] [expires: 2025-07-29]
      4541CD6683CE9B7CC17FD1836F5A20E5BB387778
sub   ed25519 2024-07-30 [A] [expires: 2025-07-29]
      248366FB94151E251C74B2B83604FD5A162A94E8
sub   ed25519 2024-07-30 [S] [expires: 2025-07-29]
      F52BA161D0B04D4D6033F41BE4380C50A72281A6
```

In my case, the fingerprint of my signing key is `F52BA161D0B04D4D6033F41BE4380C50A72281A6`.

Now that we know the fingerprint, we can configure git to use it. Rather the copy and paste the string into the command, we will instead use `sed` and `awk` to directly retrieve it when running the `git` configuration commands.

```sh
export SIGNING_KEY=$(\
  gpg --list-keys --with-subkey-fingerprints $PRIMARY | \
  sed -n '/\[S\]/,+1p' | \
  awk '/^[[:space:]]+[0-9A-F]{40}/ {print $1}'\
)
git config --global commit.gpgsign true
git config --global tag.gpgSign true
git config --global user.signingkey $SIGNING_KEY
```

`git` is now configured to use GPG to sign all commits and will prompt for passphrase input. We can check this by running the following commands to create a new temporary repository, make a commit, and then delete it.

```sh
mkdir /tmp/gpg_test
cd /tmp/gpg_test
git init
touch test.txt
git add test.txt
git commit -a -m "Testing signed commit"
git --no-pager log --show-signature
cd ~
rm -rf /tmp/gpg_test
```

Which will output something like this

```txt
commit 5852370fb076b5db0f2ffc23f798f1590ecc98a6 (HEAD -> master)
gpg: Signature made Mon Jul 29 20:47:48 2024 PDT
gpg:                using EDDSA key F52BA161D0B04D4D6033F41BE4380C50A72281A6
gpg: Good signature from "Alex Norell (Code Signing and Auth) <alex@norell.co>" [ultimate]
Author: Alex Norell <alex@norell.co>
Date:   Mon Jul 29 20:47:48 2024 -0700

    Testing signed commit
```

You will see in here the validation of the gpg signature and the key matched the signing subkey we configured.

## Publish to GitHub

Now that we have all our keys generated, lets publish them so we can use them for signing and authentication.

We'll set up SSH authentication first.

1. Export the SSH key using `ssh-add -L`
1. Go to https://github.com/settings/ssh/new and add in your public key

Next up is GPG. We'll first need to export the public key using

```sh
gpg --export --armor $PRIMARY
```

This will look something like this:

```txt
-----BEGIN PGP PUBLIC KEY BLOCK-----
...
-----END PGP PUBLIC KEY BLOCK-----
```

Open up https://github.com/settings/gpg/new and add the output from the previous command.

Your key is now uploaded to GitHub and can be visible publicly at https://github.com/${GITHUB_USERNAME}.gpg.

## Remove the Primary Key

Now that everything is set up, we need to harden our setup. Removing the primary GPG key when using subkeys is a security best practice because it minimizes the risk of the primary key being compromised. Without the primary key installed, you limit the ability to make irreversible changes, such as signing new subkeys, thus reducing the attack surface. The primary key can always be added back in for management tasks like adding new subkeys or expanding expire durations.

Everything in this section is **extremely sensitive**. If the primary private key is leaked, the subkeys have leaked and somebody would be able to generate additional valid subkey.

### Export Private Keys

The following command will export each private key to a separate file.

```sh
gpg --list-keys --with-subkey-fingerprints $PRIMARY | \
awk '/^pub|^sub/ {getline; print $1}' | \
while read -r key_id; do
  gpg --export-secret-keys --armor $key_id > ${key_id}-private.gpg
done
```

Upload these keys to your password manager or add the contents of the files to your password manager as text fields. You should also record the output of `gpg --list-keys --with-subkey-fingerprints $PRIMARY` in your password manager for reference on which key relates to which function.

### Remove Primary Private Key

**This step is unrecoverable if you don't have your private keys exported and backed up**.

This will prompt you to remove the primary key. 

```sh
gpg --delete-secret-keys $PRIMARY\!
```

```txt
sec  ed25519/1EFB4E6493D11756 2024-07-29 Alex Norell (Code Signing and Auth) <alex@norell.co>

Note: Only the secret part of the shown primary key will be deleted.

Delete this key from the keyring? (y/N) y
This is a secret key! - really delete? (y/N) y
```

Approve the prompts and your primary private key will be deleted from GPG.

### Cleaning up Private Key Files

**This step is unrecoverable if you don't have your private keys exported and backed up**.

Clean up the private key files by running the following command from your working directory.

```sh
rm ./*-private.gpg
```

Everything is cleaned up and backed up. GPG is fully configured and ready for you to start using.

## Import Private Key

If you need to import your primary private key for some reason here's how. 

1. Retrieve your private key file from your password manager and put it in your working directory
2. Run the import command: `gpg --import $PRIMARY-private.gpg`
3. Use your passphrase to unlock the key and add it back to GPG.
