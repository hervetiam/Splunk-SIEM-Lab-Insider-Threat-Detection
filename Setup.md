1. Clonage de la VM Ubuntu en splunk-server

Clonage via clic droit → Clone dans VirtualBox :

Option "Reinitialize MAC addresses" cochée
Options "Keep Disk Names" et "Keep Hardware UUID" décochées
bash
sudo hostnamectl set-hostname splunk-server

VM redémarrée après le changement de hostname.

📸 Capture des paramètres de clonage VirtualBox.

⚠️ Piège rencontré : référence MAC obsolète après clonage

Le fichier /etc/netplan/00-installer-config.yaml hérité du clone référence encore l'ancienne adresse MAC (celle d'avant le clonage) dans un bloc match: — devenu obsolète puisque l'option "Reinitialize MAC addresses" en assigne une nouvelle. Ce bloc match/set-name a été retiré de la config.

2. Configuration réseau statique de splunk-server (Netplan)

Interface enp0s3 (LAN) → IP statique. Interface enp0s8 (NAT, ajouté automatiquement par VirtualBox) → conservé en DHCP pour l'accès internet (téléchargement des paquets).

/etc/netplan/00-installer-config.yaml :

yaml
network:
  ethernets:
    enp0s3:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.1.50/24
      nameservers:
        addresses: [192.168.1.1, 8.8.8.8]
    enp0s8:
      dhcp4: true
  version: 2
bash
sudo netplan apply
⚠️ Piège rencontré : conflit de route par défaut

Une première version du fichier incluait une route par défaut explicite sur enp0s3 (via: 192.168.1.1), en plus de celle fournie automatiquement par le DHCP sur enp0s8. Résultat : ping 8.8.8.8 échouait avec Destination Host Unreachable (le trafic sortait via pfSense, qui n'a pas d'accès internet réel côté WAN). Solution : ne définir qu'une seule route par défaut, en laissant enp0s8 (DHCP) la gérer, et retirer le bloc routes: de enp0s3.

Vérification :

bash
ip a
ping -c 4 8.8.8.8
ping -c 4 google.com

📸 Capture ip a finale montrant les deux interfaces configurées correctement.

3. Installation de Splunk Enterprise sur splunk-server

Téléchargement du .deb depuis le site officiel Splunk (compte gratuit requis), transféré via un dossier partagé VirtualBox (C:\Users\...\Downloads ↔ /media/sf_Downloads) faute de presse-papiers partagé fonctionnel dans cette session.

bash
sudo dpkg -i /media/sf_Downloads/splunk-10.4.3-4174a2deda5d-linux-amd64.deb
⚠️ Piège rencontré : refus de démarrage en root
bash
sudo /opt/splunk/bin/splunk start --accept-license

→ refusé : "Running Splunk Enterprise as root is deprecated..."

Solution : changer le propriétaire du dossier Splunk vers un utilisateur non-privilégié, puis démarrer sans sudo :

bash
sudo chown -R vboxuser:vboxuser /opt/splunk
/opt/splunk/bin/splunk start --accept-license

Compte administrateur Splunk créé à cette étape (username + password, non documentés ici pour des raisons de sécurité).

📸 Capture de l'écran de démarrage Splunk confirmant "Done".

4. Accès à l'interface web Splunk (port forwarding VirtualBox)

splunk-server étant sur un réseau interne (intnet-lan), non joignable depuis le PC hôte, une règle de port forwarding NAT a été ajoutée sur l'adaptateur NAT de la VM :

Nom	Protocole	Host Port	Guest Port
splunk-web	TCP	8000	8000

Accès depuis le PC hôte : http://localhost:8000

5. Création des index dédiés

Un index par source de logs, pour une architecture claire :

bash
/opt/splunk/bin/splunk add index pfsense_logs -auth admin:<password>
/opt/splunk/bin/splunk add index ubuntu_logs -auth admin:<password>
/opt/splunk/bin/splunk add index windows_logs -auth admin:<password>

Vérification :

bash
/opt/splunk/bin/splunk list index -auth admin:<password>

📸 Capture de la liste des index confirmant la création des 3 index dédiés.

6. Input UDP pour pfSense (syslog)

Port privilégié 514 évité (Splunk tourne sous vboxuser, pas root) → port 1514 utilisé à la place.

bash
/opt/splunk/bin/splunk add udp 1514 -sourcetype pfsense -index pfsense_logs -auth admin:<password>
7. Configuration syslog côté pfSense

Interface web pfSense → Status → System Logs → Settings → Remote Logging Options :

Enable Remote Logging : coché
Source Address : Default (any)
Remote log servers : 192.168.1.50[:1514]
Remote Syslog Contents : Firewall Events coché

📸 Capture de la page "Remote Logging Options" de pfSense.

Vérification du flux

Depuis n'importe quelle VM du LAN (ex. Ubuntu cible) :

bash
ping -c 5 192.168.1.1

Côté splunk-server :

bash
/opt/splunk/bin/splunk search "index=pfsense_logs | head 5" -auth admin:<password>

Résultat confirmé : logs filterlog reçus avec succès (source, destination, protocole, action visibles).

📸 Capture du résultat de la recherche SPL confirmant la réception des logs pfSense.

8. Port de réception pour les Universal Forwarders
bash
/opt/splunk/bin/splunk enable listen 9997 -auth admin:<password>

Vérification :

bash
sudo ss -tulpn | grep 9997
⚠️ Piège rencontré : Splunk tué par le noyau (Out Of Memory)

En cours de session, splunkd a été tué par l'OOM Killer du noyau Linux (Out of memory: Killed process ... (splunkd)), à cause d'une RAM insuffisante avec trop de VMs actives simultanément (pfSense + Kali + Ubuntu + Windows + splunk-server sur 16 Go de RAM hôte au total). Symptôme observé côté client : nc -zv 192.168.1.50 9997 → Connection refused, forward Universal Forwarder passé en "Configured but inactive".

Solution appliquée : réduction du nombre de VMs actives en parallèle (utilisation de Save State plutôt que redémarrage à froid pour les VMs non utilisées), puis redémarrage de Splunk :

bash
/opt/splunk/bin/splunk start
sudo /opt/splunk/bin/splunk enable listen 9997 -auth admin:<password>

Leçon retenue : sur un hôte à ressources limitées, n'allumer que les VMs strictement nécessaires à l'étape en cours.

9. Installation du Splunk Universal Forwarder sur Ubuntu (cible)

Même méthode de transfert (dossier partagé) que pour Splunk Enterprise.

bash
sudo dpkg -i /media/sf_Downloads/splunkforwarder-10.4.3-4174a2deda5d-linux-amd64.deb
sudo /opt/splunkforwarder/bin/splunk start --accept-license

Configuration du forward-server et des sources à surveiller :

bash
sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.1.50:9997 -auth <user>:<password>
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -index ubuntu_logs -auth <user>:<password>
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log -index ubuntu_logs -auth <user>:<password>

Vérification de l'état de la connexion :

bash
sudo /opt/splunkforwarder/bin/splunk list forward-server -auth <user>:<password>

Diagnostic réseau en cas de forward "inactive" :

bash
ping -c 3 192.168.1.50
nc -zv 192.168.1.50 9997

Vérification finale côté serveur :

bash
/opt/splunk/bin/splunk search "index=ubuntu_logs | head 5" -auth admin:<password>

✅ Pipeline Ubuntu → Splunk confirmé fonctionnel de bout en bout.

📸 Capture de la recherche SPL confirmant la réception des logs Ubuntu (/var/log/syslog, /var/log/auth.log).

10. Clonage et repositionnement de Kali sur le LAN

Clonage de la VM Kali existante (préservée intacte sur le WAN pour le Lab 1) plutôt que déplacement direct :

Clic droit → Clone, nom Kali-Internal
"Reinitialize the MAC address of all network cards" coché
Type : Full Clone
VM source éteinte avant clonage (clone plus fiable)

Adaptateur réseau modifié : Adapter 1 : Internal Network → intnet-lan

Configuration IP statique (NetworkManager)
bash
nmcli connection show
sudo nmcli connection modify "wan-static" ipv4.addresses 192.168.1.20/24 ipv4.gateway 192.168.1.1 ipv4.dns 192.168.1.1 ipv4.method manual
sudo nmcli connection up "wan-static"

Vérification :

bash
ip a
ping -c 4 192.168.1.100
⚠️ Piège rencontré (à nouveau) : conflit de route par défaut

Un 2e adaptateur NAT a été ajouté à Kali-Internal pour l'accès internet (nécessaire à l'activation de Nessus). Même symptôme que sur splunk-server : deux routes par défaut en conflit.

bash
ip route
# default via 192.168.1.1 dev eth0 proto static metric 100   ← gagnait, mais sans accès internet
# default via 10.0.3.2 dev eth1 proto dhcp metric 101         ← route NAT correcte, ignorée

Solution : empêcher la connexion LAN de fournir une route par défaut :

bash
sudo nmcli connection modify "wan-static" ipv4.never-default yes
sudo nmcli connection up "wan-static"

📸 Capture ip a de Kali-Internal montrant les deux interfaces (LAN + NAT).

11. Installation de Nessus (Tenable) — scan à distance

Décision d'architecture : par manque de temps, Nessus est installé en mode scanner réseau à distance (sur Kali-Internal) plutôt qu'en déploiement d'agents sur chaque VM cible. Les agents Tenable pourront être ajoutés dans une session future.

Inscription gratuite (Nessus Essentials, jusqu'à 16 IPs) sur le site Tenable, package .deb transféré via dossier partagé.

bash
sudo dpkg -i /media/sf_Downloads/Nessus-*.deb
sudo systemctl start nessusd.service
sudo systemctl enable nessusd.service
