# Intégration de Teleport à Active Directory pour la gestion des accès à privilège

> Mémoire de Licence en Informatique Option Sécurité Informatique  
> Institut de Formation et de Recherche en Informatique (IFRI) Université d'Abomey-Calavi  
> Auteur : **Senghor Padraic VLAVONOU** | Année académique : 2025-2026

---

## Résumé

Ce projet démontre comment **Teleport** (solution open-source de PAM) peut être intégré à **Active Directory** pour pallier les limitations natives d'AD en matière de gestion des accès à privilèges. Des scénarios comparatifs ont été réalisés entre des sessions RDP classiques et des sessions médiées par Teleport, mettant en évidence des améliorations significatives en termes de sécurité, de traçabilité et de contrôle d'accès.

---

## Problématique

> *Comment améliorer la sécurité, la gestion et la traçabilité des accès privilégiés dans Active Directory en intégrant une solution comme Teleport ?*

Les mécanismes natifs d'Active Directory présentent trois limites majeures :
- Attribution **statique et permanente** des droits (pas de principe du moindre privilège)
- Journalisation **incomplète** : impossible d'identifier clairement l'utilisateur humain derrière une action
- Absence de **gestion native des accès temporaires**

---

## Solution : Teleport + Active Directory

### Architecture du laboratoire de test

| Machine | Rôle | IP |
|---|---|---|
| SRV-AD (Windows Server 2022) | Contrôleur de domaine Active Directory | 192.168.10.1 |
| teleportcluster (Ubuntu 24.02 LTS) | Cluster Teleport (Auth + Proxy Service) | 192.168.10.2 |
| WindAdmin (Windows 11 Pro) | Poste administrateur (cible de test) | 192.168.10.3 |
| WindUser (Windows 11 Pro) | Poste utilisateur standard (cible de test) | 192.168.10.4 |

Domaine Active Directory : `memoire.local`

### Ce que Teleport apporte

| Critère | RDP natif (AD seul) | Avec Teleport |
|---|---|---|
| Authentification | Simple (mot de passe) | Multi-facteur (MFA obligatoire) |
| Exposition des identifiants | Mots de passe manipulés directement | Certificats temporaires, aucun mot de passe exposé |
| Traçabilité | Journaux partiels, utilisateur non identifiable | Logs complets avec utilisateur, session, machine et IP |
| Enregistrement de session | Aucun | Enregistrement vidéo intégral, rejouable |
| Principe du moindre privilège | Non appliqué nativement | RBAC avec TTL sur les certificats |
| Point d'accès centralisé | Non (connexion directe) | Oui (proxy Teleport obligatoire) |
| Non-répudiation | Faible | Garantie (session ID unique + MFA) |

---

## Étapes d'intégration (résumé technique)

### 1. Prérequis
- Active Directory opérationnel avec AD CS (Certificate Services)
- Serveur Linux pour héberger Teleport
- Configuration DNS avec enregistrement A (`teleport`) et CNAME wildcard (`*.teleport`)
- Mise en place de **LDAPS** (LDAP over TLS) entre Teleport et AD

### 2. Installation de Teleport

```bash
curl https://cdn.teleport.dev/install.sh | bash -s 18.1.4
```

```bash
sudo teleport configure -o file \
  --cluster-name=teleport.memoire.local \
  --public-addr=teleport.memoire.local:443 \
  --cert-file=/var/lib/teleport/fullchain.pem \
  --key-file=/var/lib/teleport/privkey.pem
```

```bash
sudo systemctl enable teleport && sudo systemctl start teleport
```

### 3. Certificat TLS via AD CS

Génération d'une CSR sur Ubuntu, soumission à l'Autorité de Certification AD, conversion en PEM :

```bash
openssl req -new -newkey rsa:2048 -nodes -keyout privkey.pem -out req.csr -config openssl.cnf
openssl x509 -inform der -in cert.cer -out cert.pem
cat cert.pem ca.pem > fullchain.pem
```

### 4. Compte de service AD pour Teleport

```powershell
$SamAccountName = "svc-teleport"
New-ADUser -Name "Teleport Service Account" -SamAccountName $SamAccountName `
  -AccountPassword $SecureStringPassword -Enabled $true
```

Permissions minimales accordées sur les objets PKI dans AD, blocage des connexions interactives via GPO.

### 5. GPO - Autoriser les connexions Teleport

- Import du certificat user-CA de Teleport dans les **Autorités de certification racines de confiance**
- Publication dans `NTAuthCertificates` via `certutil`
- Activation du **service Carte à puce** (automatique)
- Activation RDP + désactivation NLA + ouverture port TCP 3389 dans le pare-feu

### 6. Rôle Teleport pour l'accès Windows Desktop

```yaml
kind: role
version: v5
metadata:
  name: full-admin
spec:
  allow:
    windows_desktop_labels:
      "*": "*"
    windows_desktop_logins: ["Administrateur", "Admin", "Jack"]
```

```bash
tctl create -f full_admin.yaml
tctl users update JackAdmin --set-roles="editor,access,full-admin"
```

---

## Résultats

Les tests ont confirmé que Teleport permet d'identifier sans ambiguïté, pour chaque accès privilégié :

- ✅ L'utilisateur humain (compte Teleport + compte Windows)
- ✅ La session concernée (identifiant unique de session)
- ✅ La machine cible et son adresse réseau
- ✅ Les actions réalisées (enregistrement vidéo + logs structurés JSON)

Les 4 objectifs spécifiques du projet ont été atteints à **100%**.

---

## Comparatif des solutions PAM étudiées

| Critère | Teleport | StrongDM | CyberArk PAM | WALLIX PAM4ALL |
|---|---|---|---|---|
| Architecture | Proxy + agents + CA interne | Proxy central | Vault + CPM + PSM | Bastion |
| Authentification | Certificats temporaires (passwordless) | Identity-based | Mots de passe + rotation | Injection de mots de passe |
| RBAC | Basé sur certificats + TTL | Dynamique | Très granulaire | Intégré au bastion |
| Audit | Replay + logs structurés | Logs centralisés | Session recording + keystrokes | Enregistrement session |
| Open Source | ✅ Oui | ❌ Non | ❌ Non | ❌ Non |

---

## Technologies utilisées

- **Teleport** 18.1.4 (Community Edition)
- **Windows Server 2022** - Active Directory Domain Services + AD CS
- **Ubuntu Server 24.02 LTS**
- **VMware Workstation 17 Pro**
- **OpenSSL**, **PowerShell**, **tctl**
- Protocoles : RDP, LDAPS, TLS, TDP (Teleport Desktop Protocol)

---


## Auteur

**Senghor Padraic VLAVONOU**  
Licence en Informatique - Option Sécurité Informatique  
IFRI, Université d'Abomey-Calavi, Bénin

Encadrant : Ing. Vladimir HOUZANME

---

## Licence

Ce projet est partagé à des fins académiques et éducatives. Teleport est distribué sous licence Apache 2.0.
