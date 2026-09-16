# Splunk-SIEM-Lab-Insider-Threat-Detection

# 1. Clonage de la VM Ubuntu en splunk-server

Clonage via clic droit → Clone dans VirtualBox :
Option "Reinitialize MAC addresses" cochée
Options "Keep Disk Names" et "Keep Hardware UUID" décochées
<img width="846" height="110" alt="image" src="https://github.com/user-attachments/assets/5fc2eb49-187f-47b4-90a0-e000b6aa8237" />

⚠️ Piège rencontré : référence MAC obsolète après clonage

Le fichier /etc/netplan/00-installer-config.yaml hérité du clone référence encore l'ancienne adresse MAC (celle d'avant le clonage) dans un bloc match: — devenu obsolète puisque l'option "Reinitialize MAC addresses" en assigne une nouvelle. Ce bloc match/set-name a été retiré de la config.
