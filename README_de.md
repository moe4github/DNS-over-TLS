# Bind9 Forwarders Anfragen mittels TLS absichern 

## Motivation

DNS Abfragen, zu Hostnamen, die der lokale DNS Server noch nicht gecached hat und Domains betreffen, die nicht in der Verwaltung des DNS Servers liegen, gehen bislang im Klartext über das Internet zu den definierten Forwarders. Jeder, der Zugriff auf das öffentliche Netz und seine Knoten hat, kann somit die Anfragen sammeln und ggf. ein Profil erstellen. Um sich davor zu schützen gibt es mehrere Möglichkeiten:

* [DNSSEC](https://de.wikipedia.org/wiki/Domain_Name_System_Security_Extensions) (Authentizität und Integrität aber nicht verschlüsselt)
* [DNS over HTTPS](https://de.wikipedia.org/wiki/DNS_over_HTTPS)
* [DNS over TLS](https://de.wikipedia.org/wiki/DNS_over_TLS)

wobei ich auf den 3. Punkt, **DNS over TLS**, eingehen möchte. Ich will dabei nicht den gesamten lokalen DNS Server absichern (wie 1. Link), sondern nur die Anfragen der definierten Forwarders, die letzen Endes das lokal Netzwerk verlassen.

## Voraussetzung
* Bind9 DNS im eigenen privaten/lokalen LAN (Version >= 9.11)
* [stunnel4](https://www.stunnel.org/index.html)

### stunnel4

Durch stunnel kann die Kommunikation von Clients oder Server, welche selbst keine Verschlüsselung unterstützen, nachträglich verschlüsselt werden.

#### 1. Installation

Die Installation unter Debian *Stretch* ist mit einem einfachen

    apt-get install stunnel4
    
erledigt.

#### 2. Konfiguration

Jetzt muss stunnel nur noch dazu gebracht werden, auf definierten Port Anfragen entgegen zu nehmen und diese, als Client, an die hinterlegten Server weiterleitet.

Die Konfiguration von stunnel erfolgt in Debian Stretch unter `/etc/stunnel/stunnel.conf`. Diese wird um zwei Services, dns4-1 und dns4-2, erweitert.

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

Die IP Adressen 1.1.1.1 und 1.0.0.1 sind DNS Server welche von [Cloudflare](https://www.cloudflare.com/de-de/dns/) bereitgestellt und als Forwarders genutzt werden sollen. Der Standardport für DNS over TLS ist 853. Die Konfiguration kann wie folgt gelesen werden: Warte auf Port 1053/2053 auf Anfragen und leite diese an 1.1.1.1/1.0.0.1 als Client weiter. Somit benötige ich für jeden separaten Forwarder einen eigenen Port.

Nachdem stunnel konfiguriert wurde, können die beiden Services wie folgt gestartet werden:

    :~#> systemctl start stunnel4

Wenn keine Fehlermeldung ausgegeben wurde, kann zusätzlich der Status des Services abgefragt werden

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

Eine Überprüfung mit netstat zeigt, dass die definierten Services an den konfigurierten Ports auf Anfragen warten.

    :~$> netstat -ant
    Aktive Internetverbindungen (Server und stehende Verbindungen)
    Proto Recv-Q Send-Q Local Address           Foreign Address         State
    ...
    tcp        0      0 0.0.0.0:1053            0.0.0.0:*               LISTEN
    tcp        0      0 0.0.0.0:2053            0.0.0.0:*               LISTEN
    ...

Gestoppt wird stunnel mittels
   
    :~#> systemctl stop stunnel4
    
wobei danach immer noch die Ports laut `netstat` durch stunnel belegt sind. Eine Überprüfung mit

    :~$> ps -A | grep stunnel
    24367 ?        00:00:01 stunnel4
    
zeigt, dass der Service noch läuft. Hier hilft ein `killall stunnel4` als root und anschließend sind die Ports auch wieder frei und stehen einem Neustart von stunnel Services nicht mehr im Weg.    

#### 3. Test

Nachdem stunnel eingerichtet wurde, sollte vor der Anpassung von Bind die Funktionsweise getestet werden. Hierzu kann `dig` verwendet werden.

    dig @127.0.0.1 -p 1053 www.msn.com

Das oben stehende Kommando fragt nach der Adresse von *www.msn.com*. Als Ergebnis sollte ein *connection timed out; no servers could be reached*. Warum? Weil ein Standard DNS Anfrage immer erst via UDP versucht wird.

    dig @127.0.0.1 -p 1053 +tcp www.msn.com
 
 Durch `+tcp` wird die Anfrage explizit mit TCP durchgeführt. Die Ausgabe von *dig* sollte jetzt aussagekräftig genug sein und unter anderen die IP Adresse (*204.79.197.203*) des angefragten Hostnames ausgeben.
 
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

### Bind anpassen

Ich benutze für diese Konfiguration Linux Debian Stretch. Im Standard APT Repository ist nur Bind9 in der Version 9.10.x. verfügbar. Allerdings gibt es in den Stretch-Backports die 9.11.x Version auf die problemlos aktualisiert werden kann.

    apt-get -t stretch-backports install bind9

Die Konfiguration der Forwarders in */etc/bind/named.conf.options* kann jetzt wie folgt angepasst werden:

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

Die Option `tcp-only` im Serverbereich stellt sicher, dass der Server `127.0.0.1` nur via TCP angesprochen wird (Standard DNS Anfragen laufen über UDP). Das Feature wurde Bind in der Version 9.11 hinzugefügt.

# Links
1. DNS over TLS: [https://kb.isc.org/docs/aa-01386](https://kb.isc.org/docs/aa-01386)
