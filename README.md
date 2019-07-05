# Bosch IoT Suite end to end setup

Example end to end setup of Bosch IoT Hub, Things, Permissions and Insights services.

Services can be booked here:

https://accounts.bosch-iot-suite.com

Book the following services:
 - asset management (things & hub)
 - permissions

 Using Postman, import the collection.json and environment.json from postman directory.

# Device provisioning using hub and things directly

### 1. First, you need to create the device in the Hub. 

Hub has its own device registry, username is in the format device-registry@{{hub.tenantId}}. Fill out the variables in the postman env: 
  - hub.tenantId and hub.deviceRegistryPassword,
  - things.namespace and device.deviceId 

Tenant id is a unique hub identified. When using IoT Things, the device id is should be by convention "things_namespace:device_id", so that the mapping to IoT things works correctly later on. things_namespace is essentially the identifer of things service, similar as what tenant id for hub represents. 

Issue a Register device Postman request.

### 2. Add device credentials

Fill out the device.authId and device.password variables and issue a Add credentials Postman request. 

The credentials issues here are of hashed-password type. Additionally, Bosch IoT Hub (Eclipse Hono) supports certificate based credentials, including parent-child certificate relation -> you can upload the root certificate to the hub, and devices can then register themselves using leaf certificates, using Subject field to prove to which tenenat they belong. Currently, upload of the root certificate is possible via Bosch support team only, not from the outside.

### 3. Test the IoT Hub connection

At this point, you can test out the connection with a Hono client, which can be downloaded here: https://www.eclipse.org/downloads/download.php?file=/hono/hono-cli-0.8-exec.jar

Replace the messaging_username, messaging_password and hub_tenant_id variables and run:

Run:

`java -jar hono-cli-0.8-exec.jar --hono.client.host=messaging.bosch-iot-hub.com --hono.client.port=5671 --hono.client.tlsEnabled=true --hono.client.username={{messaging_username}} --hono.client.password={{messaging_password}} --tenant.id={{hub_tenant_id}} --spring.profiles.active=receiver,telemetry`

In the postman, send some request over the Hub HTTP adapter. These requests should be read by the hono client.

Info: by default, the HTTP adapter uses QoS 0 (no queuing), but a header can be set for QoS 1 (at least once with queueing).
Info2: if there is no listener on the other side of the hub, you will get 503 Temporary Unavailable return codes. You need to have at least one listener connected!

### 4. Provision a device in Bosch IoT Things which maps to the Hub device

To register a device with IoT Things, you need to authenticate yourself somehow. Usually, this is a machine to machine flow, and there are generally three options here:
- using the IoT Permissions, you can create a user, and use it as a "technical user" to register devices. You also require the API token to access the things service here.
- You can have a combination of API Token / Public Key as credentials
- You can use OAuth2 applications registered here

![oauth2-clients](images/oauth2-clients.png "oauth2-clients")

Once you have a "thing" provisioned, you need to add the "policy" which will allow the hub to overwrite this device data. Using the `{{things.endpointHttp}}/api/2/policies/{{things.thingId}}/entries/hub` endpoint, you can PUT the following policy entry:

``` json
{
  "subjects": {
    "integration:51dc4384-83d9-4d29-96d2-dc92e1be55f5:hub": {
      "type": "iot-things-clientid"
    }
  },
  "resources": {
    "policy:/": {
      "grant": [],
      "revoke": []
    },
    "thing:/": {
      "grant": [
        "READ",
        "WRITE"
      ],
      "revoke": []
    },
    "message:/": {
      "grant": [
        "READ",
        "WRITE"
      ],
      "revoke": []
    }
  }
}
```

What actually happens is, when you create the asset communication suite, hub and things are provisioned incl. the connection between them. This connection defined a "default" policy, which you extended in the last step (the /entries/hub part). 


# Concept info

Usually, you would use an external identity provider, and you backend service would automatically "register" the user the first time she/he logs in. Workflow is explained here: https://permissions.s-apps.de1.bosch-iot-cloud.com/docs/developer-guide/index.html#Integration-of-OpenID-Connect-identity-providers_249117097


# Device provisioning using the Device Provisioning Service

1. Register an OAuth2 Client as already shown above. 
2. Using the swagger UI, switch to provisioning service and create a device.
3. send data, it should be automatically be visible in the Things service.

# Bosch IoT Insights integration

At this point, you might want to import data to IoT Insights. Currently, only way to do this is to write an adapter service for this job. An example (and many others) can be found here: https://github.com/bsinno/iot-things-examples/tree/master/http-forwarder