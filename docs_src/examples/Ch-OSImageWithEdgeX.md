# Creating an EdgeX Ubuntu Core Image

## Introduction
This guide walks you through creating an Ubuntu Core OS image that is preloaded with an EdgeX stack. We use [Ubuntu Core](https://ubuntu.com/core/docs) as the Linux distribution because it is optimized for IoT and is secure by design. We configure the image and bundle the current snapped versions of EdgeX components. After the deployment the snaps will continue to receive updates for the latest security and bug fixes (depending on the selected channel).

This guide is divided into three chapters to create:

- Ubuntu Core + EdgeX, with default configurations
- Ubuntu Core + EdgeX, with service configuration overrides
- Ubuntu Core + EdgeX, with custom service and device configuration files

Each chapter results in a working Ubuntu Core OS image that can be flashed on a disk and booted with the expected EdgeX stack.

In this example, we will create an `amd64` image for Intel and AMD processors. The instructions can be adapted to other architectures and even for a Raspberry Pi. We will use the Device Virtual service to simulate devices and produce synthetic events.

!!! note
    This guide has been tested on an `amd64` **Ubuntu 22.04** as the desktop OS. It may work on other Linux distributions and Ubuntu versions.

    Some commands are executed on the desktop computer, but some others on the target Ubuntu Core system. For clarity, we use **🖥 Desktop** and **🚀 Ubuntu Core** titles for code blocks to distinguish where those commands are being executed.

    An Intel NUC11TNH with 8GB RAM and 250GB NAND flash storage has been used as the target `amd64` hardware. 

We use the following tools on the desktop machine:

- [snapcraft](https://snapcraft.io/snapcraft) to manage keys in the store and build snaps
- [YQ](https://snapcraft.io/yq) to validate YAML files and convert them to JSON
- [ubuntu-image](https://snapcraft.io/ubuntu-image) v2 to build the Ubuntu Core image

Install them using the following commands:
```bash title="🖥 Desktop"
sudo snap install snapcraft --classic
sudo snap install yq
sudo snap install ubuntu-image --classic --channel=2/stable
```

Before we start, it is a good idea to read through the following documents:

- [Inside Ubuntu Core](https://ubuntu.com/core/docs/uc20/inside) to get familiar with the Ubuntu core internals
- [Getting Started using Snaps](../../getting-started/Ch-GettingStartedSnapUsers) to understand general EdgeX snap concepts

## A. Create an image with EdgeX components

In this chapter, we will create an OS image that includes the expected EdgeX components.

### Create an Ubuntu Core model assertion
The [model assertion](https://ubuntu.com/core/docs/reference/assertions/model) is a digitally signed document that describes the content of the OS image.

Refer to [this article](https://ubuntu.com/core/docs/custom-images#heading--signing) for details on how to sign the model assertion. Here are the needed steps:

1) Create a developer account

Follow the instructions [here](https://snapcraft.io/docs/creating-your-developer-account) to create a developer account, if you don't already have one.

2) Create and register a key

```bash title="🖥 Desktop"
snap login
snap keys
# continue if you have no existing keys
# you'll be asked to set a passphrase which is needed before signing
snap create-key edgex-demo
snapcraft register-key edgex-demo
```
We now have a registered key named `edgex-demo` which we'll use later.

3) Create the model assertion

First, make yourself familiar with the Ubuntu Core [model assertion](https://ubuntu.com/core/docs/reference/assertions/model).

Find your developer ID using the Snapcraft CLI:
```bash title="🖥 Desktop"
$ snapcraft whoami
...
developer-id: <developer-id>
```
or from the [Snapcraft Dashboard](https://dashboard.snapcraft.io/dev/account/).

??? info "YAML Model Assertion"
    Unlike the official documentation which uses JSON, we use YAML serialization for the model. This is for consistency with all the other serialization formats in this tutorial. Moreover, it allows us to comment out some parts for testing or add comments to describe the details inline.

Create `model.yaml` with the following content, replacing `authority-id`, `brand-id`, and `timestamp`:
```yaml
type: model
series: '16'

# set authority-id and brand-id to your developer-id
authority-id: <developer-id>
brand-id: <developer-id>

model: ubuntu-core-22-amd64
architecture: amd64

# timestamp should be within your signature's validity period
timestamp: '2022-06-21T10:45:00+00:00'
base: core22

grade: dangerous

snaps:
  - name: pc
    type: gadget
    default-channel: 22/stable
    id: UqFziVZDHLSyO3TqSWgNBoAdHbLI4dAH
  
  - name: pc-kernel
    type: kernel
    default-channel: 22/stable
    id: pYVQrBcKmBa0mZ4CCN7ExT6jH8rY1hza

  - name: snapd
    type: snapd
    default-channel: latest/candidate # temporary for latest pc-gadget compatibility; see https://github.com/canonical/edgex-ubuntu-core-testing/issues/1
    id: PMrrV4ml8uWuEUDBT8dSGnKUYbevVhc4

  # Snap base for EdgeX snaps
  - name: core22
    type: base
    default-channel: latest/stable
    id: amcUKQILKXHHTlmSa7NMdnXSx02dNeeT

  - name: edgexfoundry
    type: app
    default-channel: latest/edge # replace with latest/stable after EdgeX v3 release
    id: AZGf0KNnh8aqdkbGATNuRuxnt1GNRKkV

  - name: edgex-device-virtual
    type: app
    default-channel: latest/edge # replace with latest/stable after EdgeX v3 release
    id: AmKuVTOfsN0uEKsyJG34M8CaMfnIqxc0
```

!!! note
    We use the gadget and kernel snaps for 64bit personal computers using Intel or AMD processors. For a Raspberry Pi, you need to change the model, architecture, as well as the gadget and kernel snaps.

!!! tip "Finding Snap IDs"
    Query the unique store ID of a snap, for example the `edgexfoundry` snap:
    ```
    $ snap info edgexfoundry | grep snap-id
    snap-id: AZGf0KNnh8aqdkbGATNuRuxnt1GNRKkV
    ```

4) Sign the model assertion

We sign the model using the `edgex-demo` key created and registered earlier. 

The `snap sign` command takes JSON as input and produces YAML as output! We use the YQ app to convert our model assertion to JSON before passing it in for signing.

```bash title="🖥 Desktop"
# sign
yq eval model.yaml -o=json | snap sign -k edgex-demo > model.signed.yaml

# check the signed model
cat model.signed.yaml
```

!!! note
    You need to repeat the signing every time you change the input model, because the signature is calculated based on the model.

### Build the Ubuntu Core image
We use ubuntu-image and set the path to signed model assertion YAML file.

This will download all the snaps specified in the model assertion and build an image file called `pc.img`.

??? note "Expanding data partition for emulation"
    If you plan to use an emulator to install and run Ubuntu Core from the resulting image, it is a good idea to allocate additional writable storage. This is necessary only if you want to install additional snaps interactively or upgrade existing ones on the emulator.

    The default size of the `ubuntu-data` partition is `1G` as defined in the gadget snap. When installing on actual hardware, this partition extends automatically to take the whole remaining space on the disk volume. However, when using QEMU, the partition will have the exact same size because the image size is calculated based on the defined partition structure. The 1GB `ubuntu-data` partition will be mostly full after first boot. You can configure the image to be larger so that the installer expands the partition automatically as with a large disk volume. 

    To extend the image size, use the `--image-size` flag in the following command. For example, to add 500MB extra (the original image is around 3.5GB), set `--image-size=4G`.

```bash title="🖥 Desktop"
$ ubuntu-image snap model.signed.yaml --validation=enforce
Fetching snapd
Fetching pc-kernel
Fetching core22
Fetching pc
Fetching edgexfoundry
Fetching edgex-device-virtual

# check the created image file
$ file pc.img
pc.img: DOS/MBR boot sector, extended partition table (last)
```

!!! done
    The image file is now ready to be flashed on a medium to create a bootable drive with the needed applications!

### Boot into the OS
You can now flash the image on your disk and boot to start the installation.
However, during development it is best to boot in an emulator to quickly detect and diagnose possible issues.

Instead of flashing and installing the OS on actual hardware, we will continue this guide using an emulator. Every other step will be similar to when image is flashed and installed on actual hardware.

Refer to the following to:

- [Run in an emulator](#run-in-an-emulator) - used in this guide
- [Flash the image on disk](#flash-the-image-on-disk)

### TRY IT OUT
In this step, we connect to the machine that has the image installed over SSH, validate the installation, and do some manual configurations.

We SSH to the emulator from the previous step:
```bash title="🖥 Desktop"
ssh <user>@localhost -p 8022
```
If you used the default approach (using `console-conf`) and entered your Ubuntu account email address at the end of the installation, then `<user>` is your Ubuntu account ID. If you don't know your ID, look it up using a browser from [here](https://login.ubuntu.com/) or programmatically from `https://login.ubuntu.com/api/v2/keys/<email>`.

List the installed snaps and their services:
``` title="🚀 Ubuntu Core"
$ snap list
Name                  Version         Rev    Tracking          Publisher   Notes
core22                20230503        634    latest/stable     canonical✓  base
edgex-device-virtual  3.0.0-dev.50    669    latest/edge       canonical✓  -
edgexfoundry          3.0.0-dev.163   4452   latest/edge       canonical✓  -
pc                    22-0.3          127    22/stable         canonical✓  gadget
pc-kernel             5.15.0-71.78.1  1281   22/stable         canonical✓  kernel
snapd                 2.59.4          19361  latest/candidate  canonical✓  snapd

$ snap services
Service                                       Startup   Current   Notes
edgex-device-virtual.device-virtual           disabled  inactive  -
edgexfoundry.consul                           disabled  inactive  -
edgexfoundry.core-command                     disabled  inactive  -
edgexfoundry.core-common-config-bootstrapper  disabled  inactive  -
edgexfoundry.core-data                        disabled  inactive  -
edgexfoundry.core-metadata                    disabled  inactive  -
edgexfoundry.nginx                            disabled  inactive  -
edgexfoundry.redis                            disabled  inactive  -
edgexfoundry.security-bootstrapper-consul     disabled  inactive  -
edgexfoundry.security-bootstrapper-nginx      disabled  inactive  -
edgexfoundry.security-bootstrapper-redis      disabled  inactive  -
edgexfoundry.security-proxy-auth              disabled  inactive  -
edgexfoundry.security-secretstore-setup       disabled  inactive  -
edgexfoundry.support-notifications            disabled  inactive  -
edgexfoundry.support-scheduler                disabled  inactive  -
edgexfoundry.vault                            disabled  inactive  -
```

Everything is inactive by default. Let start the platform:
``` title="🚀 Ubuntu Core"
$ snap start --enable edgexfoundry
Started.
```

We need to also start Device Virtual, but before doing so, increase the logging verbosity using snap options to add logging for the produced data:
``` title="🚀 Ubuntu Core"
$ snap set edgex-device-virtual config.writable-loglevel=DEBUG
$ snap start --enable edgex-device-virtual
Started.
```

Inspect the logs:
``` title="🚀 Ubuntu Core"
$ snap logs edgexfoundry
...
2023-05-24T15:43:54Z edgexfoundry.consul[2785]: 2023-05-24T15:43:54.667Z [INFO]  agent: Synced check: check=support-notifications
2023-05-24T15:43:54Z edgexfoundry.consul[2785]: 2023-05-24T15:43:54.801Z [INFO]  agent: Synced check: check=core-data
2023-05-24T15:43:55Z edgexfoundry.consul[2785]: 2023-05-24T15:43:55.220Z [INFO]  agent: Synced check: check=core-command
2023-05-24T15:43:55Z edgexfoundry.consul[2785]: 2023-05-24T15:43:55.368Z [INFO]  agent: Synced check: check=core-metadata
2023-05-24T15:43:56Z edgexfoundry.consul[2785]: 2023-05-24T15:43:56.208Z [INFO]  agent: Synced check: check=support-scheduler
2023-05-24T15:44:03Z edgexfoundry.consul[2785]: 2023-05-24T15:44:03.596Z [INFO]  agent: Synced check: check=device-virtual


$ snap logs -f edgex-device-virtual
...
2023-05-24T15:44:14Z edgex-device-virtual.device-virtual[3369]: level=DEBUG ts=2023-05-24T15:44:14.269393977Z app=device-virtual source=utils.go:80 msg="Event(profileName: Random-UnsignedInteger-Device, deviceName: Random-UnsignedInteger-Device, sourceName: Uint64, id: 77701381-5bbc-404d-a9b5-f30d58182ac6) published to MessageBus on topic: edgex/events/device/device-virtual/Random-UnsignedInteger-Device/Random-UnsignedInteger-Device/Uint64"
2023-05-24T15:44:19Z edgex-device-virtual.device-virtual[3369]: level=DEBUG ts=2023-05-24T15:44:19.066059149Z app=device-virtual source=reporter.go:195 msg="Publish 0 metrics to the 'edgex/telemetry/device-virtual' base topic"
2023-05-24T15:44:19Z edgex-device-virtual.device-virtual[3369]: level=DEBUG ts=2023-05-24T15:44:19.06612871Z app=device-virtual source=manager.go:123 msg="Reported metrics..."
^C
```

All services appear healthy.
The Device Virtual logs show that the service is producing the expected synthetic data.

Let's exit the SSH session:
``` title="🚀 Ubuntu Core"
$ exit
logout
Connection to localhost closed.
```

... and query data from outside via the API Gateway:
```bash title="🖥 Desktop"
curl --insecure https://localhost:8443/core-data/api/{{api_version}}/reading/all?limit=2
```

Since the security is enabled, the access is not authorized.
You can follow the instructions from the [getting started](../../getting-started/Ch-GettingStartedSnapUsers/#adding-api-gateway-users) to add a user to API Gateway, and generate a JWT token to access the API securely.

---
{==

In this chapter, we demonstrated how to build an image that is pre-loaded with some EdgeX snaps. We then connected into a (virtual) machine instantiated with the image, verified the setup and performed additional steps to interactively start and configure the services.

In the next chapter, we walk you through creating an image that comes pre-loaded with this configuration, so it boots into a working EdgeX environment.

==}
## B. Override configurations
    
In this chapter, we will improve our OS image so that:

- EdgeX services start automatically
- EdgeX security is disabled for demonstration purposes

### Seed configurations to snaps
Overriding the snap configurations upon installation is possible with [gadget snaps](https://snapcraft.io/docs/the-gadget-snap).

The [`pc` gadget](https://snapcraft.io/pc) is available as a prebuilt snap in the store, however, in this chapter, we need to build our own to include custom configurations, passed in as default values to snaps.
We will use the source code for Core22 AMD64 gadget from [here](https://github.com/snapcore/pc-amd64-gadget/tree/22) as basis.

!!! tip
    For a Raspberry Pi, you need to use the [pi-gadget](https://github.com/snapcore/pi-gadget) instead.

Clone the repo branch:
```bash title="🖥 Desktop"
git clone https://github.com/snapcore/pc-amd64-gadget.git --branch=22
```

Add the following root level object to `pc-amd64-gadget/gadget.yml`:
```yaml
defaults:
  # edgexfoundry
  AZGf0KNnh8aqdkbGATNuRuxnt1GNRKkV: # snap id
    # automatically start all the services
    autostart: true
    # disable security
    security: false
    # override a single service's startup message
    apps.core-data.config.service-startupmsg: "Core Data Startup message from gadget!"
    # set bind address of services to all interfaces via the common config
    apps.core-common-config-bootstrapper.config.all-services-service-serverbindaddr: 0.0.0.0
    
  # edgex-device-virtual
  AmKuVTOfsN0uEKsyJG34M8CaMfnIqxc0: # snap id
    # automatically start the service
    autostart: true
    config:
      # configure the service so it does not use the secret store
      edgex-security-secret-store: false
      # override the startup message
      service-startupmsg: "Startup message from gadget!"
```

For service startup and other configuration overrides, refer to [Managing services](../../getting-started/Ch-GettingStartedSnapUsers/#managing-services) and [Config Overrides](../../getting-started/Ch-GettingStartedSnapUsers/#config-overrides).


Build:
```bash title="🖥 Desktop"
$ cd pc-amd64-gadget
$ snapcraft -v
...
Created snap package pc_22-0.3_amd64.snap

$ cd ..
```

!!! note
    You need to rebuild the snap every time you change the `gadget.yaml` file.

### Build the image
Use ubuntu-image tool again to build a new image. Use the same instructions as [before](#build-the-ubuntu-core-image) but with an additional flag to set the path to gadget snap that we locally built above.

```bash title="🖥 Desktop"
$ ubuntu-image snap model.signed.yaml --validation=enforce \
  --snap pc-amd64-gadget/pc_22-0.3_amd64.snap # sideload the gadget
Fetching snapd
Fetching pc-kernel
Fetching core22
Fetching edgexfoundry
Fetching edgex-device-virtual
WARNING: "pc" installed from local snaps disconnected from a store cannot be refreshed subsequently!
Copying "pc-amd64-gadget/pc_22-0.3_amd64.snap" (pc)
```

The warning is because we sideloaded the gadget instead of pulling it from a store.

!!! tip    
    In production settings, a custom gadget would need to be uploaded to the [IoT App Store](https://ubuntu.com/internet-of-things/appstore) to also receive OTA updates.

!!! note
    You need to repeat the build every time you change and sign the **model** or rebuild the **gadget**.

!!! done
    The image file is now ready to be flashed on a medium to create a bootable drive with the needed applications and basic configurations.

### TRY IT OUT
Refer to the following to:

- [Run in an emulator](#run-in-an-emulator) - used in this guide
- [Flash the image on disk](#flash-the-image-on-disk)

This time, as set in the gadget defaults, services are started by default and security is disabled.


SSH to the Ubuntu Core machine as before and verify some of the seeded configurations:

``` title="🚀 Ubuntu Core"
$ snap services
Service                                       Startup   Current   Notes
edgex-device-virtual.device-virtual           enabled   active    -
edgexfoundry.consul                           enabled   active    -
edgexfoundry.core-command                     enabled   active    -
edgexfoundry.core-common-config-bootstrapper  enabled   inactive  -
edgexfoundry.core-data                        enabled   active    -
edgexfoundry.core-metadata                    enabled   active    -
edgexfoundry.nginx                            disabled  inactive  -
edgexfoundry.redis                            enabled   active    -
edgexfoundry.security-bootstrapper-consul     disabled  inactive  -
edgexfoundry.security-bootstrapper-nginx      disabled  inactive  -
edgexfoundry.security-bootstrapper-redis      disabled  inactive  -
edgexfoundry.security-proxy-auth              disabled  inactive  -
edgexfoundry.security-secretstore-setup       disabled  inactive  -
edgexfoundry.support-notifications            enabled   active    -
edgexfoundry.support-scheduler                enabled   active    -
edgexfoundry.vault                            disabled  inactive  -

$ snap get edgex-device-virtual -d
{
  "autostart": true,
  "config": {
    "edgex-security-secret-store": false,
    "service-startupmsg": "Startup message from gadget!"
  }
}
```

Verify that Device Virtual has the startup message set from the gadget:
``` title="🚀 Ubuntu Core"
$ snap logs -n=all edgex-device-virtual | grep "Startup message"
2023-05-24T16:52:05Z edgex-device-virtual.device-virtual[2807]: level=INFO ts=2023-05-24T16:52:05.791386915Z app=device-virtual source=variables.go:457 msg="Variables override of 'Service/StartupMsg' by environment variable: SERVICE_STARTUPMSG=Startup message from gadget!"
2023-05-24T16:52:22Z edgex-device-virtual.device-virtual[3010]: level=INFO ts=2023-05-24T16:52:22.342760716Z app=device-virtual source=message.go:55 msg="Startup message from gadget!"
```

Since security is disabled and Core Data has been configured to listen on all interfaces (instead of just the loopback), we can now query data (insecurely) from outside:
```bash title="🖥 Desktop"
$ curl --no-progress-meter http://localhost:59880/api/{{api_version}}/reading/all?limit=2 | jq
{
  "apiVersion" : "{{api_version}}",
  "statusCode": 200,
  "totalCount": 86,
  "readings": [
    {
      "id": "66c0e3ae-70a5-41b1-931f-bf680b2814ed",
      "origin": 1684948755626088200,
      "deviceName": "Random-Boolean-Device",
      "resourceName": "Bool",
      "profileName": "Random-Boolean-Device",
      "valueType": "Bool",
      "value": "true"
    },
    {
      "id": "94ec2182-7a0b-4515-8bcd-5445b8d59d2d",
      "origin": 1684948755624763400,
      "deviceName": "Random-UnsignedInteger-Device",
      "resourceName": "Uint32",
      "profileName": "Random-UnsignedInteger-Device",
      "valueType": "Uint32",
      "value": "2463192424"
    }
  ]
}

```
We can do that only for servers that have their ports forwarded to the emulator's host as configured in [Run in an emulator](#run-in-an-emulator).

Query all registered devices from Core Metadata:
```bash title="🖥 Desktop"
$ curl --no-progress-meter  http://localhost:59881/api/{{api_version}}/device/all | jq '.devices[].name'
"Random-Boolean-Device"
"Random-Float-Device"
"Random-UnsignedInteger-Device"
"Random-Binary-Device"
"Random-Integer-Device"
```
The response shows 5 virtual devices, registered by Device Virtual.
---
{==

In this chapter, we created an OS image which comes with EdgeX components that have overridden server configurations.
We can extend the server configurations by setting other defaults in the gadget.
This mechanism is made possible via a combination of snap options and environment variable overrides implemented for EdgeX services.

Overriding configuration fields is sufficient in most scenarios.
However, there are situations in which we need to override entire configuration files instead of just some fields:

1. When we want to override entire server configuration files, rather than a few fields.
2. When we need to add or change device, profile, and provision watcher configurations.

There are different ways to tackle the above situations such as pre-populating the EdgeX Config Provider and Core Metadata with the needed data, or deploying a local agent which takes care of the provisioning on runtime. 
In the next chapter, we will address the above requirements by deploying a snap which supplies custom configuration files to applications.

==}
## C. Replace configuration files

This chapter builds on top of what we did previously and shows how to override entire configuration files supplied via a snap package, called the [config provider snap](../../getting-started/Ch-GettingStartedSnapUsers/#config-provider-snap).

### Create a config provider for Device Virtual
The EdgeX Device Virtual service cannot be fully configured using environment variables / snap options. Because of that, we need to package the modified config files and replace the defaults.
Moreover, it is tedious to override many configurations one by one, compared to having a file which contains all the needed modifications.

Since we want to create an OS image pre-loaded with the configured system, we need to make sure the configurations are there without any manual user interaction. We do that by creating a snap which provides the configuration files to the Device Virtual snap.

For this exercise, we will replace the default Device Virtual configurations with a new set of files, containing just one virtual device and profile.

We use the [config provider snap example](https://github.com/canonical/edgex-config-provider) as basis which already includes the mentioned configuration files:

```bash title="🖥 Desktop"
$ git clone https://github.com/canonical/edgex-config-provider.git

$ tree edgex-config-provider/examples/device-virtual/res/
edgex-config-provider/examples/device-virtual/res/
├── configuration.yaml
├── devices
│   └── devices.yaml
├── profiles
│   └── device.virtual.float.yaml
└── README.md
```

This example includes only Device Virtual configurations.
However, it is structured to allow the supply of configuration files to multiple EdgeX app and device services. 

We'll continue with this example snap which is named `edgex-config-provider-example`.

!!! tip
    In production settings, you would create your own snap under a unique name and [release it](https://snapcraft.io/docs/releasing-your-app) to the public snap store or a private [IoT App Store](https://ubuntu.com/internet-of-things/appstore) along with your gadget. 
    This will allow OTA updates as well as secure control of the provided configuration.

Build:
```bash title="🖥 Desktop"
$ cd edgex-config-provider
$ snapcraft -v
...
Created snap package edgex-config-provider-example_<...>.snap

$ cd ..
```

This will build for our host architecture which is `amd64`.
You can perform [remote builds](https://snapcraft.io/docs/remote-build) to build for other architectures.

Let's upload the snap and release it to the `latest/edge` channel:
```bash title="🖥 Desktop"
snapcraft upload --release=latest/edge edgex-config-provider/edgex-config-provider-example_<...>.snap
```
Uploading to the store is necessary because we need to define a connection contract on the OS between the config provider and Device Virtual snaps.

Query the snap ID from the store:
```bash title="🖥 Desktop"
$ snap info edgex-config-provider-example | grep snap-id
snap-id: WWPGZGi1bImphPwrRfw46aP7YMyZYl6w
```

### Add the config provider to the image
Perform the following:

1) Add the config provider snap to `model.yaml`:

```yaml
- name: edgex-config-provider-example
  type: app
  default-channel: latest/edge
  id: WWPGZGi1bImphPwrRfw46aP7YMyZYl6w
```

2) Sign the model as before:
```bash title="🖥 Desktop"
yq eval model.yaml -o=json | snap sign -k edgex-demo > model.signed.yaml
```

3) Add the following root level object to `pc-amd64-gadget/gadget.yaml`:
```yaml
connections:
   -  # Connect edgex-device-virtual's plug (consumer)
      plug: AmKuVTOfsN0uEKsyJG34M8CaMfnIqxc0:device-virtual-config
      # to edgex-config-provider-example's slot (provider) to override the default configuration files.
      slot: WWPGZGi1bImphPwrRfw46aP7YMyZYl6w:device-virtual-config
```
This tells the system to connect the `device-virtual-config` plug of the Device Virtual snap to the slot of the same name on the config provider snap.

4) Rebuild the gadget:
```bash title="🖥 Desktop"
$ cd pc-amd64-gadget
$ snapcraft -v
...
Created snap package pc_22-0.3_amd64.snap

$ cd ..
```

### Build the image
Use ubuntu-image tool again to build a new image. Use the same instructions as [before](#build-the-ubuntu-core-image) to build:

```bash title="🖥 Desktop"
$ ubuntu-image snap model.signed.yaml --validation=enforce \
  --snap pc-amd64-gadget/pc_22-0.3_amd64.snap
Fetching snapd
Fetching pc-kernel
Fetching core22
Fetching edgexfoundry
Fetching edgex-device-virtual
Fetching edgex-config-provider-example
WARNING: "pc" installed from local snaps disconnected from a store cannot be refreshed subsequently!
Copying "pc-amd64-gadget/pc_22-0.3_amd64.snap" (pc)
```

Note the addition of our config provider snap in the output.

!!! done
    The image file is now ready to be flashed on a medium to create a bootable drive with the needed applications and custom configuration files.

### TRY IT OUT
Refer to the following to:

- [Run in an emulator](#run-in-an-emulator) - used in this guide
- [Flash the image on disk](#flash-the-image-on-disk)

SSH to the Ubuntu Core machine and verify the installations:
    
List of snaps:
``` title="🚀 Ubuntu Core"
$ snap list
Name                           Version                   Rev    Tracking          Publisher   Notes
core22                         20230503                  634    latest/stable     canonical✓  base
edgex-config-provider-example  v3.0.0-beta+git6.1778bd4  29     latest/edge       farshidtz   -
edgex-device-virtual           3.0.0-dev.51              673    latest/edge       canonical✓  -
edgexfoundry                   3.0.0-dev.164             4455   latest/edge       canonical✓  -
pc                             22-0.3                    x1     -                 -           gadget
pc-kernel                      5.15.0-71.78.1            1281   22/stable         canonical✓  kernel
snapd                          2.59.4                    19361  latest/candidate  canonical✓  snapd
```
Note that we now also have `edgex-config-provider-example` in the list.

Verify that Device Virtual has the startup message overridden via the gadget defaults:
``` title="🚀 Ubuntu Core"
$ snap logs -n=all edgex-device-virtual | grep "Startup message"
2023-05-25T10:24:50Z edgex-device-virtual.device-virtual[2924]: level=INFO ts=2023-05-25T10:24:50.447466922Z app=device-virtual source=variables.go:457 msg="Variables override of 'Service/StartupMsg' by environment variable: SERVICE_STARTUPMSG=Startup message from gadget!"
2023-05-25T10:25:03Z edgex-device-virtual.device-virtual[3136]: level=INFO ts=2023-05-25T10:25:03.761993667Z app=device-virtual source=message.go:55 msg="Startup message from gadget!"
```

From the host machine, query the device metadata to ensure that Device Virtual has registered only a single virtual device: 
```bash title="🖥 Desktop"
$ curl --no-progress-meter  http://localhost:59881/api/{{api_version}}/device/all | jq '.devices[].name'
"Random-Float-Device"
```

---
{==

Congratulations! You deployed a system that is pre-configured to have:

- A set of EdgeX components
- Configuration overrides for the EdgeX components
- Custom configuration files for the EdgeX Device Virtual service

==}
## Run in an emulator
Running the image in an emulator makes it easier to quickly try the image and find out possible issues.

We use a `amd64` QEMU emulator. Refer to [Testing Ubuntu Core with QEMU](https://ubuntu.com/core/docs/testing-with-qemu) to setup the dependencies and learn about the various emulation options. Here, we provide the command to run without TPM emulation.

!!! warning
    The `pc.img` file passed to the emulator is used as the secondary storage. It persists any changes made to the partitions during the installation and any user modifications after the boot.
    You can stop and re-start the emulator at a later time without losing your changes.

    To do a fresh start or to flash this image on disk, your need to rebuild the image.
    Alternatively, you can make a copy before using it in QEMU.

Run the following command and wait for the boot to complete:
```bash title="🖥 Desktop"
sudo qemu-system-x86_64 \
 -smp 4 \
 -m 4096 \
 -drive file=/usr/share/OVMF/OVMF_CODE.fd,if=pflash,format=raw,unit=0,readonly=on \
 -drive file=pc.img,cache=none,format=raw,id=disk1,if=none \
 -device virtio-blk-pci,drive=disk1,bootindex=1 \
 -machine accel=kvm \
 -serial mon:stdio \
 -net nic,model=virtio \
 -net user,hostfwd=tcp::8022-:22,hostfwd=tcp::8443-:8443,hostfwd=tcp::59880-:59880,hostfwd=tcp::59881-:59881
```

The above command forwards:

- SSH port `22` of the emulator to `8022` on the host
- API Gateway's port `8433` for external access in chapter A
- Core Data's port `59880`
- Core Metadata's port `59881`

!!! failure "Could not set up host forwarding rule 'tcp::8443-:8443'"
    This means that the port 8443 is not available on the host. Try stopping the service that uses this port or change the host port (left hand side) to another port number, e.g. `tcp::18443-:8443`.
    
!!! success
    Once the installation is complete, you'll see the initialization interface; Refer [here](#initialization) for details.

## Flash the image on disk
!!! warning
    If you have used `pc.img` to install in QEMU, the image has changed. You need to rebuild a new copy before continuing.

The installation instructions are device specific. You may refer to Ubuntu Core section in [this page](https://ubuntu.com/download/iot). For example:

- [Intel NUC](https://ubuntu.com/download/intel-nuc) - applicable to most computers with an attached secondary storage
- [Raspberry Pi](https://ubuntu.com/download/raspberry-pi)

A precondition to continue with some of the instructions is to compress `pc.img`. This speeds up the transfer and makes the input file similar to official images, improving compatibility with the available instructions.

To compress with the lowest compression rate of zero:
```bash title="🖥 Desktop"
$ xz -vk -0 pc.img
pc.img (1/1)
  100 %     817.2 MiB / 3,309.0 MiB = 0.247    10 MiB/s       5:30             

$ ls -lh pc.*
-rw-rw-r-- 1 ubuntu ubuntu 3.3G Sep 16 17:03 pc.img
-rw-rw-r-- 1 ubuntu ubuntu 818M Sep 16 17:03 pc.img.xz
```
A higher compression rate significantly increases the processing time and needed resources, with very little gain.

Follow the device specific instructions.

!!! success
    You may refer [here](#initialization) for the initialization steps appearing by default.

## Initialization
Once the installation is complete, you will see the interface of the `console-conf` program. It will walk you through the networking and user account setup. You'll need to enter the email address of your Ubuntu account to create a OS user account with your registered username and have your SSH public keys deployed as authorized SSH keys for that user.
If you haven't done so, follow the instructions [here](https://snapcraft.io/docs/creating-your-developer-account) to add your SSH keys before doing this setup.

Read [here](https://ubuntu.com/core/docs/system-user) to know how the manual account setup looks like and how it can be automated.

## References
- [Getting Started using Snaps](../../getting-started/Ch-GettingStartedSnapUsers/)
- [EdgeX Core Data](../../microservices/core/data/Ch-CoreData/)
- [Inside Ubuntu Core](https://ubuntu.com/core/docs/uc20/inside)
- [Gadget snaps](https://snapcraft.io/docs/gadget-snap)
- [Testing Ubuntu Core with QEMU](https://ubuntu.com/core/docs/testing-with-qemu)
- [Ubuntu Core - Image building](https://ubuntu.com/core/docs/image-building#heading--testing)
- [Ubuntu Core - Custom images](https://ubuntu.com/core/docs/custom-images)
- [Ubuntu Core - Building a gadget snap](https://ubuntu.com/core/docs/gadget-building)
