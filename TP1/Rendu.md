# TP 1 - Remise dans le bain !

## Sommaire
* [I. Exploration du réseau d'une machine CentOS](#i-exploration-du-réseau-dune-machine-centos)
    * [1. Mise en place](#1-mise-en-place)
        * [Configuration](#configuration)
    * [2. Basics](#2-basics)
        * [Routes](#routes)
        * [Table ARP](#table-arp)
        * [Capture Reseau](#capture-reseau)
* [II. Communication simple entre deux machines](#ii-communication-simple-entre-deux-machines)
    * [1. Mise en place](#1-mise-en-place-1)
    * [2. Basics](#2-basics-1)
        * [Ping && ARP](#ping-&&-arp)
        * [UDP](#udp)
        * [TCP](#tcp)
        * [Firewall](#firewall)
* [III. Routage statique simple](#iii-routage-statique-simple)


# I. Exploration du réseau d'une machine CentOS

## 1. Mise en place 

### Configuration

* Combien y a-t-il d'adresses disponibles dans un `/24` ?
    * Il y en a 254 !

* Combien y a-t-il d'adresses disponibles dans un `/30` ?
    * Il y en a 2 !

* Quelle est l'utilité d'un /30 ?
    * Vu que la nous n'avons que 2 hôtes capable de se connecter cela va nous permettre de faire des tables de routage plus simple et ceci permet aussi de ne pas gaspiller d'adresses IP.

[ ] s'assurer que les 3 cartes réseaux fonctionnent :

* Carte NAT : 
    * On fait un `curl google.com` 
    * On obtient donc un retour HTML avec le code d'erreur 301 qui signifie **Moved Permanetly** qui veut dire **redirection permanente**.

* Carte net1 :
    * ping 10.1.1.1
    * `PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
        64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.037 ms`

* Carte net2 :
    * ping 10.1.2.1 
    * `PING 10.1.2.2 (10.1.2.2) 56(84) bytes of data.
64 bytes from 10.1.2.2: icmp_seq=1 ttl=64 time=0.035 ms`

## 2. Basics

### Routes

* Afficher les routes :

    ```
    ip route show

    default via 10.0.2.2 dev enp0s3 proto static metric 100 
    10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100 
    10.1.1.0/24 dev enp0s8 proto kernel scope link src 10.1.1.2 metric 100 
    10.1.2.0/30 dev enp0s9 proto kernel scope link src 10.1.2.2 metric 100 
    ```
    * La première route la route par défault qui va permettre l'accès au réseau `n'importe lequel`.

    * La route route est la route via  enp0s3 (NAT) qui va permettre l'accès à `10.0.2.0/24 `.

    * La troisième route est la route via la enp0s8 (net1) qui va permettre l'accès à `10.1.1.0/24 `.

    * La quatrième route est la route via enp0s9 (net2) qui va permettre l'accès à `10.1.2.0/30 `.
  
* Supprimer une route :

    `sudo ip route del 10.1.2.0/30`

    ```
    ping 10.1.2.1

    PING 10.1.2.1 (10.1.2.1) 56(84) bytes of data.
    From 10.1.1.2 icmp_seq=1 Destination Host Unreachable
    ```
    * Elle ne fonctionne plus, on ne peut plus utiliser le reseau concerné

* Remettre une route :

    `sudo ip route add 10.1.2.0/30 via 10.1.2.2 dev enp0s9`

    ```
    ip route show 

   default via 10.0.2.2 dev enp0s3 proto static metric 100 
    10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100 
    10.1.1.0/24 dev enp0s8 proto kernel scope link src 10.1.1.2 metric 100 
    10.1.2.0/30 dev enp0s9 proto kernel scope link src 10.1.2.2 metric 100 
    ```
    * On voit bien que la route `10.1.2.0/30` c'est bien rajouter.

### Table ARP

* Afficher les voisins que connaît notre machine = la table ARP :

    ```
    ip neigh show

    10.1.1.0 dev enp0s8 lladdr 0a:00:27:00:00:06 REACHABLE
    ```

    * Cette ligne veut dire que j'ai déjà discuté avec la machine en `10.1.1.0` qui est sur le même réseau.

* Vider la table ARP :

    `sudo ip neigh flush all`

    * Quand l'on fait un `ip neigh show`il n'y a plus rien.

* Effectuer une requête simple vers l'hôte :

    `ping 10.1.1.2`

    ```
    ip neigh show

    10.1.1.0 dev enp0s8 lladdr 0a:00:27:00:00:06 REACHABLE
    ```

### Capture réseau

* On capture 10 packets

    ```
    [slbordas@client1 ~]$ sudo tcpdump -i enp0s9 -w ping.pcap
    [sudo] Mot de passe de slbordas : 
    tcpdump: listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
    ^C10 packets captured
    10 packets received by filter
    0 packets dropped by kernel
    ```

* Screen du Wireshark :

    [Voir ping.pcap](/TP1/pcap/ping.pcap)

    ![alt text](/TP1/screens/ping.png "Whireshark")

    * La ligne **1** va être une requête ARP vers l'ip `10.1.2.1` pour lui demander son adresse MAC, vu qu'au préalable nous avions vider la table ARP.
    * La ligne **2** c'est la réponse avec l'adresse MAC.
    * De la ligne **3 à 10** ce sont des requêtes `ping 'pong'`.

## Communication simple entre deux machines

## 1. Mise en place

* Nouveau clone de VM 

    * Configuration d'une nouvelle ip statique
    ```
    [slbordas@client2 ~]$ cat /etc/sysconfig/network-scripts/ifcfg-enp0s8
    TYPE=Ethernet
    BOOTPROTO=static

    NAME=enp0s8
    DEVICE=enp0s8

    ONBOOT=yes

    IPADDR=10.1.1.3
    NETMASK=255.255.255.0
    ```

    * On valide
        ```
        ifdown enp0s8
        ifup enp0s8
        ```

    * Changement du nom de domaine : 
    ```
    sudo hostname client2.tp1.b2
    ```

    * Edition du fichier hosts de client2 :
    ```
    [slbordas@client2 ~]$ cat /etc/hosts
    127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
    ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
    10.1.1.3    localhost client2.tp1.b2
    ```

## 2. Basics

### Ping && ARP

* Vider les tables ARP des deux machines (#LaClassique):

    ```
    sudo ip neigh flush all
    ```

* On va ping **client2** depuis **client1** :

    ```
    ping -c 4 client2 (10.1.1.3)
    ```
    * Ensuite on va aller check la table ARP : 

        ```
        slbordas@localhost ~]$ ip neigh show
        10.1.1.1 dev enp0s8 lladdr 0a:00:27:00:00:06 REACHABLE
        10.1.1.3 dev enp0s8 lladdr 08:00:27:26:2e:dc REACHABLE
        ```
        _10.1.1.3 vient de s'ajouter !_

* On va refaire la même opération en faisant une capture réseau : 

    * Sur SSH1 client1 :

        ```
        sudo tcpdump -i enp0s8 -w ping-2.pcap
        ```

     * Sur SSH2 client1 :

        ```
        ping -c 4 10.1.1.3
        ```

    * Sur SSH1 client1 : 

        ```
        ^C10 packets captured
        10 packets received by filter
        0 packets dropped by kernel
        ```

    * On récupère le fichier ping-2.pcap

        ```
        scp slbordas@10.1.1.2:/home/Matthieu.Bordas/ping-2.pcap ~/Desktop
        ```

* On passe le fichier dans Wireshark : 

    [Voir ping-2.pcap](/TP1/pcap/ping-2.pcap)

    ![alt text](/TP1/screens/ping-2.png "Whireshark2")

    * La ligne **1** va être une requête ARP vers l'ip `10.1.2.1` pour lui demander son adresse MAC, vu qu'au préalable nous avions vider la table ARP.
    * La ligne **2** c'est la réponse avec l'adresse MAC.
    * De la ligne **3 à 10** ce sont des requêtes `ping 'pong'`.

___
### UDP

* Sur client1 :

    * Ouverture du port 8888 : 

        ```
        [slbordas@localhost ~]$ sudo firewall-cmd --add-port=8888/udp --permanent
        [sudo] password for slbordas:
        success

        [slbordas@localhost ~]$ sudo firewall-cmd --reload
        success
        ```

    * On lance netcat pour qu'il écoute sur le port UDP 8888 : 

        ```
        [slbordas@localhost ~]$ nc -u -l 8888

        ```

* Sur client2 : 

    ```
    [slbordas@client2 ~]$ nc -u 10.1.1.2 8888

    ```

    ![alt text](/TP1/screens/chat.png "chat")
    _Le tchat marche enfin_

* Sur client1 (2nd shell) : 

    ```
    [slbordas@localhost ~]$ ss -unp
    Recv-Q Send-Q Local Address:Port               Peer Address:Port              
    0      0      10.1.1.2:8888               10.1.1.3:43889              users:(("nc",pid=1512,fd=4))
    ```

* Sur client2 (2nd shell) : 

    ```
    [slbordas@client2 ~]$ ss -unp
    Recv-Q Send-Q Local Address:Port               Peer Address:Port              
    0      0      10.1.1.3:43889             10.1.1.2:8888                users:(("nc",pid=1494,fd=3))
    ```

* Sur le client1 (3eme shell) : 

    ```
    [slbordas@localhost ~]$ sudo tcpdump -i enp0s8 -w nc-udp.pcap
    tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
    ^C15 packets captured
    15 packets received by filter
    0 packets dropped by kernel
    ```

    [Voir nc-udp.pcap](/TP1/pcap/nc-udp.pcap)

    ![alt text](/TP1/screens/nc-udp.png "nc-udp")

    * Nous voyons des transmission de données faites entre un client et un serveur par le protocole UDP (aucun tunnel).

___
### TCP 

* Sur client1 :

    * Ouverture du port 8888 : 

        ```
        [slbordas@localhost ~]$ sudo firewall-cmd --add-port=8888/tcp --permanent
        [sudo] password for slbordas :
        success

        [slbordas@localhost ~]$ sudo firewall-cmd --reload
        success
        ```

    * On lance netcat pour qu'il écoute sur le port TCP 8888 : 

        ```
        [slbordas@localhost ~]$ nc -l 8888

        ```

* Sur client2 : 

    ```
    [slbordas@client2 ~]$ nc 10.1.1.2 8888

    ```

* Sur client1 (2nd shell) : 

    ```
    [slbordas@localhost ~]$ ss -tnp
    State       Recv-Q Send-Q Local Address:Port               Peer Address:Port              
    ESTAB       0      0      10.1.1.2:8888               10.1.1.3:45716              users:(("nc",pid=1474,fd=5))
    ESTAB       0      0      10.1.1.2:22                 10.1.1.1:50020              
    ESTAB       0      0      10.1.1.2:22                 10.1.1.1:50071  
    ```

* Sur client2 (2nd shell) : 

    ```
    [slbordas@client2 ~]$ ss -tnp
    State       Recv-Q Send-Q Local Address:Port               Peer Address:Port              
    ESTAB       0      0      10.1.1.3:45716             10.1.1.2:8888                users:(("nc",pid=1190,fd=3))
    ESTAB       0      0      10.1.1.3:22                 10.1.1.1:50072              
    ESTAB       0      0      10.1.1.3:22                 10.1.1.1:50021 
    ```

* Sur client1 (3eme shell) : 

    ```
    [slbordas@localhost ~]$ sudo tcpdump -i enp0s8 -w nc-tcp.pcap
    [sudo] Mot de passe de slbordas : 
    tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
    ^C15 packets captured
    15 packets received by filter
    0 packets dropped by kernel
    ```

    [Voir nc-tcp.pcap](/TP1/pcap/nc-tcp.pcap)

    ![alt text](/TP1/screens/nc-tcp.png "nc-tcap")

    * Ici nous avons des requêtes TCP qui passe par un tunnel cette fois-ci et nous avons un 'accusé de réception' a contrario du protocole UDP

___
### Firewall

* Sur client1 :

    * Fermeture du port 8888 UDP : 

        ```
        [slbordas@localhost ~]$ sudo firewall-cmd --remove-port=8888/udp --permanent
        [sudo] password for slbordas :
        success

        [slbordas@localhost ~]$ sudo firewall-cmd --reload
        success
        ```

    * On lance netcat pour qu'il écoute sur le port TCP 8888 : 

        ```
        [slbordas@localhost ~]$ nc -u -l 8888

        ```

* Sur client2 : 

    ```
    [slbordas@client2 ~]$ nc -u 10.1.1.2 8888

    ```

* Sur client1 (2nd shell) : 

    ```
    [slbordas@localhost ~]$ sudo tcpdump -i enp0s8 -w firewall.pcap
    [sudo] Mot de passe de slbordas : 
    tcpdump: listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
    ^C10 packets captured
    10 packets received by filter
    0 packets dropped by kernel
    ```

    [Voir firewall.pcap](/TP1/pcap/firewall.pcap)

    ![alt text](/TP1/screens/firewall.png "nc-tcap")

    * La ligne **10** nous indique que la destination est inaccessible, tout simplement parce que nous avons fermer le port UDP.

# III. Routage statique simple 

* Sur client1 : 

    * Transformation de notre machine en routeur

        ```
        [slbordas@localhost ~]$ sudo sysctl -w net.ipv4.ip_forward=1
        [sudo] Mot de passe de slbordas : 
        net.ipv4.ip_forward = 1
        ```

* Sur client2 : 

    * Ajout d'une route statique sur net2 :

        ```
        [slbordas@client2 ~]$ sudo vi /etc/sysconfig/network-scripts/route-enp0s9
        ```

        ```
        10.1.2.0/30 via 10.1.1.2 dev enp0s9
        ```

        _Route statique permanente_

    * Maintenant nous allons essayer de ping l'ip de l'hôte dans net2 : 

        ```
        [slbordas@client2 ~]$ ping 10.1.2.1
        PING 10.1.2.1 (10.1.2.1) 56(84) bytes of data.
        64 bytes from 10.1.2.1: icmp_seq=1 ttl=63 time=0.595 ms
        64 bytes from 10.1.2.1: icmp_seq=2 ttl=63 time=0.328 ms
        ^C
        --- 10.1.2.1 ping statistics ---
        2 packets transmitted, 2 received, 0% packet loss, time 1001ms
        rtt min/avg/max/mdev = 0.328/0.461/0.595/0.135 ms
        ```

    * Nous allons voir le chemin parcouru par nos paquets : 

        ```
        [slbordas@client2 ~]$ traceroute 10.1.2.1
        traceroute to 10.1.2.1 (10.1.2.1), 30 hops max, 60 byte packets
        1  gateway (10.0.2.2)  0.227 ms  0.205 ms  0.122 ms
        2  10.1.2.1 (10.1.2.1)  0.320 ms  0.243 ms  0.464 ms
        ``` 

        On voit qu'ils passent par la gateway (10.0.2.2) et ensuite notre net2 10.1.2.1.