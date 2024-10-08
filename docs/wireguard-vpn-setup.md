# Wireguard VPN setup

### Installing Wireguard

```bash
sudo apt install wireguard
```

### Configuring Wireguard (Server)

We'll need to configure the VPN server for clients to connect to.

Generate public and private keys

```bash
wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

Private keys should never be shared with anyone and should always be kept secure!

This will create two files in the `/etc/wireguard` directory called **privatekey** and **publickey**. You can view the contents by using the `cat` command.

#### Create a configuration file for the tunnel device

```bash
sudo nano /etc/wireguard/wg0.conf
```

It should have the below contents in it:

```
[Interface]
Address = 10.0.0.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE; iptables -A FORWARD -o %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens18 -j MASQUERADE; iptables -D FORWARD -o %i -j ACCEPT
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>
```

> * **Address** - The IP address for the tunnel device (wg0) - it can be separate to your home LAN subnet.
> * **SaveConfig** - When set to true, the current state of the interface is saved to the configuration file when shutdown.
> * **ListenPort** - The listening port.
> * **PrivateKey** - A private key generated by the wg genkey command. (To see the contents of the file type: `sudo cat /etc/wireguard/privatekey`)
> * **PostUp** - Command or script that is executed before bringing the interface up. In this example, we’re using iptables to enable masquerading. This allows traffic to leave the server, giving the VPN clients access to the Internet.
> * **PostDown** - command or script which is executed before bringing the interface down. The iptables rules will be removed once the interface is down. {: .prompt-info }

Make sure to replace `ens3` after `-A POSTROUTING` to match the name of your public network interface. You can easily find the interface with:

```bash
ip -o -4 route show to default | awk '{print $5}'
```

> The configuration above was adapted from what you see on most guides. There was an issue where the tunnel was created but no traffic was traversing. Had to include additional rules in the iptables for PostUp and PostDown.
>
> [https://www.reddit.com/r/WireGuard/comments/bkt4wl/connected\_but\_no\_internet/](https://www.reddit.com/r/WireGuard/comments/bkt4wl/connected\_but\_no\_internet/)
>
> PostUp - iptables -A FORWARD -o %i -j ACCEPT
>
> PostDown - iptables -D FORWARD -o %i -j ACCEPT {: .prompt-info }

#### Good Security practices

The `wg0.conf` and `privatekey` files should not be readable to normal users. Use `chmod` to set the permissions to `600`:

```bash
sudo chmod 600 /etc/wireguard/privatekey
sudo chmod 600 /etc/wireguard/wg0.conf
```

#### Server Network and Firewall Configuration(?)

IP forwarding must be enabled for NAT to work. Open the `/etc/sysctl.conf` file and add or uncomment the following line:

```
net.ipv4.ip_forward=1
```

Save and the apply the changes:

```bash
sudo sysctl -p
```

#### Firewall configurations

If you are using UFW to manage your firewall, you'll need to open UDP traffic on port 51820.

Check if you are using UFW firewall:

```bash
sudo ufw status verbose
```

If you are using it, you'll need to add the below:

```bash
sudo ufw allow 51820/udp
```

#### Test the tunnel device

```bash
sudo wg-quick up wg0
```

Check the interface has been created.

```bash
sudo wg show wg0
```

```
chunaki@wireguard-vpn:~$ sudo wg show wg0
interface: wg0
  public key: spMapi0CqSVM6scC9bQcoeS3EGagYp5B6DADFaVClVw=
  private key: (hidden)
  listening port: 51820
```

#### Enable Wireguard to start automatically

```bash
sudo systemctl start wg-quick@wg0.service
sudo systemctl enable wg-quick@wg0.service
systemctl status wg-quick@wg0.service
```

### Wireguard Client configuration

There's sort of two ways this can be done. You can create the client on the server itself and then set it up on the client. Alternatively, you can create everything on the client itself and then update the server to include the client. In this example, I'll show how you create it on the server and add to the client.

I'll create a new folder for the clients and generate a new public and private key pair for it.

```bash
sudo mkdir -p /etc/wireguard/clients
wg genkey | sudo tee /etc/wireguard/clients/user1.key | wg pubkey | sudo tee /etc/wireguard/clients/user1.key.pub
```

> I'll call this client "user1" but you'll need to change it depending on your use cases.

#### Create a new configuration file for the new client.

```bash
sudo nano /etc/wireguard/clients/user1.conf
```

```
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.0.0.2/24
[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
AllowedIPs = 192.168.0.0/24
Endpoint = <WAN_IP_ADDRESS>:32645
```

> * **PrivateKey** - To see the client private key, run this on the server: `sudo cat /etc/wireguard/clients/privatekey`
> * **Address** - A comma-separated list of v4 or v6 IP addresses for the `wg0` interface. This will be the IP address that the client uses, can't be the same as the server!
> * **PublicKey** - A public key of the peer you want to connect to. (The contents of the server’s `/etc/wireguard/publickey` file).
> * **Endpoint** - An IP or hostname of the peer you want to connect to followed by a colon, and then a port number on which the remote peer listens to. (This can also be a domain if you have it setup)
> * **AllowedIPs** - A comma-separated list of v4 or v6 IP addresses from which incoming traffic for the peer is allowed and to which outgoing traffic for this peer is directed. I'm using 192.168.0.0/24 as I only want to use the VPN to access my home internal network. If you want all traffic to go over this interface, use 0.0.0.0/0. This is basically split-tunnelling. {: .prompt-info }

> You'll need to do this for every client. Each client will also have its own unique IP address in the subnet.

#### Setting up Port forwards

You'll need to open ports to the Wireguard VPN server on the ports that you have specified.

In my case, I've given the server the IP address of 192.168.0.10 and the Wireguard server is configured to Listen on port 51820. In the client configuration above, I've asked it to try and get into my home using the WAN IP address on port 32645.

> Traffic coming into port 32645 on my WAN IP Address will be pointed to 192.168.0.10 on port 51820

#### Add the client as a peer on the server

In order for the client to be able to route traffic, we'll need to add the client as peer on the server. This tells the server that this client is allowed to open a connection with the server. There's two ways you can do this:

* Editing the configuration manually on `/etc/wireguard/wg0.conf`

```
[Peer]
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32
```

> In this case, AllowedIPs will be the IP address of the client's tunnel interface.

* Using the below command:

```bash
sudo wg set wg0 peer <CLIENT_PUBLIC_KEY> allowed-ips 10.0.0.2
```

> This needs to be done whether you setup the client on the client device itself or on the server!

#### Setting up QR codes for easy setup

Rather than having to enter the information manually whenever you want to setup the VPN on a client, we can generate a QR code which just needs to be scanned to allow for the VPN to be setup. This requires the client configuration to already be setup beforehand.

**Install the QR code package**

```bash
sudo apt install qrencode
```

**Generating a QR code**

```bash
qrencode -t ansiutf8 < /etc/wireguard/clients/user1.conf
```

As we've already created a **user1.conf** file earlier, we'll generate a QR code for this client. It'll produce a QR code which you can use to setup the VPN connection on any compatible Wireguard device.

#### Final tests and checks

Using the QR code, setup the VPN and check that it all works OK. If you've allowed it to route all traffic using **ALLOWEDIPS=0.0.0.0/0**, make sure the public IP shows the WAN IP address of the server. If you've only allowed for it to access a particular subnet, check you can access or ping it.

On the server, you can see that stats of the clients using `wg show`

```
root@wireguard-vpn:/home/chunaki# wg show
  interface: wg0
  public key: spMapi0CqSVM6scC9bQcoeS3EGagYp5B6DADFaVClVw=
  private key: (hidden)
  listening port: 51820

  peer: U6BBdGlMduHoP+A/gySQZ5hcZCD+1EnU6vTrI564WVg=
  endpoint: 148.252.132.163:9128
  allowed ips: 10.0.0.2/32
  latest handshake: 1 hour, 10 minutes, 35 seconds ago
  transfer: 2.28 MiB received, 121.94 MiB sent
```

#### Removing peers

You can find (KEY) under `wg show` for the peer you want to remove

```bash
sudo wg set wg0 peer (KEY) remove
sudo wg show
```
