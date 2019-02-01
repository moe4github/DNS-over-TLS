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

### Bind Konfiguration

    options {
      ...
      
      forwarders { 127.0.0.1 port 1053; };
  
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
