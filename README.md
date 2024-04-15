# Introduction
J'ai décidé de créer un cluster de raspberry pi. Le but de ce projet est d'avoir à portée de main un cluster physique pour me former en administration système et en réseau sur un cluster de machines.

Ce repository évoluera et grandira au gré des projets et nouvelles idées : Je vais me concentrer sur la partie systèmes et réseaux pour l'instant. Je garde en tête l'idée originale du projet de tester un cluster MongoDB, ELK ou encore Kubernetes sur plusieurs noeuds.

<img src="./Images_Readme/cluster_metadat_del.JPG" width=800px/>

<br/>

# Coûts et démarche personnelle

Il est à noter que ce projet a un certain coût financier et environnemental : J'ai décidé d'acheter l'ensemble des équipements dont j'avais besoin sur LeBonCoin pour économiser et ne pas acheter inutilement des équipements neufs.

Attention, si les cartes constituent le facteur de coût prépondérant, il faudra tout de même penser aux câbles ethernet, à la/les multiprises éventuelles, au switch, etc ...

Les coûts sont ceux auquels j'ai achété mon matériel sur LeBonCoin.
| Equipements        | Utilité | Nombre      |Commentaires      | Coût unitaire | Coût total |
| ------| ------|-----|-----|-----|-----|
| raspberry pi 4 Go RAM|Evidente|3|Le coût comprend aussi l'alimentation, les boîtiers et les cartes SD 32 Go|61.6€|185€|
| TL-WR902AC|Communiquer avec le routeur principal en mode client |1||8€|8€|
| Archer C50|Routeur principal connecté en filaire avec la box |1||3€|3€|
| TL-SF-1008D 10/100Mbps|Switch pour connecter le routeur client avec les cartes en filaire  |1||3€|3€|
| Câbles ethernet||3||1€|3€|
| Boîte en bois|Stockage |1|Je vous conseille de chercher les boîtes pour bouteilles de vin|5€|5€|

Au total, on atteint environ 210€.

# Réseau

Je conseille une installation qui ne soit pas directement connectée en filaire à une box pour des raisons pratiques.

Pour cela, je conseille un routeur qui soit connecté en filaire à la box et un routeur client connecté en filaire aux cartes. L'intérêt est de pouvoir transporter et d'installer aisèment le cluster où l'on souhaite et d'y avoir accès à distance. 

Evidemment, le débit sera dépendant des performance du routeur client et du switch avec une telle installation.

J'ai gardé le DHCP sur le Archer C50 et j'ai associé une adresse ip fixe pour chaque interface réseau de chaque carte (y compris les interfaces wifi, c'est utile lors du setup pour savoir à qui on s'adresse).

<br/>
<img src="./Images_Readme/reseau_metadat_del.svg" width=600px/>

Notes : Petite erreur, les adresses ip des cartes sont 102, 104, 106.
Il y a écrit TL-SF-1080D mais c'est bien évidemment TL-WR902AC et son adresse ip est X.X.X.100.

Au niveau du TL-WR902AC, il faut se connecter au réseau du routeur principal et passer en mode client :

<img src=".\Images_Readme\Client_configuration_metadat_del.JPG" width=600px/>

<br/>

# Accès à distance 

Il faut activer le SSH lors de la configuration de la carte SD ou dans le menu de configuration (et VNC si nécessaire).

Pour créer et partager votre clé publique depuis un OS windows vers une des cartes pour vous connecter sans mot de passe :

```powershell
ssh-keygen
```

```powershell
type $env:USERPROFILE\.ssh\id_rsa.pub | ssh X.X.X.101 "cat >> .ssh/authorized_keys"
```

## VNC

Pour les releases de TigerVNC :
https://github.com/TigerVNC/tigervnc/releases

<img src="./Images_Readme/TigerVNC_metadat_del.JPG" width=400px />
<img src="./Images_Readme/TigerVNC_2_metadat_del.JPG"  width=400px  />

## RDP

```bash
sudo apt install xrdp
```

Attention, à bien quitter les autres terminaux (VNC et/ou ssh) pour ne pas avoir d'écrans blancs.

Ecran de connexion :

<img src="./Images_Readme/RDP_connection_metadat_del.JPG"  width=600px />
<img src="./Images_Readme/RDP_connection_enter_metadat_del.JPG"  width=600px />

<br>

# LDAP

## Coté serveur 

Installation des packets
```bash
sudo apt-get install slapd ldap-utils
```

Reconfiguration du paquet slapd pour ajouter le nom de domaine et le compte administrateur de l'admin 
```bash
sudo dpkg-reconfigure slapd
```

On peut faire une recherche dans l'annuaire et constater que le nom de domaine clusterpi.org a été choisi :
```bash
linuxperso@raspberrypi:~ $ sudo ldapsearch -Q -L -Y EXTERNAL -H ldapi:/// -b dc=clusterpi,dc=org
version: 1

#
# LDAPv3
# base <dc=clusterpi,dc=org> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# clusterpi.org
dn: dc=clusterpi,dc=org
objectClass: top
objectClass: dcObject
objectClass: organization
o: clusterpi
dc: clusterpi

# search result

# numResponses: 2
# numEntries: 1
```

Il ne reste plus qu'à définir un fichier de configuration ldif contenant les users et groups à rajouter.

Mon fichier ldif contient 2 users et un groupe d'utilisateurs (cluster_users) :
```bash
dn: ou=People,dc=clusterpi,dc=org
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=clusterpi,dc=org
objectClass: organizationalUnit
ou: Groups

dn: cn=cluster_users,ou=Groups,dc=clusterpi,dc=org
objectClass: posixGroup
cn: cluster_users
gidNumber: 5000

dn: uid=patrickl,ou=People,dc=clusterpi,dc=org
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: patrickl
sn: patrickl
givenName: Patrick
cn: patrickl
displayName: Patrick L.
uidNumber: 10000
gidNumber: 5000
userPassword: XXXXXXXXX
gecos: Patrick L.
loginShell: /bin/bash
homeDirectory: /home/ldap/patrickl

dn: uid=rodgerf,ou=People,dc=clusterpi,dc=org
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: rodgerf
sn: rodgerf
givenName: Rodger
cn: rodgerf
displayName: Rodger F.
uidNumber: 10001
gidNumber: 5000
userPassword: XXXXXXXXX
gecos: Rodger F.
loginShell: /bin/bash
homeDirectory: /home/ldap/rodgerf
```

On ajoute ce fichier de configuration à l'aide de la commande ldapadd 
```bash
root@raspberrypi:~# ldapadd -x -D cn=admin,dc=clusterpi,dc=org -W -f tree.ldif 
Enter LDAP Password: 
adding new entry "ou=People,dc=clusterpi,dc=org"

adding new entry "ou=Groups,dc=clusterpi,dc=org"

adding new entry "cn=cluster_users,ou=Groups,dc=clusterpi,dc=org"

adding new entry "uid=patrickl,ou=People,dc=clusterpi,dc=org"

adding new entry "uid=rodgerf,ou=People,dc=clusterpi,dc=org"
```

## Cote client 

J'ai suivi ce tutoriel :
https://www.instructables.com/Make-Raspberry-Pi-do-LDAP-Authentication/

On ajoute le nom de domaine contenant le serveur ldap sur chacune des cartes :
```
echo "X.X.X.10{2/4/6}      clusterpi.org" >> /etc/hosts
``` 

On installe le paquet nécessaire
```
sudo apt-get install libnss-ldapd -y
```

Lors de l'installation du packet, il est demandé le nom de domaine de l'annuaire ainsi que la partie de l'arbre que l'ou souhaite interroger (search base). Cela revient à configurer /etc/nslcd.conf

On édite le fichier /etc/nsswitch.conf qui spécifie les bases de données à interroger selon le service (dans notre cas passwd, group, shadow, gshadow ), c'est à dire les utilisateurs/groupes et l'authentification associée.
```
sudo vim /etc/nsswitch.conf
```

Le fichier ressemble au final à ceci :
```
linuxperso@raspberrypi2:~ $ cat /etc/nsswitch.conf 
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         files ldap
group:          files ldap
shadow:         files ldap
gshadow:        files ldap

hosts:          files mdns4_minimal [NOTFOUND=return] dns
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis
```


On peut maintenant se connecter sur une autre machine avec les users et les groupes dans le ldap :
```
linuxperso@raspberrypi2:~ $ su - rodgerf
Password: 
su: warning: cannot change directory to /home/ldap/rodgerf: No such file or directory

Wi-Fi is currently blocked by rfkill.
Use raspi-config to set the country before use.

rodgerf@raspberrypi2:/home/linuxperso$ 
logout
linuxperso@raspberrypi2:~ $ su - patrickl
Password: 
su: warning: cannot change directory to /home/ldap/patrickl: No such file or directory

Wi-Fi is currently blocked by rfkill.
Use raspi-config to set the country before use.
```
On remarque que les homes des utilisateurs n'existent pas ce qui est normal.

```bash
rodgerf@raspberrypi:/$ id
uid=10001(rodgerf) gid=5000(cluster_users) groups=5000(cluster_users)
```

BUG : Encore non résolu, VNC comme xrdp refuse la création d'un bureau à distance pour les users sur le LDAP (pourtant l'authentification marche bien). Je ne semble pas être le seul à avoir des problèmes à ce niveau.



# Rajout des homes des utilisateurs lors de leur connexion à la plateforme

J'ai acheté un SSD de 512 Go pour avoir des homes partagés sur l'infrastructure. Il est branché en USB à la rasberrypi2. Le but est de partager des partitions aux autres cartes via NFS pour avoir des données centralisées.

Note : On remarque les partitions de boot(512M) et de data(29.2G) qui sont sur la carte SD. 

## Lister disques & partitions 

```bash
linuxperso@raspberrypi2:~ $ lsblk -o model,name,type,fstype,size,label
MODEL                      NAME        TYPE FSTYPE            SIZE LABEL
SAMSUNG YYYYYYYYYYYY-00000 sda         disk ddf_raid_member 476.9G
                           `-sda1      part                  12.4M
                           mmcblk0     disk                  29.7G
                           |-mmcblk0p1 part vfat              512M bootfs
                           `-mmcblk0p2 part ext4             29.2G rootfs
```

## Suppression des données sur le disque dur existant 

Etant donné que le SSD a été acheté à un particulier, je ne me pose pas de questions et je supprime tout immédiatement en réécrivant le disque (ce qui altère la durée de vie du SSD par ailleurs).

```bash
linuxperso@raspberrypi2:~ $ sudo dd status=progress if=/dev/zero of=/dev/sda
512087400960 bytes (512 GB, 477 GiB) copied, 22976 s, 22.3 MB/s
dd: writing to '/dev/sda': No space left on device
1000215217+0 records in
1000215216+0 records out
512110190592 bytes (512 GB, 477 GiB) copied, 22986.5 s, 22.3 MB/s
```

Il n'y a plus de partitions dans /dev/sda 
```bash
linuxperso@raspberrypi2:~ $ lsblk -o model,name,type,fstype,size,label
MODEL                      NAME        TYPE FSTYPE   SIZE LABEL
SAMSUNG YYYYYYYYYYYY-00000 sda         disk        476.9G
                           mmcblk0     disk         29.7G
                           |-mmcblk0p1 part vfat     512M bootfs
                           `-mmcblk0p2 part ext4    29.2G rootfs
```

## Création des partitions

Création de la table de partionnement 


```bash
linuxperso@raspberrypi2:~ $ sudo fdisk /dev/sda
Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier ZZZZZZZZZ.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

Je vais créer une partition pour les homes de 150 GB (139 GiB), la deuxième partition sera dédiée aux données et accesible à tous les utilisateurs autorisés (360 GB).
```bash
linuxperso@raspberrypi2:~ $ sudo fdisk /dev/sda

Welcome to fdisk (util-linux 2.38.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-1000215215, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-1000215215, default 1000215215): +139G

Created a new partition 1 of type 'Linux' and of size 139 GiB.

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2):
First sector (291506176-1000215215, default 291506176):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (291506176-1000215215, default 1000215215):

Created a new partition 2 of type 'Linux' and of size 337.9 GiB.

Command (m for help): p
Disk /dev/sda: 476.94 GiB, 512110190592 bytes, 1000215216 sectors
Disk model: YYYYYYYYYYYY-000
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: ZZZZZZZZZ

Device     Boot     Start        End   Sectors   Size Id Type
/dev/sda1            2048  291506175 291504128   139G 83 Linux
/dev/sda2       291506176 1000215215 708709040 337.9G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

Enfin, on formatte les partitons en ext4
```bash
linuxperso@raspberrypi2:~ $ sudo mkfs -t ext4 -L HOMES /dev/sda1
linuxperso@raspberrypi2:~ $ sudo mkfs -t ext4 -L DATA /dev/sda2
```

On constate les résultats : 
```bash
linuxperso@raspberrypi2:~ $ lsblk -o model,name,type,fstype,size,label
MODEL                      NAME        TYPE FSTYPE   SIZE LABEL
SAMSUNG YYYYYYYYYYYY-00000 sda         disk        476.9G
                           |-sda1      part ext4     139G HOMES
                           `-sda2      part ext4   337.9G DATA
                           mmcblk0     disk         29.7G
                           |-mmcblk0p1 part vfat     512M bootfs
                           `-mmcblk0p2 part ext4    29.2G rootfs
```

## On peut donc maintenant monter les partitions :

super tuto : https://debian-facile.org/doc:systeme:fstab

Pour trouver les PARTUUID :
```bash
linuxperso@raspberrypi2:~ $ sudo blkid
/dev/mmcblk0p1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="EF6E-C078" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="739840f3-01"
/dev/mmcblk0p2: LABEL="rootfs" UUID="4aa56689-dcb4-4759-90e6-179beae559ac" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="739840f3-02"
/dev/sda2: LABEL="DATA" UUID="acad9d66-6fdc-4a2b-920a-2591bee5cb64" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c92f7114-02"
/dev/sda1: LABEL="HOMES" UUID="aca3b0ca-22ae-47fe-abb2-216b4746b26e" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c92f7114-01"
```

Définir les points de montage ( le fichier /etc/fstab permet le montage à chaque reboot).
Il est à noter que les partitions ne peuvent définir des umask, et autres permissions depuis le fichier /etc/fstab, ce qui fait foi ce sont les permissions définies dans le système de fichiers.
```bash
linuxperso@raspberrypi2:/home $ cat /etc/fstab
proc            /proc           proc    defaults          0       0
PARTUUID=739840f3-01  /boot/firmware  vfat    defaults          0       2
PARTUUID=739840f3-02  /               ext4    defaults,noatime  0       1
PARTUUID=c92f7114-02  /data           ext4    defaults          0       2
PARTUUID=c92f7114-01  /home           ext4    defaults          0       2

# a swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
```

Execution des montages dans /etc/fstab et relance du démon systemctl

```bash
linuxperso@raspberrypi2:/data $ sudo mount -a
mount: (hint) your fstab has been modified, but systemd still uses
       the old version; use 'systemctl daemon-reload' to reload.
linuxperso@raspberrypi2:/home $ systemctl daemon-reload
==== AUTHENTICATING FOR org.freedesktop.systemd1.reload-daemon ====
Authentication is required to reload the systemd state.
Authenticating as: ,,, (linuxperso)
Password:
==== AUTHENTICATION COMPLETE ====
```

On peut constater les points de montage /data et /home 
```bash
linuxperso@raspberrypi2:~ $ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 476.9G  0 disk 
|-sda1        8:1    0   139G  0 part /home
`-sda2        8:2    0 337.9G  0 part /data
mmcblk0     179:0    0  29.7G  0 disk 
|-mmcblk0p1 179:1    0   512M  0 part /boot/firmware
`-mmcblk0p2 179:2    0  29.2G  0 part /

```

Il faut maintenant définir la création des homes lors de l'authentification des users (si le home n'existe pas ).

Il faut rajoute la ligne suivante dans le fichier /etc/pam.d/common-session
```bash
session required        pam_mkhomedir.so skel=/etc/skel umask=027
```
Note : Le fichier /etc/skel définit comment les homes seront crée (pas de modifications à faire la dessus de notre coté). Le umask enlève les droits de modifications au groupe et n'attribue aucun droits aux autres utilisateurs.


On peut constater la création des homes dans le point de montage /home (notamment les users du ldap)
```bash
linuxperso@raspberrypi:/home $ ssh linuxperso@X.X.X.104
linuxperso@X.X.X.104's password:
Creating directory '/home/linuxperso'.
Linux raspberrypi2 6.1.0-rpi7-rpi-v8 #1 SMP PREEMPT Debian 1:6.1.63-1+rpt1 (2023-11-24) aarch64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Apr 14 04:28:53 2024 from X.X.X.102
-bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)

Wi-Fi is currently blocked by rfkill.
Use raspi-config to set the country before use.

linuxperso@raspberrypi2:~ $ touch test.txt
linuxperso@raspberrypi2:~ $ ls -la
total 28
drwxr-x--- 2 linuxperso linuxperso 4096 Apr 14 04:40 .
drwxr-xr-x 4 root       root       4096 Apr 14 04:40 ..
-rw-r----- 1 linuxperso linuxperso  220 Apr 14 04:40 .bash_logout
-rw-r----- 1 linuxperso linuxperso 3523 Apr 14 04:40 .bashrc
-rw-r----- 1 linuxperso linuxperso 5290 Apr 14 04:40 .face
lrwxrwxrwx 1 linuxperso linuxperso    5 Apr 14 04:40 .face.icon -> .face
-rw-r----- 1 linuxperso linuxperso  807 Apr 14 04:40 .profile
-rw-r--r-- 1 linuxperso linuxperso    0 Apr 14 04:40 test.txt
linuxperso@raspberrypi2:~ $ ls -la /home
total 28
drwxr-xr-x  4 root       root        4096 Apr 14 04:40 .
drwxr-xr-x 19 root       root        4096 Apr 14 03:36 ..
drwxr-x---  2 linuxperso linuxperso  4096 Apr 14 04:40 linuxperso
drwx------  2 root       root       16384 Apr 14 02:33 lost+found
linuxperso@raspberrypi2:~ $ su rodgerf
Password:
bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
rodgerf@raspberrypi2:/home/linuxperso $ cd
rodgerf@raspberrypi2:~ $ ls -la
total 28
drwxr-x--- 2 rodgerf cluster_users 4096 Apr 14 04:41 .
drwxr-xr-x 3 root    linuxperso    4096 Apr 14 04:41 ..
-rw-r----- 1 rodgerf cluster_users  220 Apr 14 04:41 .bash_logout
-rw-r----- 1 rodgerf cluster_users 3523 Apr 14 04:41 .bashrc
-rw-r----- 1 rodgerf cluster_users 5290 Apr 14 04:41 .face
lrwxrwxrwx 1 rodgerf cluster_users    5 Apr 14 04:41 .face.icon -> .face
-rw-r----- 1 rodgerf cluster_users  807 Apr 14 04:41 .profile

```

D'ailleurs, on remarque le user group associé au dossier /home/ldap crée est linuxperso (ce qui est assez intriguant).

## Exposer le point de montage aux autres cartes


tuto : https://opensource.com/article/20/5/nfs-raspberry-pi
<br>
Options /etc/exports : https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/deployment_guide/s1-nfs-server-config-exports

Nous allons utiliser le protocole NFS. Installation des paquets et activation du démon nfs-kernel-server :
```bash
sudo apt-get install nfs-common nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```

Modifications du fichier /etc/exports (il aurait été possible de spécifier un réseau et un masque associé)
Note : il est possible de désactiver les droits root sur la partiton partagée.
```bash
linuxperso@raspberrypi2:~ $ cat /etc/exports
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/home X.X.X.102(rw) X.X.X.106(rw)
/data X.X.X.102(rw) X.X.X.106(rw)
```

## Coté client (pour X.X.X.102 & X.X.X.106)

Réalisation du montage coté client :
```bash
linuxperso@raspberrypi:/ $ sudo mkdir /data
linuxperso@raspberrypi:/ $ sudo cat /etc/fstab
proc            /proc           proc    defaults          0       0
PARTUUID=61bc64d0-01  /boot/firmware  vfat    defaults          0       2
PARTUUID=61bc64d0-02  /               ext4    defaults,noatime  0       1
X.X.X.104:/home /home            nfs     defaults          0       0
X.X.X.104:/data /data            nfs     defaults          0       0
#swapfile is not a swap partition, no line here
#   use  dphys-swapfile swap[on|off]  for that
linuxperso@raspberrypi:/ $ sudo mount -a
linuxperso@raspberrypi:/ $ sudo systemctl daemon-reload
```

On peut vérifier que les homes sont partagés :
```bash
linuxperso@raspberrypi:~ $ touch test_nfs.txt
linuxperso@raspberrypi:~ $ ls -la
total 44
drwxr-x--- 4 linuxperso linuxperso 4096 Apr 15 02:10 .
drwxr-xr-x 5 root       root       4096 Apr 14 04:41 ..
-rw------- 1 linuxperso linuxperso  587 Apr 15 01:35 .bash_history
-rw-r----- 1 linuxperso linuxperso  220 Apr 14 04:40 .bash_logout
-rw-r----- 1 linuxperso linuxperso 3523 Apr 14 04:40 .bashrc
drwx------ 3 linuxperso linuxperso 4096 Apr 15 01:47 .config
-rw-r----- 1 linuxperso linuxperso 5290 Apr 14 04:40 .face
lrwxrwxrwx 1 linuxperso linuxperso    5 Apr 14 04:40 .face.icon -> .face
-rw-r----- 1 linuxperso linuxperso  807 Apr 14 04:40 .profile
-rw-r--r-- 1 linuxperso linuxperso    0 Apr 14 19:50 .sudo_as_admin_successful
-rw-r--r-- 1 linuxperso linuxperso    0 Apr 15 02:10 test_nfs.txt
-rw-r--r-- 1 linuxperso linuxperso    0 Apr 14 04:40 test.txt
-rw------- 1 linuxperso linuxperso  716 Apr 15 00:02 .viminfo
drwx------ 3 linuxperso linuxperso 4096 Apr 14 19:50 .vnc
linuxperso@raspberrypi:~ $ ssh linuxperso@X.X.X.104
linuxperso@X.X.X.104's password:

[...]

linuxperso@raspberrypi2:~ $ ls -la
total 48
drwxr-x--- 5 linuxperso linuxperso 4096 Apr 15 02:12 .
drwxr-xr-x 5 root       root       4096 Apr 14 04:41 ..
-rw------- 1 linuxperso linuxperso  587 Apr 15 01:35 .bash_history
-rw-r----- 1 linuxperso linuxperso  220 Apr 14 04:40 .bash_logout
-rw-r----- 1 linuxperso linuxperso 3523 Apr 14 04:40 .bashrc
drwx------ 3 linuxperso linuxperso 4096 Apr 15 01:47 .config
-rw-r----- 1 linuxperso linuxperso 5290 Apr 14 04:40 .face
lrwxrwxrwx 1 linuxperso linuxperso    5 Apr 14 04:40 .face.icon -> .face
-rw-r----- 1 linuxperso linuxperso  807 Apr 14 04:40 .profile
drwx------ 2 linuxperso linuxperso 4096 Apr 15 02:12 .ssh
-rw-r--r-- 1 linuxperso linuxperso    0 Apr 14 19:50 .sudo_as_admin_successful
-rw------- 1 linuxperso linuxperso  716 Apr 15 00:02 .viminfo
drwx------ 3 linuxperso linuxperso 4096 Apr 14 19:50 .vnc
-rw-r--r-- 1 linuxperso linuxperso    0 Apr 14 04:40 test.txt
-rw-r--r-- 1 linuxperso linuxperso    0 Apr 15 02:10 test_nfs.txt
```

Mainteant, les utilisateurs ont leur home partagés sur l'infrastructure. Ils ont aussi accès à une partiton /data qui contiendra les données partagés (images docker pour kubernetes par exemple). Il faudra gérer les droits sur celle-ci.







