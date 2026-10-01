# 📄 Trame de Contexte de Projet — Épreuve U6 (BTS SIO SISR)

---

## 1. 📌 Fiche synthétique du projet

| Élément | Description |
| :--- | :--- |
| **Intitulé du projet** | Mise en place d'une infrastructure Web sécurisée : Cloisonnement DMZ (Production) et VLAN dédié (Pré-production) sous WordPress |
| **Cadre de réalisation** | Épreuve E6 / U6 — Conception et maintenance de solutions informatiques |
| **Organisation support** | PME fictive ou réelle *(ex: InnovTech / Clinique / Collectivité)* |
| **Rôle de l'étudiant** | Administrateur Systèmes & Réseaux / Chargé de projet infrastructure |
| **Périmètre** | Segmentation réseau (VLAN / DMZ), routage & filtrage (Pare-feu), déploiement de serveurs web LAMP/LEMP, sécurisation et gestion du cycle de vie du CMS WordPress |

---

## 2. 🏢 Contexte organisationnel et problématique

### 2.1 Présentation de l'entreprise
L'organisation est une PME de 40 collaborateurs qui souhaite refondre sa présence numérique et disposer d'un site web institutionnel / vitrine moderne basé sur **WordPress**.  
L'entreprise dispose d'un réseau local d'entreprise (LAN) hébergeant les postes clients et des services internes critiques (fichiers, annuaire Active Directory).

### 2.2 Problématique constatée
- **Risque de sécurité majeur** : Héberger un site web public directement sur le réseau interne expose l'ensemble des données de l'entreprise en cas de compromission (failles de sécurité web, extensions WordPress vulnérables).
- **Absence d'environnement de test** : Les modifications, ajouts d'extensions et montées de version du CMS sont actuellement appliquées à chaud ou sans validation préalable, causant des interruptions de service potentielles.
- **Manque d'étanchéité réseau** : Les flux d'administration et les flux publics ne sont pas isolés.

### 2.3 Objectifs du projet
1. **Isolation de l'accès public (DMZ)** : Déployer le site web de **Production** dans une Zone Démilitarisée (VLAN DMZ) isolée du LAN d'entreprise.
2. **Création d'un environnement de Pré-production (Staging)** : Déployer une réplique du site dans un **VLAN Web / Pré-prod** distinct, accessible uniquement par l'équipe interne (communication / webmestre / administration).
3. **Contrôle strict des flux (Pare-feu)** : Appliquer une politique de sécurité empêchant formellement toute connexion initiée depuis la DMZ vers le reste de l'infrastructure interne.
4. **Procédure de validation et mise en production** : Définir une méthode fiable pour tester les changements en pré-production avant bascule en production.
5. **Continuité de service et sauvegardes** : Automatiser la sauvegarde des fichiers et des bases de données SQL.

---

## 3. 🌐 Architecture Réseau et Plan d'Adressage

### 3.1 Découpage des zones et VLANs

| Zone / VLAN | ID VLAN | Réseau IP (exemple) | Passerelle (Pare-feu) | Rôle & Description |
| :--- | :---: | :--- | :--- | :--- |
| **WAN** | — | *IP Publique / FAI* | IP Passerelle FAI | Accès Internet public entrant et sortant |
| **VLAN ADMIN / LAN** | `VLAN 10` | `192.168.10.0/24` | `192.168.10.254` | Postes internes, équipe informatique et webmestres |
| **VLAN WEB (Pré-prod)** | `VLAN 20` | `192.168.20.0/24` | `192.168.20.254` | Serveur WordPress de test / validation interne |
| **VLAN DMZ (Prod)** | `VLAN 30` | `192.168.30.0/24` | `192.168.30.254` | Serveur WordPress de production ouvert sur le Web |

### 3.2 Schéma logique simplifié

```text
               +-----------------------+
               |        INTERNET       |
               +-----------+-----------+
                           | (WAN)
                   +-------+-------+
                   |    PARE-FEU   | (pfSense / OPNsense / Linux Router)
                   +---+-------+---+
                       |       |
      (VLAN 30 - DMZ)  |       |  (VLAN 20 - Web / Pré-prod)
    +------------------+       +-------------------+
    |                                              |
    v                                              v
+------------------------+             +------------------------+
| Srv-Web-PROD (DMZ)     |             | Srv-Web-PREPROD        |
| - IP: 192.168.30.10    |             | - IP: 192.168.20.10    |
| - WordPress public     |             | - WordPress interne    |
| - Nginx/Apache + SQL   |             | - Accessible LAN seul  |
+------------------------+             +------------------------+
            ^                                      ^
            | (Déploiement / SSH restreint)        | (Accès recette / dev)
            +------------------+-------------------+
                               | (VLAN 10 - LAN / Admin)
                    +----------+-----------+
                    | Postes Admin / Dév   |
                    | IP: 192.168.10.x     |
                    +----------------------+
```

---

## 4. 🛡️ Matrice des Flux et Politique de Sécurité

La sécurité repose sur le principe de **moindre privilège** :

| Source | Destination | Ports / Protocoles | Action | Justification |
| :--- | :--- | :--- | :---: | :--- |
| **WAN (Internet)** | **DMZ (Srv Prod)** | `80/TCP (HTTP)`, `443/TCP (HTTPS)` | **Autoriser** | Accès public au site web de production (NAT / Port Forwarding) |
| **WAN (Internet)** | **VLAN Pré-prod** | Tout | **Bloquer** | La pré-production ne doit jamais être accessible depuis l'extérieur |
| **VLAN LAN / Admin**| **VLAN Pré-prod** | `80, 443, 22 (SSH)` | **Autoriser** | Travail, intégration de contenu et administration du site de test |
| **VLAN LAN / Admin**| **DMZ (Srv Prod)** | `22/TCP (SSH restreint)` | **Autoriser** | Administration distante sécurisée (filtrage par IP admin) |
| **DMZ (Srv Prod)** | **LAN / Pré-prod** | Tout | **Bloquer (STRICT)**| **Règle d'or de la DMZ** : un serveur compromis ne peut pas rebondir vers l'interne |
| **DMZ & Pré-prod** | **WAN (Internet)** | `80, 443 (HTTP/S)`, `53 (DNS)` | **Autoriser** | Mises à jour des paquets Linux et des extensions WordPress |

---

## 5. 🧰 Stack Technique et Outils

- **Virtualisation / Hyperviseur** : Proxmox VE, VMware ESXi ou VirtualBox (pour la maquette).
- **Routeur / Pare-feu** : pfSense ou OPNsense (gestion des VLANs 802.1Q, DHCP, règles de pare-feu et translation NAT).
- **Système d'exploitation des serveurs** : Debian GNU/Linux ou Ubuntu Server LTS.
- **Serveur Web & Base de données** :
  - Stack LAMP (*Linux, Apache2, MariaDB/MySQL, PHP*) ou LEMP (*Nginx*).
  - Possibilité d'utiliser des conteneurs **Docker** avec Docker Compose pour isoler les services.
- **Applicatif** : CMS WordPress (thème personnalisé ou adapté, extensions nécessaires).
- **Mécanisme de bascule / Déploiement** :
  - Outil de migration (ex: *WP-CLI*, *All-in-One WP Migration*, ou script Bash combinant `rsync` et export `mysqldump`).
- **Sauvegarde** : Scripts automatisés via tâche planifiée `cron` vers un stockage sécurisé.

---

## 6. 📋 Compétences BTS SIO SISR mobilisées

En référence aux épreuves du BTS SIO SISR :

1. **Concevoir une solution d'infrastructure** :
   - Analyser le besoin de sécurisation d'un service exposé publiquement.
   - Élaborer un plan d'adressage et segmenter le réseau en VLANs / DMZ.
2. **Installer, tester et déployer** :
   - Configurer les interfaces et règles de filtrage sur le pare-feu.
   - Déployer et durcir deux serveurs Linux d'hébergement web.
   - Configurer les bases de données et les hôtes virtuels (vHosts).
3. **Assurer la sécurité et la continuité de service** :
   - Mettre en œuvre le principe d'étanchéité de la DMZ.
   - Configurer des certificats SSL/TLS (HTTPS).
   - Mettre en place un plan de sauvegarde et de restauration (BDD + fichiers).
4. **Documenter et exploiter** :
   - Rédiger les procédures techniques d'installation et d'exploitation.
   - Rédiger la procédure de mise en production (recette -> prod).

---

## 7. 🗓️ Découpage prévisionnel des étapes (Roadmap)

- [ ] **Étape 1 : Cadrage & Cahier des charges** (Validation des besoins, architecture, plan d'adressage).
- [ ] **Étape 2 : Configuration Réseau & Pare-feu** (Création des interfaces, VLANs, règles de routage et NAT).
- [ ] **Étape 3 : Déploiement du serveur de Pré-production** (Installation OS, stack web, déploiement du WordPress de dev).
- [ ] **Étape 4 : Déploiement du serveur de Production en DMZ** (Installation OS, stack web, durcissement de sécurité).
- [ ] **Étape 5 : Mise en place du processus de synchronisation / déploiement** (Script ou outil de transfert Pré-prod ➔ Prod).
- [ ] **Étape 6 : Stratégie de sauvegarde & Tests de validation** (Tests de cloisonnement réseau, tests d'intrusion basiques, tests de restauration).
- [ ] **Étape 7 : Finalisation de la documentation technique** (Dossier U6, schémas finaux et fiches d'incidents/recette).