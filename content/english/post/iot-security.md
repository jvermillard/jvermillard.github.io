+++
author = "Julien Vermillard"
title = "The 4 Elements of Internet-of-Things Security"
tags = [ "Internet-of-Things", "security"]
date = "2015-01-21"
+++

The Internet of Things (IoT) is fascinating: it is a game changer for health-care, connected homes and cities, ground transportation, and many other domains. It is also fascinating due to its unprecedented scale, and the billions of things that are in play.

From a technical point of view, IoT is very challenging: hardware design, embedded software, internet scale servers, or a groundbreaking user-experience are the core ingredients of any successful connected product. It’s an immense challenge to master all these topics, even more so if you add security as a critical constraint.

Many IoT vulnerabilities have been found and showcased at security events recently, from connected thermostats to power plants. Insecurity has even become a subject of choice for creating catchy headlines: [“Connected killer toaster”](http://www.eweek.com/blogs/security-watch/internet-of-things-could-bring-on-attack-of-the-killer-toaster-botnet.html), [“Fridges changed into spamming machines”](http://www.bbc.com/news/technology-25780908), [“The Nightmare on connected home street”](http://www.wired.com/2014/06/the-nightmare-on-connected-home-street/).

So let’s explore this topic, what is exactly needed for building a secure infrastructure for any Internet of Things or Machine-to-Machine application?

## 1st Element: Secure your hardware

This one is maybe the less obvious one and also one of the hardest to master. Why so difficult? Because when you deal with hardware security you need to deal with physics. The “real” world has much more stringent rules than the digital world. Everything is a matter of compromises: do I need to protect from local physical access? Can I trust my users? Should I increase the price of my object by using this secure chip, when I can use this less secure one?

Usually, the first reaction is: why should I care? If I correctly designed my security scheme, physically accessing one device should only compromise that particular device, right? Yes! But it’s a big deal, if you look at [this example](https://www.blackhat.com/docs/us-14/materials/us-14-Jin-Smart-Nest-Thermostat-A-Smart-Spy-In-Your-Home.pdf) with the very popular Nest thermostat.

In a nutshell, anyone can buy an IoT device off-the-shelf, install a spyware and put them back on the market by selling it on eBay, or just bringing it back to the shop.

Hopefully, hardware security features are very common nowadays: secure boot, secure flash, CPU that is powerful enough to handle asymmetric cryptography or even tamper detection/prevention features.

Hardware security is a vast topic, and while I’m no expert at it I can only but stress the importance of thinking about it early in your development process. Hardware isn’t like software: it can’t be updated over-the-air!

## 2nd Element: You can't secure what you can't update

If you only have time for only one element of this list, it must be this one! If you have the budget for developing only one feature, it should be remote firmware upgrade “over the air”. This is the key for pushing new security features or simply for patching holes in the future.

But it’s maybe the hardest feature of any device management stack: it must be bullet proof: power supply, batteries, networks and NAND flash are not reliable you need to mitigate all those risks in your design. A 0.1% failure rate on a 1 million fleet is 1000 devices to fix and the last thing you want to do is to take your car for fixing bricked devices around the world☺

Update is data intensive: even a small radio firmware for a 3G or 4G cellular modem can be 10 to 20 MB, embedded Linux root filesystem are becoming larger and larger. So patch or delta update make a lot of sense: a smaller download is a faster and more robust download.

To delta update robustly, whether it be a bare metal microcontroller or an embedded Linux system, you need to do it from the bootloader or you will need twice the flash space. Most of the time it means you need to write a custom bootloader or at least customize one. It’s an expensive development, it needs to be designed upfront.

## 3rd Element: Secure communication

As opposed to the early days of Machine-to-Machine (M2M) where many existing industrial protocols where “recycled”, we now start to have simple and efficient protocols that really address the constraints of IoT, like MQTT or CoAP. They are simple and compact enough to be implemented on 8-bit processors that have only a few kilobytes of RAM and flash memory.

If you want to secure those protocols by tunneling them through SSL/TLS, DTLS or IPSec, it certainly has a cost. Actually, most 8-bit processors are not really able to run a modern TLS implementation, and so are some 32-bit processors when they have limited flash or RAM available.

If we consider TLS, the successor of SSL, a protocol for creating a secure tunnel for your communications, there are basically two possible security schemes:

* **Pre-Shared-Keys** (PSK) where the two peers share a common secret which has been provisioned securely thus most likely “offline”. Security is accomplished using symmetric ciphers like AES and SHA-1, the same secret being used for encrypting or decrypting the communications. The general idea is that you will store a secret key in your device during the manufacturing process, and store the same secret on the server. Usually, the key will be stored in a “write-only” secured-chip that won’t let the key be read afterwards. An implementation of TLS based on a pre-shared key is not difficult to fit in an ARM Cortex M3 or M4 microcontroller, especially since those architectures already have hardware support for AES. You can find RAM/flash estimate in this great document: https://tools.ietf.org/html/draft-tschofenig-lwig-tls-minimal-03.
* The next level of security is to use **asymmetric cryptography** (public/private key). It’s a better scheme because you don’t have to move sensitive secrets like PSK around, you just need to exchange public keys and not the sensitive private keys.
* Another level exists, with of course more complexity: **public key certificates**. You add it on top of asymmetric cryptography. The idea is to chain cryptographic signature of different third party for authenticating an identity. The price in code size is higher but you can delegate the certificate provisioning to trusted third parties. You need to parse the whole certificate, check the signature chain and also check for possible revocation which is not a simple matter: https://www.imperialviolet.org/2014/04/19/revchecking.html

It is important to understand the different possible security schemes that are available in order to choose the most secure approach given your hardware constraints (code size, RAM, hardware acceleration). A common requirement to every security scheme is to regularly rotate the credentials, which brings us to the next element.

## 4th Element: Key distribution

If you use pre-shared secrets you need to choose a way to generate them. The credentials will probably be generated and put in your device as part of your manufacturing process, probably during the initial flashing of the firmware.

Provisioning credentials for thousands or millions of devices is certainly not a trivial matter, and you will likely have to delegate the generation of the credentials to the factory, which will hand over the list to you, so you can reintegrate them in your IoT backend. Moving credentials from oversea factory back to your office is not a comfortable situation. So you will be tempted to use a predictable way to regenerate the list on your side.

If you look at the slide 37 of this [https://www.blackhat.com/docs/us-14/materials/us-14-Solnik-Cellular-Exploitation-On-A-Global-Scale-The-Rise-And-Fall-Of-The-Control-Protocol.pdf] presentation at BlackHat 2014, you will understand why it is critical that each device’s key is different, in addition to be seeded with a random (or pseudo-random) secret. The used formula is:

`String password = Base64(MD5(IMEI+CARRIER_NOT_SO_SECRET)`

This is wrong! IMEI is a public phone identifier easy to figure out. So if you know the “NOT_SO_SECRET” value you are able to guess the password of any phone.

Nice trick: use a secret AES key securely stored in a HSM as a seed for generating the credentials. It’s the same seed but pseudo randomized — convenient and secure. For example, use a well-known public identifier, like the serial number and generate the credential using a private AES key. The key can be secured using a Hardware Security Module, so nobody can extract it. It’s simple to operate and much more secure than the “NOT_SO_SECRET” salt used in the previous slide.

Once your key provisioning mechanism is in place, you need to think about how you are going to change the keys on the field because let’s face it, one day your secret will be outdated, or the crypto you use will be proven flawed. Just like backup restorations, device credentials rotations need to be tested on a regular basis: don’t wait to be forced to press the “change 1 million credentials over-the-air” button to start sweating! The best way to be sure you can actually change the credentials is probably to do it for every device you connect, at least once in their life. A good practice, for example, is to replace the factory provided credentials right after the first connection. You can take inspiration from the Lightweight M2M device initiated bootstrap. If you apply this mechanism you are sure you will be able to change your device credentials in the future because you already did it once.

## Is it your core business?

In the world of Iot, all the good old web server security rules still apply. You need to secure your IoT cloud server just like any other serious cloud application or web service. The major difference when IoT devices are involved should now clear to you: you need a strong device to cloud communication security and you must be ready to upgrade the device keys and software quickly.

Now that you have a list of security features you should implement to secure your IoT application, it’s time to ask your product management the appropriate budget to tackle all the aforementioned issues! There is a high probability that they will reply:

![ship it!](/images/shipit.jpeg)

All hope is not lost, a lot of vendors concentrate on providing turn-key solutions around device-management and security (I am actually employed by [one of them](https://www.sierrawireless.com/) ☺).

Also, standards and open-source software stacks exist for providing a solid foundation for your IoT application infrastructure. My favourite one is [Open Mobile Alliance Lightweight M2M](http://technical.openmobilealliance.org/Technical/technical-information/release-program/current-releases/oma-lightweightm2m-v1-0) (a.k.a. LWM2M). Its scope is to provide a solution for monitoring, configuring, over-the-air firmware and software upgrades and also credential distribution.

Active open-source implementations with business friendly licenses exist:

* [Eclipse Wakaama](https://eclipse.org/wakaama): a C client for embedded devices, that can easily fit in ARM Cortex processors, and even some 8-bit Arduino-like microcontrollers.
* [Eclipse Leshan](https://eclipse.org/leshan): a Java server implementation, to be used as a library for building your cloud IoT server.

I’ll discuss this topic further at the next [EclipseCon](https://www.eclipsecon.org/na2015/session/five-elements-iot-security-open-source-rescue), along why it’s in the interest of the industry at large to build a common open-source infrastructure for IoT security and device management.

Follow me on twitter: [@vrmvrm](https://twitter.com/vrmvrm)

P.S.: Thanks [@kartben](https://twitter.com/kartben) and [@manusangoi](https://twitter.com/manusangoi) for proof reading the article! ❤
