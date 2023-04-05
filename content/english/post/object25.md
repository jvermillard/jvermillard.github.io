+++
author = "Julien Vermillard"
title = "Lightweight M2M Object 25: gateway"
tags = [ "Internet-of-Things", "LWM2M", "LPWA"]
date = "2023-03-08"
+++

A separate specification was added to the Lightweight M2M 1.2 release: object 25 named "Gateway." This object is used by a Lightweight M2M device to act as a proxy for devices without a direct IP/CoAP connection to the management server.

The "gateway" is a proxy for the "endpoint IoT devices" in the specification. Those nodes are usually non-IP devices like Bluetooth low-energy beacons and LORA sensors attached to a beefier device like a cellular gateway.

This standard extension helps manage devices that are not powerful enough or without a practical way to connect them to the Internet.
How does it work in practice?

![LWM2M gateway object](/images/gateway.png)

The gateway provides a list of the managed nodes by advertising an instance of the object 25 for each device. Then on the different object instances, the server can retrieve the device identifier, the CoAP prefix to use, and the list of supported objects.

Once this object is read, the server can reach any device the gateway manages by prefixing the LWM2M CoAP request with the advertised prefix.

The gateway handles the request for the node using any protocol. For example, the CoAP TLV payload can be pushed to the device using ad-hoc BLE messages for simple object implementation or some other object like firmware update or location, totally abstract the communication, and use another way to do it.

Now that we better understand the Lightweight M2M gateway architecture let's discuss the practical implementation on the server side and precisely what I'm implementing for [Eclipse Leshan](https://eclipse.org/leshan).

First, to interact with nodes managed by a gateway, we need to read object 25. Once we know the node's detail: the "prefix," and the "list of objects," we can create a device registration for the node. This registration lifecycle (update lifetime, deregistration) must follow the lifecycle of the gateway. If we reuse the IP and DTLS session of the gateway, then we can easily route the node requests to the gateway. We add the prefix and magic! We have a working registration for a node behind a proxy.

Now, where it's getting more complicated is how to handle node changes: when the list of object instances changes for a regular Lightweight M2M device, the server detects it because the client updates the registration with the new list of object instances. For nodes managed thru object 25, we don't have this capability.

The solution I use for now is reading `/25` for every gateway registration update. This allows the server to get a new view of the nodes and update their registration if needed.

Handling devices roaming from one gateway to another can also be tricky without reading the whole object 25 at each registration update. Because with only the list of object 25 instance identifiers from the registration payload, it's impossible to identify the managed nodes uniquely.

So by default, in Leshan, we do nothing. In the user server code, I added a registration handler to read the complete object 25 during a registration or a registration update to add and remove the managed node.

```java
/**
 * Class in charge of registering end-iot-device if the registered client advertise for some object 25.
 */
 public class GatewayRegistrationHandler implements RegistrationListener {

    private static final Logger LOG = LoggerFactory.getLogger(GatewayRegistrationHandler.class);

    private final LeshanServer server;

    private final LinkSerializer linkSerializer = new DefaultLinkSerializer();

    public GatewayRegistrationHandler(LeshanServer server) {
        this.server = server;
    }

    @Override
    public void registered(Registration registration, Registration registration1, Collection<Observation> collection) {
        readGatewayUpdates(registration, null);
    }

    @Override
    public void updated(RegistrationUpdate registrationUpdate, Registration registration,
            Registration previousRegistration) {
        readGatewayUpdates(registration, registrationUpdate);
    }

    @Override
    public void unregistered(Registration registration, Collection<Observation> collection, boolean expired,
            Registration registration1) {
    }

    private void readGatewayUpdates(Registration registration, RegistrationUpdate gatewayRegUpdate) {
        if (registration.getSupportedObject().get(25) != null) {
            // read and register end iot devices supported by this gateway
            var readRequest = new ReadRequest(25);
            server.send(registration, readRequest, (response) -> {
                if (response != null && response.isSuccess()) {
                    LOG.debug("read /25 response: {}", response.getContent());
                    try {
                        LwM2mObject object25 = (LwM2mObject) response.getContent();
                        object25.getInstances().forEach((id, objectInstance25) -> {
                            registerObject25Instance(registration, objectInstance25, gatewayRegUpdate);
                        });
                    } catch (RuntimeException e) {
                        LOG.error("Error while processing object 25 value", e);
                    }
                }
            }, (e) -> {
                LOG.warn("Invalid response to read object 25 for device '{}': '{}'", registration.getEndpoint(), e.getMessage());
            });
        }
    }

    private void registerObject25Instance(Registration registration, LwM2mObjectInstance objectInstance25, RegistrationUpdate gatewayRegUpdate) {
        String deviceId = objectInstance25.getResource(0).getValue().toString();
        String prefix = objectInstance25.getResource(1).getValue().toString();
        Link[] iotDeviceObjects = (Link[]) objectInstance25.getResource(3).getValue();

        // If a valid end node was defined in the LwM2M gateway object, register a new
        // device using the endpoint for the gateway plus the CoAP device prefix
        if (deviceId.isEmpty() || prefix.isEmpty() || iotDeviceObjects == null) {
            LOG.warn("Invalid device ID, prefix, or IoT Device Objects, gateway '{}', object25 instance: '{}'", registration.getEndpoint(), objectInstance25);
            return;
        }
        prefix = prefix.replaceAll("/", "");

        // add a new endpoint
        LOG.debug("Adding endpoint '{}'", deviceId);

        // we decide if it's a registration update only if the information didn't changed
        if (gatewayRegUpdate != null) {
            var oldDeviceReg = server.getRegistrationService().getByEndpoint(deviceId);
            if (oldDeviceReg != null && oldDeviceReg.getGatewayRegId().equals(registration.getId()) && oldDeviceReg.getEndpoint().equals(deviceId) && oldDeviceReg.getGatewayPrefix().equals(prefix)) {
                server.registrationUpdateEndIotDevice(gatewayRegUpdate, oldDeviceReg.getId(), deviceId, prefix, iotDeviceObjects);
                return;
            }
        }
        server.registerEndIotDevice(registration.getId(), deviceId, prefix, iotDeviceObjects);
    }
}
```

For now, I didn't enforce the read /25 behavior, mainly to provide some freedom to the users who use a static assignation of the node and gateway or have a very static object model for the nodes.

You can try the Leshan server implementation by building this branch [https://github.com/eclipse/leshan/tree/obj25](https://github.com/eclipse/leshan/tree/obj25). As the client, don't hesitate to correct me in the comments), only the Zephyr OS LWM2M client claims some support for object 25 for now.
