---
title: "Setting Up Jumphost with Wireguard"
date: 2023-08-15
tags: "linux,vpn,networking"
---

# Setting Up Jumphost with Wireguard

The note contains step by step to setup VPN machine with Wireguard as jumphost for connecting to services within private network in Cloud.

## Configuration

```
 -------------        -------------        -------------   
|             |      |             |      |             |  
|   MacOS     | -->  |   Wireguard |  --> |   Nginx     |  
|             |      |    VM       |      |    VM       |  
 -------------        -------------        -------------   
```

* MacOS: Local network
* Wireguard VM: VPC `payment-vpc` subnet `payment-private-us-central1-public`
* Nginx VM: VPC `payment-vpc` subnet `payment-private-us-central1`

## Notes

1. Setting up new GCP Project
1. Setting up new custom VPC with 2 subnets:

    1. `payment-private-us-central1-public` . CIDR: `10.1.2.0/24`
    1. `payment-private-us-central1`. CIDR: `10.1.1.0/24`

1. Setting up following firewall rules:

    1. `payment-private-allow-http` allows `tcp:80` with `http-server` network tags
    1. `payment-allow-user-vpn` allows `udp:51820` with `wireguard` network tags
    1. `payment-private-allow-ssh` allows `tcp:22`
    1. `payment-private-allow-custom` allows all ports between subnets

1. Setting up NAT Gateway for private
1. Create 1 Nginx VM without public IP address.

    ```shell
    sudo apt update
    sudo apt install nginx
    curl localhost
    ```

1. Create 1 VPN VM and install wireguard with public IP address. Follow [this blog](https://www.procustodibus.com/blog/2022/11/wireguard-jumphost/) to configure wireguard on the jumphost

    ```shell
    sudo apt update
    sudo apt install wireguard
    sudo su
    cd /etc/wireguard
    wg genkey > jumphost.key
    wg pubkey < jumphost.key > jumphost.pub
    ```

    Create jumphost configuration
    ```txt
    # local settings for the jumphost
    [Interface]
    PrivateKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEE=
    Address = 10.0.0.1/24
    ListenPort = 51820
    PreUp = sysctl -w net.ipv4.conf.all.forwarding=1
    ```

    Run jumphost as systemd service
    ```shell
    sudo systemctl start wg-quick@wg0.service
    sudo systemctl enable wg-quick@wg0.service
    ```

    Check wireguard
    ```shell
    sudo wg
    ```

1. Testing access to nginx vm from jumphost box

    ```shell
    curl <nginx_private_ip_address>
    ```

1. Install and configure wireguard client in macos.

    ```txt    
    [Interface]
    PrivateKey = <CLIENT_PRIVATE_KEY>
    Address = 10.0.0.11 #client wireguard ip address

    # remote settings for the jumphost
    [Peer]
    PublicKey = <SERVER_PUBLIC_KEY>
    Endpoint = <WIREGUARD_VM_PUBLIC_IP_ADDRESS>:51820
    AllowedIPs = 10.1.1.0/24, 10.1.2.0/24, 10.0.0.1/32
    ```

    * `10.1.1.0/24` and `10.1.2.0/24` are CIDR from the subnets in `payment` vpc.

    * `10.0.0.1/32` should be the wireguard server ip address in `wg0` interface

1. Configure peer in jumphost

    ```txt    
    [Interface]
    PrivateKey = AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEE=
    Address = 10.0.0.1/24
    ListenPort = 51820
    PreUp = sysctl -w net.ipv4.conf.all.forwarding=1

    [Peer]
    PublicKey = <CLIENT_PUBLIC_KEY>
    AllowedIPs = 10.0.0.11/32 # CLIENT WIREGUARD IP ADDRESS
    ```

    * `CLIENT_WIREGUARD_IP_ADDRESS` is not the same as gcp vm private ip address. Look like this is ip address assigned to the virtual network interface.

1. Start VPN client on the macos

1. Testing ssh to jumpbox by using its private ip address

    ```shell
    ssh -i ~/.ssh/google_compute_engine imre.nagi@<wireguard_private_ip>
    ```

1. Setting up traffic forwarding

    ```shell
    sudo nano /etc/sysctl.conf
    ```

    uncomment `net.ipv4.ip_forward=1`

    ```shell
    sudo sysctl -p
    ```

    Find the network interface that can route the traffic to internal subnet

    ```shell
    ip route list default

    # default via 10.x.x.x dev ens4 proto dhcp src 10.y.y.y metric 100
    # ens4 is the network interface name
    ```

    Update `wg0.conf`
    ```
    PreUp = sysctl -w net.ipv4.conf.all.forwarding=1
    PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens4 -j MASQUERADE
    PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens4 -j MASQUERADE
    ```

    ```shell
    systemctl restart wg-quick@wg0.service
    ```

1. Test access to nginx webserver by using its ip address from macos

    ```shell
    ssh -i ~/.ssh/google_compute_engine imre.nagi@<nginx_private_ip>
    ```

## Open Questions

1. How to dynamically configure peer in both server and client? We need to assign the wireguard ip address to each client, how to do this dynamically?