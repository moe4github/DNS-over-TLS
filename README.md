# Use TLS to secure request of Bind9 Forwardes 

## Motivation

DNS Abfragen, zu Namen, die mein lokaler DNS Server noch nicht gecached hat bzw. Domains betreffen, die nicht in der Verwaltung meines DNS Servers liegen. gehen bislang im Klartext über das Internet zu meinen definierten Forwarders. Jeder, der zugriff auf die Routen hat, kann somit meine Anfragen sammeln und ggf. ein Profil erstellen. Um sich davor zu schützen gibt es mehrere Möglichkeiten:

* [DNSSEC](https://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions)
* [DNS over HTTPS](https://en.wikipedia.org/wiki/DNS_over_HTTPS)
* [DNS over TLS](https://en.wikipedia.org/wiki/DNS_over_TLS)

wobei ich auf den 3. Punkt, DNS over TLS, eingehen möchte. Ich will dabei nicht den gesamten lokalen DNS Server absichern, sondern nur die Anfragen der definierten Forwarders.

## Requirements
* Bind DNS in your private/local Network
* stunnel

### stunnel

#### 1. Install

#### 2. Configuration

#### 3. Test

### Bind Configuration
