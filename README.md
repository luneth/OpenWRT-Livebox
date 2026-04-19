# Remplacer une Livebox par un routeur OpenWrt (ipv4,ipv6 & TV).

Guide lourdement basé (pour ainsi dire, copié à 99% sur le travail de ubune (sujet sur [lafibre.info](https://lafibre.info/remplacer-livebox/remplacement-de-la-livebox-par-un-routeur-openwrt-18-dhcp-v4v6-tv) & Github [repo](https://github.com/ubune/openwrt-livebox) ), [chat_roux](https://lafibre.info/remplacer-livebox/remplacement-de-la-livebox-par-un-routeur-openwrt-18-dhcp-v4v6-tv/msg1147909/#msg1147909) (aka golfromeo-fr) (Github [repo](https://github.com/golfromeo-fr/openwrt-turris-omnia-orange-france) ), [l'expertise](https://lafibre.info/remplacer-livebox/durcissement-du-controle-de-loption-9011-et-de-la-conformite-protocolaire/) de levieuxatorange et des differents posts sur [lafibre.info](https://lafibre.info/index.php)

Le but étant de centraliser toutes ces informations à un seul endroit.


**Prerequis :**  
  
Avoir un ONT Orange ou un ONT perso bien connecté/auth avec l'OLT, Leox LXT-010H-D, fs.com GPON-ONU-34-20BI, etc…  
Avoir récupéré ses identifiants FTI.  
Un routeur compatible openwrt et avec assez de perf pour gérer le débit de votre offre.  
  
**Informations sur Openwrt =>**  
  
Pour récupérer votre version, il faut identifier le soc de votre routeur, exemple sur un ubiquiti edgerouterx => <https://openwrt.org/toh/hwdata/ubiquiti/ubiquiti_edgerouter_x>  
On retrouve dans target : ramips et subtarget : mt7621.  
Une fois ces éléments identifiés, on peut aller sur <https://downloads.openwrt.org> pour récupérer la version correspondante.  
  
**Présentation rapide de l'architecture Openwrt :**  
  
Sur Openwrt, vos fichiers de configurations se trouvent dans **/etc/config**, par exemple par défaut on retrouve :  
**/etc/config/network** pour la configuration des interfaces réseau, des routes statiques...  
**/etc/config/Firewall** pour la configuration du parefeu, du nat, des zones etc.  
**/etc/config/dhcp** pour la config des serveurs dhcp/dhcpv6 et les annonces RA ipv6...).

Nativement, l'interface Wan est en dhcp (client) et l'interface Lan (ou br-lan) est configurée en 192.168.1.1/24 en dhcp (serveur) et avec annonce d'un préfixe ULA, le routeur est joignable en ssh/https  
user **root** mdp vide.  
  
  
# __PARTIE 1 : INTERNET__
  
Récupérez votre login/pass FTI, et direction <https://jsfiddle.net/kgersen/3mnsc6wy/> (Merci !) pour générer votre option 90.  
Dans ce tuto, nous simulons le login suivant :  
fti/qpq8888  
12345622

Qui donne :

`00000000000000000000001a090000055801034101116674692F6674692F717071383838383c1231323334353637383930313233343536031341302f6f2d83fc857d7829d65ddea775d7`  

Pour remplacer la livebox par votre propre routeur nous avons besoin :  
  
- De votre option 90 générée grâce au script de kgersen.  
- De configurer l'interface Wan en utilisant le VLAN 832 en dhcp => **/etc/config/network**  
- D'envoyer les bonnes options dhcp permettant l'authentification auprès d'orange  => **/etc/config/network**  
- D'envoyer tout le flux qui ne passe pas par nftables en priorité L2 (802.1p) CS6 => **/etc/config/network (egress mapping)**   
- Gérer les files pour éviter les pbs de débit (on remet bien notre traffic internet avec des priorités cohérentes) => **nft rules**  
- De l'adresse mac de votre livebox (ou d'une adresse mac fixe, qui sera associé à l'interface vlan 832 **__et__** aux options "client ID" envoyées.  
  
  
**nano /etc/config/network**  
  
Dans le fichier network on doit retrouver :

- Config Lan :
```
config interface 'lan'`
        option proto 'static'`
        option ipaddr '192.168.1.1'`
        option netmask '255.255.255.0'`
        option ip6assign '64'
        option ip6ifaceid '::bad'
        option device 'br-lan'
```

- Config eth0.832 avec mapping :  
  
*Attention, n'hésitez pas à creer la sous interface wan "vlan832" depuis l'interface web (luci) et ensuite d'aller voir votre fichier network, car en fonction du modèle de routeur ce n'est pas exactement le même nom d'interface (intf physique).*  
*Dans notre cas, le wan du routeur est "eth0", mais chez certains, il se nomme "wan" ou "eth1".*
```
config device
        option type '8021q'
        option ifname 'eth0'
        option vid '832'
        option name 'eth0.832'
        list egress_qos_mapping '6:6'
        option macaddr 'A2:34:56:78:19:26' # A remplacer par l'adresse mac de votre livebox, ne pas oublier le client id plus bas qui doit avoir la meme valeur que l'adresse mac de votre interface wan !
```
  
- Config Wan ipv4 :
```
config interface 'wan4'
        option proto 'dhcp'
        option broadcast '1'
        option hostname 'livebox'
        option vendorid 'sagem'
        option userclass 'FSVDSL_livebox.Internet.softathome.livebox6'
        option reqopts '1 3 6 15 28 51 58 59 90 119 120 125'
        option macaddr 'A2:34:56:78:19:26' # Remplacer par l'adresse mac sans les : de votre interface vlan
        option auto '1'
        list sendopts '61:01A23456781926' # Remplacer par 01 + l'adresse mac sans les : de votre interface vlan
        list sendopts '12:6c697665626f78'# Hostname: 6c697665626f78 = livebox
        list sendopts '60:736167656d' # Vendor Class: 736167656d = sagem
        list sendopts '77:2b46535644534c5f6c697665626f782e496e7465726e65742e736f66746174686f6d652e4c697665626f7836' # User Class: 2b46535644534c5f6c697665626f782e496e7465726e65742e736f66746174686f6d652e4c697665626f7836 = FSVDSL_livebox.Internet.softathome.livebox6
        list sendopts '90:00000000000000000000001a090000055801034101116674692F6674692F717071383838383c1231323334353637383930313233343536031341302f6f2d83fc857d7829d65ddea775d7' # XAUT: à remplacer par votre chaine générée précédemment
        option force_link '1'
        option device 'eth0.832' # A remplacer par votre interface
        option norelease '1' 
        option clientid '01A23456781926' # Doit correspondre à 01 + l'adresse mac sans les : de votre interface vlan 832 !
```
- Config Wan ipv6
```
config interface 'wan6'
        option proto 'dhcpv6'
        option auto '0'
        option reqaddress 'none'
        option reqprefix 'auto'
        option defaultreqopts '0'
        list sendopts '15:FSVDSL_livebox.Internet.softathome.livebox6' # User Class
        list sendopts '16:0000040e0005736167656d' # Vendor Class: 0000040e0005736167656d = sagem
        list sendopts '17:000005580006000e495056365f524551554553544544' # IPv6_REQUESTED
        list sendopts '11:00000000000000000000001a090000055801034101116674692F6674692F717071383838383c1231323334353637383930313233343536031341302f6f2d83fc857d7829d65ddea775d7' # XAUT: à remplacer par votre chaine générée précédemment
        option reqopts '11 17 23 24'
        option noclientfqdn '1'
        option noacceptreconfig '1'
        option iface_dslite '0'
        option device 'eth0.832' # A remplacer par votre interface
        option dscp '6'
        option skpriority '6'
        option clientid '00030001A23456781926' # Doit correspondre à 00030001 + l'adresse mac sans les : de votre interface vlan 832 !
```

NFT rules, on créé un fichier contenant nos règles (pour le remapping l2 des flux), qui sera lancé en même temps que le firewall

**Installez kmod-nft-netdev:**
```
apk update && apk install kmod-nft-netdev
```
*Pensez à modifier “type filter hook egress device "eth0.832" priority 0; policy accept;” en fonction de votre interface*

**nano /etc/nftables.d/orange-prio.include**

```
table netdev orange-rules
flush table netdev orange-rules

table netdev orange-rules {
        chain orange-rules-chain {
                type filter hook egress device "eth0.832" priority 0; policy accept; 
                vlan type ip6 udp dport 547 vlan pcp set 6 ip6 dscp set cs6 counter accept
                vlan type ip6 icmpv6 type { echo-request, echo-reply, nd-neighbor-solicit, nd-neighbor-advert, nd-router-solicit } vlan pcp set 6 ip6 dscp set cs6 counter accept
                vlan type ip6 ip6 dscp set cs0 counter accept
                vlan type ip udp dport 67 vlan pcp set 6 ip dscp set cs6 counter accept
                vlan type ip ip dscp set cs0 counter accept
                vlan type arp vlan pcp set 6 counter accept
        }
}
```

  
  
**nano /etc/config/firewall**

```
config include 'orange_rules'
        option enabled '1'
        option type 'nftables'
        option path '/etc/nftables.d/orange-prio.include'
        option position 'ruleset-post'
```

Zone Wan
```
config zone
	option name 'wan'
	option output 'ACCEPT'
	option family 'ipv4' ## Zone ipv4 only pour ne pas mélanger les règles sur les deux mondes
	list network 'wan4'
	option forward 'DROP'
	option masq '1' ## On remplace l'ip source "privée" des flux sortant sur internet par l'ip publique reçu sur le wan4.
	option input 'DROP'
```
Zone Wan6
```
config zone
	option name 'wan6'
	option input 'DROP'
	option output 'ACCEPT'
	option forward 'DROP'
	option family 'ipv6' ## Zone ipv6 only pour ne pas mélanger les règles sur les deux mondes
	list network 'wan6'
	list device 'eth0.832'
```
Config des Rules :
```
config rule
	option name 'Allow-DHCP-Renew'
	option src 'wan'
	option proto 'udp'
	option dest_port '68'
	option target 'ACCEPT'
	option family 'ipv4'

config rule
	option name 'Allow-Ping'
	option src 'wan'
	option proto 'icmp'
	option icmp_type 'echo-request'
	option family 'ipv4'
	option target 'ACCEPT'

config rule
	option name 'Allow-IGMP'
	option src 'wan'
	option proto 'igmp'
	option family 'ipv4'
	option target 'ACCEPT'

config rule
	option name 'Allow-DHCPv6'
	option proto 'udp'
	option dest_port '546'
	option family 'ipv6'
	option target 'ACCEPT'
	option src 'wan6'
	list src_ip 'fc00::/6'
	list dest_ip 'fc00::/6'

config rule
	option name 'Allow-MLD'
	option proto 'icmp'
	option family 'ipv6'
	option target 'ACCEPT'
	option src 'wan6'
	list src_ip 'fe80::/10'

config rule
	option name 'Allow-ICMPv6-Input'
	option proto 'icmp'
	option limit '1000/sec'
	option family 'ipv6'
	option target 'ACCEPT'
	list icmp_type 'bad-header'
	list icmp_type 'destination-unreachable'
	list icmp_type 'echo-reply'
	list icmp_type 'echo-request'
	list icmp_type 'neighbour-advertisement'
	list icmp_type 'neighbour-solicitation'
	list icmp_type 'packet-too-big'
	list icmp_type 'router-advertisement'
	list icmp_type 'router-solicitation'
	list icmp_type 'time-exceeded'
	list icmp_type 'unknown-header-type'
	option src 'wan6'

config rule
	option name 'Allow-ICMPv6-Forward'
	option dest '*'
	option proto 'icmp'
	option limit '1000/sec'
	option family 'ipv6'
	option target 'ACCEPT'
	list icmp_type 'bad-header'
	list icmp_type 'destination-unreachable'
	list icmp_type 'echo-reply'
	list icmp_type 'echo-request'
	list icmp_type 'packet-too-big'
	list icmp_type 'time-exceeded'
    list icmp_type 'parameter-problem'
	list icmp_type 'unknown-header-type'
	option src 'wan6'  
```
Config Forwarding :
```
config forwarding
	option src 'lan'
	option dest 'wan'

config forwarding
	option src 'lan'
	option dest 'wan6'
```

DHCPv4 + RA (ipv6, Router Advertisement), pour le LAN:

**nano /etc/config/dhcp**  
*Ici on ne fait pas de dhcpv6, uniquement du RA, on annonce le premier /64 du /56 au lan (pour l'autoconfiguration des machines) + serveur dhcpv4.*
```
config dhcp 'lan'
	option interface 'lan'
	option start '100'
	option limit '59'
	option leasetime '12h'
	option dhcpv4 'server'
	option ra 'server'
	list ra_flags 'none'
```

Une fois tout ceci fait, vous pouvez supprimer votre livebox et connecter directement votre nouveau routeur openwrt sur l'ont orange. Vous devriez recevoir votre ip publique ainsi que votre /56 sur l'eth0.832.  
  
Commandes utiles pour vérifier tout ça  :  
**ifstatus wan4**  
**ifstatus wan6** 

# __PARTIE 2 : Auto-gen de l'option 90/11 & Heathchecks__
## Génération automatique de l'option 90

Même si pour certains (moi y compris) garder l'option 90 générée au début du tutoriel ne pose pas de soucis, l'identification auprès d'orange est sensée être re-générée régulièrement.  

*Modifiez Login et PASSWORD avec vos identifiants*

**nano /etc/config/orange-auth**

```
# Orange FTI/Livebox credentials - AUTH renews automatically when orange-gen-auth.sh runs
LOGIN="abcdefg" # without fti/
PASSWORD="motdepasse"
BYTE="A"
SALT_PREFIX="1234567890123"   # first 13 chars fixed
```

**nano /etc/config/orange-gen-auth.sh**

```
#!/bin/sh

# Orange France FTI/Livebox authentication script
# IMPORTANT: Orange rejects AUTH strings used for >5-8 weeks - run this periodically to renew
# Can be triggered by: /etc/init.d/orange-auth (if enabled) or hotplug on wan ifdown

# run with debug
# sh -x orange-gen-auth.sh

. /etc/config/orange-auth

SALT_PREFIX="1234567890123"
SALT_SUFFIX=$(cat /dev/urandom | tr -dc '0-9' | head -c 3)
SALT="${SALT_PREFIX}${SALT_SUFFIX}"

str_hex() {
    echo -n "$1" | hexdump -v -e '/1 "%02X:"' | sed 's/:$//'
}

tl() {
    printf "%s:%02X" "$1" "$2"
}

ORANGE="fti/${LOGIN}"
ORANGE_LEN=$(echo -n "$ORANGE" | wc -c)
SALT_LEN=16

MD5S=$(printf '%s' "${BYTE}${PASSWORD}${SALT}" \
    | md5sum | cut -d' ' -f1 \
    | sed 's/../&:/g;s/:$//' \
    | tr 'a-f' 'A-F')

AUTH="00:00:00:00:00:00:00:00:00:00:00\
:1A:09:00:00:05:58:01:03:41\
:$(tl 01 $((2+ORANGE_LEN))):$(str_hex "$ORANGE")\
:$(tl 3C $((2+SALT_LEN))):$(str_hex "$SALT")\
:$(tl 03 $((2+1+SALT_LEN))):$(str_hex "$BYTE"):${MD5S}"

# Remove colons and swap uppercase > lowercase
AUTH=$(printf '%s' "$AUTH" | tr 'A-Z' 'a-z' | tr -d ':')

# Remove only the auth entries, keep everything else
uci -q del_list network.wan4.sendopts="$(uci get network.wan4.sendopts | tr ' ' '\n' | grep '^90:')"
uci add_list network.wan4.sendopts="90:${AUTH}"

uci -q del_list network.wan6.sendopts="$(uci get network.wan6.sendopts | tr ' ' '\n' | grep '^11:')"
uci add_list network.wan6.sendopts="11:${AUTH}"

uci commit network


logger -t orange-auth "Auth applied salt=${SALT} len=$(echo -n "$AUTH" | wc -c)"
```

**nano /etc/config/orange-auth-init.sh**

```
#!/bin/sh /etc/rc.common

# Init script to generate AUTH on boot - renews auth to avoid Orange expiry

# integration in openwrt 25.12
# ln -s /etc/config/orange-auth-init.sh /etc/init.d/orange-auth
# /etc/init.d/orange-auth enable

START=09

start() {
    /etc/config/orange-gen-auth.sh
}
```

**nano /etc/config/97-orange-config**
```
#!/bin/sh

# Orange hotplug script - renews AUTH on wan ifdown to avoid Orange rejecting old auth

# integration in openwrt 25.12
# ln -s /etc/config/97-orange-config /etc/hotplug.d/iface/97-orange-config

case "$ACTION/$INTERFACE" in
    ifdown/wan4)
        logger -t orange-config "wan4 down, pre-generating auth for next ifup"
        /etc/config/orange-gen-auth.sh
        ifdown wan6
        ;;
esac
```

**nano /etc/config/99-wan6-delay**
```
#!/bin/sh

# Delays wan6 ifup until wan DHCP completes - prevents wan6 failure on Orange network

# integration in openwrt 25.12
# ln -s /etc/config/99-wan6-delay /etc/hotplug.d/iface/99-wan6-delay

case "$ACTION/$INTERFACE" in
    ifup/wan4)
        logger -t wan6-delay "wan DHCPv4 up >>>>>>>>>>>>>>>>>>>> sleeping before wan6 <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<"
        sleep 3
        ifup wan6
        ;;
    ifdown/wan4)
        ifdown wan6
        ;;
esac
```

Activer les service:

```
chmod +x /etc/config/orange-auth-init.sh /etc/config/orange-gen-auth.sh /etc/config/97-orange-config /etc/config/99-wan6-delay
ln -s /etc/config/97-orange-config /etc/hotplug.d/iface/97-orange-config
ln -s /etc/config/99-wan6-delay /etc/hotplug.d/iface/99-wan6-delay
ln -s /etc/config/orange-auth-init.sh /etc/init.d/orange-auth
/etc/init.d/orange-auth enable
```

## Tests de vie

Comme expliqué [ici](https://lafibre.info/remplacer-livebox/durcissement-du-controle-de-loption-9011-et-de-la-conformite-protocolaire/), il est recommandé de vérifier l'état de votre connections. Le script suivant suit exactement ces recommandations:  
- IPv4 : faire une séquence ARP Request / Reply vers l'adresse du routeur donné en DHCPv4  
- IPv6 : faire une séquence ICMP6 NS/NA de fe80::ba0:bab  
- Pour chacun des deux stack  
  - faire une séquence toutes les 120s  
  - en cas de non réponse au bout de 10s , faire 2 répétitions  
  - au 3ème timeout (donc au total 150s de timeout), considérer que la liaison est en échec  
relancer CE stack

*Penser à modifier DEV=”eth0.832” en fonction de votre interface*
**nano /etc/config/wan-watchdog.sh**

```
#!/bin/sh

DEV="eth0.832"
IF4="wan4"
IF6="wan6"


MAX_RETRY=3
TIMEOUT=10

get_gw4() {
    ubus call network.interface.$IF4 status | jsonfilter -e '@["route"][0].nexthop'
}

get_gw6() {
    ubus call network.interface.wan6 status | jsonfilter -e '@.route[@.target="::" && @.mask=0].nexthop'
}

check_ipv4() {
    GWv4=$(get_gw4)
    [ -z "$GWv4" ] && return 1

    COUNT=0
    while [ $COUNT -lt $MAX_RETRY ]; do
        arping -I $DEV -c 1 -w $TIMEOUT $GWv4 >/dev/null 2>&1 && return 0
        COUNT=$((COUNT+1))
    done
    return 1
}

check_ipv6() {
    GWv6=$(get_gw6)
    [ -z "$GWv6" ] && return 1

    GWv6="${GWv6}%${DEV}"

    COUNT=0
    while [ $COUNT -lt $MAX_RETRY ]; do
        ping6 -c 1 -W $TIMEOUT $GWv6 >/dev/null 2>&1 && return 0
        COUNT=$((COUNT+1))
    done
    return 1
}

restart_ipv4() {
    logger -t wan-watchdog "wan4 failure > restarting"
    ifdown $IF4
    sleep 3
    ifup $IF4
}

restart_ipv6() {
    logger -t wan-watchdog "wan6 failure > restarting"
    ifdown $IF6
    sleep 3
    ifup $IF6
}

while true; do
    check_ipv4 || restart_ipv4
    check_ipv6 || restart_ipv6
    sleep 120
done
```

**nano /etc/config/wan-watchdog-init.sh**

```
#!/bin/sh /etc/rc.common

# Init script to check for upstream connectivity

# integration in openwrt 25.02
# ln -s /etc/config/wan-watchdog-init.sh /etc/init.d/wan-watchdog
# /etc/init.d/wan-watchdog enable
START=99
USE_PROCD=1

start_service() {
    procd_open_instance
    procd_set_param command /etc/config/wan-watchdog.sh
    procd_set_param respawn
    procd_set_param respawn 3600 5 5
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}
```

Activer le service:

```
chmod +x /etc/config/wan-watchdog.sh /etc/config/wan-watchdog-init.sh
ln -s /etc/config/wan-watchdog-init.sh /etc/init.d/wan-watchdog
/etc/init.d/wan-watchdog enable
``` 
# __PARTIE 3 : Télévision__
Installation d'igmp proxy :
```
apk update && apk install igmpproxy
```

On modifie le fichier /etc/config/igmpproxy
```
config igmpproxy
	option quickleave 1
#	option verbose [0-3](none, minimal[default], more, maximum)

config phyint
    option network tvorange
    option zone wantv
    option direction upstream
        list altnet "0.0.0.0/0"

config phyint lan
    option network lan
    option zone lan
    option direction downstream
```

On modifie le fichier /etc/config/network pour créer l'interface vlan 840, et ajouter l'igmp snooping sur le lan.
```
config device
        option name 'eth0.840'
        option type '8021q'
        option ifname 'eth0'
        option vid '840'
        list egress_qos_mapping '0:5'
        list egress_qos_mapping '1:5'
        list egress_qos_mapping '2:5'
        list egress_qos_mapping '3:5'
        list egress_qos_mapping '4:5'
        list egress_qos_mapping '5:5'
        list egress_qos_mapping '6:5'
        list egress_qos_mapping '7:5'

config interface 'tvorange'
        option device 'eth0.840'
        option proto 'static'
        option ipaddr '192.168.255.254'
        option netmask '255.255.255.255'
        option delegate '0'
```

Toujours dans /etc/config/network
On ajoute à la conf lan la ligne suivante :
```
option igmp_snooping '1'
```

On modifie le fichier /etc/config/dhcp pour envoyer les dns d'orange, et l'option 125 :

Pour les "XX", convertissez votre numéro de serie de la livebox tv (ascii vers hex)
Exemple : IA2022323438110 devient 494132303232333233343338313130

Pour les "YY", on prend les 3 premiers octects de la mac, à convertir en Hex également.
Exemple : 08:87:C6:B2:D1:90 soit 0887C6 devient 303838374336
Cet exemple, donnerait : '125,00:00:0d:e9:24:04:06:30:38:38:37:43:36:05:0f:49:41:32:30:32:32:33:32:33:34:33:38:31:31:30:06:09:4c:69:76:65:62:6f:78:20:34'
```
config dhcp 'lan'
	option interface 'lan'
	option start '101'
	option limit '150'
	option leasetime '12h'
	option dhcpv4 'server'
	option ra 'server'
	list ra_flags 'none'
        list dhcp_option '6,81.253.149.10,80.10.246.3'
        list dhcp_option '15,orange.fr'
        list dhcp_option '125,00:00:0d:e9:24:04:06:YY:YY:YY:YY:YY:YY:05:0f:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:XX:06:09:4c:69:76:65:62:6f:78:20:34'
```

Et pour finir, on modifie le fichier /etc/config/firewall pour créer la zone WanTV et les règles associées :
```
config zone
	option name 'wantv'
	option output 'ACCEPT'
	option masq '1'
	option network 'tvorange'
	option input 'DROP'
	option forward 'DROP'
	option family 'ipv4'
	list device 'eth0.840'

config forwarding
	option src 'lan'
	option dest 'wantv'

config rule
	option target 'ACCEPT'
	option name 'igmp'
	option family 'ipv4'
	option proto 'igmp'
	option src 'wantv'

config rule
	option target 'ACCEPT'
	option name 'multicast'
	option family 'ipv4'
	option proto 'udp'
	option src 'wantv'
	option dest 'lan'
	option dest_ip '224.0.0.0/4'
```
