# Scenario

To service or maintain a PLC or HMI in a factory, we need to connect remotely. Often it is complicated to access directly to the PLC and in addition to an internet access, a public IP or a port forward are required which are often not available. IOhubOS integrates a VPN client to make this unnecessary.
If IOhubOS is connected to the internet (the only thing required) it becomes easily and safely reachable by the customer who needs to make maintenance to the PLC or access to the resources of IOhubOS or of the installed applications.

[![IOhubOS Scenario](https://github.com/iohubos/iohubos/blob/media/iohubos_scenario.svg?raw=true)](https://github.com/iohubos/iohubos)

## Architecture

To accomplish this scenario you need to configure a VPN server that is publicly reachable. IOhubOS will automatically connect to the VPN server and John will be able to connect when needed to the same VPN server to reach the PLC/HMI or the applications installed on IOhubOS.

[![IOhubOS VPN](https://github.com/iohubos/iohubos/blob/media/iohubos_vpn_server.svg?raw=true)](https://github.com/iohubos/iohubos)

So you need:

- IOhubOS with version 1.1.0 or higher
- VPN server with public IP
- Client to connect the PLC

## IOhubOS setup

Starting from version 1.1.0, IOhubOS provides auto-configuration to a Wireguard VPN server.

### Configuration

VPN configuration can be found in the `/iohub/envvars.d/1.1.0-envars` file.

```text
IOHUBOS_VPN_ENABLED='false'                 # enable wireguard vpn
IOHUBOS_VPN_SERVER_PUBLIC_KEY=''            # wireguard server public key
IOHUBOS_VPN_BOX_NUMBER=''                   # a number > 1; must be unique for each box and > 1 (1 is the server)
IOHUBOS_VPN_NETWORK=''                      # internal wireguard vpn network, e.g. 100.90.56.0/24
IOHUBOS_VPN_SERVER_IP_ADDRESS=''            # public server ip address
IOHUBOS_VPN_SERVER_IP_PORT=''               # public server port, e.g. 51820
IOHUBOS_VPN_EXPOSE='auto'                   # 'auto' to expose automatically the wireguard interface, otherwise set manually to set the AllowedIPs
IOHUBOS_VPN_ACCESS_HOST='false'             # if 'true', access to the IOhubOS host through VPN is granted. Denied otherwise.
VPN configuration examples
```

- To enable the VPN, you need to set the variable `IOHUBOS_VPN_ENABLED` to `true`.
- `IOHUBOS_VPN_SERVER_PUBLIC_KEY` set the server public key, see below ho to create a key for the server.
- To work, the VPN needs to rely on a class of dedicated private IP addresses that are not used by other networks. An example could be 100.90.56.0/24, if the server uses the address 100.90.56.1, IOhubOS could use 100.90.56.2. In this case `IOHUBOS_VPN_BOX_NUMBER` is the number `2`.
- `IOHUBOS_VPN_NETWORK` set the IP class of the Wireguard VPN network, for example `100.90.56.0/24`
- `IOHUBOS_VPN_SERVER_IP_ADDRESS` set the public Wireguard server ip address reachable from internet
- `IOHUBOS_VPN_SERVER_IP_PORT`  set the public Wireguard server port
- `IOHUBOS_VPN_EXPOSE` use  `auto` to expose automatically the Wireguard interface, otherwise set manually to set the AllowedIPs
- `IOHUBOS_VPN_ACCESS_HOST` if `true`, access to the IOhubOS host through VPN is granted. Denied otherwise.

### How to read IOhubOS Wireguard keys

You can find the private and public key (to copy and use in the server configuration) in folder `/iohub/vpn`

## VPN Server

We will use [Wireguard](https://www.wireguard.com/) as VPN technology, it's fast, modern, secure and easy to implement, open source and it's available for all platforms, you can download it from [here](https://www.wireguard.com/install/).

We assume you have the wireguard server installed and ready to use, you can find a good start from [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04), it's not the goal of this readme to build a Wireguard server.

Wireguard is based on a very simple concept, the server and the client exchange keys and if the keys match communication takes place over an encrypted tunnel. So we need to create two keys (one public and one private) for each device that needs to connect. Each client, IOhubOS and server must have and use a unique key pair.

### How to create Wireguard public and private keys

A simple docker container is available in [Github](https://github.com/iohubos/wg-keygenerator) to create random public and private keys.

```text
docker run --rm iohubos/wg-keygenerator
```

Output

```text
=================== Wireguard Key Pair ==================

Private key: iPDOhmjCWGoZvRDpK1dJM8IIUpmQdJS/UEtkZO1Blm0=
Public key:  OGvu+4liaX4GSD9SeJ4tYpt3LwehYsvETg54iK/X0CU=

=========================================================
```

These keys are an example, I recommend not to use them but to create new ones.
Create a key for the server and one for each client if needed. Some clients such as windows, if you add a new tunnel automatically create keys.
The IOhubOS VPN container automatically generates the keys so you don't need to create them and are available on `/iobub/vpn`

### Server configuration

This is an example of `/etc/wireguard/wg0.conf`

```text
[Interface]
Address = 100.90.56.1/32
ListenPort = 51820
PrivateKey = iPDOhmjCWGoZvRDpK1dJM8IIUpmQdJS/UEtkZO1Blm0=

#IOhubOS
[Peer]
PublicKey = x19RRA/q7Clh0qqafiD8zQsxyJk1WJcwqZZUAv7bmg8=
AllowedIPs = 100.90.56.2/32, 192.168.10.0/24

#client
[Peer] 
PublicKey = Zy3UHgQpWqhlk7QqzYn1tLGL6s7RREijnuG9BK1xJmg=
AllowedIPs = 100.90.56.3/32
```

- `[Interface]` definition server variables

- `Address` IP server for private Wireguard network

- `ListenPort` Listen UDP Wireguard port

- `PrivateKey` Wireguard Server private key

- `[Peer]` definition client variables (one per clients)

- `PublicKey` public key of the client
- `AllowedIPs` defines the IP address of the client. For IOhubOS, add the classes that must be reachable by other clients. For example 192.168.10.0/24 is added to AllowedIPs represents the PLC network that must be reachable by other clients in VPN.

After editing the file, activate the vpn tunnel with `wg-quick up wg0`

## Client example

This is an example configuration of a client that once connected accesses the PLC network 192.168.10.0/24

```text
[Interface]
PrivateKey = 0MzAMdPs1jHopdX4YUyFtWBz/kg45CJ4rEEXmBHgrWI= 
Address = 100.90.56.3/24 

[Peer] 
PublicKey = pa4jMqH04MpWjk0ujNPKSHdhkcVkBvTkMhORS0V1ZgQ= 
AllowedIPs = 100.90.56.0/24, 192.168.10.0/24 
Endpoint = myserver.com:51820 
PersistentKeepalive = 25
```

- `[Interface]` client variables definition

- `PrivateKey` Wireguard client private key

- `Address` IP client for private Wireguard network

- `[Peer]` peer variables definition, the server

- `PublicKey` Wireguard server public key

- `AllowedIPs` define the IP/classes that need to be reached remotely

- `Endpoint` public VPN server and port

- `PersistentKeepalive` keepalive for firewall NAT in seconds

The `AllowedIPs` on the client allow you to define the destinations are then the routes that are dynamically added to the client connection.
