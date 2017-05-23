# OpenAirInterface5G - SDR and Cloud based deployment
This tutorial is intended to setup all the OpenAirInterface5G components using the S1 interface, placing the core components in a cloud based system. The connection between OAI-UE and the OAI-eNB will be carried out by using USRP B210 SDR cards connected using wires and the selected cloud system where cloud components will be deployed will be OpenStack.

## Hardware setup
The proposed deployment has been tested using the hardware showed in the following figure:

![Alt text](/hw_configuration.PNG?raw=true "Optional Title")

The attenuation between UE and eNB should be adjusted between 40 and 60 dB.

## Core components deployment (HSS, MME and SP-GW).
The core components will be deployed in a cloud based system. In this tutorial, we are going to use OpenStack as a virtual machines manager. For this deployment we need three virtual machines, one per OpenAirCN componet with the following configuration:
* Ubuntu 14.04 LTS (32-bits or 64-bits)
* Kernel setup: Install 4.7.x kernel from pre-compiled debian package
  * `git clone https://gitlab.eurecom.fr/oai/linux-4.7.x.git`
  * `cd linux-4.7.x`
  * `sudo dpkg -i linux-headers-4.7.7-oaiepc_4.7.7-oaiepc-10.00.Custom_amd64.deb linux-image-4.7.7-oaiepc_4.7.7-oaiepc-10.00.Custom_amd64.deb`
* Install git 
  * `sudo apt-get install git`
* Download the latest development branch of OpenAir-cn
  * `git clone https://gitlab.eurecom.fr/oai/openair-cn.git`
  * `git checkout develop`
  * `git fetch && git pull`

<details>
<summary>Core Components Network Configuration</summary>

### Core Components Network Configuration
Once we have our virtual machines running, we have to configure all the network devices in order to allow the communication between them. We propose the following network configuration (OpenStack):

![Alt text](/oai-cn_network_config.PNG?raw=true "Optional Title")

</details>


<details>
<summary>Setup communication between eNB and Cloud based Core Network</summary>

### Setup communication between eNB and Cloud based Core Network
Before start configuring the core components, we have to setup a VPN between eNB, MME and SP-GW in order to allow IP level communication between them. In order to do so, we are going to setup an OpenVPN server in the eNB and generate certificates for both MME and SP-GW following this tutorial [How To Set Up an OpenVPN Server on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-14-04).

</details>

<details>
<summary>HSS Configuration and Building</summary>

### HSS Configuration and Building
Log into the HSS virtual machine and follow the instructions below:

* `sudo apt-get install mysql-server`
* `cd ~/openairinterface5g/SCRIPTS/`
* `./build_hss -i`
* Create folder for configuration files
  * `sudo mkdir -p /usr/local/etc/oai/freeDiameter`
* Copy Configuration files to the folder
  * `sudo cp OPENAIRCN_DIR/ETC/hss.conf /usr/local/etc/oai`
  * `sudo cp OPENAIRCN_DIR/ETC/acl.conf OPENAIRCN_DIR/ETC/hss_fd.conf /usr/local/etc/oai/freeDiameter`
* Set FQDN for the HSS by modifying `/etc/hosts` file
  * `127.0.1.1 hss.openair4G.eur hss`
* Generate Certificates for HSS
  * `./check_hss_s6a_certificate /usr/local/etc/oai/freeDiameter hss.openair4G.eur`
* Modify HSS configuration file `/usr/local/etc/oai/hss.conf`
```
HSS :
{
  ## MySQL mandatory options
  MYSQL_server = "127.0.0.1";
  MYSQL_user = "@MYSQL_user@";
  MYSQL_pass = "@MYSQL_pass@";
  MYSQL_db = "oai_db";

  -- HSS options
  OPERATOR_key = "1006020f0a478bf6b699f15c062e42b3"; # OP key for oai_db.sql
  RANDOM = "true";

  -- Freediameter options
  FD_conf = "/usr/local/etc/oai/freeDiameter/hss_fd.conf";
};
```
 * Modify freediameter configuration file `/usr/local/etc/oai/freeDiameter/hss_fd.conf`
```
  -- Identity and realm configured on /etc/hosts
  Identity = "hss.openair4G.eur";

  Realm = "openair4G.eur";

  -- Certificates
  TLS_Cred = "/usr/local/etc/oai/freeDiameter/hss.cert.pem", "/usr/local/etc/oai/
  freeDiameter/hss.key.pem";
  TLS_CA = "/usr/local/etc/oai/freeDiameter/hss.cacert.pem";
```
 * Load example database only in the first run
   * `sudo ./run_hss -i ~/openairinterface5g/SRC/OAI_HSS/db/oai_db.sql`
</details>

<details>
<summary>MME Configuration and Building</summary>

### MME Configuration and Building
The hostname of this machine must be its Identity, in our case "mme". We can change this identifier by modifying "/etc/hosts" and "/etc/hostsname". Then, restart the virtual machine.
Log into the MME virtual machine and follow the instructions below:

* `cd ~/openairinterface5g/SCRIPTS/`
* `sudo mkdir -p /usr/local/etc/oai/freeDiameter`
* `./build_mme -i`
* `sudo cp OPENAIRCN_DIR/ETC/mme.conf /usr/local/etc/oai`
* `sudo cp OPENAIRCN_DIR/ETC/mme_fd.conf /usr/local/etc/oai/freeDiameter/`
* `./check_mme_s6a_certificate /usr/local/etc/oai/freeDiameter/ mme.openair4G.eur`
* `vim /usr/local/etc/oai/mme.conf`
```
NETWORK_INTERFACES :
{
  # MME binded interface for S1-C or S1-MME communication (S1AP), can be ethernet interface, virtual ethernet interface
  MME_INTERFACE_NAME_FOR_S1_MME = "TUN_IFACE";               # YOUR NETWORK CONFIG HERE
  MME_IPV4_ADDRESS_FOR_S1_MME = "IP_ADDR_OF_TUN_IFACE";      # YOUR NETWORK CONFIG HERE

  # MME binded interface for S11 communication (GTPV2-C) 
  MME_INTERFACE_NAME_FOR_S11_MME = "eth2";               # YOUR NETWORK CONFIG HERE
  MME_IPV4_ADDRESS_FOR_S11_MME = "192.168.30.2/24";      # YOUR NETWORK CONFIG HERE
  MME_PORT_FOR_S11_MME = 2123;                           # YOUR NETWORK CONFIG HERE

};

S-GW :
{
  # S-GW binded interface for S11 communication (GTPV2-C), if none selected the ITTI message interface is used
  SGW_IPV4_ADDRESS_FOR_S11 = "192.168.30.3/24";           # YOUR NETWORK CONFIG HERE
};
```
* `vim /usr/local/etc/oai/freeDiameter/mme_fd.conf`
```
  # Identity of MME
  Identity = "mme.openair4G.eur";
  Realm = "openair4G.eur";

  # TLS configuration
  TLS_Cred = "/usr/local/etc/oai/freeDiameter/mme.cert.pem",
  "/usr/local/etc/oai/freeDiameter/mme.key.pem";
  TLS_CA = "/usr/local/etc/oai/freeDiameter/mme.cacert.pem";

  # HSS information
  ConnectPeer= "hss.openair4G.eur" { ConnectTo = "192.168.40.3"; No_SCTP ; No_IPv6;
  Prefer_TCP; No_TLS; port = 3868; realm = "openair4G.eur";};
```
</details>


<details>
<summary>SP-GW Configuration and Building</summary>

### SP-GW Configuration and Building
* `cd ~/openairinterface5g/SCRIPTS/`
* `sudo mkdir -p /usr/local/etc/oai/freeDiameter`
* `./build_spgw -i`
* `sudo vim /etc/hosts`
```
127.0.1.1    sgpw.openair4G.eur spgw
```
* `sudo vim /usr/local/etc/oai/spgw.conf`
```
S-GW :
{
 NETWORK_INTERFACES :
 {
  # S-GW binded interface for S11 communication (GTPV2-C)
  SGW_INTERFACE_NAME_FOR_S11 = "eth2";             # YOUR NETWORK CONFIG HERE
  SGW_IPV4_ADDRESS_FOR_S11 = "192.168.30.3/24";    # YOUR NETWORK CONFIG HERE

  # S-GW binded interface for S1-U communication (GTPV1-U) can be ethernet interface, virtual ethernet
  SGW_INTERFACE_NAME_FOR_S1U_S12_S4_UP = "eth1";
  SGW_IPV4_ADDRESS_FOR_S1U_S12_S4_UP = "192.168.20.10/24";

  SGW_IPV4_PORT_FOR_S1U_S12_S4_UP = 2152; 
  
  # S-GW binded interface for S5 or S8 communication, not implemented, so leave it to none 
  SGW_INTERFACE_NAME_FOR_S5_S8_UP = "none";       # STRING, interface name, DO NOT CHANGE (NOT IMPLEMENTED YET)
  SGW_IPV4_ADDRESS_FOR_S5_S8_UP = "0.0.0.0/24";   # STRING, CIDR, DO NOT CHANGE (NOT IMPLEMENTED YET)
};

P-GW =
{
 NETWORK_INTERFACES :
 {
  # P-GW binded interface for S5 or S8 communication, not implemented, so leave it to none
  PGW_INTERFACE_NAME_FOR_S5_S8 = "none"; # STRING, interface name, DO NOT CHANGE (NOT IMPLEMENTED YET)

  # P-GW binded interface for SGI (egress/ingress internet traffic)
  PGW_INTERFACE_NAME_FOR_SGI = "eth0";  # STRING, YOUR NETWORK CONFIG HERE
  PGW_MASQUERADE_SGI = "yes";           # STRING, {"yes","no"}. YOUR NETWORK CONFIG HERE, will do NAT for you if you put "yes".
  UE_TCP_MSS_CLAMPING = "no";           # STRING, {"yes","no"}.
};

```
</details>

## OAI eNB deployment
Under construction

## OAI UE deployment
Under construction

## Authors
Under construction



