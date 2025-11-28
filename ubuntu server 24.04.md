<!-- 
MIT License

Copyright (c) 2025 Rafał Kiepiela

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
-->

---
title: "Konfiguracja Ubuntu Server 24.04"
author: [Rafał Kiepiela]
listings-no-page-break: true
titlepage: true
titlepage-background: "background2.pdf"
---

github.com/TexturedPolak

# Konfiguracja Ubuntu Server 24.04

## 0. Słowem Wstępu

Dokument ten przedstawia przykładową konfigurację Ubuntu Server 24.04.
Konfigurowane usługi to:

- sieć (netplan)
- serwer DHCP (isc-dhcp-server)
- serwer DNS (bind9)
- routing (iptables)
- serwer plików FTP (vsftpd)
- serwer plików Samba (samba)
- serwer WWW (apache2)
- serwer SSH (openssh-server)
- serwer telnet (telnetd)

Autor zaznacza, że zrzuty ekrany były kadrowane tylko i wyłącznie dla czytelności dokumentu oraz, że jest świadomy, iż na egzaminie z kwalifikacji INF.02 nie można kadrować zrzutów ekranów.

## 1. Konfiguracja sieci

Przyjęte założenia:

- automatycznie przydzielany adres IP (DHCP) na pierwszej karcie sieciowej
- statyczny adres IP klasy B (w moim przypadku 172.16.0.1), maska podsieci 16 bitowa na drugiej karcie sieciowej

### Krok 1 - Sprawdzenie stanu i nazw kart sieciowych

```sh
ip a
```

![](/run/media/rafal/ECD5-E984/photos/ip-a-przed.png)

Widzimy dwie karty sieciowe:

- enp0s3 - WAN (pierwsza karta sieciowa)
- enp0s8 - LAN (druga karta sieciowa)

### Krok 2 - Edycja pliku konfiguracyjnego netplan'a

Nazwa pliku konfiguracyjnego może nieznacznie się różnić w zależności od edycji systemu, dlatego zastosowano znak "*":

```sh
sudo nano /etc/netplan/*.yaml
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-netplan.png)

Omówię teraz tą konfigurację:

```yaml
network:
  version: 2
  ethernets:
    # Pierwsza karta sieciowa
    enp0s3: 
      # Automatyczne przydzielanie adresu
      dhcp4: true 
    # Druga karta sieciowa
    enp0s8:
      # Wyłączenie automatycznego przydzielania adresów
      dhcp4: false 
      # Ustawienie statycznego adresu IP (172.16.0.1, maska 16 bitowa)
      addresses: [172.16.0.1/16]
      # Ustawianie adresów serwerów DNS
      nameservers: 
        # Adresy serwerów DNS
        addresses: [172.16.0.1, 1.1.1.1] 
```

Należy zwrócić uwagę na zachowanie wcięć.

### Krok 3 - Zastosowanie konfiguracji oraz sprawdzenie poprawności wykonanej konfiguracji

```sh
sudo netplan apply
```

```sh
ip a
```

![](/run/media/rafal/ECD5-E984/photos/netplan-apply.png)

Żaden błąd nie wyskoczył, a konfiguracja sieci wygląda poprawnie.

## 2. Konfiguracja serwera DHCP

### Krok 0 - Instalacja pakietu isc-dhcp-server

```sh
sudo apt update
```

```sh
sudo apt install isc-dhcp-server
``` 

Ja już ten pakiet zainstalowałem szybciej, przejdźmy do konfiguracji.

### Krok 1 - Edycja pliku konfiguracyjnego isc-dhcp-server

```sh
sudo nano /etc/default/isc-dhcp-server
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-isc-dhcp-server.png)

Omówienie konfiguracji:

```toml
# (...) Komentarze (...)

# Podajemy nazwę karty sieciowej LAN'owej 
# (w moim przypadku enp0s8)
INTERFACESv4="enp0s8"
# Zostawiamy puste 
INTERFACESv6=""
```

### Krok 2 - Edycja pliku konfiguracyjnego dhcpd.conf

```sh
sudo nano /etc/dhcp/dhcpd.conf
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-dhcpd2.png)

Omówienie konfiguracji:

```sh
# (...) Komentarze i inne domyślne ustawienia (...)

# Ustawienie serwera DHCP jako jedyny w sieci
authoritative;

# (...) Komentarze (...)

# subnet - adres sieci - netmask - maska podsieci
subnet 172.16.0.0 netmask 255.255.0.0 {
  # Zakres sieci (bez serwera)
  # W moim przypadku 172.16.0.10-172.16.0.20
  range 172.16.0.10 172.16.0.20;
  # Adresy serwerów DNS
  option domain-name-servers 172.16.0.1, 1.1.1.1;
  # Nazwa domeny (opcjonalnie)
  option domain-name "kiepiela.local";
  # Maska podsieci
  option subnet-mask 255.255.0.0;
  # Adres rozgłoszeniowy
  option broadcast-address 172.16.255.255;
  # Czas zalokowania adresu
  default-lease-time 600;
  # Maksymalny czas zalokowania adresu
  max-lease-time 7200;
}
```

### Krok 3 - Zastosowanie konfiguracji oraz sprawdzenie poprawności wykonanej konfiguracji

```sh
sudo systemctl restart isc-dhcp-server
```

```sh
sudo systemctl status isc-dhcp-server
```

![](/run/media/rafal/ECD5-E984/photos/systemctl-dhcp.png)

Jest na zielono? - to oznacza, że wszystko działa :)

## 3. Konfiguracja serwera DNS

### Krok 0 - Instalacja pakietu bind9 i bind9-utils

```sh
sudo apt update
```

```sh
sudo apt install bind9 bind9-utils
```

Ja już te pakiety zainstalowałem szybciej, przejdźmy do konfiguracji.

### Krok 1 - Edycja pliku konfiguracyjnego named.conf.options

```sh
sudo nano /etc/bind/named.conf.options
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-named-conf-options.png)

Omówienie konfiguracji:

```java
options {
  // Wartość domyślna
  directory "/var/cache/bind";

  // (...) Komendarze (...)

  forwarders {
    // Adresy publicznych serwerów DNS
    1.1.1.1;
    8.8.8.8;
  }

  // (...) Komentarze (...)

  // Wartości domyślne
  dnssec-validation auto;
  listen-on-v6 {any; };
}
```

### Krok 2 - Edycja pliku konfiguracyjnego named.conf.local

```sh
sudo nano /etc/bind/named.conf.local
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-named-conf-local2.png)

Omówienie konfiguracji:

```java
// (...) Komentarze (...)

// Domena (strefa do przodu)
zone "kiepiela.local"{
  type master;
  file "/etc/bind/strefaprzod";
};
// Adres sieci bez zer od tyłu + ".in-addr.arpa" (strefa do tyłu)
zone "16.172.in-addr.arpa"{
  type master;
  file "/etc/bind/strefatyl";
};
```

### Krok 3 - Utworzenie i edycja pliku konfiguracyjnego strefaprzod

Kopiowanie domyślnego pliku konfiguracyjnego do naszego:
```sh
sudo cp /etc/bind/db.local /etc/bind/strefaprzod
```

```sh
sudo nano /etc/bind/strefaprzod
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-strefaprzod2.png)

Omówienie konfiguracji:

```lisp
; (...) komentarze (...)

$TTL  604800;     użytkownik.domena.    root.użytkownik.domena.
@     IN  SOA     rafal.kiepiela.local. root.rafal.kiepiela.local.(
                        2         ; Serial
                   604800         ; Refresh
                    86400         ; Retry
                  2419200         ; Expire
                   604800 )       ; Negative Cache TTL
;
;                 użytkownik.domena.
@     IN  NS      rafal.kiepiela.local.
;                 adres ip serwera
@     IN  A       172.16.0.1
;@     IN  AAAA    ::1
; użytkownik      adres ip serwera
rafal IN  A       172.16.0.1
```

### Krok 4 - Utworzenie i edycja pliku konfiguracyjnego strefatyl

Kopiowanie pliku konfiguracyjnego strefaprzod do strefatyl:
```sh
sudo cp /etc/bind/strefaprzod /etc/bind/strefatyl
```

```sh
sudo nano /etc/bind/strefatyl
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-strefatyl2.png)

Omówienie konfiguracji:

```lisp
; (...) komentarze (...)

$TTL  604800;     użytkownik.domena.    root.użytkownik.domena.
@     IN  SOA     rafal.kiepiela.local. root.rafal.kiepiela.local.(
                        2         ; Serial
                   604800         ; Refresh
                    86400         ; Retry
                  2419200         ; Expire
                   604800 )       ; Negative Cache TTL
;
;                 użytkownik.domena.
@     IN  NS      rafal.kiepiela.local.
;                 adres ip serwera
@     IN  A       172.16.0.1
;@     IN  AAAA    ::1
; użytkownik      adres ip serwera
rafal IN  A       172.16.0.1

; nowości od tego momentu:
; adres serwera od tyłu bez części sieciowej | użytkownik.domena.
1.0   IN  PTR     rafal.kiepiela.local.
;                 domena.
@     IN  PTR     kiepiela.local.
```

### Krok 5 - Zastosowanie konfiguracji oraz sprawdzenie poprawności wykonanej konfiguracji

```sh
sudo systemctl restart bind9
```

```sh
sudo systemctl status bind9
```

![](/run/media/rafal/ECD5-E984/photos/systemctl-bind.png)

Jest na zielono? - to oznacza, że wszystko działa :)

### Krok 6 - Edycja pliku konfiguracyjnego resolv.conf

```sh
sudo nano /etc/resolv.conf
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-resolv.png)

Omówienie konfiguracji:

```sh
# (...) Komentarze (...)

# Adres serwera - na serwerze można ustawić localhost
nameserver 127.0.0.1
# Domena
search kiepiela.local
```

Warto też zabezpieczyć plik przed nadpisywaniem przez system podczas restartu:

```sh
realpath /etc/resolv.conf
```

```sh
sudo chattr +i /run/systemd/resolve/stub-resolv.conf
``` 

![](/run/media/rafal/ECD5-E984/photos/chattr-resolv.png)

Aktualizacja systemu jednak może powodować "zdjęcie" tej "ochrony".

### Krok 7 - Zastosowanie konfiguracji oraz sprawdzenie poprawności wykonanej konfiguracji (nslookup)

Warto sprawdzić czy wszystko działa narzędziem nslookup:

```sh
nslookup kiepiela.local
```

```sh
nslookup rafal.kiepiela.local
```

```sh
nslookup wp.pl
```

```sh
nslookup 172.16.0.1
```

```sh
nslookup rafal
```

![](/run/media/rafal/ECD5-E984/photos/nslookup3.png)

Jak widać strefa do przodu jak i do tył zwraca pożądany wynik.

### Krok 8 (w razie problemów) - Edycja pliku konfiguracyjnego hosts

Plik /etc/hosts omija serwery DNS (tak nawet nasz, który konfigurowaliśmy) i wymusza swoje własne wartości. Takie trochę rozwiązanie "na chama".

```sh
sudo nano /etc/hosts
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-hosts.png)

Omówienie konfiguracji:

```sh
127.0.0.1 localhost
127.0.1.1 kiepiela
# adres serwera | użytkownik
172.16.0.1 rafal

# (...) domyślne ustawienia (...)
```

## 4. Konfiguracja routingu (NAT)

### Krok 1 - włączenie routingu w systemie (plik sysctl.conf)

```sh
sudo nano /etc/sysctl.conf
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-sysctl.png)

Omówienie konfiguracji:

```sh
# Należy odkomentować linijkę poniżej:
net.ipv4.ip_forward=1
```

Zastosowanie konfiguracji:

```sh
sudo sysctl -p
```

![](/run/media/rafal/ECD5-E984/photos/sysctl.png)

### Krok 2 - Konfiguracja maskarady w firewall'u

```sh
                                  # nazwa karty WAN
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

![](/run/media/rafal/ECD5-E984/photos/iptables.png)

### Krok 3 - Zachowanie routingu po restarcie

Aby zachować ustawienia routingu po restarcie należy zainstalować pakiet iptables-persistent.

```sh
sudo apt update
```

```sh
sudo apt install iptables-persistent
```

Podczas instalacji wyskoczą 2 monity. Należy wybrać "yes".

![](/run/media/rafal/ECD5-E984/photos/iptables-monit-v4.png)

![](/run/media/rafal/ECD5-E984/photos/iptables-monit-v6.png)

Konfigurację warto jeszcze raz ręczenie zapisać:

```sh
sudo iptables-save
```
![](/run/media/rafal/ECD5-E984/photos/iptables-save.png)

## 5. Serwer plików FTP

### Krok 0 - Instalacja pakietu vsftpd

```sh
sudo apt update
```

```sh
sudo apt install vsftpd
```

Ja już ten pakiet zainstalowałem szybciej, przejdźmy do konfiguracji.

### Krok 1 - Edycja pliku konfiguracyjnego vsftpd.conf

Na potrzeby ćwiczeniowe przyjmujemy, że użytkownik anonimowy będzie miał dostęp do zapisu jak i odczytu.

```sh
sudo nano /etc/vsftpd.conf
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/vsftpd.png)

Omówienie konfiguracji:

```sh
# Włączenie serwera
listen=YES
# Wyłaczenie nagłaśniania na ipv6 (musi być NO inaczej powoduje problemy)
listen_ipv6=NO
# Włączenie dostępu anonimowego
anonymous_enable=YES
# Włączenie logowania bez hasła
no_anon_password=YES
# Włączenie zapisu
write_enable=YES
# Coś z uprawnieniami
local_umask=022
anon_umask=022
# Włączenie zapisu dla anonimów
anon_upload_enable=YES
# Włączenie tworzenia folderów dla anonimów
anon_mkdir_write_enable=YES
# Włączenie innych uprawnień dla anonimów
anon_other_write_enable=YES
# Ścieżka do folderu (zaraz go utworzymy)
anon_root=/srv/anon

# (...) Reszta domyślnie (...)
```

### Krok 2 - Utworzenie odpowiednich folderów i ustawienie odpowiednich uprawnień

Tworzymy dwa foldery:

```sh
sudo mkdir /srv/anon
```

```sh
sudo mkdir /srv/anon/folder-do-zapisu
```

I ustawiamy odpowiednie uprawnienia:

```sh
sudo chmod 755 /srv/anon # Tak, ten folder ma mieć uprawnienia zapisu tylko dla właściciela
```

```sh
sudo chmod 777 /srv/anon/folder-do-zapisu
```

![](/run/media/rafal/ECD5-E984/photos/vsftpd-folders.png)

### Krok 3 - Zastosowanie konfiguracji oraz sprawdzenie poprawności wykonanej konfiguracji

```sh
sudo systemctl restart vsftpd
```

```sh
sudo systemctl status vsftpd
```

![](/run/media/rafal/ECD5-E984/photos/systemctl-vsftpd.png)

Jest na zielono? - to oznacza, że wszystko działa :)

## 6. Serwer plików Samba

### Krok 0 - Instalacja pakietu samba

```sh
sudo apt update
```

```sh
sudo apt install samba
```

Ja już ten pakiet zainstalowałem szybciej, przejdźmy do konfiguracji.

### Krok 1 - Edycja pliku konfiguracyjnego smb.conf

Na potrzeby ćwiczeniowe przyjmujemy, że użytkownik anonimowy będzie miał dostęp do zapisu jak i odczytu.

```sh
sudo nano /etc/samba/smb.conf
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-smb.png)

Omówienie konfiguracji:

```ini
# (...) Domyślne ustawienia i kometarze (...)

[anon]
  # Ścieżka do folderu który zaraz utworzymy
  path = /srv/samba
  # Dostęp anonimowy
  guest ok = yes
  # Przeglądanie zawartości
  browseable = yes
  # Włączenie zapisu
  read only = no
  writeable = yes
  # Czy widoczny na liście
  public = yes 
  # Coś z uprawnieniami
  create mask = 0666
  directory mask = 0777
  # Wymuszenie danego użytkownika do obsługi serwera 
  # (NIE STOSOWAĆ ROOT'a NA PRODUKCJI!)
  force user = root
```

### Krok 2 - Utworzenie odpowiednich folderów i ustawienie odpowiednich uprawnień

```sh
sudo mkdir /srv/samba
```

```sh
sudo chmod 777 /srv/samba
```

![](/run/media/rafal/ECD5-E984/photos/mkdir-samba.png)

### Krok 3 - Zastosowanie konfiguracji oraz sprawdzenie poprawności wykonanej konfiguracji

```sh
sudo systemctl restart smbd
```

```sh
sudo systemctl status smbd
```

![](/run/media/rafal/ECD5-E984/photos/systemctl-smbd.png)

Jest na zielono? - to oznacza, że wszystko działa :)

## 7. Konfiguracja serwera WWW

### Krok 0.0 - Instalacja pakietu apache2

```sh
sudo apt update
```

```sh
sudo apt install apache2
``` 

Ja już ten pakiet zainstalowałem szybciej, przejdźmy do konfiguracji.

### Krok 0.1 - Założenia

Scenariusz zakłada, że strona internetowa (plik inny.html jako strona główna) znajduje się w folderze /srv/www a katalog ma uprawnienie odczytu dla serwera www (najlepiej 755). Uprawnienia można zmienić w ten sposób:

```sh
sudo chmod 755 /srv/www
```

### Krok 1 - Konfiguracja pliku konfiguracyjnego 000-default.conf

```sh
sudo nano /etc/apache2/sites-available/000-default.conf
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/sites.png)

Omówienie konfiguracji:

```sh
<VirtualHost *:80>

   # (...) Kometarze (...)

  ServerAdmin webmaster@localhost # Adres e-mail admina (do celów ćwiczeniowych można zostawić to co jest)
  DocumentRoot /srv/www # ścieżka do folderu ze stroną

  # (...) Kometarze i inne domyślne ustawienia (...)

  DirectoryIndex inny.html # ustawienie strony głównej na inny.html zamiast domyślnego index.html
</VirtualHost>
```

### Krok 2 - Konfiguracja pliku konfiguracyjnego apache2.conf

```sh
sudo nano /etc/apache2/apache2.conf
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/apache2-conf.png)

Omówienie konfiguracji:

```sh
# (...) Komentarze i inne domyślne ustawienia (...)

#       ścieżka do folderu ze stroną
<Directory /srv/www/>
    # 3 ustawienia domyślne:
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

# (...) Komentarze i inne domyślne ustawienia (...)
```

### Krok 3 - Zastosowanie konfiguracji oraz sprawdzenie poprawności wykonanej konfiguracji

```sh
sudo systemctl restart apache2
```

```sh
sudo systemctl status apache2
```

Dodatkowo możemy sprawdzić czy strona dotarła by po wpisaniu adresu w przeglądarce za pomocą narzędzia curl:

```sh
curl localhost
```

Polecenie powinno zwrócić kod html naszej strony, w tym przypadku pliku inny.html

![](/run/media/rafal/ECD5-E984/photos/curl.png)

Jest na zielono? Curl wyświetla kod strony? - to oznacza, że wszystko działa :)

## 8. Konfiguracja serwera SSH

### Krok 0 - Instalacja pakietu openssh-server

```sh
sudo apt update
```

```sh
sudo apt install openssh-server
``` 

Ja już ten pakiet zainstalowałem szybciej, przejdźmy do konfiguracji.

### Krok 1 - Uruchomienie usługi SSH

```sh
sudo systemctl enable ssh --now
```

```sh
sudo systemctl status ssh
```

![](/run/media/rafal/ECD5-E984/photos/ssh.png)

Jest na zielono? - to oznacza, że wszystko działa :)

## 9. Konfiguracja serwera Telnet (używać SSH, chyba, że precyzowano, iż należy używać Telneta)

### Krok 0 - Instalacja pakietu telnetd

```sh
sudo apt update
```

```sh
sudo apt install telnetd
``` 

Ja już ten pakiet zainstalowałem szybciej, przejdźmy do konfiguracji.

### Krok 1 - Edycja pliku konfiguracyjnego inetd.conf

```sh
sudo nano /etc/inetd.conf
```

Edytujemy plik do pożądanego rezultatu:

![](/run/media/rafal/ECD5-E984/photos/nano-inetd.png)

Omówienie konfiguracji:

```sh
# Należy odkomentować linijkę:
telnet  stream  tcp   nowait  root    /usr/sbin/tcpd  /usr/sbin/telnetd
```

### Krok 3 - Zastosowanie konfiguracji oraz sprawdzenie poprawności wykonanej konfiguracji

```sh
sudo systemctl restart inetd
```

```sh
sudo systemctl status inetd
```

![](/run/media/rafal/ECD5-E984/photos/systemctl-inetd.png)

Jest na zielono? - to oznacza, że wszystko działa :)


## Bonus - Aktualizacja systemu

### Krok 0 - Aktualizacja repozytoriów

```sh
sudo apt update
```

![](/run/media/rafal/ECD5-E984/photos/apt-update.png)

### Krok 1 - Aktualizacja systemu 

```sh
sudo apt upgrade
```

![](/run/media/rafal/ECD5-E984/photos/apt-upgrade.png)

```sh
reboot
```

lub 

```sh
sudo reboot
```

## Bonus - Aktualizacja Ubuntu Server do nowszej wersji

### Krok 0 - Aktualizacja repozytoriów

```sh
sudo apt update
```

![](/run/media/rafal/ECD5-E984/photos/apt-update.png)

### Krok 1 - Aktualizacja Ubuntu Server do nowszej wersji

```sh
sudo apt dist-upgrade
```