# Splunk-SIEM-Lab-Insider-Threat-Detection

# 1. Clonage de la VM Ubuntu en splunk-server

Clonage via clic droit → Clone dans VirtualBox :
Option "Reinitialize MAC addresses" cochée
Options "Keep Disk Names" et "Keep Hardware UUID" décochées
<img width="846" height="110" alt="image" src="https://github.com/user-attachments/assets/5fc2eb49-187f-47b4-90a0-e000b6aa8237" />

⚠️ Piège rencontré : référence MAC obsolète après clonage

Le fichier /etc/netplan/00-installer-config.yaml hérité du clone référence encore l'ancienne adresse MAC (celle d'avant le clonage) dans un bloc match: — devenu obsolète puisque l'option "Reinitialize MAC addresses" en assigne une nouvelle. Ce bloc match/set-name a été retiré de la config.

# 2. Configuration réseau statique de splunk-server (Netplan)

Interface enp0s3 (LAN) → IP statique. Interface enp0s8 (NAT, ajouté automatiquement par VirtualBox) → conservé en DHCP pour l'accès internet (téléchargement des paquets).

/etc/netplan/00-installer-config.yaml :
<img width="851" height="542" alt="image" src="https://github.com/user-attachments/assets/6e4031c4-2bc7-46b5-8a4d-338ef76e0537" />

⚠️ Piège rencontré : conflit de route par défaut

Une première version du fichier incluait une route par défaut explicite sur enp0s3 (via: 192.168.1.1), en plus de celle fournie automatiquement par le DHCP sur enp0s8. Résultat : ping 8.8.8.8 échouait avec Destination Host Unreachable (le trafic sortait via pfSense, qui n'a pas d'accès internet réel côté WAN). Solution : ne définir qu'une seule route par défaut, en laissant enp0s8 (DHCP) la gérer, et retirer le bloc routes: de enp0s3.

<img width="857" height="167" alt="image" src="https://github.com/user-attachments/assets/6a187532-629b-458c-9617-5b1bd61eb465" />

# 3. Installation de Splunk Enterprise sur splunk-server

Téléchargement du .deb depuis le site officiel Splunk (compte gratuit requis), transféré via un dossier partagé VirtualBox (C:\Users\...\Downloads ↔ /media/sf_Downloads) faute de presse-papiers partagé fonctionnel dans cette session.


<img width="836" height="117" alt="image" src="https://github.com/user-attachments/assets/f2c39fba-8c0d-4225-9d23-6a53834b9d91" />

# 4. Accès à l'interface web Splunk (port forwarding VirtualBox)

splunk-server étant sur un réseau interne (intnet-lan), non joignable depuis le PC hôte, une règle de port forwarding NAT a été ajoutée sur l'adaptateur NAT de la VM :

Accès depuis le PC hôte : http://localhost:8000 

<img width="807" height="97" alt="image" src="https://github.com/user-attachments/assets/c6163838-f720-43ce-93ee-3a9a9fe78f60" />

# 5. Création des index dédiés

Un index par source de logs, pour une architecture claire :


<img width="841" height="166" alt="image" src="https://github.com/user-attachments/assets/ecda62da-e330-4cf5-adcf-ed42bdb04393" />


<img width="857" height="151" alt="image" src="https://github.com/user-attachments/assets/09830240-286d-46e7-b373-aa7faa299a14" />

# 6. Input UDP pour pfSense (syslog)

Port privilégié 514 évité (Splunk tourne sous vboxuser, pas root) → port 1514 utilisé à la place.
/opt/splunk/bin/splunk add udp 1514 -sourcetype pfsense -index pfsense_logs -auth admin:<password>

# 7. Configuration syslog côté pfSense

Interface web pfSense → Status → System Logs → Settings → Remote Logging Options :

Enable Remote Logging : coché
Source Address : Default (any)
Remote log servers : 192.168.1.50[:1514]
Remote Syslog Contents : Firewall Events coché


<img width="897" height="352" alt="image" src="https://github.com/user-attachments/assets/fdb09623-8528-489c-ad87-ce30ed003205" />


# 8. Port de réception pour les Universal Forwarders

<img width="872" height="337" alt="image" src="https://github.com/user-attachments/assets/087f666a-35b8-48ba-91a8-f5a2291edb89" />

⚠️ Piège rencontré : Splunk tué par le noyau (Out Of Memory)

En cours de session, splunkd a été tué par l'OOM Killer du noyau Linux (Out of memory: Killed process ... (splunkd)), à cause d'une RAM insuffisante avec trop de VMs actives simultanément (pfSense + Kali + Ubuntu + Windows + splunk-server sur 16 Go de RAM hôte au total). Symptôme observé côté client : nc -zv 192.168.1.50 9997 → Connection refused, forward Universal Forwarder passé en "Configured but inactive".

Solution appliquée : réduction du nombre de VMs actives en parallèle (utilisation de Save State plutôt que redémarrage à froid pour les VMs non utilisées), puis redémarrage de Splunk :

/opt/splunk/bin/splunk start
sudo /opt/splunk/bin/splunk enable listen 9997 -auth admin:<password>

# 9. Installation du Splunk Universal Forwarder sur Ubuntu (cible)

Même méthode de transfert (dossier partagé) que pour Splunk Enterprise.

sudo dpkg -i /media/sf_Downloads/splunkforwarder-10.4.3-4174a2deda5d-linux-amd64.deb
sudo /opt/splunkforwarder/bin/splunk start --accept-license

Configuration du forward-server et des sources à surveiller :
sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.1.50:9997 -auth <user>:<password>
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -index ubuntu_logs -auth <user>:<password>
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -index ubuntu_logs -auth <user>:<password>

Vérification de l'état de la connexion :

sudo /opt/splunkforwarder/bin/splunk list forward-server -auth <user>:<password>

# 10. Installation de Nessus (Tenable) — scan à distance

Décision d'architecture : par manque de temps, Nessus est installé en mode scanner réseau à distance (sur Kali-Internal) plutôt qu'en déploiement d'agents sur chaque VM cible. Les agents Tenable pourront être ajoutés dans une session future.

Inscription gratuite (Nessus Essentials, jusqu'à 16 IPs) sur le site Tenable, package .deb transféré via dossier partagé.

Interface accessible directement depuis le navigateur de la VM Kali : https://localhost:8834
