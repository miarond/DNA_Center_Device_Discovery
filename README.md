# DNA Center Device Onboarding Process

The DNA Center platform includes the capability to automatically discover network devices and onboard them into its managed inventory.  This process takes advantage of the Plug and Play Agent software, which has been incorported into Cisco's IOS-XE software for many years.  If all of the proper steps are taken to prepare your network environment, it is possible to connect a router or switch (in a factory default "out-of-the-box" state) to your production network and have it completely onboarded into DNA Center's inventory without any manual intervention on the device itself.

Throughout this guide we will break down the process and explain each step in simple terms.  The Plug and Play Agent software will follow a pre-configured workflow in its attempt to discover and connect to the Plug and Play server, so there are several decisions that will be made as it progresses.

  > *Disclaimer: The processes and configurations in this document are dependent up on platform type, model number, and IOS-XE code version.  We have tested these configurations on Catalyst 9300 switches running IOS-XE 17.x code.  Support for these features are subject to change.*

## Table of Contents

* [Initial Device Bootup](#initial-device-bootup)
  * [Workflow Overview](#workflow-overview)
  * [Initial Discovery Communication](#initial-discovery-communication)
* [Device Bootstrap Config](#device-bootstrap-config)
* [Changing PnP Startup VLAN](#changing-pnp-startup-vlan)
* [DHCP Discovery of PnP Server](#dhcp-discovery-of-pnp-server)
* [DNS Discovery of PnP Server](#dns-discovery-of-pnp-server)
* [Plug and Play Connect Cloud Portal Discovery](#plug-and-play-connect-cloud-portal-discovery)
  * [Preparing to Use Plug and Play Portal](#preparing-to-use-plug-and-play-portal)
  * [Device Phone-Home Procedure](#device-phone-home-procedure)
* [PnP Connecting to DNA Center](#pnp-connecting-to-dna-center)

---

## Initial Device Bootup

### Workflow Overview

![Initial_Bootup_Workflow.png](/assets/Initial_Bootup_Workflow.png)

[Return to ToC](#table-of-contents)

---

### Initial Discovery Communication

During the initial stage of discovering the Plug and Play server (Catalyst Center), the Plug and Play Agent software in IOS-XE must establish an HTTP connection with the server.  Many customers are concerned that this initial connection uses the unencrypted HTTP protocol over TCP port 80, as this creates a potential security threat.  However, it is important to understand the steps necessary to establish a trusted, secure and encrypted HTTPS connection between the Plug and Play Agent software and Catalyst Center - this can't happen without first establishing a mutual trust between client and server, using SSL certificates.

Cisco network devices (primarily routers and switches) have hardware certificates issued and installed when they are built.  Those certificates are issued by Cisco's trusted Certificate Authority, which is also part of the trust chain installed on other Cisco platforms like Catalyst Center.  However, Catalyst Center, by default, will create its own self-signed SSL certificate during initial setup...and this certificate is not trusted by *anyone*.  You can install your own Enterprise SSL certificate after initial setup however, that certificate is also unlikely to be trusted by anyone outside your organization.  So, if the client (network device) and the server (Catalyst Center) can not establish a mutual chain of trust for each other's SSL certificates, they can not successfully build a secure HTTPS connection.

For that reason, the initial connection between client and server must be over the unencrypted HTTP protocol so that the Plug and Play Agent can download and install the Catalyst Center server's SSL certificate.  Once that certificate has been installed as a "trustpoint" in IOS-XE, the Plug and Play Agent will reconnect to Catalyst Center using the HTTPS protocol on TCP port 443, and all further communications will be encrypted.

Here is a diagram of this communication flow:

![PnP_Server_Authentication_Process.png](/assets/PnP_Server_Authentication_Process.png)

[Return to ToC](#table-of-contents)

---

## Device Bootstrap Config

Some IOS-XE devices (including ISR4K routers, Catalyst 8K routers, Catalyst 3850 and 9K series switches) support the USB Autoinstall feature, which will allow the device to be "bootstrapped" with a configuration file that is stored on a USB Flash Drive connected to the device.  **This option would be useful if you are unable to configure DHCP on your management network, to allow new devices to obtain management IP addresses dynamically.**  

Interrupting the PnP Agent on a device as it boots up, by breaking into Enable Mode from the serial console, will cause the PnP Agent process to stop.  If any Startup configuration is saved to the device's NVRAM, the PnP Agent process will abort and will no longer automatically discover a PnP Server.  To work around this scenario while still allowing the configuration of a static IP address and default route or gateway, we can leverage the USB Autoinstall feature and a "bootstrap" configuration file.

The USB Flash Drive should be formatted using the FAT filesystem and a plain text file using one of the following filenames should be saved at the root directory of the Flash Drive:

* `router-confg`
* `ciscortr.cfg`

This text file should be formatted exactly as any device configuration file would look.  At a minimum, the `router-confg` or `ciscortr.cfg` file should contain these CLI commands (edit according to your configuration needs):

```
! If DNS is needed
ip name-server <dns_server_ip>
!
interface Vlan1
 ip address <static_ip> <subnet_mask>
 no shutdown
!
! If Layer 2
ip default-gateway <gateway_ip>
! If Layer 3
ip routing
ip route 0.0.0.0 0.0.0.0 <next_hop_ip>
!
pnp profile dnac
 transport http <host | ipv4 | ipv6> <dnac_hostname_or_ip> port <tcp_port>
!
end
```

  > *Note: Bootstrap configuration ONLY supports the use of VLAN 1 or a routed uplink interface.  It is not possible to create a new VLAN from the bootstrap configuration file.*

  > *Note: Bootstrap configuration and `pnp startup-vlan <id>` configurations are ***mutually exclusive***.  You can either use the bootstrap method on VLAN 1, or use the PnP Startup VLAN configuration on the upstream device, but NOT both.*

When the device boots up you will eventually see entries similar to this in the log, and on the Serial Console:

```
*Jun 22 17:06:21.978: %SYS-5-CONFIG_I: Configured from usbflash0:ciscortr.cfg by console
*Jun 22 17:06:21.978: : File bootup successful. Autoinstall will not start now.
```

[USB Autoinstall Documentation](https://www.cisco.com/c/dam/en/us/td/docs/routers/access/800/hardware/deployment/guide/deployguide800.pdf) </br>
[Cisco Blogs Documentation](https://blogs.cisco.com/developer/dna-center-pnp-day-0)

[Return to ToC](#table-of-contents)

---

## Changing PnP Startup VLAN

By default, the PnP agent on a switch running IOS-XE will use VLAN 1 as the management VLAN and it will attempt to obtain an IP address via DHCP.  Generally, it is best practice *not* to use VLAN 1 in a production network, so at some point you may want to shift management to another VLAN on the switch.  This can be done as part of the Day-0 Onboarding template however, it's much easier to incorporate this change into the PnP process.  To do this, you can apply a single command to the upstream device which the new, factory default network switch will be connected to:

`pnp startup-vlan <vlan_id>`

The upstream network device uses Cisco Discovery Protocol (CDP) to communicate this new PnP Startup VLAN to the downstream switch.  The downstream switch then disables the DHCP Client on VLAN 1 and *enables* it on the new PnP Startup VLAN.  Finally, the dynamic trunk link that is established between the downstream and upstream devices changes the allowed VLAN to match the new PnP Startup VLAN:

![PnP_Startup_VLAN.png](/assets/PnP_Startup_VLAN.png)

The CDP protocol transmits this new VLAN ID to downstream devices in a special payload section, named `Web mgmt port`.  Inside this section, a field named `Platform` contains the VLAN ID specified in the `pnp startup-vlan <vlan_id>` command:

![cdp_pnp_pcap.png](/assets/cdp_pnp_pcap.png)

[Cisco Blogs Documentation](https://blogs.cisco.com/developer/dna-center-pnp-day-0) </br>
[Cisco Live Presentation](https://www.ciscolive.com/c/dam/r/ciscolive/emea/docs/2019/pdf/LTRNMS-2007.pdf)

[Return to ToC](#table-of-contents)

---

## DHCP Discovery of PnP Server

The first method that the PnP Agent in IOS-XE uses to discover its PnP Controller server is DHCP.  The factory default network device will establish communication with an upstream network device, then attempt to obtain a management IP address from DHCP by going through the normal DHCP Discover/Offer/Request/Ack process.

The PnP Agent will include Option 60 in its DHCP Discover message, containing the plain text value `ciscopnp`.  If the receiving DHCP server is properly configured, it should respond with a DHCP Offer message that includes Option 43.  Option 43 is an ASCII string value that is semicolon delimited, and contains encoded values which give the network device information about how to reach the PnP Server.  For example:

```
option 43 ascii 5A1N;B2;K4;I172.19.45.222;J80;
```

Which translates to:

* `5A1N` 
  * `5`: DHCP suboption indicating Plug and Play
  * `A`: Active operation (PnP Agent initiates communication with PnP server).  For Passive operation, use `P` instead.
  * `1`: Version of the template that the PnP Agent should use
  * `N`: Disable debugging information.  To enable debugging information, use `D` instead.
* `B2`
  * PnP Server Address Type:
    * `1`: Hostname
    * `2`: IPv4 Address
    * `3`: IPv6 Address
* `K4`
  * Transport protocol to be used:
    * `4`: HTTP
    * `5`: HTTPS
    * *Note: HTTP is the default option.  While the RFC for DHCP Option 43 specifies option numbers for the XMPP protocol, the HTTP protocol options should be used instead.*
* `I172.19.45.222`
  * Capital letter "I" followed by IP Address or DNS Hostname of PnP server.
* `J80`
  * Capital Letter "J" followed by the TCP Port number used by the PnP server.

> *Note: There are additional optional values that can be added to this string.  A `T` option followed by a URL can be used to specify where to download the Trust Pool certificate bundle (**mandatory** if HTTPS protocol - option `K5` - is used).  Also, a `Z` option followed by the IP Address of an NTP server can be included (**mandatory** when using option `T` for trustpool security, to ensure device time is sychronized).*

![PnP_DHCP_Process.png](/assets/PnP_DHCP_Process.png)

[DNA Center Documentation](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/dna-center/2-3-3/user_guide/b_cisco_dna_center_ug_2_3_3/m_onboard-and-provision-devices-with-plug-and-play.html#id_90877)

[Return to ToC](#table-of-contents)

---

## DNS Discovery of PnP Server

In the event that DHCP was successful and the device has obtain a management IP, but Option 43 was not present in the DHCP Offer message, the PnP Agent software falls back to using DNS to attempt discovery of the PnP server IP Address.

  > It is important to note that unless the USB Bootstrap method is used, DHCP is ***required*** for the PnP Agent to continue functioning.  If the network device can not obtain an IP Address for communicating with the PnP server, the process will halt.

DNS Discovery relies upon DHCP to provide the network device with an IP Address, DNS Server(s) ***and*** the DNS Domain Name of the client's network.  The PnP Agent will construct a Fully-Qualified Domain Name (FQDN) record using the DNS Domain Name supplied by DHCP, and it will use the DNS server(s) provided by DHCP to attempt to resolve this A Record.  For example:

DHCP Pool Config
```
<omitted>
domain-name customer.com
dns-server 172.16.200.200
```

The network device will construct the FQDN `pnpserver.customer.com` based on this information, and it will send a DNS Query to DNS server `172.16.200.200` attempting to resolve the IP Address of this FQDN.  Additionally, the network device will attempt to resolve the IP Address of an NTP server using the FQDN `pnpntpserver.customer.com`.

  > :warning: If this method is used to discover DNA Center, the FQDN `pnpserver.<domain>.<tld>` **MUST** be included in DNA Center's SSL certficate, in either the "CN" (Common Name) field or the "SAN" (Subject Alternative Name) field.  When the PnP Agent in IOS-XE attempts to contact DNA Center, it will do so using the HTTP protocol toward this FQDN.  The initial HTTP connection will be established to download the SSL certificate from DNA Center, and then an HTTPS connection will be attempted.  **It is at THIS point** where the connection attempt will fail if the `pnpserver.<domain>.<tld>` FQDN is *not* included in DNA Center's SSL certificate.  The identity of DNA Center can not be confirmed by the PnP Agent, and the HTTPS connection will be terminated as a result.

![PnP_DNS_Process.png](/assets/PnP_DNS_Process.png)

[DNA Center Documentation](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/dna-center/2-3-3/user_guide/b_cisco_dna_center_ug_2_3_3/m_onboard-and-provision-devices-with-plug-and-play.html#id_90879)

[Return to ToC](#table-of-contents)

---

## Plug and Play Connect Cloud Portal Discovery

As a final fallback measure for the PnP Agent, Cisco provides a cloud-hosted "Plug and Play Connect" portal and a phone-home FQDN that devices can use to discover their PnP Controller server.  There is quite a bit of initial setup required for this process to work, and it begins at the time the device is ordered and purchased from Cisco.  The portal can be accessed by visiting https://software.cisco.com and clicking on the "Manage Devices" link underneath the **Network Plug and Play** section - or you can click on the link below:

[Plug and Play Connect Portal](https://software.cisco.com/software/csws/ws/platform/login?route=module/pnp)

### Preparing to Use Plug and Play Portal

The following preparatory steps are necessary to utilize the Plug and Play Portal with your network devices:

1. You must first create a **Smart Licensing Account**, or "Smart Account", for your organization.  This can be done at https://software.cisco.com
2. At the time of order/purchase of network devices, your Cisco Partner will need to link your Smart Account to the Purchase Order.
3. A **Controller Profile** must be created in the Plug and Play Portal, which defines the IP Address or Hostname of the PnP Server (in this case, DNA Center) and, optionally, the SSL Certificate of the PnP Server.
4. The Controller Profile should be assigned as the "Default Profile" for a specific **Virtual Account** (a sub-division of the overall Smart Account), so that devices added to that Virtual Account are redirected to the appropriate Controller.
5. To automatically load the purchased devices into the Plug and Play Portal, an additional feature license called `NETWORK-PNP-LIC` must be added to all devices that support it. 
    1. *At the time of this writing, the `NETWORK-PNP-LIC` option is a free, $0.00 licensed feature. This can be subject to change in the future.*
    2. Alternatively, devices can be manually added to the Plug and Play Portal ***however***, this may require physical access to the device in order to obtain the SUDI Identifier from the Serial Console interface.
6. Once devices are added to the Plug and Play Portal, they should be moved to the appropriate **Virtual Account** within your organization's Smart Account.

The following documentation explains the procedure in detail (note that while the document is outdated, the information is still mostly accurate): [Plug and Play Connect Documentation](https://www.cisco.com/c/dam/en_us/services/downloads/plug-play-connect.pdf)

### Device Phone-Home Procedure

In the event that DNS Discovery of the PnP Controller server was not successful, the PnP Agent will attempt to contact the Plug and Play Connect Cloud Portal, provided by Cisco.  It will follow this procedure:

1. Obtain a management IP Address and DNS Server via DHCP.
2. Attempt to download the Cisco Trustpool Bundle, which contains Certificate Authority (CA) certificates used to establish trust with Cisco devices via HTTPS.
3. Attempt to synchronize device clock with the following public NTP servers: `time-pnp.cisco.com` or `pool.ntp.org`
4. Attempt a connection with the Plug and Play Portal server at `devicehelper.cisco.com`
5. The Plug and Play Portal queries the global PnP device database for a match of the device's Serial Number and its SUDI Identifier.
6. If a match is found (an associated customer Smart Account), the service looks for a suitable Controller Profile in the customer's Smart Account.
7. If a Controller Profile is found, the controller's HTTP or HTTPS URL is returned, along with the TCP port number and, if HTTPS is used, the Controller's SSL Certificate.
8. The Plug and Play Portal server returns the Controller details to the network device.
9. The network device attempts to contact the local PnP Controller server.

![PnP_Portal_Process.png](/assets/PnP_Portal_Process.png)

[DNA Center Documentation](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/dna-center/2-3-3/user_guide/b_cisco_dna_center_ug_2_3_3/m_onboard-and-provision-devices-with-plug-and-play.html#id_90888)

[Return to ToC](#table-of-contents)

---

## PnP Connecting to DNA Center

Here is what you can expect to see on the Serial Console CLI, including when DHCP is used for Plug and Play:

```
Autoinstall trying DHCPv6 on Vlan1

Autoinstall trying DHCPv4 on Vlan1

Acquired IPv4 address 172.16.2.25 on Interface Vlan1
Received following DHCPv4 options:
        vendor          : 5A1D;B2;K4;I172.16.202.101;J80;

OK to enter CLI now...

pnp-discovery can be monitored without entering enable mode

Entering enable mode will stop pnp-discovery

Press RETURN to get started!

*Jul 20 19:16:02.933: %PKI-6-TRUSTPOINT_CREATE: Trustpoint: TP-self-signed-745695 created succesfully
*Jul 20 19:16:02.994: %IOXN_APP-6-PRE_INIT_DAY0_GS_INFO: Day0 Guestshell de-initilization API is being invoked

*Jul 20 19:16:03.253: %PNP-6-HTTP_CONNECTED: PnP Discovery connected to PnP server (profile=pnp-zero-touch, ip=172.16.202.101, port=80)
*Jul 20 19:16:03.306: %CRYPTO_ENGINE-5-KEY_ADDITION: A key named TP-self-signed-745695 has been generated or imported by crypto-engine
*Jul 20 19:16:03.307: %SSH-5-ENABLED: SSH 1.99 has been enabled
*Jul 20 19:16:03.351: %PKI-4-NOCONFIGAUTOSAVE: Configuration was modified.  Issue "write memory" to save new IOS PKI configuration
*Jul 20 19:16:03.353: %SYS-5-CONFIG_P: Configured programmatically by process PnP Agent Discovery from console as vty0Guestshell destroyed successfully 

*Jul 20 19:16:04.538: %CRYPTO_ENGINE-5-KEY_ADDITION: A key named TP-self-signed-745695.server has been generated or imported by crypto-engine
*Jul 20 19:16:05.360: %SYS-5-CONFIG_P: Configured programmatically by process PnP Agent Discovery from console as vty0
*Jul 20 19:16:05.361: %PNP-6-PNP_SAVING_TECH_SUMMARY: Saving PnP tech summary (/pnp-tech/pnp-tech-discovery-summary)... Please wait. Do not interrupt.
*Jul 20 19:16:05.526: %UICFGEXP-6-SERVER_NOTIFIED_STOP: Switch 1 R0/0: psd: Server iox has been notified to stop
*Jul 20 19:16:06.521: %PNP-6-PNP_TECH_SUMMARY_SAVED_OK: PnP tech summary (/pnp-tech/pnp-tech-discovery-summary) saved successfully (elapsed time: 1 seconds).
*Jul 20 19:16:06.521: %PNP-6-PNP_DISCOVERY_DONE: PnP Discovery done successfully (PnP-DHCP-IPv4)
Jul 20 19:16:14.004: %SYS-5-CONFIG_P: Configured programmatically by process XEP_pnp-zero-touch from console as vty0
Jul 20 19:16:14.040: %PKI-6-TRUSTPOINT_CREATE: Trustpoint: pnplabel created succesfully
Jul 20 19:16:14.044: %PKI-4-NOCONFIGAUTOSAVE: Configuration was modified.  Issue "write memory" to save new IOS PKI configuration
Jul 20 19:16:14.045: %PNP-6-PNP_TRUSTPOINT_INSTALLED: Trustpoint (pnplabel) installed from (/pnp-info/pnp-xsvc-cert) by (pid=590, pname=XEP_pnp-zero-touch, time=19:16:14 UTC Thu Jul 20 2023)
Jul 20 19:16:42.446: %IOXN_APP-6-PRE_INIT_DAY0_GS_INFO: Day0 Guestshell de-initilization API is being invoked

Jul 20 19:16:42.446: AUTOINSTALL: script execution not successful for Vl1.
Jul 20 19:17:32.516: %PNP-6-PNP_BACKOFF_NOW: PnP Backoff now for (59) seconds requested (1/3) by (profile=pnp-zero-touch, ip=172.16.202.101, port=443)
Switch>
Switch>
Jul 20 19:25:41.732: %PNPA-DHCP Op-43 Msg: Op43 has 5A. It is for PnP
Jul 20 19:25:41.732: %PNPA-DHCP Op-43 Msg: After stripping extra characters in front of 5A, if any: 5A1D;B2;K4;I172.16.202.101;J80; op43_len: 31

Jul 20 19:25:41.732: %PNPA-DHCP Op-43 Msg: _pdoon.2.ina=[Vlan1]
Jul 20 19:25:41.732: %PNPA-DHCP Op-43 Msg: _papdo.2.eRr.ena
Jul 20 19:25:41.732: %PNPA-DHCP Op-43 Msg: _pdoon.2.eRr.pdo=-1
Jul 20 19:25:49.769: %PNP-6-PNP_CONFIG_ARCHIVE: Config (flash:pnp-archive-Jul-20-19-25-45.450-0) archive (1/3) by (pid=482, pname=XEP_pnp-zero-touch, time=19:25:49 UTC Thu Jul 20 2023)
000207: Jul 20 19:25:50.951: %CRYPTO_ENGINE-5-KEY_ADDITION: A key named dnac-sda has been generated or imported by crypto-engine

000215: Jul 20 19:26:08.457: %SEC_LOGIN-5-LOGIN_SUCCESS: Login Success [user: dnac] [Source: 172.16.202.101] [localport: 22] at 19:26:08 UTC Thu Jul 20 2023
000216: Jul 20 19:26:08.974: %SEC_LOGIN-5-LOGIN_SUCCESS: Login Success [user: dnac] [Source: 172.16.202.101] [localport: 22] at 19:26:08 UTC Thu Jul 20 2023
000217: Jul 20 19:26:09.074: %SEC_LOGIN-5-LOGIN_SUCCESS: Login Success [user: dnac] [Source: 172.16.202.101] [localport: 22] at 19:26:09 UTC Thu Jul 20 2023
000218: Jul 20 19:26:09.296: %SYS-6-LOGOUT: User dnac has exited tty session 3(172.16.202.101)
000230: Jul 20 19:26:53.791: %SYS-6-LOGGINGHOST_STARTSTOP: Logging to host 172.16.202.101 port 0 CLI Request Triggered
000231: Jul 20 19:26:53.792: %SYS-6-LOGGINGHOST_STARTSTOP: Logging to host 172.16.202.101 port 514 started - CLI initiated
000232: Jul 20 19:26:53.867: %SYS-5-CONFIG_I: Configured from console by dnac on vty6 (172.16.202.101)
000234: Jul 20 19:26:58.885: %SYS-6-LOGGINGHOST_STARTSTOP: Logging to host 172.16.202.101 port 514 started - reconnection
000239: Jul 20 19:27:13.658: %PKI-6-TRUSTPOINT_CREATE: Trustpoint: DNAC-CA created succesfully
```

[Return to ToC](#table-of-contents)

---