# Use TLS to secure request of Bind9 Forwardes 

## Motivation

* [DNSSEC](https://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions)
* [DNS over HTTPS](https://en.wikipedia.org/wiki/DNS_over_HTTPS)
* [DNS over TLS](https://en.wikipedia.org/wiki/DNS_over_TLS)

## Requirements
* Bind DNS in your private/local Network (version >= 9.11)
* [stunnel4](https://www.stunnel.org/index.html)

### stunnel

#### 1. Install

    apt-get install stunnel4

#### 2. Configuration

`/etc/stunnel/stunnel.conf`

    ; Sample stunnel configuration file for Unix by Michal Trojnara 1998-2018
    ; Some options used here may be inadequate for your particular configuration
    ; This sample file does *not* represent stunnel.conf defaults
    ; Please consult the manual for detailed description of available options
    
    ; **************************************************************************
    ; * Global options                                                         *
    ; **************************************************************************

    ; It is recommended to drop root privileges if stunnel is started by root
    setuid = stunnel4
    setgid = stunnel4
    
    ; **************************************************************************
    ; * Service definitions (remove all services for inetd mode)               *
    ; **************************************************************************
    [dns4-1]
    client = yes
    accept = 1053
    connect = 1.1.1.1:853
    verifyChain = yes
    CApath = /etc/ssl/certs
    checkHost = cloudflare-dns.com

    [dns4-2]
    client = yes
    accept = 2053
    connect = 1.0.0.1:853
    verifyChain = yes
    CApath = /etc/ssl/certs
    checkHost = cloudflare-dns.com

    :~#> systemctl start stunnel4


    :~#> systemctl status stunnel4    
        stunnel4.service - LSB: Start or stop stunnel 4.x (TLS tunnel for network daemons)
        Loaded: loaded (/etc/init.d/stunnel4; generated; vendor preset: enabled)
        Active: active (running) since Mon 2019-02-04 14:46:42 CET; 22min ago
        Docs: man:systemd-sysv-generator(8)
        Process: 24329 ExecStop=/etc/init.d/stunnel4 stop (code=exited, status=0/SUCCESS)
        Process: 24352 ExecStart=/etc/init.d/stunnel4 start (code=exited, status=0/SUCCESS)
        Tasks: 1 (limit: 4915)
        ...
        
    Feb 04 15:08:58 dns stunnel[24367]: LOG5[663]: Service [dns4-2] connected remote server from x.x.x.x:34908
    Feb 04 15:08:58 dns stunnel[24367]: LOG5[664]: Service [dns4-2] accepted connection from x.x.x.x:44883
    Feb 04 15:08:58 dns stunnel[24367]: LOG5[663]: Connection closed: 73 byte(s) sent to TLS, 938 byte(s) sent to socket
    Feb 04 15:08:58 dns stunnel[24367]: LOG5[664]: s_connect: connected 1.0.0.1:853
    ...

    :~$> netstat -ant
    Aktive Internetverbindungen (Server und stehende Verbindungen)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State
    ...
    tcp        0      0 0.0.0.0:1053            0.0.0.0:*               LISTEN
    tcp        0      0 0.0.0.0:2053            0.0.0.0:*               LISTEN
    ...
   
    :~#> systemctl stop stunnel4
   
    :~$> ps -A | grep stunnel
    24367 ?        00:00:01 stunnel4
    
#### 3. Test


    dig @127.0.0.1 -p 1053 www.msn.com

    dig @127.0.0.1 -p 1053 +tcp www.msn.com
 
    ; <<>> DiG 9.10.3-P4-Debian <<>> @127.0.0.1 -p 1053 +tcp www.msn.com
    ; (1 server found)
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25174
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1
    
    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 1452
    ; OPT=12: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00     00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00     ("................................................................................................................................................................................................................................................................  ..........................................................................")
    ;; QUESTION SECTION:
    ;www.msn.com.			IN	A
    
    ;; ANSWER SECTION:
    www.msn.com.		264	IN	CNAME	www-msn-com.a-0003.a-msedge.net.
    www-msn-com.a-0003.a-msedge.net. 135 IN	CNAME	a-0003.a-msedge.net.
    a-0003.a-msedge.net.	227	IN	A	204.79.197.203
    
    ;; Query time: 31 msec
    ;; SERVER: 127.0.0.1#1053(127.0.0.1)
    ;; WHEN: Fri Feb 01 19:44:31 CET 2019
    ;; MSG SIZE  rcvd: 468

### Bind Configuration

    apt-get -t stretch-backports install bind9

    options {
      ...
      
      forwarders {
        // 1.1.1.1
        // 1.0.0.1
        127.0.0.1 port 1053;
        127.0.0.1 port 2053;
      };
  
      forward only;
      ...
    };

    server 127.0.0.1 {
      tcp-only yes;
    };

    ...

# Links
1. DNS over TLS: [https://kb.isc.org/docs/aa-01386](https://kb.isc.org/docs/aa-01386)

