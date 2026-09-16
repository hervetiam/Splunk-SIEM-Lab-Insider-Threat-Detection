# Splunk-SIEM-Lab-Insider-Threat-Detection
Ce lab est la suite directe de pfsense-firewall-ips-lab. Alors que le premier projet se concentre sur la défense périmétrique (bloquer les attaques externes), celui-ci simule un scénario où un attaquant a déjà un accès interne au réseau (poste compromis), et démontre comment un SIEM permet de détecter ce type de menace que le firewall ne détecte 
Lab personnel de cybersécurité simulant la détection d'une menace interne (insider threat) à l'aide d'un SIEM Splunk, dans un environnement 100 % virtualisé (VirtualBox).

# Objectifs
Centraliser les logs de plusieurs sources hétérogènes (pare-feu, Linux, Windows) dans un SIEM Splunk
Simuler une attaque interne (brute-force SMB) invisible pour le pare-feu périmétrique
Détecter cette attaque via des requêtes SPL (Search Processing Language)
Croiser les résultats d'un scan de vulnérabilités (Tenable Nessus) avec les événements de sécurité observés
Documenter une architecture SIEM réaliste, du déploiement à la détection

# Architecture

<img width="1846" height="860" alt="image" src="https://github.com/user-attachments/assets/97f6e5eb-d8e1-4e80-a2a7-bd998f56407c" />

<img width="887" height="607" alt="image" src="https://github.com/user-attachments/assets/b6b14cdf-8fe9-4a6a-a802-f3aef5959bcf" />

# Sources de logs centralisées

<img width="857" height="206" alt="image" src="https://github.com/user-attachments/assets/f05674d6-5148-4371-ae54-a144a6413a2d" />
