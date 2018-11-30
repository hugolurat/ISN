# Ports and services scanning - Methodology

### Nmap

The great Nmap ("Network Mapper") tool is the most popular, versatile, and
robust port scanners to date.  
It has been actively developed for over a decade, and has numerous features
beyond port scanning.

Nmap uses raw IP packets to determine what hosts are available on the network,
what services (application name and version) those hosts are offering, what
operating systems (and OS versions) they are running, what type of packet
filters/firewalls are in use, and dozens of other characteristics.

In addition to the classic command-line Nmap executable, the Nmap suite
includes an advanced GUI and results viewer (Zenmap), a flexible data transfer,
redirection, and debugging tool (Ncat), a utility for comparing scan results
(Ndiff), and a packet generation and response analysis tool (Nping).

###### Usage

```
nmap [Scan Type(s)] [Options] (<IP> | <FQDN> | <CIDR> | <RANGE>)
```

###### Single host scanning

To scan a single host:

```
# TCP - all ports
nmap -v -sS -Pn -A -p- <IP/FQDN>
nmap -v -sT -Pn -A -p- <IP/FQDN>
nmap -v -sS -Pn -A -oA nmap_<FILENAME> -p- <IP/FQDN>

# UDP - Top 1000
nmap -v -sU -Pn -A <IP/FQDN>
nmap -v -sU -Pn -A -oA nmap_<FILENAME> <IP/FQDN>

# Script engine
# For more information about the nmap scripts to use for a given service refer to the service note (L7/<SERVICE>)
nmap -v -sT -Pn -p <SERVICE_PORT> --script=vuln <IP/FQDN>
```

###### Network scanning

*Host Discovery*

Generate a live hosts list trough a nmap ping sweep (ARP ping if on same subnet,
ICMP echo and TCP packets on ports 80 and 443 otherwise)

```
nmap -sn -T4 -oG Discovery.gnmap <RANGE/CIDR>
grep "Status: Up" Discovery.gnmap | cut -f 2 -d ' ' > LiveHosts.txt
```

*Port Discovery*

```
# Most Common Ports
nmap -sS -T4 -Pn -A -oG TopTCP -iL LiveHosts.txt
nmap -sU -T4 -Pn -A -oN TopUDP -iL LiveHosts.txt

# Full Port Scans (UDP is very slow)
nmap -sS -T4 -Pn -A -p- -oN FullTCP -iL LiveHosts.txt
nmap -sU -T4 -Pn -A -p- -oN FullUDP -iL LiveHosts.txt
```

*Print results*

```
grep "open" FullTCP | cut -f 1 -d ' ' | sort -nu | cut -f 1 -d '/' |xargs | sed 's/ /,/g'|awk '{print "T:"$0}'
grep "open" FullUDP | cut -f 1 -d ' ' | sort -nu | cut -f 1 -d '/' |xargs | sed 's/ /,/g'|awk '{print "U:"$0}'
```

*Specific service vulnerabilites*

```
nmap -v -sT -Pn -p <SERVICE_PORT> -oA <FILEOUT> --script=vuln <RANGE/CIDR>
nmap -v -sT -Pn -p <SERVICE_PORT> -oA <FILEOUT> --script=vuln -iL LiveHosts.txt
```

###### Scan Types

-sS : TCP SYN scan

A SYN packet is sent.
In response, a SYN/ACK indicates the port is listening (open), while a RST
(reset) is indicative of a non-listener. 
No response or an ICMP unreachable error means the port is filtered.

-sT : TCP connect scan

Does not require admin privilege.
Instead of writing raw packets as most other scan types do, Nmap asks the
underlying operating system to establish a connection with the target machine.
Works the same way as the TCP SYN scan, only closing the TCP handsake.

-sU : UDP scan

Can be combined with a TCP scan.
A UDP packet is sent.
Open and filtered ports rarely send any response.
If an ICMP port unreachable error (type 3, code 3) is returned, the port is 
closed.

-sN, -sF, -sX : TCP NULL, FIN, and Xmas scans.

Null scan : Does not set any bits (TCP flag header is 0).  
FIN scan : Sets just the TCP FIN bit.  
Xmas scan : Sets the FIN, PSH, and URG flags, lighting the packet up like a
Christmas tree.  
If the system scanned is RFC complient, a RST packet will be received if the
port is closed and no response at all if the port is open.  
The port is marked filtered if an ICMP unreachable error (type 3, code 0, 1, 2,
 3, 9, 10, or 13) is received.  
The key advantage to these scan types is that they can sneak through certain
non-stateful firewalls and packet filtering routers.

-sI <zombie host> : idle scan

Reference : https://nmap.org/book/idlescan.html

###### Target Specification

Nmap supports multiple way to specify a target host :

- IP address or hostname : 192.168.15.15 / www.google.com
- IP range : 192.168.0.*
- CIDR-style : 192.168.0.0/24
- Input file : -iL <inputfilename>

######  Usefull Options

```
-p <port ranges> : scan specified ports

  Individual port numbers, comma separated list of ports or hyphen separated
  range can be used.
  When scanning a combination of protocols, a particular protocol can be
  specified by preceding the port numbers by T: for TCP, U: for UDP.
  Ex: -p U:53,111,137, T:21-25,80,139,443,8080

-sn : No port scan (ping scan)

  Tells Nmap not to do a port scan after host discovery, and only print out the
  available hosts that responded to the host discovery probes.

-Pn : No ping

  By default, Nmap use a ping scan to determine if the host is up before starting
  the specified scan.
  Tells Nmap to skip the ping scan and directly start the specified scan.

-n : No DNS resolution

  Tells Nmap to never do reverse DNS resolution on the active IP addresses it
  finds. Can slash scanning times.

-PR : ARP Ping

  Use ARP request to conduct host discovery on LA-T4N network.

-sV : Version detection

 	Tells Nmap to try to determine the service protocol, the application name,
  the version number, hostname, device type and OS family of the target.

-O : Enable OS detection)

  Tells Nmap to try to determine the OS and OS details of the target.

-sC : script scanning

  Performs a script scan using the default set of scripts

-A : Aggressive scan options

  Tells Nmap to perform OS detection (-O), version scanning (-sV), script
  scanning (-sC) and traceroute (--traceroute).

-T paranoid/0 | sneaky/1 | polite/2 | normal/3 | aggressive/4 |
insane/5 : timing template
 		
  Paranoid and Sneaky are for IDS evasion and are incredibly slow.  
  Polite mode slows down the scan to use less bandwidth and target machine
  resources. A Polite scan may be 10 times slower than a normal scan.  
  Normal mode is the default.  
  Aggressive mode speeds scans up by making the assumption that you are on a
  reasonably fast and reliable network.  
  Insane mode assumes that you are on an extraordinarily fast network or are
  willing to sacrifice some accuracy for speed.  
```

###### Nmap Scripting Engine (NSE)

NSE scripts define a list of categories they belong to.
Currently defined categories are auth, broadcast, brute, default. discovery,
dos, exploit, external, fuzzer, intrusive, malware, safe, version, and vuln.

NSE Options:

```
--script <filename>|<category>|<directory>|<expression>[,...]
```

Runs a script scan using the comma-separated list of filenames, script
categories, and directories.

```
nmap --script "http-\*"
```

Loads all scripts whose name starts with http-, such as http-auth and 
http-open-proxy. The argument to --script had to be in quotes to protect the
wildcard from the shell.

```
nmap --script "not intrusive"
```

Loads every script except for those in the intrusive category.

```
nmap --script "default or safe"
```

This is functionally equivalent to nmap --script "default,safe". It loads all
scripts that are in the default category or the safe category or both.

```
--script-args <n1=<v1>,<n2>={<n3>=<v3>},<n4>={<v4>,<v5>}
```

Lets you provide arguments to NSE scripts.
Arguments are a comma-separated list of name=value pairs.
Names and values may be strings not containing whitespace or the characters
‘{’, ‘}’, ‘=’, or ‘,’. To include one of these characters in a string, enclose
the string in single or double quotes.
The online NSE Documentation Portal at https://nmap.org/nsedoc/ lists the
arguments that each script accepts, including any library arguments that may
influence the script.

```
--script-args-file <filename>
```

Lets you load arguments to NSE scripts from a file.
Any arguments on the command line supersede ones in the file. 

```
--script-help <filename>|<category>|<directory>|<expression>|all[,...]
```

Shows help about scripts.
Can be used to list all scripts in a given category/directory/exprossion. 

```
--script-updatedb
```

This option updates the script database found in scripts/script.db 

###### Proxies

Use Proxychains to scan through a proxy.  
Supported proxies types : http, socks4 and socks5.

HTTP/socks4 can only be used to conduct TCP scan:

  - ICMP ping can not be performed. Use -Pn.
  - Never perform DNS resolution to prevent DNS leaks. Use -n.
  - RAW/PACKET or UDP sockets cannot be redirected through these kind of proxies
    as they are designed to relay full TCP connections only.   
  - OS fingerprinting based on features of the IP stack is not possible.
  - TCP connect scan (-sT) and service fingerprint on TCP (-sV) can be proxyfied.

To use nmap through Proxychains:

```
Edit the /etc/proxychains.conf to use the proxy  
<http | socks4 | socks5> <IP> <PORT>

# Start the scan using Proxychains
proxychains nmap -v -n -Pn -sT -A ...
```