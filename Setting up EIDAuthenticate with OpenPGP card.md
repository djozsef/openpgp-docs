Setting up EIDAuthenticate with OpenPGP card
============================================

This document will guide you trough the process of initializing a blank OpenPGP card and setting it up to be used for smart card based Windows logon with My Smart Logo EIDAuthenticate. 

**[Jump to TL;DR](#tldr)**

**Pre-requisites:**

 - Blank OpenPGP card (OR access to private key backup)
 - My Smart Logon's EIDAuthenticate credential provider
 - Ubuntu Desktop or Live CD (or any distro that you can get the tools below up and runing)
 - GnuPG v2.x 
 - up-to-date OpenSSL
 - openpgp2ssh ([The Monkeysphere Project](http://web.monkeysphere.info/), part of the monkeysphere package on Ubuntu)

**What we will do:**

 1. [Generate keys for the OpenPGP card](#generate-keys) (or import existing secure keyring)
 2. [Write keys to the card](#write-keys-to-card)
 3. [Create an X509 certificate in binary DER format and write it to the card](#create-cert).
 4. [Set up EIDAuthenticate](#set-up-eidauthenticate)

Actually you would be able to do the whole process in Windows environment except exporting the GPG keys into PEM format. For that we need the openpgp2ssh tool that is part of the monkeysphere package on main Linux distros. AFAIK this tool is written in Perl so it could be possible to port i to Windows but for now I'll go for the stable package.
You can skip the key generation steps if you already have a working GPG keyring either on your computer or your card. You will still have to have access to the actual private keys to be able to export one into PEM format. So if you exiled your private key into a safe it's time to fetch it for a short time.

<a name="generate-keys">1. Generate keys</a>
----------------
If you have a blank OpenPGP card you should start with generating keys:

    $ gpg --gen-key

Complete the key generation process. Make sure you choose "`RSA and RSA (default)`" for key type and at least a 2048 bit length. Latest OpenPGP V2 cards support 4096 bit length keys, I recommend using this size.
At the end of the key generation process, you will see something like this:

    pub   4096R/ABCD1234 2015-01-01
          Key fingerprint = 1234 5678 9ABC DEF0 1234 5678 9ABCD EF01 ABCD 1234
    uid       [ultimate] John Doe
    sub   4096R/4321DCBA 2015-01-01

You should note your main key ID, that is the `ABCD1234` part in the above example. 
The inner psychics of the GPG system is out of the scope of this document, but you should know that GPG works with one main key and several subkeys. The main key is only used for a narrow set of purposes, while the subkeys are used in real life situations. 
After key generation you have a main key and one subkey for encryption. Keys are assigned one or more capabilities of the following:  Sign, Encrypt, Authenticate. The OpenPGP card fits this scheme: it has 3 slots for a Signature, Encryption and an Authentication key. Since after the key generation you already have an Encryption key, you just have to add a Sign and an Encrypt capable key. 

> Please note that one key can be assigned multiple capabilities. 
> **This is a key factor to our goal to set up EIDAuthenticate: You have to assign all capabilities to your Authentication key.**

So our goal is to have a Signature and an Encryption key, plus an additional one that has all three capabilities: Sign, Encrypt, Authenticate. To add additional subkeys you have to edit your main key. To do so:

    $ gpg --expert --edit-key ABCD1234

You have to provide the main key ID to edit, so replace `ABCD1234` with your actual main key ID. We need the `--expert` option to be able manually select key capabilities. In the GPG console issue:

    gpg> addkey

You will be asked for the passphrase for the key, then the type of the subkey you want to add. Select:

    (8) RSA (set your own capabilities)

In the next step you can select the capabilities. Use the menu to assemble the desired capability sets and select `Q` when done. You will be asked a key length and a validity period too, these must be familiar after the main key generation. 
Repeat this step until you have 3 subkey with the following flags: `S`, `E`, `SEA`.

> **Please not that each time you make changes to your keyring you have to save them by issuing:**

    gpg> save

This is a good point where you make backups of yout GPG keyring:

    cp ~/.gnupg/secring.gpg ~/.gnupg/secring.gpg.backup
    cp ~/.gnupg/pubring.gpg ~/.gnupg/pubring.gpg.backup

> If you already have a working keyring with private keys on the card this is the point where you should re-import your backups to be able to export the private key.

Our final goal is to have an X509 certificate written to the OpenPGP card into the designated slot. But wait, GPG and X509 are two different worlds! No worries, here is the point where The Monkeysphere Project comes to our help. This tool does the translation between the two worlds: converts the exported GPG keys into PEM format. To do so first we have to temporarily remove the passphrase form the keyring because the openpgp2ssh tool cannot handle encrypted keys:

    $ gpg --edit-key ABCD1234
    gpg> passwd
Enter empty input for the new passphrase to clear it. 

> Please note that we need to export the Authentication subkey with all (`SEA`) capabilities. To get the Auth subkey ID just edit the main key and GPG console tool it will list the key ID's with their capabilities:

    $ gpg --edit-key ABCD1234
Copy the key ID with marked with "`usage: SEA`". Now you are ready to do the magic, export the subkey. **Use your subkey ID in the following command**:

    $ gpg --export-options export-reset-subkey-passwd,export-minimal,no-export-attributes --export-secret-keys --no-armor 0xEFGH5678! | openpgp2ssh EFGH5678 > EFGH5678.key
Now we have the private key of our Authentication key in PEM format. 
We should re-set a passphrase to our keyring before we forgot it:

    $ gpg --edit-key ABCD1234
    gpp> passwd

Enter the original or desired passphrase. 

> If you used key backups this is where you can remove the secret key from the keyring and put your backups back to exile.

<a name="write-keys-to-card">Write keys to the card</a>
----------------------
OpenPGP card has 3 key slots, each capable to store a maximum 4096 bit long key. You can store keys to these slots with the corresponding capabilities: Signature, Encrypt and Authenticate. When you add a key to your card, the secret key material is written to the corresponding card slot and removed from the local keyring, leaving a so-called *stub* in the keyring pointing to the card slot. This way each time you want to use the private key, the card is asked for.
To add your keys to the card use the GPG console and **edit your main key**:

    $ gpg --edit-key ABCD1234
After entering the GPG console you will see your keys listed:

    pub  4096R/ABCD1234  created: 2015-02-12  expires: never        usage: SC
    				     trust: unknown validity: unknown mode
    sub  4096R/CCCC3333  created: 2015-02-12  expires: never        usage: S
    sub  4096R/DDDD4444  created: 2015-02-12  expires: never        usage: E
    sub  4096R/EFGH5678  created: 2015-02-12  expires: never        usage: SEA
    [unknown] (1). John Doe (jdoe) <john@doe.net>

To write (and move) a key to the card first you have to select which subkey you would like to move. To do so use the `key KEYNUM` command where KEYNUM is the number of the key index counting from the top and starting with zero. So to select the Signature key in the example above I use:

    gpg> key 1
        pub  4096R/ABCD1234  created: 2015-02-12  expires: never        usage: SC
        				     trust: unknown validity: unknown mode
        sub* 4096R/CCCC3333  created: 2015-02-12  expires: never        usage: S
        sub  4096R/DDDD4444  created: 2015-02-12  expires: never        usage: E
        sub  4096R/EFGH5678  created: 2015-02-12  expires: never        usage: SEA
        [unknown] (1). John Doe (jdoe) <john@doe.net>
Note that the selected key is marked with a `*`. Now that you selected the key you want to transfer to the card, issue the actual command that does so:

    gpg> keytocard
You will be asked which slot to write the key to, select the option that fits your key's capabilities.
Iterate over these steps to transfer all 3 keys to the card. Please note that you can toggle selected flag on multiple keys, so make sure you select only the one you want to transfer. 
After a successful key transfer you should see a key list like this:

    $ gpg -K
    sec   4096R/ABCD1234 2015-01-01
    uid                  John Doe (jdoe) <john@doe.net>
    ssb>  4096R/CCCC3333 2015-01-01
    ssb>  4096R/DDDD4444 2015-01-01
    ssb>  4096R/EFGH5678 2015-01-01
Note that a `>` has appeared in your subkey entries, this means that your local secure keyring does not have that private key, only a stub that points to the actual private key on the your card.

<a name="create-cert">Create an X509 certificate</a>
--------------------------
At this point we have our Authentication private key in PEM format. What we need is an X509 certificate in binary DER format, because that is what OpenPGP card can store in its special certificate slot. The theory of certificates is way beyond of the scope of this document, but in short:

 - we need to generate a Certificate Signing Request (CSR)
 - need to sign the certificate
 - convert the base64 encoded PEM certificate into binary DER

To create a CSR use the OpenSSL console tool with the `EFGH5678.key` PEM private key file we fetched before:

    $ openssl req -new -key EFGH5678.key -out EFGH5678.csr
You will have to provide entity details for the CSR. For the sake of simplicity you can skip setting a request challenge since we are making a self-signed certificate. This will create the `EFGH5678.csr` containing the the certification request along with the public key. Now it is time to self-sign the CSR with the same private key:

    $ openssl x509 -req -days 3650 -in EFGH5678.csr -signkey EFGH5678.key -out EFGH5678.crt
This will create the PEM certificate in `EFGH5678.crt`. You can override validity period by setting the `-days` parameter. Next step is to convert our fresh and crispy PEM certificate into binary DER format:

    $ openssl x509 -outform der -in EFGH5678.crt -out EFGH5678.der

This will result in a `EFGH5678.der` file which is ready to be written to the card. To do so we need to edit the card with GPG console tool:

    $ gpg --card-edit

To be able to make actual modifications to the card switch to admin mode:

    gpg> admin

Then use this command to write the certificate to the card:

    gpg> writecert 3 < EFGH5678.der

This is it! Now you have an OpenPGP card filled with keys and an X509 certificate of your Authentication key with full capabilities. 

 

<a name="set-up-eidauthenticate">Set up EIDAuthenticate</a>
----------------------
Now that you have a properly configured OpenPGP card you can proceed to set up EIDAuthenticate. I will not cover this process while Vincent Le Toux from [My Smart Logon](http://www.mysmartlogon.com/) already made a pretty straightforward video presentation about it: https://www.youtube.com/watch?v=FsjlTxKL1x8

<a name="tldr">TL;DR</a>
-----

The most important factors to get OpenPGP card working with EIDAuthenticate:

 - You need to have an Authentication key with *ALL* capabilities (`SEA` flags) on your card.
 - you need to export your private Auth subkey carefully without any additional info, just the bare key material. Use openpgp2ssh tool from monkeysphere package.
 - Create a self signed certificate with that private key and write it to the card in binary DER  format.

To export the Auth subkey into PEM:

    $ gpg --export-options export-reset-subkey-passwd,export-minimal,no-export-attributes --export-secret-keys --no-armor 0x{SUBKEY_ID}! | openpgp2ssh {SUBKEY_ID} > {SUBKEY_ID}.key

To write the cert to the card:

    $ gpg --card-edit
    gpg> admin
    gpg> writecert 3 < certfile.der

## _ ##

> Written by [Dubravszky József](https://twitter.com/djozsef), CTO at [Chili Creative Solutions](http://chilicreative.hu/).
> Licensed under GPLv3 
> Any comments, improvements or bug report are welcome. 