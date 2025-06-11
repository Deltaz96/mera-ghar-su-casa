# OPNsense IPsec IKEv2 Road Warrior Setup on AWS

This project documents the setup of **OPNsense** as a virtual firewall/router within the **AWS cloud environment**, designed to securely provide **IPsec IKEv2 Road Warrior VPN** access with modern encryption and authentication practices.

> 🏠 **Note:** While this guide focuses on AWS deployment, the configuration and steps are equally applicable to **homelab** environments — whether running OPNsense on bare metal, virtual machines (Proxmox), or mini-PCs.

## Features

- 🛡️ **OPNsense on AWS**  
  Deploy and configure OPNsense on an EC2 instance, with proper networking.

- 🔐 **Let's Encrypt SSL Certificates**  
  Automatically issue and renew certificates via the built-in **ACME client** for IPsec authentication.

- 🌐 **IPsec IKEv2 Road Warrior VPN**  
  Set up a strongSwan-based IKEv2 VPN server for secure remote access across devices (macOS, iOS, Android, Linux).<br>
    (unfortunately it also works on windows).

- 🔥 **Firewall Rules for IPsec**  
  Define granular rules to control and secure access between VPN clients.

## Use Case

Ideal for individuals or small teams looking to deploy a scalable, secure, and certificate-based VPN solution in the cloud using open-source tools.

## 🎯 Goal

The goal of this project is to create a **secure, reliable, and scalable VPN access point** using OPNsense and IPsec IKEv2, suitable for both **cloud-based (AWS)** and **homelab** environments.

Specifically, this setup aims to:

- Enable **remote access** to internal resources through a **certificate-based IPsec IKEv2 VPN**.
- Leverage **Let's Encrypt** to automate secure certificate issuance via OPNsense's ACME client.
- Harden the deployment with **firewall rules** to strictly control VPN access.
- Offer a reproducible, documented configuration that can serve as a base for both personal and small team deployments.

This provides a modern alternative to traditional OpenVPN or L2TP setups, offering improved compatibility, performance, and security — especially on mobile and modern OS platforms.


## Start an instance with
1. OPNsense OS.
2. set a key pair.
3. in network, you need to configure to open these ports according to your needs
    - `22` TCP (ssh)
    - `80` TCP (HTTP)
    - `443` TCP (HTTPS)
    - `500` UDP (for ipsec)
    - `4500` UDP (for ipsec)

## Basic configuration

1. log into Web UI and go to
    - `DO NOT USE THE WIZARD FOR AWS` [¹](#1)
2. System: `Settings: General` and SET
    - `Hostname:`
	- `Domain:`
    - `Timezone:`
3. Go to `Interfaces: [WAN]` (most likely autoconfigured in AWS, but still check)
4. Go to `System: Firmware`
    - update OPNsense to latest and then reboot
    - install plugin `os-acme-client`

## Set up ACME ##
1. Go to `Services: ACME Client: Settings`
	- choose the `settings` tab
  	- enable plugin
  	- `log level` = `debug 3` [²](#2)
  	- uncheck `Show introduction pages`
	- click `Apply`
2. Go to `Services: ACME Client: Accounts`
    - add an `Email` in account (any working email)
	- click `Save`
3. Go to `Services: ACME Client: Challenge Types`
    - add a challenge
        - `challenge Type`= `DNS-01` [³](#3)
        - `DNS service`= `route53` [³](#3)
        - `AWS ID` [⁴](#4)
        - `AWS Secret` [⁴](#4)
4. Go to `Services: ACME Client: Certificates`
    - we need `2` certificates one for opnsense web UI and other for ipsec
    - add a cert request.
		- in `Common Name` add the FQDN for this instance. 
        - example `opnsense.lab.com`. [⁵](#5)
      	- Change the rest according to your needs
    - clone the above-created cert request
    	- in `Common Name` replace with what you prefer. 
        - example `ipsec.lab.com`. [⁶](#6)
5. Go to `System: Settings: Administration`
    - in `SSL Certificate` select the certificate for web UI created `opnsense.lab.com` 

## Set up ipsec IKEv2 ##
1. Go to `VPN: IPsec: Pre-Shared Keys`
    - create user
      	- `Local Identifier` = username
      	- `Pre-Shared Key` = password
      	- `type` = `EAP`
      	- click `save`
    - click on `Apply`
2. Go to `VPN: IPsec: Connections`
    - in `pools` tab
		- create new
      	- In Name set `Name` for the ip pool(obviously)
      	- In Network set ip range (for me `10.0.1.0/24`)<a id="pool"></a>
		- click `save`
    - in `Connections` tab
      	- create new 
        - turn on `advanced mode`
        - `Proposals` = `aes256-sha256-modp2048(DN14)` [⁷](#7)
        - `Version` = `IKEv2`
        - `Local addresses` = `ipsec.lab.com` [⁸](#8)
        - enable `UDP encapsulation`
        - `Rekey time(s)` = 2400 [⁹](#9)
        - `DPD delay(s)` = 30 [⁹](#9)
        - `Pools` = choose the one created one step before.
        - `Send certificate` = Always [¹⁰](#10)
        - `Keyingtries` = 0
        - `Description` = name of this `Connection`.
        - click `save`
    - new options should appear below the save button
        - `Local Authentication`
        - `Remote Authentication`
        - `Children`
    - In `Local Authentication`, create new 
        - `Authentication` = `Public key`
        - `id` = `ipsec.lab.com` [⁸](#8)
        - `Certificates` = `ipsec.lab.com` [¹¹](#11)
        - `Description` = name this `Local Authentication`
        - click `save`
    - In `Remote Authentication`, create new
        - `Authentication` = `EAP-MSCHAPv2`
        - `EAP id` = `%any`
        - `Description` = name this `Remote Authentication`
        - click `save`
    - in `Children` create new
        - `Start action` = `none`
        - `ESP proposals` = `aes256-sha256-modp2048(DN14)` [⁷](#7)
        - `Rekey time(s)` = 600
        - `Description` = name this `child`
        - click `save`
    - save the connection
    - check `Enable IPsec` and `click Apply`

##  Firewall ##
1. Go to `Firewall: Aliases`
    - add first alias
    	- `Name` = `internet_ipv4`<a id="FW-Alias1"></a>
      	- `type` = `Network(s)`
      	- `content` = `10.0.0.0/8` `172.16.0.0/12` `192.168.0.0/16` `127.0.0.0/8` [¹²](#12)
      	- Description = `IPsec-use as inverted`
    - add a second alias
    	- `Name` = `ipsec_ports`<a id="FW-Alias2"></a>
      	- `type` = `Port(s)`
      	- `content` = `500` `4500` [¹²](#12)
      	- `Description` = `IPsec-ports`
2. Go to `Firewall: NAT: Outbound`
    - `Mode`:  `Hybrid outbound NAT rule generation`
    	- click `Save`
    - add `Manual rules`
    	- `Source address` = `Single host or network`
        	- in the field under `Source address` enter your [`pool ip`](#pool) range created in IPsec.
      	- `Translation/target` = `WAN address`
      	- click `Save`
3. Go to `Firewall: Rules: WAN`
    - add new
      - `Protocol` = `UDP`
      - `Destination` = `Wan address`
      - `Destination port range` = [`ipsec_ports`](#FW-Alias2)
      - click `Save`
4. Firewall: Rules: IPsec
    - add new
    	- `Protocol` = `TCP/UDP`
      	- `Source` = `Single host or network`
    		- in the field under `Source` enter your [`pool ip`](#pool) range created in IPsec.
      	- enable/check `Destination/Invert` 
      	- `Destination` = [`internet_ipv4`](#FW-Alias1)
      	- click `Save`
    - clone the above-created rule [¹³](#13)
        - disable/uncheck `Destination/Invert` 
        - replace `internet_ipv4` in `Destination` with `LAN net`

---
<a id="1"></a>¹ Breaks opnsense's interfaces sometimes<br>
<a id="2"></a>² Used for debugging of `Acme client`<br>
<a id="3"></a>³ I am using AWS route53 for this.<br>
<a id="4"></a>⁴ needed to be made from AWS - `iam` with access to the hosted zone of domain.<br> 
<a id="5"></a>⁵ This Cert is for Web UI. Needs `A` record in `route53`.<br>
<a id="6"></a>⁶ This Cert is for IPsec. Needs `A` record in `route53`.<br>
<a id="7"></a>⁷ Using `aes256-sha256-modp2048(DN14)`, recommended by apple.<br>
<a id="8"></a>⁸ Should be same as the Common name in point 6.<br>
<a id="9"></a>⁹ Recommended by IPsec.<br>
<a id="10"></a>¹⁰ Recommended by Apple.<br>
<a id="11"></a>¹¹ Select the Cert created for IPsec in point 6.<br>
<a id="12"></a>¹² Press enter after each entry.<br>
<a id="13"></a>¹³ if you want to give `ipsec interface` access to `LAN interface`.



Reference <br>
https://docs.opnsense.org/manual/how-tos/ipsec-swanctl-rw-ikev2-eap-mschapv2.html