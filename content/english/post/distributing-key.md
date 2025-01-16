+++
author = "Julien Vermillard"
title = "Generating and distributing symmetrical IoT authentication keys"
tags = [ "IoT", "Security", "Cryptography", "HSM"]
date = "2025-01-16"
images = ["/images/hsm-banner.png"]
+++
![HSM banner](/images/hsm-banner.png)

To ensure secure communication between devices, you need to distribute authentication keys. If your device and your transport can't work with asymmetrical cryptography or PKI, you must build on symmetrical authentication secret (password or pre-shared-keys).

For example, if your device uses MQTT to authenticate, it must provide a username and a password at connection, if you use DTLS-PSK with CoAP, the device will need to provide a PSK Identity and a pre-shared-key during the handshake. A common solution is to use the serial number as the login. For the password, you must share the same unique value between the device and the server. 

Now, the challenge is how to "share" those secrets in a complex manufacturing environments? The factory line doing the device building and provisioning is probably a different company, in a different country, with limited or no willingness to hook their production line to the Internet, creating a potential source of production line interruptions.

One way to do that is to let the factory generate the passwords and exchange back flat files which is often a nightmare in term of security. A better trick I learned from my predecessors, is to derive the password from the serial number and use the same derivation method in the factory and in the server in charge of managing the device. For that, in the past, I inherited expensive SafeNet HSM storing a couple of AES-128 master keys.

![Key generation](/images/key-generation.png)

This way a unique password is generated for each unique serial number, and if you use the same master key in the factory and in the cloud you can generate the exact same password for the same serial number.

Now, using AES-128 for hashing is looking a bit strange (why not using a secret and HMAC?), the reason is because storing and using AES key commonly supported by Hardware Security Module (HSM). HSM are physical devices in charge of managing secrets (like symmetrical or asymmetrical cryptographic keys). Their main goal, in this case, is to be able to deploy the key in your factory or data-center without allowing someone with physical access to extract the key.

Now industrial grade HSM have some drawbacks: slow, very expensive (above 20k€, and you need a backup and proper training or professional services to setup it). Also, if you generate the key in the HSM, you can backup and restore it in another HSM, but only from the same vendor, you are locked with them!

For some time I was thinking if we can use cheaper alternatives like smart-cards and I stumbled on Nitrokeys (https://www.nitrokey.com/products/nitrokeys) which have an USB stick sized HSM with open-source hardware and software for around 100€!

Now my idea is to do something not conventional: generate the AES-128 key outside of the HSM, to be able to import it in other hardware (like AWS cloud HSM or Azure key vault). It's obviously safer to generate the key in the HSM and use the backup/restore mechanism to duplicate it, but it's locking your key usage to a single type of HSM.

First get 16 random bytes from a computer:


`hexdump -vn16 -e'4/4 "%08X" 1 "\n"' /dev/random`

First initialize the HSM key (slect your how SO and PIN secrets):
`sc-hsm-tool --initialize --so-pin 3537363231383830 --pin 648219
`
With the SmartCard starter kit https://www.cardcontact.de/download/sc-hsm-starterkit.zip run the `scsh3gui` shell to create the AES128 key on your device:

(replace 00112233445566778899AABBCCDDEEFF by the random value your generated)
```javascript
var aes = new Key();
aes.setComponent(Key.AES, new ByteString("00112233445566778899AABBCCDDEEFF", HEX));

// Use default crypto provider
var crypto = new Crypto();

// Create card access object
var card = new Card(_scsh3.reader);

card.reset(Card.RESET_COLD);

// Create SmartCard-HSM card service
var sc = new SmartCardHSM(card);

// Attach key store
var ks = new HSMKeyStore(sc);

// Initialize with key domain
var sci = new SmartCardHSMInitializer(card);
sci.setKeyDomains(1);
sci.initialize();

// Create DKEK domain with 00.00 DKEK
sc.createDKEKKeyDomain(0, 1);
var share = new ByteString("0000000000000000000000000000000000000000000000000000000000000000", HEX);
sc.importKeyShare(0, share);

// Create DKEK encoder and import share
var dkek = new DKEK(crypto);
dkek.importDKEKShare(share);

// Encode AES key into blob
var blob = dkek.encodeAESKey(aes);
dkek.dumpKeyBLOB(blob);

var key = ks.importAESKey("MyAESKey", blob, 128);
```

You should be able to find the AES key using the PKCS11 interface:

```
kcs11-tool --module /usr/local/lib/libsc-hsm-pkcs11.so -l --pin 648219 --list-objects
Using slot 0 with a present token (0x1)
Certificate Object; type = unknown cert type
  label:      C.DevAut
Certificate Object; type = unknown cert type
  label:      C.DICA
Secret Key Object; AES length 16
  label:      MyAESKey
  ID:         01
  Usage:      encrypt, decrypt
  Access:     sensitive, always sensitive, never extractable, local
```

By the way, PKCS11 is a standard C API to access this kind of cryptographic token like HSMs. This tool use the .so implementation pointed by the `--module` parameter. The used implementation is https://github.com/cardcontact/sc-hsm-embedded 

Now we have the key imported in the Nitrokey HSM, we can use a program to generate some keys. I built this simple CLI proof-of-concept, using the github.com/miekg/pkcs11 PKCS11 library for Go:

```Go
package main

import (
	"crypto/md5"
	"encoding/hex"
	"os"

	"github.com/miekg/pkcs11"
)

func main() {
	if len(os.Args) <= 3 {
		println("usage: [AES128 key label] [PIN] [serial number 1] [serial number 2] [serial number ...]")
		return
	}

	keyLabel := os.Args[1]
	pin := os.Args[2]
	serialNumbers := os.Args[3:]

	//p := pkcs11.New("/usr/lib/x86_64-linux-gnu/opensc-pkcs11.so") this one doesn't support AES
	p := pkcs11.New("/usr/local/lib/libsc-hsm-pkcs11.so")
	e := p.Initialize()
	if e != nil {
		panic(e)
	}

	defer p.Destroy()
	defer p.Finalize()

	slots, e := p.GetSlotList(true)
	if e != nil {
		panic(e)
	}

	session, e := p.OpenSession(slots[0], pkcs11.CKF_SERIAL_SESSION|pkcs11.CKF_RW_SESSION)
	if e != nil {
		panic(e)
	}
	defer p.CloseSession(session)

	// Login as SO
	e = p.Login(session, pkcs11.CKU_USER, pin)
	if e != nil {
		panic(e)
	}
	defer p.Logout(session)

	template := []*pkcs11.Attribute{
		pkcs11.NewAttribute(pkcs11.CKA_LABEL, keyLabel),
	}
	if e := p.FindObjectsInit(session, template); e != nil {
		panic(e)
	}
	obj, _, e := p.FindObjects(session, 1)
	if e != nil {
		panic(e)
	}
	if e := p.FindObjectsFinal(session); e != nil {
		panic(e)
	}

	// try encrypt

	for _, sn := range serialNumbers {
		// hash the serial number to get a 16bytes value
		plaintext := md5.Sum([]byte(sn))

		// use the same empty IV, we can't use ECB, it's not supported
		iv := []byte{0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0}
		if e = p.EncryptInit(session, []*pkcs11.Mechanism{pkcs11.NewMechanism(pkcs11.CKM_AES_CBC, iv)}, obj[0]); e != nil {
			panic(e)
		}
		var ciphertext []byte
		if ciphertext, e = p.Encrypt(session, plaintext[:]); e != nil {
			panic(e)
		}
		println(sn, hex.EncodeToString(ciphertext))
	}
}
```

Sample output:
```
go run ./main.go MyAESKey 648219 SN12345 SN134535 SN1241451 SN14145
SN12345 0493eb8f2dde631cee38b11d78e11a3c
SN134535 3bdd47535bb4ac42d3c94549e3757988
SN1241451 44a3984de49e309a51bfdfdaa4a89f33
SN14145 aa0659ded4eab1677bc4bc8eb09ae47b
```

And it's not so slow! I was expecting worse performance but it's looking fast enough to use in a factory line.

Using symmetric keys for IoT device authentication is cost-effective when asymmetrical cryptography isn’t viable, but it make key distribution difficult to secure. 

By deriving keys from serial numbers and using flexible tools like Nitrokey HSM, we can simplify key management and ensure each device has a unique password without relying on factory to cloud communications.

PS: Big thanks Nitrokey forum support for helping me with this strange use-case!
