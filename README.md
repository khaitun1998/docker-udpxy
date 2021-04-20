# UDPXY (RPi) WITH 2 NETWORK INTERFACES

Udpxy is UDP-to-HTTP translator for IGMP streams. <br/>
Forked from [agrrh/docker-udpxy](https://github.com/agrrh/docker-udpxy)

In this version, i will guide you how to config UDPXY to proxy multicast stream into HTTP, using Raspberry Pi with 2 network interfaces, one for getting multicast stream from IPTV Provider (ISP), and other is for internet connection/your LAN network.

This has been tested to work perfectly with ARM, ARM64, x86-64, Raspberry Pi 4 (2GB).

# Config

- First, we need to make sure that our 2 network interfaces are all up and running normally. To do that, we type this command in terminal. If not, you will have to check it by yourself :-)

```bash
pi@raspberrypi:~ $ ifconfig
...

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.0.155  netmask 255.255.255.0  broadcast 172.16.0.255
        inet6 fe80::107d:9fde:4188:c9db  prefixlen 64  scopeid 0x20<link>
        ether dc:a6:32:8b:xx:xx  txqueuelen 1000  (Ethernet)
        RX packets 18598195  bytes 2729582863 (2.5 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 35367023  bytes 1813772789 (1.6 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.11.xx.xx  netmask 255.255.254.0  broadcast 10.11.xx.xx
        inet6 fe80::d195:ae9c:edc6:3906  prefixlen 64  scopeid 0x20<link>
        ether 00:e0:4c:36:xx:xx  txqueuelen 1000  (Ethernet)
        RX packets 31205084  bytes 3658871636 (3.4 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1064  bytes 65938 (64.3 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
...
```

- As we can see here in my case, <b>eth0</b> is the interface that will connect to the internet, as well as my local LAN network. And <b>eth1</b> is the interface that connect directly to the IPTV Provider. 

- Our mission here is to proxy multicast stream from <b>eth1</b> to <b>eth0</b>. Let's begin!

# Dockerfile Config

```Docker
FROM buildpack-deps as builder

ARG UDPXY_SRC_URL
ENV UDPXY_SRC_URL=${UDPXY_SRC_URL:-"http://www.udpxy.com/download/udpxy/udpxy-src.tar.gz"}

WORKDIR /tmp
RUN wget -O udpxy-src.tar.gz ${UDPXY_SRC_URL}
RUN tar -xzvf udpxy-src.tar.gz
RUN cd udpxy-* && make && make install

FROM debian:stable

COPY --from=builder /usr/local/bin/udpxy /usr/local/bin/udpxy
COPY --from=builder /usr/local/bin/udpxrec /usr/local/bin/udpxrec

# service will run at port 4022
# receive multicast stream through eth1 interface
# serve max 50 CCU
# Renew subscription each 180 secs (i.e. 3 mins)
ENTRYPOINT ["/usr/local/bin/udpxy"]
CMD ["-m", "eth1", "-v", "-T", "-p", "4022", "-S", "-M", "180", "-c", "50"]

```

- In the Dockerfile, replace <b>eth1</b> with your current multicast-enabled interface in config step.
- You can also change the service port 4022 with your desire port.

# Build docker

- After each time you change the Dockerfile, you have to build/re-build the docker image in order to apply those new changes.
- To build docker image, simply just run:

```bash
docker build . \
  --build-arg UDPXY_SRC_URL=http://www.udpxy.com/download/udpxy/udpxy-src.tar.gz \
  -t khaidq/udpxy
```

# Usage

- Hit this command into your terminal

```bash
# Run container as daemon

docker run -d --name udpxy \
  --network host \
  --restart unless-stopped \
  khaidq/udpxy \
```

- Example of opening a multicast link in VLC:

| Multicast | Unicast (HTTP) |
| ------ | ------ |
| `rtp://@239.0.5.185:8208` | `http://172.16.0.155:4022/rtp/239.0.5.185:8208` |
