+++
author = "Julien Vermillard"
title = "Bootstrapping device security with Lightweight M2M"
tags = [ "LWM2M", "leshan", "security", "Internet-of-Things"]
date = "2015-02-15"
+++

During the last **OMA LWM2M Test Fest** I was quite surprised the **bootstrap** procedure wasn’t present in the test cases proposed by the organizers. Furthermore only few vendors implemented it, or even understand what’s the purpose of this feature. So I decided to write this little introduction.

Bootstrapping is the procedure for the device to get the secret keys and URL for reaching the servers. It’s also useful for re-keying, upgrading security scheme or redirecting your device to another server.

## Why not “hardcode” final server URL and credentials at factory?

Well it’s possible, it’s called **“factory bootstrap”**, but it’s not a good practice: if you need to change the server URL or credentials (passwords) you will need to have physical access to the device. This is not so IoT ☺.

Here come the so-called “device initiated bootstrap”. During the factory flashing you just put “bootstrap credentials” for reaching a bootstrap server. The bootstrap server only feature is to answer bootstrap request (POST on /bs) and to configure the device with servers URL and credentials (private key, certificate, pre-shared-keys, etc..).

The parameters configured during bootstrap are:

**Server security credentials:**

* pre-shared keys for secure **DTLS PSK** communications
* or private key and certificate for **DTLS RPK** or **X.509** communications
* **URLs** for reaching the server like “coaps://my.server.com:5684"
* **SMS security** parameters for communicating with the device or the server

**Communication parameters:**

* communication frequency (lifetime)
* binding mode: does the device is always connected or not, does it uses UDP and/or SMS

**Access Control Lists:**

* which server can read/write which objects, like firmware update or connectivity parameters, (only when you have more than one device management server)

The device can trigger a bootstrap when:

* when the device contains only bootstrap credentials and no device-management server credentials,
* when the device, for any reason, fails to start a communication with the configured device management servers (server not responding, authentication failure, etc..).

![The initial bootstrap sequence: how a device get the final device management credentials](/images/bootstrap.gif)

Why doing security configuration in two steps? Why not putting the key in the factory and wait for the need to change it for triggering a device bootstrap?

The first advantage: you are sure configuring device credential “over-the-air” is working because you use it at least once during the device lifetime.

You want to change the credentials in the device because you had a breach in your servers or because you want to rotate your keys/certificate? It’s easy, tested and running! Just invalidate the previous credentials on the server, the device will trigger a re-bootstrap and get the new credentials from the bootstrap server. This procedure can also be used for re-configuring a device for connecting to another server.

Once past the initial bootstrap, the factory doesn’t know the final secrets used for communicating. So someone breaking into the factory and gathering bootstrap credentials won’t be able to eavesdropping final device-management communications. At least, they could observe the bootstrap communications and discover the device management key for few devices, but they won’t directly have access to your whole fleet.

Today [Leshan](https://github.com/eclipse/leshan) server contains the code for supporting device initiated bootstrap. It’s still an early work, but it’s functional.

[Wakaama](https://github.com/eclipse/wakaama) client code doesn’t support the bootstrap procedure, but we are working on it, maybe in a couple of weeks an early support will land in the git repository.
