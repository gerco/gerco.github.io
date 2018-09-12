---
layout: post
title: What every developer must know about TLS (no excuses) 
---

Transport Layer Security (TLS) or, as many still call it: SSL, is a cornerstone technology of the internet. It enables internet banking, many b2b applications and secure communications with your doctor. With the advent of free certificates from [Let's Encrypt](https://letsencrypt.org), there are no more excuses for websites not to support it. As a result many websites have adopted HTTPS, the secure version of HTTP (or HTTP over TLS). Sadly, many website administrators and developers don't quite understand how it all works and stumble through tutorials, settings, certificate files and checkboxes hoping to get it to work. When it finally works, they leave it all alone until it inevitably breaks and the process of trial and error begins anew.

To prevent too many people from making the same mistakes over and over again, I decided to write this article. Much of this is old news to many people but it's my hope that there is a nugget of missed understanding here and there that will be useful. This article is a high-level overview and in no way attempts to convey technical details about the underlying protocols.

## TITLE HERE

TLS is a communications protocol with two goals: authentication and privacy. The goal is to know who you're talking to and to make sure nobody else can listen in on the conversation. To understand how TLS achieves these goals, we first have to go over some basics.

### Privacy

When two parties want to exchange information without anyone finding out what that information is, that information can be encoded in a way that can be decyphered by the communicating parties but not by anyone else. An uncountable number of schemes have been divised for achieving this, ranging from the simple Ceasar Cipher to full blown quantum cryptography. Most of these schemes are not (yet) relevant to TLS, however, so we'll limit ourselves to the RSA system.

In the RSA cryptography system, the two parties (traditionally called Alice and Bob) that want to communicate first both create something called a **key pair**, this key pair consists of a **public key** and a **private key** (both are essentially very large numbers). The specifics are not important here, but suffice it to say that when Alice encrypts a message with Bob's **public key**, it can only be decrypted by Bob's **private key**. Conversely when Bob wants to send a message to Alice, he needs to encrypt it with her **public key** so Alice can later decrypt it with her **private key**.

Aside from encrypting messages, RSA can also be used to *sign* messages. A message signature proves who wrote the signed message and that the messages has not been altered after having been signed. In order to sign a message, Alice will compute a [hash](https://en.wikipedia.org/wiki/Cryptographic_hash_function) of the message and encrypt that hash with her **private key**. Anyone that has Alice's **public key** can then verify the signature and know the message was signed by Alice and has not been modified.

To learn more about RSA and other asymmetric crypto systems, see [Public-key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography) on Wikipedia.

### Authentication

Privacy by itself is not enough for secure communications. An eavesdropper may not be able to listen in on the connection after it's been established, but it's still perfectly possible for a [Man in the Middle](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) (MITM) to intercept the connection and establish key pairs with both sides of the connection. The MITM is then able to decrypt all traffic from Alice and re-encrypt it before sending it off to Bob. Alice and Bob are none the wiser without the second important property of TLS: authentication.

In TLS, authentication is achieved by using certificates (see [X.509](https://en.wikipedia.org/wiki/Certificate_authority)). A certificate is simply a **public key** bundled with some information about its owner that is signed by a trusted third party (a [certificate authority](https://en.wikipedia.org/wiki/Certificate_authority)). Think of it like a passport, a passport can be used to prove someone's identity because it was issued by a trusted third party (that person's government).

Alice can prove that her certificate (that contains her public key) is hers because she has the corresponding private key. A man in the middle cannot pretend to be Alice because they do not have Alice's private key.

## Basics of a TLS connection

A TLS connection begins when a client (i.e. a web browser) sends a [`Client Hello`](https://tools.ietf.org/html/rfc5246#section-7.4.1.2) message to a server (a web server). This message contains some basic information about the connection the client wants to establish: the protocol versions supported, a list of supported encryption protocols, hashing algorithms, etc. It contains everything the server could possibly want to know about what the client supports so it's free to choose the most secure combination of algorithms both client and server support.

After receiving the `Client Hello`, the server will reply with its own [`Server Hello`] (https://tools.ietf.org/html/rfc5246#section-7.4.1.3) message that informs the client of the chosen encryption- and hashing algorithms, various parameters and the all important **Server Certificate**.

The server certificate contains the server's public key and information about the server that owns that key (such as the company that owns it, etc). Most importantly, it contains the server's name (called the "Common Name" or CN). Any TLS client *must* compare the name in the certificate with the name they were trying to connect to and reject the connection if they don't match to prevent man in the middle attacks.

After the `Server Hello` and associated messages, the client must now decide whether to trust the certificate the server has offered or abort the connection, more about that later. The client can also send a certificate of its own in a [`Client Certificate`](https://tools.ietf.org/html/rfc5246#section-7.4.6) message. This is used for [Mutual authentication](https://en.wikipedia.org/wiki/Mutual_authentication), which is more rare and outside the scope of this article.

## Chain of trust

How does a TLS client know whether to trust the certificate a server has offered? As you will remember, the certificate contains a public key and some information about the owner of that key, such as its name. In the case of a web server this name will be the domain name of the server, something like "somecompany.com". The certificate can also contain some alternative names, such as "www.somecompany.com" so the certificate can be used for multiple domain names.

A certificate also contains validity dates, an **issuer name** and a **signature** from that issuer. The certificate for "somecompany.com" therefore looks something like this:

| Item        | Example                              |
| ----------- | :----:                               |
| Name(s)     | somecompany.com, www.somecompany.com |
| Valid from  | December 1, 2017                     |
| Valid until | December 1, 2020                     |
| Public key  | *a very large number*                |
| Issuer      | TrustUs Inc. Certificate Authority   |
| Signature   | *--- signed by TrustUs Inc. ---*     |

In order for a client to trust a certificate it must be valid at the time of connection; it must belong to the server the client is trying to connect to and; it must have been signed by a certificate authority the client trusts. Every TLS client is configured with a **trust store**, a list of certificate authorities that it trusts. Only certificates signed by a trusted authority will be accepted.

For web browsers, the trust store can be built in to the operating system (Internet Explorer, Google Chrome or Safari) or the browser vendor can provide its own (Firefox). Typically, there exists some management interface to add and remove certificates manually (Keychain Access on the Mac, the Certificate Manager SMC snap-in) on Windows but this store is most often maintained by the operating system or browser vendor. For non-web browser applications, the trust store is often configured separately. However the setup for any particular operating system or application, there is always a trust store containing the trusted certification authorities.

Let's examine a real world certificate. For illustration purposes, we'll use the certificate for github.com:

![github.com certificate]({{ site.baseurl }}/images/github-dot-com-certificate.png)

This certificate's chain of trust is a little more complicated than the above description. The certificate for "github.com" was signed by the issuer "DigiCert SHA2 Extended Validation CA", which in turn is signed by the issuer "DigiCert High Assurance EV Root CA". In many situations, certificate authorities use "intermediate" certificates. These intermediate certificates often have shorter validity dates than their main certificate and are only used for signing certain types of certificates (in this case only for [Extended Validation](https://en.wikipedia.org/wiki/Extended_Validation_Certificate) certificates, a type of certificate with strict validation of the company it represents).

When connecting to github.com, the server will send us **both** its own certificate and also the above intermediate certificate. It may send more, enough to form a "chain of trust" all the way up to a certificate that is included in the client's trust store. In order for a client to check the validity of the server's certificate it has to be able to check the validity of all certificates leading up to a trusted root certificate (the ones in its trust store). When connecting to github.com, a web browser will first check whether the github.com certificate's signature is valid and then move up the chain to the intermediate certificate (in this case "DigiCert SHA2 Extended Validation CA"). It will then validate that intermediate certificate's signature and notice that it's signed by the trusted root "DigiCert High Assurance EV Root CA". This root certificate is explicitly trusted because it's included in the browser's trust store.

## Useful commands

## Troubleshooting

