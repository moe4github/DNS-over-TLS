# Bind9 Forwarders anfragen mittels TLS absichern 

## Motivation

DNS Abfragen, zu Hostnamen, die mein lokaler DNS Server noch nicht gecached hat und Domains betreffen, die nicht in der Verwaltung meines DNS Servers liegen, gehen bislang im Klartext über das Internet zu meinen definierten Forwarders. Jeder, der Zugriff auf das öffentliche Netz und seine Knoten hat, kann somit die Anfragen sammeln und ggf. ein Profil erstellen. Um sich davor zu schützen gibt es mehrere Möglichkeiten:

* [DNSSEC](https://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions)
* [DNS over HTTPS](https://en.wikipedia.org/wiki/DNS_over_HTTPS)
* [DNS over TLS](https://en.wikipedia.org/wiki/DNS_over_TLS)

wobei ich auf den 3. Punkt, DNS over TLS, eingehen möchte. Ich will dabei nicht den gesamten lokalen DNS Server absichern (wie 1. Link), sondern nur die Anfragen der definierten Forwarders.

## Voraussetzung
* Bind9 DNS im eigenen privaten/lokalen LAN (Version >= 9.11)
* stunnel

### stunnel

#### 1. Installation

#### 2. Konfiguration


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

Die Konfiguration der Forwarders in */etc/bind/named.conf.options* kann jetzt wie folgt angepasst werden:

    options {
      ...
      
      forwarders {
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
