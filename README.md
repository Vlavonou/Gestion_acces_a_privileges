# Intégration de Teleport à Active Directory pour la gestion des accès à privilège

> Mémoire de Licence en Informatique - Option Sécurité Informatique
> Institut de Formation et de Recherche en Informatique (IFRI), Université d'Abomey-Calavi
> Auteur : **Senghor Padraic VLAVONOU** | Encadrant : Ing. Vladimir HOUZANME | Année académique 2025-2026

---

## Résumé

Ce projet étudie comment une solution open-source de gestion des accès à privilèges (Privileged Access Management, PAM), **Teleport**, peut être intégrée à **Active Directory (AD)** pour combler les limites natives de ce dernier en matière de traçabilité et de contrôle des accès privilégiés. Après une analyse comparative des principales solutions PAM du marché, Teleport a été retenu, déployé dans un laboratoire virtuel reproduisant une infrastructure d'entreprise, puis intégré à Active Directory. Un protocole d'évaluation comparatif a ensuite été appliqué entre des sessions RDP natives et des sessions médiées par Teleport, afin de mesurer objectivement les gains apportés en matière d'authentification, de traçabilité et d'auditabilité.

---

## Contexte et problématique

Les organisations s'appuient massivement sur Active Directory pour centraliser la gestion des identités et des accès. Cependant, à mesure que les infrastructures se complexifient et que le nombre de comptes à privilèges élevés augmente, le suivi précis des actions et le contrôle effectif des accès aux ressources sensibles deviennent difficiles à assurer avec les seuls mécanismes natifs d'AD. Trois limites structurelles ont été identifiées :

- une **attribution statique et permanente** des droits, contraire au principe du moindre privilège ;
- une **journalisation incomplète**, qui ne permet pas d'identifier avec certitude l'utilisateur humain à l'origine d'une action ;
- l'**absence de gestion native des accès temporaires**, la fonctionnalité TTL disponible depuis Windows Server 2016 restant complexe à activer et limitée en pratique (modification irréversible du schéma AD, absence d'interface ou de workflow d'approbation).

Ces constats posent la problématique centrale du mémoire :

> *Comment améliorer la sécurité, la gestion et la traçabilité des accès privilégiés dans Active Directory en intégrant une solution comme Teleport ?*

## Objectifs

**Objectif général** : mettre en place une solution de gestion des accès privilégiés intégrée à Active Directory, afin de renforcer la sécurité des ressources sensibles et d'améliorer la traçabilité et le contrôle des activités des utilisateurs.

**Objectifs spécifiques :**

1. Analyser et comparer plusieurs solutions PAM afin de justifier le choix de Teleport dans le cadre du projet ;
2. Installer et configurer Teleport dans un environnement de test local ;
3. Intégrer Teleport à Active Directory afin de centraliser l'authentification et la gestion des accès utilisateurs ;
4. Élaborer et appliquer un cadre d'évaluation de l'auditabilité des accès privilégiés, comparant les mécanismes natifs d'AD à ceux introduits par Teleport.

---

## Choix technique et justification

### Panorama des solutions PAM étudiées

Quatre solutions ont été analysées de manière comparative : **Teleport**, **StrongDM**, **CyberArk Privileged Access Manager** et **WALLIX PAM4ALL**.

| Critère | Teleport | StrongDM | CyberArk PAM | WALLIX PAM4ALL |
|---|---|---|---|---|
| Architecture | Proxy + agents + CA interne | Proxy central (gateways, agentless) | Vault + CPM + PSM | Bastion centralisé |
| Gestion des secrets | Aucun stockage de mot de passe | Centralisée, identity-aware | Coffre-fort (Vault) sécurisé | Vault / Bastion |
| Authentification | Certificats temporaires (passwordless) | Identity-based (SSO, LDAP, OIDC) | Mots de passe + rotation automatique | Injection de mots de passe |
| RBAC | Basé sur certificats + TTL | Dynamique | Très granulaire | Intégré au bastion |
| Audit | Rejeu de session + logs structurés JSON | Logs centralisés, intégration SIEM | Enregistrement vidéo + keystrokes | Enregistrement de session |
| Dépendance aux mots de passe | Non | Faible | Forte | Forte |
| Licence | Open source | Propriétaire | Propriétaire | Propriétaire |

Cette analyse comparative a montré que les solutions concurrentes reposent majoritairement sur des licences propriétaires et onéreuses, peu compatibles avec les contraintes d'un environnement académique. **Teleport se distingue par son caractère open source**, son architecture fondée sur le paradigme **Zero Trust**, et une authentification entièrement passwordless basée sur des certificats éphémères - ce qui en fait la solution la plus adaptée au contexte du projet.

### Principe de fonctionnement de Teleport

Teleport repose sur trois composants principaux :

- **Auth Service** : autorité de certification (CA) interne du cluster ; il délivre les certificats aux utilisateurs et services, et tient le journal d'audit complet des activités.
- **Proxy Service** : point d'entrée unique du cluster depuis l'extérieur ; il relaie le trafic vers les ressources internes sans jamais décrypter ni authentifier directement les connexions.
- **Agents** : déployés au plus près des ressources cibles, ils communiquent via les protocoles natifs des services (SSH, RDP via le Teleport Desktop Protocol, API Kubernetes, bases de données, etc.) et vérifient les certificats émis par l'Auth Service.

Après une première authentification sécurisée par MFA, reliée à un fournisseur d'identité (Active Directory, Okta, GitHub…), l'utilisateur reçoit un certificat temporaire à durée de vie configurable (TTL). Ce certificat porte à la fois l'identité de l'utilisateur et ses permissions (RBAC), qui expirent automatiquement à l'issue de la session - ce qui permet d'appliquer nativement le principe du moindre privilège, contrairement à l'attribution statique des droits observée dans Active Directory.

---

## Architecture du laboratoire de test

L'ensemble du laboratoire a été hébergé dans **VMware Workstation 17 Pro**, installé sur un ordinateur portable Lenovo (Intel Core i5-3437U @ 1.90 GHz, 16 Go de RAM, SSD 256 Go). Les machines virtuelles sont interconnectées sur un même réseau virtuel (VMnet4), simulant une infrastructure d'entreprise classique.

| Machine | Rôle | IP |
|---|---|---|
| SRV-AD (Windows Server 2022) | Contrôleur de domaine Active Directory (AD DS + AD CS) | 192.168.10.1 |
| teleportcluster (Ubuntu Server 24.02 LTS) | Cluster Teleport (Auth + Proxy Service) | 192.168.10.2 |
| WindAdmin (Windows 11 Pro) | Poste représentant un profil administrateur | 192.168.10.3 |
| WindUser (Windows 11 Pro) | Poste représentant un profil utilisateur standard | 192.168.10.4 |

Domaine Active Directory : `memoire.local`. Deux comptes de domaine ont été créés pour représenter les deux profils d'usage : **Jack** (administrateur du domaine) et **Celia** (utilisateur standard).

Dans cette architecture, aucun accès aux postes Windows ne s'effectue directement : toute demande de connexion transite obligatoirement par le proxy Teleport, qui s'appuie sur Active Directory pour vérifier l'identité de l'utilisateur avant d'établir la connexion distante vers la machine cible, tout en assurant le contrôle et la traçabilité de la session. Cette organisation dissocie la gestion des identités (assurée par AD) de la gestion effective des accès (prise en charge par Teleport).

---

## Méthodologie de mise en œuvre

La mise en œuvre s'est déroulée en plusieurs étapes structurées :

1. **Préparation de l'infrastructure Active Directory** : déploiement du contrôleur de domaine, intégration des postes clients Windows au domaine, création des comptes utilisateurs.
2. **Configuration DNS** : ajout d'un enregistrement A (`teleport` → 192.168.10.2) et d'un enregistrement CNAME wildcard (`*.teleport`) pour permettre l'adressage des ressources via Teleport.
3. **Obtention d'un certificat TLS via l'autorité de certification interne (AD CS)** : génération d'une CSR sur le serveur Ubuntu, soumission à l'AD CS depuis le serveur Windows, conversion des certificats au format PEM requis par Teleport, puis intégration du certificat racine de l'AD CS dans le magasin de confiance d'Ubuntu.
4. **Installation et configuration de Teleport Community Edition (version 18.1.4)** : installation du binaire, génération du fichier de configuration principal référençant le certificat TLS, mise en service via systemd, puis création d'un premier utilisateur administrateur (`JackAdmin`) avec activation du MFA (application d'authentification).
5. **Intégration de Teleport à Active Directory**, elle-même décomposée en plusieurs sous-étapes :
   - mise en place d'une connexion sécurisée **LDAPS** entre Teleport et AD ;
   - création d'un **compte de service restrictif** dédié (`svc-teleport`), avec permissions minimales sur les objets PKI et blocage de toute connexion interactive via GPO ;
   - création d'une **GPO d'autorisation** important le certificat CA de Teleport dans les autorités racines de confiance du domaine et publiant ce certificat dans les magasins `RootCA` et `NTAuthCertificates` ;
   - activation du **service Carte à puce** (Teleport émule une authentification par carte à puce basée sur certificat) ;
   - activation des **connexions Bureau à distance**, avec désactivation de la NLA (Network Level Authentication) et de la demande systématique de mot de passe, ces deux paramètres étant incompatibles avec le code PIN temporaire généré par Teleport pour chaque session ;
   - ouverture du **port RDP (TCP 3389)** dans le pare-feu Windows et activation de **RemoteFX** pour optimiser le rendu graphique des sessions distantes ;
   - configuration du service `windows_desktop_service` dans le fichier de configuration Teleport, référençant l'annuaire AD et le compte de service ;
   - création d'un **rôle Teleport** (`full-admin`) accordant l'accès à l'ensemble des postes du domaine avec les logins `Administrateur` et `Jack`, assigné à l'utilisateur `JackAdmin`.
6. **Définition d'un cadre d'évaluation de l'auditabilité**, appliqué ensuite de façon comparative aux deux mécanismes d'accès (RDP natif vs RDP médié par Teleport).

---

## Protocole d'évaluation comparative

Un accès privilégié a été considéré comme *auditable* dans ce travail lorsque les journaux de sécurité permettent d'identifier sans ambiguïté : l'utilisateur humain à l'origine de l'action, la session concernée, la machine cible, et les actions réalisées. Ce cadre a été appliqué à deux scénarios identiques (création d'un répertoire et d'un compte de domaine ajouté au groupe des administrateurs), l'un exécuté via une session RDP native, l'autre via une session médiée par Teleport.

### Scénario 1 - Accès RDP natif via Active Directory

L'analyse des journaux de sécurité Windows (Observateur d'événements) après la session a révélé un événement de type *Security Group Management* (ID 4799), associé non pas à un utilisateur humain mais au **compte machine** (`WINDUSER$`) et à un processus système (`svchost.exe`). L'identifiant de session technique relevé (Logon ID) ne permettait aucune correspondance explicite avec une session RDP identifiable. Ces observations mettent en évidence les limites constatées des journaux natifs : absence d'identification claire de l'utilisateur réel, informations dispersées et purement techniques, nécessité d'une corrélation manuelle longue entre plusieurs événements, et donc une faible valeur probante pour établir qui a réalisé une action, dans quel cadre, et avec quelle intention.

### Scénario 2 - Accès médié par Teleport

La même opération, réalisée via une session Teleport (authentification `JackAdmin` + MFA, puis connexion au compte `Jack` sans saisie de mot de passe grâce à l'authentification par certificat), a généré dans le journal d'audit de Teleport (*Audit Log*) une série d'événements horodatés et liés à un identifiant de session unique : démarrage de la session bureau Windows, émission des certificats utilisateur, fin de la session. Le détail de l'événement *Windows Desktop Session Started* mentionne explicitement le compte Teleport (`JackAdmin`), le compte Windows utilisé (`Jack`), le domaine (`memoire.local`), les adresses réseau locale et distante, ainsi que le protocole utilisé (Teleport Desktop Protocol). La section *Session Recording* permet en complément de rejouer intégralement la session sous forme de vidéo.

---

## Résultats

Les tests ont confirmé que Teleport permet d'identifier sans ambiguïté, pour chaque accès privilégié :

-  l'utilisateur humain (compte Teleport + compte Windows) ;
-  la session concernée (identifiant unique) ;
-  la machine cible et son adresse réseau ;
-  les actions réalisées (enregistrement vidéo intégral + logs structurés JSON).

### Synthèse comparative RDP natif vs Teleport

| Critère | RDP natif (AD seul) | Avec Teleport |
|---|---|---|
| Authentification | Simple (mot de passe) | Multi-facteur (MFA obligatoire) |
| Exposition des identifiants | Mots de passe manipulés directement | Certificats temporaires, aucun mot de passe exposé |
| Traçabilité | Journaux partiels, utilisateur non identifiable | Logs complets (utilisateur, session, machine, IP) |
| Enregistrement de session | Aucun | Enregistrement vidéo intégral, rejouable |
| Principe du moindre privilège | Non appliqué nativement | RBAC avec TTL sur les certificats |
| Point d'accès centralisé | Non (connexion directe) | Oui (proxy Teleport obligatoire) |
| Non-répudiation | Faible | Garantie (session ID unique + MFA) |

L'évaluation a également mobilisé des métriques reconnues en sécurité des systèmes d'information : réduction de la surface d'attaque (élimination du stockage et de la transmission de mots de passe), nombre de facteurs d'authentification (1 pour le RDP natif contre 2 pour Teleport), accountability, complétude des journaux, traçabilité des sessions, non-répudiation, et temps moyen d'analyse (réduit par la centralisation et la structuration des journaux Teleport).

Sur cette base, les quatre objectifs spécifiques du projet ont été atteints à 100 % : justification du choix de Teleport à l'issue de la comparaison des solutions PAM, installation et configuration réussies dans l'environnement de test, intégration effective à Active Directory via LDAPS, et application concrète du cadre d'évaluation de l'auditabilité.

---

## Limites et perspectives

Le travail s'est limité à un environnement de test virtualisé et à la **version communautaire (Community Edition)** de Teleport. Une démarche d'industrialisation nécessiterait d'approfondir l'analyse des éditions commerciales, d'évaluer leur intégration dans des environnements hybrides ou multi-cloud, et d'en étudier les performances à grande échelle.

Le mémoire ouvre plusieurs pistes de poursuite : intégration de Teleport dans une approche plus globale combinant PAM, Zero Trust et gestion des identités (IAM) ; automatisation des contrôles de sécurité ; intégration avec des solutions SIEM ; et application de l'intelligence artificielle à la détection proactive de comportements anormaux liés aux comptes privilégiés.

---

## Technologies utilisées

- **Teleport** 18.1.4 (Community Edition)
- **Windows Server 2022** - Active Directory Domain Services + Active Directory Certificate Services (AD CS)
- **Ubuntu Server 24.02 LTS**
- **VMware Workstation 17 Pro**
- **OpenSSL**, **PowerShell**, **tctl** (CLI d'administration Teleport)
- Protocoles : RDP, LDAPS, TLS, TDP (Teleport Desktop Protocol)

---

## Auteur

**Senghor Padraic VLAVONOU**
Licence en Informatique - Option Sécurité Informatique
IFRI, Université d'Abomey-Calavi, Bénin

Encadrant : Ing. Vladimir HOUZANME

---

## Licence

Ce projet est partagé à des fins académiques et éducatives.
---

> 📄 Ce README propose une synthèse du travail. Pour une lecture complète (revue de littérature, analyse détaillée, bibliographie), consulter le mémoire complet au format PDF disponible dans ce dépôt.
