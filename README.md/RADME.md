# DHCP - Dynamic Host Configuration Protocol

### Wprowadzenie

Protokół DHCP umożliwia automatyczną konfigurację sieciową hosta podłączającego się do sieci. Od serwera DHCP może on uzyskać swój adres IP i maskę sieci, adres domyślnej bramy, adres lokalnego serwera nazw i wiele innych parametrów.

Parametry sieciowe są dzierżawione na określony czas (godziny, dni). Klient zgłasza się do serwera w celu odnowienia dzierżawy. Jeżeli tego nie zrobi dzierżawa wygasa i serwer może przypisać adres innemu klientowi.

### Ustawienia początkowe

Będziemy korzystać z dwóch maszyn wirtualnych. Można użyć maszyn H2 i R2 z poprzednich zajęć.

Aby pobrać pakiet instalacyjny, konieczne będzie podłączenie R2 do rzeczywistej sieci. W tym celu zmieniamy ustawienia karty sieciowej interfejsu `eth0` na NAT (pierwsza karta w ustawieniach R2 w VirtualBox). Drugą (lub trzecią) kartę pozostawiamy podłączoną do sieci wewnętrznej `siknet2`. Interfejsowi eth1 przypisujemy adres `10.11.0.1/24`.

### Instalacja i konfiguracja serwera DHCP

Instalację i konfigurację serwera przeprowadzimy na maszynie R2. Serwer będzie obsługiwał podsieć `10.11.0.0`

Poniżej przedstawiamy dwa różne serwery DHCP.

### Serwer ISC (tylko DHCP)

```console
R2:# apt update
R2:# apt install isc-dhcp-server
```

Demon serwera DHCP nosi nazwę `dhcpd`, a jego plikiem konfiguracyjnym jest `/etc/dhcp/dhcpd.conf`, w którym należy zdefiniować:

- początkowy i maksymalny okres dzierżawy w sekundach (tutaj wartości domyślne):
```
default-lease-time 600;
max-lease-time 7200;
```
- podsieci, w których serwer zarządza adresami wraz z pulą adresów do rozdzielenia i ewentualnie dodatkowymi parametrami takimi jak adres domyślnej bramy:
 ```cpp
subnet 10.11.0.0 netmask 255.255.255.0 {
  range 10.11.0.2 10.11.0.20;
  option routers 10.11.0.1;
}
```

Po zmianach w pliku konfiguracyjnym należy zrestartować serwer dhcpd:

```console
R2:# /etc/init.d/isc-dhcp-server restart
```

W pliku konfiguracyjnym możemy umieścić również inne parametry, które serwer będzie przekazywać klientom, takie jak nazwa domeny czy adres serwera nazw:

```console
option domain-name "siklab.mimuw.edu.pl"; 
option domain-name-servers 10.10.0.10; 
```

Można przydzielić hostom adresy na stałe, przykładowo:

```cpp
host H8 {
  hardware ethernet XX:XX:XX:XX:XX:XX;
  fixed-address 10.11.0.21;
}
```

W miejsce `XX:XX:XX:XX:XX:XX` należy wpisać adres sprzętowy odpowiedniego interfejsu hosta.

Więcej informacji na temat konfiguracji w `man dhcpd.conf` oraz w komentarzach w pliku `dhcpd.conf`.

### Dnsmasq (DNS+DHCP)

Serwer Dnsmasq spełnia rolę serwer DHCP oraz uproszczonego serwera DNS (tylko przekazywanie i buforowanie zapytań). Zastosowanie ma przede wszystkim w małych sieciach domowych.

```console
R2:# apt update
R2:# apt install dnsmasq
```

Plikiem konfiguracyjnym jest `/etc/dnsmasq.conf`, w którym należy odkomentować i uzupełnić następujące opcje:

```console
domain-needed
bogus-priv
listen-address=127.0.0.1,10.11.0.1
dhcp-range=10.11.0.2,10.11.0.20,255.255.255.0,12h
dhcp-option=option:router,10.11.0.1
dhcp-authoritative
dhcp-leasefile=/var/lib/dhcp/dnsmasq.leases
```

Powyższe opcje dotyczą usługi DHCP. Konfigurację DNS i połączenie tych dwóch usług pozostawiamy jako ćwiczenie.

Sprwadzenie poprawności składniowej pliku konfiguracyjnego:

```console
R2:# dnsmasq --test
dnsmasq: syntax chech OK.
```

Po zmianach w pliku konfiguracyjnym należy uruchomić usługę na przykład w ten sposób:

```console
R2:# systemctl restart dnsmasq
```

Wszystkie istotne opcje konfiguracji można znaleźć w komentarzach w pliku `/etc/dnsmasq.conf`.

### Test konfiguracji

Maszyna H2 powinna mieć podłączoną jedną kartę sieciową do sieci wewnętrznej `siknet2`, a w pliku `etc/network/interfaces` ustawienia konfiguracji przez serwer dhcp:

```cpp
auto eth0
iface eth0 inet dhcp
```

Następnie możemy zrestartować maszynę H2 lub wykonać:
```console
H2:# ifdown eth0
H2:# ifup eth0
```
albo

```console
H2:# dhclient eth0
```

Maszyna powinna uzyskać adres z puli `10.11.0.2 - 10.11.0.20`. Jaki adres został przydzielony?

Stan dzierżawy można obejrzeć w plikach `/var/lib/dhcp/dhclient`.leases w systemie klienta oraz `/var/lib/dhcp/dhcpd.leases` lub `/var/lib/dhcp/dnsmasq.leases` na serwerze.

### Uwagi

Jaki może być problem przy jednoczesnym zastosowaniu serwera DNS do rozwiązywania nazw maszyn w sieci lokalnej i serwera DHCP do dynamicznego przydzielania im adresów IP?

Jednym ze sposobów rozwiązania problemu statycznych baz DNS i dynamicznej konfiguracji adresów IP przez DHCP jest przypisanie ogólnych nazw dla każdego dzierżawionego adresu i przydzielanie ich wraz z adresami IP. Inna możliwość to skonfigurowanie dhcpd w taki sposób, aby aktualizował bazę DNS przy rozdzielaniu adresów. Jeszcze inna, to zastosowanie protokołu multicast Domain Name System (mDNS).

### Ćwiczenie punktowane (1 pkt)

- Utwórz sieć złożoną z maszyn wirtualnych H2, H3 i R2 zgodnie ze schematem (można przekształcić maszynę R1 w H3).
```cpp

                                  H2
                                   |  
                                   |
                                10.11.0.0 (siknet2)
                                   |
                                   | eth1
         H3--------10.13.0.0-------R2
                   (siknet3)   eth0
```
- Na R2 przypisz adresy `10.11.0.1/24` interfejsowi `eth1` oraz `10.13.0.1/24` interfejsowi `eth0`. Ustaw opcje przekazywania pakietów (`ip_forward`, zajęcia 11).
- Skonfiguruj i uruchom (dowolny) serwer DHCP na maszynie R2. Serwer ma przydzielać adresy z pul `10.11.0.2 - 10.11.0.20` i `10.13.0.3 - 10.13.0.20` z maską `255.255.255.0` oraz domyślne bramy odpowiednio `10.11.0.1` i `10.13.0.1`.

- Skonfiguruj maszyny H2 i H3 tak, aby korzystały z serwera DHCP do konfiguracji parametrów sieciowych. Uruchom maszynę H2 i sprawdź jaki adres dostała.

- Zapisz do pliku `start-ping-dump` wynik wykonania `tcpdump -i eth0` na maszynie R2 w trakcie uruchamiania maszyny H3 i wykonywania na niej polecenia `ping <adres-przydzielony-H2>`.

- Zidentyfikuj trzy różne protokoły, których pakiety znajdują się wśród przechwyconych poleceniem `tcpdump`. Dla każdego z protokołów wybierz po jednym pakiecie, określ protokół, adresy IP nadawcy i odbiorcy oraz w jednym zadaniu napisz do czego dany pakiet służy.

