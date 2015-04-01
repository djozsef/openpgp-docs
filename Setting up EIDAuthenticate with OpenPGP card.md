Setting up EIDAuthenticate with OpenPGP card
============================================

This document will guide you trough the process of initializing a blank OpenPGP card and setting it up to be used for smart card based Windows logon with My Smart Logo EIDAuthenticate. 

**[Jump to TL;DR](#tldr)**

**Pre-requisites:**

 - OpenPGP card
 - My Smart Logon's [EIDAuthenticate](http://www.mysmartlogon.com/products/eidauthenticate.html) credential provider
 - My Smart Logon [OpenPGP minidriver](http://www.mysmartlogon.com/products/openpgp-card-mini-driver.html)
 - GnuPG v2.x 
 - up-to-date OpenSSL

**What we will do:**

 1. [Generate keys for the OpenPGP card](#generate-keys) 
 2. [Write keys to the card](#write-keys-to-card)
 3. [Create an X509 certificate in binary DER format and write it to the card](#create-cert).
 4. [Set up EIDAuthenticate](#set-up-eidauthenticate)

You can skip the key generation steps if you already have a working GPG keyring either on your computer or your card.

<a name="generate-keys">1. Generate keys</a>
----------------

> **READ BEFORE YOU START:** 
> It is strongly recommended that you do not do any key management on an online computer. Best is to set up a secure environment without any persistent storage or network connectivity (including Wifi, Bluetooth, IrDA etc.). I use a specially prepared Ubuntu Live CD to generate keys and a USB thumb drive (as key storage) that I never ever plug into any device except the computer running my Live CD. A notebook computer with an easily removable HDD and hardware radio switch is a good choice.

If you have a blank OpenPGP card you should start with generating keys. You have two options here: generate on the card or on the computer. 

**A. Generating on the card**
This method is pretty straightforward while GnuPG will take care of everything for you. If you do not want to have a backup of your private keys, go for this method.

First enter card edit mode:

    gpg --card-edit

Switch to admin mode

    gpg/card> admin

Start generation:

    gpg/card> generate

You will be asked several standard questions, starting with if you want a backup of the first encryption key. 

**Generating on your computer**

When you do want to have a backup of your secret key, go for this method.

To start generation issue:

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

Our goal is to have a Signature and an Encryption key, plus an additional one that can Authenticate. To add additional subkeys you have to edit your main key. To do so:

    $ gpg --expert --edit-key ABCD1234

You have to provide the main key ID to edit, so replace `ABCD1234` with your actual main key ID. We need the `--expert` option to be able manually select key capabilities. In the GPG console issue:

    gpg> addkey

You will be asked for the passphrase for the key, then the type of the subkey you want to add. Select:

    (8) RSA (set your own capabilities)

In the next step you can select the capabilities. Use the menu to assemble the desired capability sets and select `Q` when done. You will be asked a key length and a validity period too, these must be familiar after the main key generation. 
Repeat this step until you have 3 subkey with the following flags: `S`, `E`, `A`.

> **Please note that each time you make changes to your keyring you have to save them by issuing:**

    gpg> save

This is a good point where you make backups of yout GPG keyring:

    cp ~/.gnupg/secring.gpg ~/.gnupg/secring.gpg.backup
    cp ~/.gnupg/pubring.gpg ~/.gnupg/pubring.gpg.backup


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
    sub  4096R/EFGH5678  created: 2015-02-12  expires: never        usage: A
    [unknown] (1). John Doe (jdoe) <john@doe.net>

To write (and move) a key to the card first you have to select which subkey you would like to move. To do so use the `key KEYNUM` command where KEYNUM is the number of the key index counting from the top and starting with zero. So to select the Signature key in the example above I use:

    gpg> key 1
        pub  4096R/ABCD1234  created: 2015-02-12  expires: never        usage: SC
        				     trust: unknown validity: unknown mode
        sub* 4096R/CCCC3333  created: 2015-02-12  expires: never        usage: S
        sub  4096R/DDDD4444  created: 2015-02-12  expires: never        usage: E
        sub  4096R/EFGH5678  created: 2015-02-12  expires: never        usage: A
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
Our final goal is to have an X509 certificate written to the OpenPGP card into the designated slot. But wait, GPG and X509 are two different worlds! No worries, GnuPG has the gpgsm tool that brings the two worlds closer. Among others it is capable of generating a signed CSR for your keys. 

To create the CSR issue:

    $ gpgsm --armor --output EFGH5678.csr --gen-key

First select your key that you want to create the certificate for. You can choose to use a secret key on your local keyring or a key stored on your card.  If you choose `Existing key` you have to enter the keygrip (key ID) of the subkey that you want to create a CSR for. If you choose `Existing key from card`, you have to select the card key. 
At the next step you will be asked for possible actions, choose  `sign, encrypt`. 
Next you have to provide the X509 Subject Name in the standard Distinguished Names ([RFC1779](https://tools.ietf.org/rfc/rfc1779.txt)) format. In this exaple I will use:

    CN=John Doe,EMail=john@doe.net,OU=IT operations,O=Doe and Partners Ltd.,L=New York,C=US
In your Subject Name (SN) string instead of the standard `emailAddress` parameter name use `EMail` otherwise `gpgsm` will reject the SN. Though `gpgsm` will ask for multiple e-mails, SAN's and URL's to include in the CSR, you have to put your e-mail in the Subject Name directly with `emailAddress=` to be able to request an S/MIME certificate. 

If everything goes well, you should have a CSR ready for your Authentication key.

What we need is an X509 certificate in binary DER format, because that is what OpenPGP card can store in its special certificate slot. You can take your CSR to a Certificate Authority or you can sign it for yourself. Technically both works, but a CA issued certificate additionally proves your identity.

To create a  self-signed certificate, you will need a a CA key that you can generate easily. Setting up a local CA for self-signing is out of the scope of this document, but it is well [documented](http://www.freebsdmadeeasy.com/tutorials/freebsd/create-a-ca-with-openssl.php) over the Internets. 

To create your certificate signed with `CA.key` issue:
    
    $ openssl x509 -req -in EFGH5678.csr -CA root.pem -CAkey root.key -CAcreateserial -out EFGH5678.crt -days 3650

This will create the PEM certificate in `EFGH5678.crt`. You can override validity period by setting the `-days` parameter. Next step is to convert our fresh and crispy PEM certificate into binary DER format:

    $ openssl x509 -outform der -in EFGH5678.crt -out EFGH5678.der

This will result in a `EFGH5678.der` file which is ready to be written to the card. To do so we need to edit the card with GPG console tool:

    $ gpg --card-edit

To be able to make actual modifications to the card switch to admin mode:

    gpg> admin

Then use this command to write the certificate to the card:

    gpg> writecert 3 < EFGH5678.der

This is it! Now you have an OpenPGP card filled with keys and an X509 certificate of your Authentication key. 

 
<a name="set-up-eidauthenticate">Set up EIDAuthenticate</a>
----------------------
**Install OpenPGP card middleware**
Out of the box OpenPGP card is not recognized in Windows, so you need to download and install [OpenPGP card mini driver](http://www.mysmartlogon.com/products/openpgp-card-mini-driver.html) from My Smart Logon website. 
To test if the mini driver is installed properly and your card can be used, open a Command Line prompt and enter:

    certutil -scinfo

If you see your reader and a `Card: OpenPGP card` entry, then installation went fine and you are ready to continue. A detailed mini driver test is available on [My Smart Logon website](http://www.mysmartlogon.com/test-the-presence-of-a-minidriver-or-a-csp/).

Now that you have a properly configured OpenPGP card you can proceed to [obtain](http://www.mysmartlogon.com/download/#EIDAuthenticate) and set up EIDAuthenticate. I will not cover this process while Vincent Le Toux from [My Smart Logon](http://www.mysmartlogon.com/) already made a pretty straightforward video presentation about it: https://www.youtube.com/watch?v=FsjlTxKL1x8

<a name="tldr">TL;DR</a>
-----

The most important factors to get OpenPGP card working with EIDAuthenticate:

 - You need to create a CSR with the `gpgsm` tool
 - have your CSR signed by a trust provier CA *OR* create a self-signed certificate with a separate CA key 
 - write the certificate to the card in binary DER format.

To make the CSR:

    $ gpgsm --armor --output EFGH5678.csr --gen-key

To create the certificate:

    $ openssl x509 -req -in EFGH5678.csr -CA root.pem -CAkey root.key -CAcreateserial -out EFGH5678.crt -days 3650

To convert PEM certificate to DER:

    $ openssl x509 -outform der -in EFGH5678.crt -out EFGH5678.der

To write the cert to the card:

    $ gpg --card-edit
    gpg> admin
    gpg> writecert 3 < EFGH5678.der

## _ ##

> Written by [Dubravszky József](https://twitter.com/djozsef), CTO at [Chili Creative Solutions](http://chilicreative.hu/).
> Licensed under GPLv3 
> Any comments, improvements or bugs reports are welcome. Please use [GitHub](https://github.com/djozsef/openpgp-docs/issues).

