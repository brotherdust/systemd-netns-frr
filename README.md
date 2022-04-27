# What is This?

There are use cases where it is desirable to isolate network forwarding and control plane functions into a dedicated Linux `netns`. For instance, a routing appliance with a management interface and several interfaces that are only for forwarding network traffic.

These systemd units make it easy to create a `netns`, attach the desired interfaces, and start FRR.

## Why Not Use DPDK+FRR?

The objective of this configuration is not primarily performance, but simplicity. It's much easier to attach a network interface to a `netns` than it is to configure it to use DPDK (if it's supported). Also, recent Linux kernels have very good forwarding performance, in some cases as good as what DPDK can offer. So, it made sense for my use-case to do it this way.

## Why Not Use VRF?

It takes longer to set up, and, by default, VRF's don't run in their own `netns`, so one loses the benefit of the implicit security context that a `netns` affords. It is possible to have VRF's *inside* of a `netns`, but that is beyond the scope of this setup.

# Audience

The user of this document is expected to have experience with SystemD-based Linux distributions.

# Setup

## 1. Install FRR

Follow the [FRR docs](http://docs.frrouting.org/en/latest/) to install for your distro.

Ensure FRR service is stopped:

```
systemctl stop frr.service
```

## 2. Identify Interfaces

Create a list of interfaces that will be dedicated to routing. The names will vary depending on distribution. In our example, we'll use `enps0f0` and `enps0f1`.

## 3. Copy The Units

### `netns` Template Unit File and FRR Unit Drop-In

```
sudo mkdir -p /etc/systemd/system/frr.service.d; \
sudo cp ./frr.service.d/frr.conf /etc/systemd/system/frr.service.d/frr.conf; \
sudo cp netns@.service /etc/systemd/system/netns@.service
```

### Interface Units

For each interface identified in step 2, copy `attach-example@.service` template unit file, replacing `example` int the filename with the interface name:

```
sudo cp attach-example@.service /etc/systemd/system/attach-enps0f0@.service; \
sudo cp attach-example@.service /etc/systemd/system/attach-enps0f1@.service
```

## 4. Enable Services

```
sudo systemctl enable netns@frr.service; \
sudo systemctl enable attach-enps0f0@frr.service; \
sudo systemctl enable attach-enps0f1@frr.service; \
sudo systemctl enable frr.service
```

## 5. Modify FRR Unit Drop-In

Edit the FRR unit drop-in configuration to add the interfaces we identified in step 2.

```
sudo nano /etc/systemd/system/frr.service.d/frr.conf
```

Change this:
```
### MODIFY
Requires=attach-example@frr.service
After=attach-example@frr.service
###
```

To this, matching our example:
```
### MODIFY
Requires=attach-enps0f0@frr.service
Requires=attach-enps0f1@frr.service

After=attach-enps0f0@frr.service
After=attach-enps0f1@frr.service
###
```

## 6. Rerfresh SystemD

```
sudo systemctl daemon-reload
```

## 7. Start FRR

```
sudo systemctl start frr.service
```

## 8. Confirm

The result of this command:
```
sudo ip netns exec frr ip link
```

Should now list the interfaces we attached to the `netns`
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enps0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
3: enps0f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether xx:xx:xx:xx:xx:xx brd ff:ff:ff:ff:ff:ff
```

## 9. Finish

Configure FRR as required.