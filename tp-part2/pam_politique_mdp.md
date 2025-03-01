# PAM - Politique de mot de passe

## Objectif  
Configurer une politique de mot de passe avec PAM respectant les contraintes suivantes :  
- 14 caractères minimum  
- Au moins : une majuscule, une minuscule, un caractère spécial et un chiffre  

## 🔧 Configuration  

### Modifications du fichier `/etc/security/pwquality.conf`  
```bash
sudo nano /etc/security/pwquality.conf
```
Modifications apportées :
```
minlen = 14
ucredit = -1
lcredit = -1
dcredit = -1
ocredit = -1
retry = 3
minclass = 4
usercheck = 1
dictcheck = 1
```

### Modifications des fichiers PAM  
#### `/etc/pam.d/system-auth`
```bash
sudo nano /etc/pam.d/system-auth
```
Ajout à la ligne "password requisite" :
```
password requisite ... minlen=14 minclass=4
```

#### `/etc/pam.d/password-auth`
```bash
sudo nano /etc/pam.d/password-auth
```
Ajout à la ligne "password requisite" :
```
password requisite ... minlen=14 minclass=4
```

## 🔎 Test de la configuration  

### Création d’un utilisateur de test
```bash
sudo adduser testuser
sudo passwd testuser
```

### Tentatives de mot de passe (résultats)  
| Mot de passe testé       | Accepté ❌/✅ |
|--------------------------|-------------|
| `abcDEF45697`             | ❌ Trop court |
| `abcdef759!@#`           | ❌ Pas de majuscule |
| `ABCdefgh!@#$%^`         | ❌ Pas de chiffre |
| `ABCdefgh456789`         | ❌ Pas de caractère spécial |
| `ABCdef123!@#xyz`        | ✅ Conforme |

### Suppression de l’utilisateur test  
```bash
sudo userdel -r testuser
```
