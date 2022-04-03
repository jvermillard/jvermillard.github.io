+++
author = "Julien Vermillard"
title = "Notes: X.509 Certificates and the Internet-of-Things"
tags = [ "Internet-of-Things", "X509", "PKI", "security"]
date = "2016-04-13"
+++

Today most of IoT/M2M applications are using passwords, pre-shared keys or maybe no security for device communications. Due to the expected massive deployment scale and the complex logistic behind it, more and more IoT shops want to deploy [Public Key Infrastructure](https://en.wikipedia.org/wiki/Public_key_infrastructure) (PKI) based security. For example the new AWS IoT MQTT broker is enforcing using X.509 certificates for device authentication.

**The goal of this post is to share and write down the current state of my notes on the subject. Don’t expect something very well written :)**

In a nutshell PKI security is based on [Certificate Authorities](https://en.wikipedia.org/wiki/Certificate_authority) (CA) delivering digital certificates to entities. For example an HTTPS based website will be certified by a CA for being the owner of a given Fully Qualified Domain Name (FQDN). If you click on the fancy lock icon in your browser you will see an example of a digital certificate issued by a CA :) For example the one of medium.com is issued by DigiCert and your browser trust DigiCert as a CA.

Now back to Internet-of-Thing: the idea is to put a certificate on all the devices you produce and want to connect. Also your server will get a certificate.

### The first challenge is computation power

Certificates authentication is based on [public key cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography). Implementing the needed algorithms (RSA, DSA, ECDSA) can be a challenge on some targets. Hopefully more and more devices contains hardware accelerators or are not so constrained (32bit MCUs). This one is probably the most frightening at start but ECDSA is not so hard to fit inside embedded targets. Some interesting numbers on real hardware (Cortex-M): https://www.ietf.org/proceedings/92/slides/slides-92-lwig-3.pdf

### Increased network bandwidth usage

This can be critical for cellular connectivity where you pay per consumed kilobyte but also for or Low Power wireless technologies (like 6LowPAN, Thread or LTE-MTC) where the network packet size is constrained.

For example a DTLS handshake using PSK (pre-shared-key) security, will cost ~830 bytes. For a handshake based on X.509 certificates, it’ll cost at least 1500 bytes for a self signed ECDSA one, and more if you are using a complex certification chain. [The Client Certificate URLs extension](https://tools.ietf.org/html/rfc6066#page-9) and [TLS Cached Information Extension](https://datatracker.ietf.org/doc/draft-ietf-tls-cached-info/) can help in one way but not totally solve this problem.

### The hardest issue: Verifying certificate validity

First step is to check the validity time range. A certificate have two dates: “not before” and “not after”. So your device needs to have a time source, and a secure one. Neither NTP, GPS , cellular network may be considered as secure time source. It letting you with little options, like relying on local network or hardware, or a PSK based authentication for a time service. Or simply combining all those strategy in a safe enough heuristic.

More details on this topic: https://www.ietf.org/mail-archive/web/dtls-iot/current/msg00657.html

The second step for validity check is too look for possible revocation of any element in the certification chain. Revocation is the idea of deprecating a certificate before it’s end-of-life date. Possible reasons are a security issue with one of the element of your PKI chain:

* server security issue compromising your private key: for example any bug like OpenSSL [“Heartbleed”](https://en.wikipedia.org/wiki/Heartbleed)
* some cryptography issues, like you want to deprecate weak cryptographic algorithm, example: banning SHA-1 certificate for stronger hash algorithm
* device security issue (tampering detection), so you want to revoke the device certificate
* or simply you want to rotate a key, because it was used for too long or someone who had access to it left your company

Anyway rotating your keys can be a good idea and you don’t want to have very short certificate validity period for devices because it can offline in a warehouse for quite some time.

How to check for certificate non-revocation?

### Certificate Revocation List (CRL):

In your certificates you have some URLs for downloading the CRL, for example, the medium ones are:

```
URI: http://crl3.digicert.com/sha2-ev-server-g1.crl
URI: http://crl4.digicert.com/sha2-ev-server-g1.crl
```

This is the compiled list of the certificate revoked by DigiCert. So for validating the certificate you receive from your peer wasn’t revoked, you need to download those lists and check if the received certificate is not here.

This method is now deprecated in favor of: **The Online Certificate Status Protocol (OCSP)**.

If you check the certificate you will find another URL, here for medium:

```OCSP: URI: http://ocsp.digicert.com```

This is a API URL, where you can ask for the revocation status of a given certificate. This is much more convenient than downloading and checking big CRL files. But OCSP is a little brittle, it can be easily took down with a DoS attack or simply not accessible in your private network.

Also it’s a another server to contact for your device. If your device is connected to your server using a VPN you need to provide a route for contacting all the possible OCSP API server to contact for validating all the possible certificate chains.

### OCSP stapling

An efficient way for the client to use OCSP is to ask the server to do the job and present the OCSP result directly via the (D)TLS protocol. In addition to his certificate, the server will present a non-revocation certificate using a TLS extension defined in [RFC6961](https://tools.ietf.org/html/rfc6961).

So the client don’t have to contact unreliable external OCSP servers. The TLS server will send a certificate status using a TLS extension. This status is a non-revocation certificate signed by the CA.

Stapling remove the main issues of OCSP: no more brittle API server to contact, no more external service to query and the server can cache his OCSP status for few minutes.

Another benefit: only one protocol on the client side, you don’t have to implement both (D)TLS and HTTP. You can do everything with your (D)TLS protocol, for example LWM2M. Can be useful for constrained devices and also for very constrained network were TCP is hardly working.

Keep in mind OCSP stapling is an optional extension and I’m pretty sure only few DTLS implementation support it.
