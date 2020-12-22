# Learn Ethical Hacking from Scratch

## Network Hacking
1. Pre-connection attacks
2. Gaining access
3. Post-connection attacks

## Changing MAC address
* Why change it?
	* Increase anoymmity
	* Impersonate other devices
	* Bypass filters
* How to choose a MAC address
```bash
ifconfig <interface> down
ifconfig <interface> hw ether <new MAC address>
ifconfig <interface> up
```

# Pre-connection attacks
## Wirless modes
* To see what mode each wireless adapter has user: iwconfig
* It will also show the wirless mode each adapter is in.
* `Monitor mode` allows you to look at different networks.
### Monitor mode
* Putting wireless adapter in monitor mode:
```bash
ifconfig <interface> down
airmon-ng check kill
iwconfig <interface> mode monitor
ifconfig <interface> up
```
or

```bash
airmon-ng start <interface>
```

## Packet sniffing
### Discover all nearby networks
```bash
# network card has to be in monitor mode
airodump-ng <interface> #discovers networks around
# BSSID, PWR, Beacons, #Data, #/s, CH, MB, ENC, CIPHER, AUTH, ESSID
```
### WIFI Bands
* Decide the frequency range that can be used
* Determine the channels that can be used
	* Most common 2.4GHz and 5GHz

| value | description               |
|-------|---------------------------|
| a     | uses 5GHz freq only       |
| b,g   | both use 2.4GHz freq only |
| n     | uses 5 and 2.4GHz         |
| ac    | Uses freq lower than 6GHz |

```bash
airodump-ng --band <value> <interface> # look on that value
```
### Targeted Packet sniffing (look at devices on that network)
```bash
airodump-ng --bssid <BSSID> --channel <ch> --write <file> <interface> #write to files captured packets
```
* The above could have captured the handshake in `<file>.cap`

## Deauthentication Attack
* Disconnect any clients from any network
	* Works on encrpyted networks(WEP, WPA/2)
	* No need to know the network key (password)
	* No need to be connected to the network

use:
```bash
airplay-ng --deauth <# of deauth packets> -a <network mac/bssid> -c <target Mac> <interface>
# pretty much the hacket sends out packets saying im the access point and im going to disconnect you
```

# Gaining Access

## WEP
* Uses an alogirthm called RC4
* Still used in some networks
* Can be cracked easily
* Client/Access point encrypts data using a key stream
 	* Random Initalization vectors (IV) - 24bits
	* key stream = IV + key (password)
	* Encrypted data = keystream + "data"
	* Packets include the encrypted data with the IV
* Client/Access point decrypts data

### WEP cracking
1. Capture a large # of packets (IVS)
	* Use `airodump-ng`
2. Analyze the captured IV's and crack the key
 	* Use `aircrack-ng`

**What if the AP isnt busy?**
 * We need to force the AP to generate new IV's
 * APs only communicate with connected clients (we arent connected yet)
 	* We can associate with the AP, and still force the AP to generate IV's

#### How to associate with the AP
```bash
airplay-ng --fakeauth 0 -a <bssid>  -h <adapter MAC> <int>
```
#### How to force AP to generate IV's
* ARP Request Replay attack
	* wait for an arp packet
	* capture it, and replay it (retransmit it)
	* This causes the AP to produce new packets with a new IV
	* Keep doing this still we have enough IV's to crack the key
```bash
airplay-ng --arpreplay -b <bssid> -h <MAC of adapter> <interface>
aircrack-ng <file>.cap
```
## WPA/WPA2
* Much more secure than WEP
* Each packet is encrypted using a unique temporary key
* Packets contain no useful information

### WPS (hacking WPA/2 with out wordlist)
* Is a feature of WPA/2
* Allows clients to connect with out the password
* Authentication is done using an 8 digit pin
	* This pin can be used to compute the actual password
	* Doesnt work if push button auth is enabled on the router for WPS

```bash
wash --interface <interface> # see all networks using WPS
airplay-ng --fakeauth 30 -a <bssid> -h <adapters mac> <interface> # associate
reaver --bssid <bssid> --channel <ch> --interface <int> --vvv --no-associate # crack using wps
```

### Capturing WPA/2 handshake (hacking with wordlist)
* Packets of WPA encryption contain no useful information
* Packts from the handshake only contain useful information
* Handshake information:
	* SP address
	* SJA address
	* AP nonce
	* SJA nonce
	* EAPOL
	* Payload
	* MIC
* Use the handshake information excluding the MIC to try a plaintext password and compare to MIC
* To force a new handshake use:
	* `aireplay-ng --deauth 4 -a <bssid> -c <target client> <interface>`
* Capture this handshake using `airodump-ng` save get cap file:
	* aircrack-ng <file>.cap -w [wordlist]


#### Creating a wordlist
```bash
crunch [min] [max] [chars] -t [pattern] -o [filename]
```


# Post-connection attacks
* works against wifi and ethernet
* Gather more info
* intercept data (usernames, passwords, ect)
* Modify data on the fly

## Information gathering
1. `netdiscover`: netdiscover -r <target/range IP>
2. zenmap/nmap:
   * Open ports
   * Runnning services
   * operating systems
   * conneted clients
   * more

## MITM attacks
* You act as the router by changing your mac address on the target machines arp table
	* ARP spoofing:
		* arpsoof -i [interface] -t [clientIP] [gatewayIP]
		* arpsoof -i [interface] -t [gatewayIP] [clientIP]
	* Need to enable port forwarding so requests can flow through you
		*  `echo 1 > /proc/sys/net/ipv4/ip_forward`

### Bettercap
* Framework to run network attacks
* Can be used to:
	* Arp spoof targets (redirect the flow of packets)
	* sniff data (passwords, usernames)
	* bypass Https
	* redirect domains requests (DNS spoofing)
	* inject code in loaded pages
	* and more
* usage:
	* `bettercap -iface <interface>`

#### Bettercap shell
* help : shows all modules
* help [module] : shows all paramters of that module
* <module> on: turn that module on
* set <parm> <value> : set parm in module to that value
* Modules:
	* net.probe on : discover devices on network
	* net.show : show Ip,mac,nam,vendor,sent,recv,seen
	* arp.spoof:
		* set arp.spoof.fullduplex true
		* set arp.spoof.targets [Ips]
		* arp.spoof on
		* net.sniff on : capture data from arpsoofing
#### Using bettercap script
`bettercap -iface <interface> -caplet "file.cap"`

### HTTPS
* Data in HTTP is sent as plain text (can be easily read)
* Uses TLS or SSL to encrypt the data

#### Bypassing HTTPS
* Most websites use this
* Downgrade HTTPS to HTTP
* In bettercap:
	* caplet.show # show name of each caplet and size
	* <capletName> # to run it
	* hstshijack/hstshijack # to downgrade https to http
	* Some websites use HSTS:
		* In a browser contains a list of known websites and will not recieve it unless its https
### DNS spoofing
* The user requests google.com and you can respond with any IP address
* In bettercap: dns.spoof

## injecting javascript code
* inject js code in loaded pages
* Code gets executed by the targeted browser
	* Replace links
	* Replace images
	* Insert html elements
	* Hook target browser to exploitation frameworks


## Create a fake access point
* Use mana-toolkit
* It can :
	* automatically configure and create fake AP's
	* automatically sniff data
	* Automtically bypass https

# Gaining access - server side attacks
* neeed an IP address
* Very simple if its on same network use zenmap
* If target has a domain, then simple ping will get its IP
* methods:
	* Try default password
	* some services might contain a backdooor
	* code exec vulneralbilties: buffer overflow

## Metasploit
* its an sploit development and exec tool it can be used to carry out other pen testing tasks as port scans, service id's and post exploitation tasks
* `msfconsole`
* `show [something] `: something can be exploits payloads auxilaries or options
* `use [comething]`
* `set [option] [value]` : config option to have a value
* `exploit` - runs current tasks



# Gaining access - client side attacks
* Use `Veil`
* It can product non detected viruses
* To test if your virus is detectable use [website](https://nodistribute.com/)
* use `evasion`
* show : show payloads
* options: show options or params
* set [options] [value]
* generate: generate a backdoor virus

* use msfconsole to listen to backdoor
* set PAYLOAD to same as veils'
* set other options



# windows password
Passw0rd!
# kali
user: root
pass: toor
# metasploit
user: msfadmin
pass: msfadmin
