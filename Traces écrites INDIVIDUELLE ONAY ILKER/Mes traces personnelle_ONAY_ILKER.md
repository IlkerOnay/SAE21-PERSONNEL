# <ins> SAE 21 : PARTIE INDIVIDUELLE

Dans ce document je vais vous montrez ma partie individuelle du travail. (J'ai fait toute la SAE21 seul)
<br> Nous allons voir comment j'ai déclarer ou crée mais serveur ect.. et à la fin voir les résultat ping commande qui permet de voir le bon fonctionnement ect ...
***

## <ins> GNS 3 :
***
<ins> Mon GNS3 :

<img src=".\screen1\conf_switch_vlan.png">

***

### <ins>Serveur interne et externe:

Le serveur interne est un simple server apache2, pour le mettre en place il faut installer avec la commande suivante :
``apt-get install apache2``

Une fois installer on se rend dans le fichier html : ``cd /var/www/html`` et on place notre dossier qui contient le serveur soit le html, le css , et les images.

<img src='.\screen1\emplacement_serveur_interne.png'>

Par la suite on se rend dans le fichier de configuration par défaut : ``cd /etc/apache2/sites-enabled/000-default.conf`` et on modifie une ligne pour indiquer le chemin de notre dossier, on modifie : `` DocumentRoot /var/www/html/SAE-WEB-INTERNE ``

<img src='.\screen1\conf_def_interne.png'>

<br>

Maintenant que le serveur est crée il ne reste plus cas le lancer : `` service apache2 start ``<br>
Il ne reste plus cas faire ça sur le pc du server externe.
***

### <ins> DHCP :

Le DHCP va directement être fait sur le routeur cisco gns3 :

Les commandes sont hyper simple a déclarer dans mon cas j'ai fait comme ceci :
> D'abord on déclare notre pool, puis lui attribut un nom (ici 10,20,30 et 40 correspondant a mes vlans), puis on dit a quelle network ça correspond, on peut indiquer l'ip du DNS ici sera le notre et le router par défault donc sa gateway.
````
!
ip dhcp pool 10
   network 192.168.10.0 255.255.255.0
   dns-server 192.168.2.248
   default-router 192.168.10.254 
!
ip dhcp pool 20
   network 192.168.20.0 255.255.255.0
   dns-server 192.168.2.248
   default-router 192.168.20.254 
!
ip dhcp pool 30
   network 192.168.30.0 255.255.255.0
   dns-server 192.168.2.248
   default-router 192.168.30.254 255.255.255.0 
!
ip dhcp pool 40
   network 192.168.40.0 255.255.255.0
   dns-server 192.168.2.248
   default-router 192.168.40.254 
!
````

Ici on peut voir que ça marche pour nous : (il faut d'abord déclarer nos vlans, interface ect...)

<img src='.\screen1\dhclient.png'>


***

### <ins> VLAN :

On va voir la déclaration des vlan et de leur ACL

Sur notre switch dans un premier temps nous allons déterminer quel port correspond a quel VLAN pour ce faire avant de commencer notre GNS3, il faut conceptualiser notre serveur sur une feuille ou alors avec GNS3 en écrivant bien quel partie appartient a quel réseaux ect ...

<ins>Voici mes vlans sur mon switch :

> Ils sont déclarer en interface graphique car j'avais pas accès sur GNS3 à la console du switch
<img src=".\screen1\conf_switch_vlan.png">

<ins>Sur mon routeur leur déclaration est la suivante :
>J'ai mis dans les interfaces l'encapsulation dot1Q . Il permet de répandre plusieurs VLAN sur un lien,
J'ai aussi mis les interfaces de chacun en .254 en /24, puis déclarer leur groupe d'acces en entrer et en sortie sachant que pour chaque VLAN L'entrée et la sortie sera pour chacun le même. Mais aussi j'ai déclarer les nat mais on verra ça plus tard.
````
!
interface FastEthernet0/0
 ip address 192.168.1.254 255.255.255.0
 duplex auto
 speed auto
!
interface FastEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.254 255.255.255.0
 ip access-group VLAN10ADMIN in
 ip access-group VLAN10ADMIN out
 ip nat inside
 ip virtual-reassembly
!
interface FastEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.254 255.255.255.0
 ip access-group VLAN20SI in
 ip access-group VLAN20SI out
 ip nat inside
 ip virtual-reassembly
!
interface FastEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.254 255.255.255.0
 ip access-group VLAN30COM in
 ip access-group VLAN30COM out
 ip nat inside
 ip virtual-reassembly
!
interface FastEthernet0/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.254 255.255.255.0
 ip access-group VLAN40SERV in
 ip access-group VLAN40SERV out
 ip nat inside
 ip virtual-reassembly
!
interface FastEthernet0/1
 ip address 192.168.2.254 255.255.255.0
 ip nat outside
 ip virtual-reassembly
 duplex auto
 speed auto
!
````

<ins>Les groupes d'accès :

Pour expliquez dans les grandes lignes certaines de nos vlans devront pouvoir être accessible par ssh depuis les SI donc on autorise le port ssh donc le 22, d'autre devront avoir accès à internet donc on va autoriser de l'udp et tcp (par exemple 443 le html), et certaines VLAN ne devront pas avoir accès à d'autre c'est la ou entre en compte les deny ip ou on refuse les ip par exemple de la 30 vers la 10 et on laisse passer les bootps et bootpc pour tous qui correspond au DHCP.... et on appliquera nos règles ACL dans nos interface FastEthernet de nos VLANS.

>VLAN 10 
````
ip access-list extended VLAN10ADMIN
 permit udp any any eq bootps
 permit udp any any eq bootpc
 permit ip host 192.168.10.1 any
 permit tcp 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255 eq 22
 deny   ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit tcp any any eq www
 permit tcp any any eq 443
 permit icmp any any
 permit ip any any
 permit tcp 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 established
 permit tcp any any eq 22
````

>VLAN 20
`````
ip access-list extended VLAN20SI
 permit udp any any eq bootps
 permit udp any any eq bootpc
 permit ip host 192.168.20.1 any
 permit tcp 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255 eq 22 established
 permit tcp 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 eq 22 established
 permit tcp 192.168.10.0 0.0.0.255 any established
 permit tcp 192.168.30.0 0.0.0.255 any established
 permit tcp any any eq www
 permit tcp any any eq 443
 permit icmp any any
 permit ip any any
 permit tcp any any eq 22
`````

>VLAN 30

````
ip access-list extended VLAN30COM
 permit udp any any eq bootps
 permit udp any any eq bootpc
 permit ip host 192.168.30.1 any
 permit tcp 192.168.20.0 0.0.0.255 192.168.30.0 0.0.0.255 eq 22
 deny   ip 192.168.10.0 0.0.0.255 192.168.30.0 0.0.0.255
 permit tcp any any eq www
 permit tcp any any eq 443
 permit icmp any any
 permit ip any any
 permit tcp 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255 established
 permit tcp any any eq 22
````

>VLAN 40

````
ip access-list extended VLAN40SERV
 permit udp any any eq bootps
 permit udp any any eq bootpc
 permit ip host 192.168.40.1 any
 permit tcp 192.168.10.0 0.0.0.255 any eq www
 permit tcp 192.168.20.0 0.0.0.255 any eq www
 permit tcp 192.168.30.0 0.0.0.255 any eq www
 permit icmp any any
 permit ip any any
 permit tcp any any established
````


***

### <ins> NAT :

Comme sont nom l'indique Network address translation, il permet la traduction d'ip d'un réseaux à un autre.

Comme on a vue avant on à déclarer sur nos interfaces routeur la partie interne (nos vlans) et déclarer en ``ip nat inside`` et la partie externe en ``ip nat outside`` (FastEthernet 0/1)

Par la suite nous avons dû crée nos access-list pour notre NAT:
>1,2,3 et 4 correspondent au vlan 10,20,30 et 40

````
access-list 1 permit 192.168.10.0 0.0.0.255
access-list 2 permit 192.168.20.0 0.0.0.255
access-list 3 permit 192.168.30.0 0.0.0.255
access-list 4 permit 192.168.40.0 0.0.0.255
````

En suite on crée une pool d'adresse en sortie de notre routeur :
> Mon routeur sort sur une adresse de réseaux 192.168.2.0/24
après on déclare que nos nat inside exemple :l'acces-list 1 a pour POOL d'adressage 'pool1' (pool1= pool d'address de 192.168.2.0 à 2.252 sachant que 0 pas disponible)
On applique la même pool pour tous car ils n'y a qu'une seul sortie

`````
ip nat pool pool1 192.168.2.0 192.168.2.252 netmask 255.255.255.0
ip nat inside source list 1 pool pool1
ip nat inside source list 2 pool pool1
ip nat inside source list 3 pool pool1
ip nat inside source list 4 pool pool1
`````

***

### <ins> SSH:

Pour le ssh, il nous reste une dernière manipulation à faire il faut se rendre sur tout les pc et server gns3et faire ``nano /etc/ssh/sshd_config`` pour modifier la ligne PermitRootLogin en ``PermitRootLogin Yes`` car tout nos pc sont par défauts en root lors de la connections.

<img src='.\screen1\sshd.png'>

***

### <ins> Firewall :

Sur notre routeur mikrotik il faut configurer notre pare-feu (FireWall).

Pour ce faire voilà la configuration de notre routeur :

>Il est en 192.168.2.253, il a un DHCP , et le NAT automatique

<img src='.\screen1\quick_set_mikrotik.png'>

Notre firewall graphique correspond à ça :

<img src=".\screen1\firewall_mikrotik_graphic.png">

>Nos commandes firewall :
Notre firewall drop par défaut les nouvelles connections.<br>
Et accepte les connections déjà établie.<br>
Il accepte les ports tcp 80 et 443 pour l'accès à 'internet' au site.<br>
Le port UDP 1194 est celui par défaut pour openVPN. <br>
On autorise l'icmp pour le ping <br>
Et on autorise le port 53 UDP pour le dns <br> (pour les connections des pc de l'IUT au site externe sae.stark.fr)

````
/ip firewall filter

add action=accept chain=forward connection-state=established

add action=accept chain=forward connection-state=new dst-port=80 protocol=tcp

add action=accept chain=forward connection-state=new dst-port=443 protocol=tcp

add action=accept chain=forward connection-state=new port=1194 protocol=udp

add action=accept chain=forward connection-state=established,new protocol=icmp

add action=accept chain=forward connection-state=new port=53 protocol=udp

add action=drop chain=forward connection-state=new
````

****

### <ins> DNS:

Voici comment j'ai déclaré mon serveur dns.

Dans un premier temps il faut ``apt-get purge bind9`` pour supprimer tous les configuration si d'autre groupes sont passer avant puis ``apt-get install bind9``.

Par la suite faut se rendre dans ``/etc/bind/`` puis éditer les textes ``named.conf.local ; named.conf.options ; db.sae.stark.fr ``


<ins>named.conf.local :

> On nomme la zone , ici sae.stark.fr et indique le fichier /etc/bind/db.sae.stark.fr et on laisse le reste par défaut.<br> Type correspond à server maître ici, le file correspond au chemin du fichier qui décrit la zone. Ce fichier permet de juste définir quel fichier nous allons utiliser pour décrire notre zone

````
zone "sae.stark.fr" IN {
type master;
file "/etc/bind/db.sae.stark.fr";
allow-update { none; };
allow-query{any;};
};
````

<ins>named.conf.options :

>directory par défault <br>
forwarders on mais 8.8.8.8 cela permet de renvoyer que toutes les résolutions dns que notre dns ne connaît pas il interroge le dns google pour l'apprendre.

````
options {
        directory "/var/cache/bind";
        forwarders {8.8.8.8;};
};
````

<ins>db.sae.stark.fr :

>TTL : durée de validité que le server fourni aux resolver <br>
Serial Le numéro de série est à incrémenter dès qu’une
modification est effectuée<br>
Refresh : Nombre de secondes entre 2 demandes de mise à
jour entre le serveur maitre et esclave<br>
Retry : Nombre de secondes que le serveur esclave attend
avant de ré-émettre une demande si la précédente
a échoué <br>
Expire : Nombre de secondes qu’un serveur attend avant
de considérer une donnée comme indisponible <br>
Negative Cache TTL : obsolète

`````
; sae.stark.fr
$TTL    604800
@       IN      SOA     sae.stark.fr. root.sae.stark.fr. (
                        2               ; Serial
                        604800          ; Refresh
                        86400           ; Retry
                        2419200         ; Expire
                        604800 )        ; Negative Cache TTL
;
$ORIGIN sae.stark.fr.
@       IN      NS      sae.stark.fr.
@       IN      A       192.168.2.248
www     IN      A       192.168.2.248

``````

>$ORIGIN sae.stark.fr. remplie à notre les nom symbolique par exemple :@ par @sae.stark.fr. et www par www.sae.stark.fr.
<br>
On peut lire ici que sae.stark.fr et wwww.sae.stark.fr a pour ip 192.168.2.248

Pour que notre server prend en compte les modifications on peut utiliser la commande ``systemctl reload bind9`` et pour voir ci celui ci fonctionne bien et renvoie pack d'erreur on peut utiliser la commande ``named.checkconf -z``

<img src=".\screen3\db.sae.stark.fr.png">
<img src=".\screen3\named.conf.local.png">
<img src=".\screen3\named.conf.options.png">


Une fois lancer pour vérifier si votre pc à bien pour dns 192.168.2.248 on peut allez voir avec la commande ``nano /etc/resolv.conf`` si il y'a bien marquer ``nameserver 192.168.2.248`` comme l'image ci-dessous :

<img src=".\screen1\resolv dns.png">
<br>
<br>

***


# Résultat

Nous allons maintenant tester nos configuration <br>
(Rappel 192.168.2.248 et l'ip DNS et l'ip du site externe,<br> mais aussi que 192.168.40.1 le serveur web interne et en 40.2 car j'ai utiliser plusieurs fois le dhclient dessus pour les screens)

Commençons par le plus simple par le dhcp :

Je vais faire un ``dhclient`` sur tout les pc GNS3.

<img src=".\screen1\dhclient.png">


***

Les accès :

Les administratifs devront pouvoir accéder à internet et aux serveurs internes et
externe.
Pour voir les accès au sites interne et externe nous allons utiliser la commande ``wget -O- [IP ou lien]`` mais il faut l'installer avec la commande ``apt-get install wget`` celui peut aussi être une preuve d'accès à internet. (On utilise wget car on a pas d'interface graphics)
<br>
( en sachant q'un seul mot change dans le index.html du site interne et externe qui précise web interne ou web externe)<br><br>

<ins>Administratifs :

On observe que notre dns marche en car on peut ping le 8.8.8.8 et google.com<br>
Et on peut aussi ping sae.stark.fr et 192.168.40.2
<img src=".\screen1\admin_ping.png">
<br>
On peut observer ici qu'on a accès au site externe
<img src=".\screen1\admin wget externe.png">
<br>

On peut observer ici qu'on a accès au site interne

<br>
<img src=".\screen1\admin_wget_interne.png">
<br>
On peut observer les ping vers les vlans: <br>-On accède au 20 car celui-ci est laisser ouvert pour le ssh<br>-On a pas accès au 30 car on est pas censer avoir accès à celui-ci selon le cahier des charges <br> -On peut ping le vlan 40 car celui-ci est le server interne et on doit y avoir accès<br>Donc on constate que le VLAN 10 administratifs, répond bien au demande des cachiers des charges.

<img src=".\screen1\admin ping vlan.png">

***

<ins>Les commerciaux : 

Devront pouvoir accéder aux serveurs interne et externes (et internet d'après ce que j'ai compris)

On observe comme tout à l'heure le bon fonctionnement du dns et des accès internet
Avec les ping sae.stark.fr google.com 8.8.8.8 et 192.168.40.2

<img src=".\screen1\com_ping.png">

On observe ici l'accès au web externe

<img src=".\screen1\com wget externe.png">

On observe ici l'accès au web interne

<img src=".\screen1\com wget interne.png">

On peut voir ici encore un accès vers la VLAN 20 qui sont les SI ( ils ont accès a tous en ssh)<br>
On peut aussi voir l'accès à la vlan 40 donc web interne<br>
On peut voir un refus de connection vers la 10 ce qui est normal
<br> On peut dire alors que la configuration du vlan commerciaux correspond bien au cahier des charges
<img src=".\screen1\com ping vlan.png">

<br>
<br>

***

<ins>Le SI:

Doit pouvoir avoir accès à toutes les machines en SSH


Les pings classique comme précédent pour vérifier le dns et l'accès internet
<img src=".\screen1\si ping.png">

On observe un accès au site interne

<img src=".\screen1\si wget interne.png">

On observe un accès au site externe

<img src=".\screen1\si wget externe.png">

On peut voir les pings de SI vers les VLANS:<br>
Et il peut tous les ping donc il a accès à tous 

<img src=".\screen1\si ping vlan.png">

Maintenant nous allons prouver avec le ssh :

On voit ici qu'il arrive bien a ssh la vlan 30

<img src=".\screen1\si ssh com.png">

On voit ici aussi qu'il arrive bien a ssh la vlan 40 et 10

<img src=".\screen1\si ssh server et admin.png">

***

L’entreprise souhaite avoir un serveur web externe qui contiendra une simple page
html statique avec le nom de l’entreprise. Ce serveur sera placé dans la DMZ et sera
accessible à la fois pour les personnes à l’intérieur du réseau comme pour celle à
l’extérieur du réseau (partie IUT).

<ins>Démonstration :

On voit bien que mon pc a une ip de la salle 10.213.8.1 mais on mets sa route par défaut en 10.213.0.230 qui est la pate externe de mon routeur firewall et en lui mettant en dns nameserver 192.168.2.248 (on peut voir qu'il n'y pas de petit * en haut de la page ce qui signifie qu'elle n'est pas en cours de modifications), on a bien accès depuis l'exterieur au sites externe depuis un pc extérieur


<img src=".\screen2\externe.png">


<br><br>
On ne doit pas avoir accès à l'intérieur du réseaux

<ins> Démonstration :

Toujours depuis le même pc voici mes pings, je peut ping sae.stark.fr qui est dans la dmz mais je peut pas ping les vlan à l'intérieur du réseaux

<img src=".\screen2\pc iut ping.png">