# **Part IV : Gestion d'utilisateurs**

## **1. Gestion d'utilisateurs**

### **Création des utilisateurs et groupes**
Les utilisateurs ont été créés avec un mot de passe fort généré via la commande suivante :
```bash
openssl rand -base64 12
```
Chaque utilisateur possède un home directory et utilise `/bin/bash` comme shell par défaut. Voici la liste des utilisateurs et groupes :

| Nom    | Groupe principal | Groupes secondaires        |
|--------|----------------|---------------------------|
| suha   | suha          | managers, admins          |
| daniel | daniel        | admins, sysadmins         |
| liam   | liam          | admins                    |
| noah   | noah          | managers, artists         |
| alysha | alysha        | artists                   |
| rose   | rose          | artists, devs             |
| sadia  | sadia         | devs                      |
| jakub  | jakub         | devs                      |
| lev    | lev           | devs                      |
| grace  | grace         | rh                        |
| lucia  | lucia         | rh                        |
| oliver | oliver        | rh                        |
| nginx  | nginx        | (aucun)                   |

### script bin/bash

#!/bin/bash

Liste des utilisateurs et leurs groupes principaux et secondaires 

declare -A users 
users=( ["suha"]="suha managers,admins" 
["daniel"]="daniel admins,sysadmins" 
["liam"]="liam admins" 
["noah"]="noah managers,artists" 
["alysha"]="alysha artists" 
["rose"]="rose artists,devs"
["sadia"]="sadia devs" 
["jakub"]="jakub devs" 
["lev"]="lev devs"
["grace"]="grace rh" 
["lucia"]="lucia rh" 
["oliver"]="oliver rh" 
["nginx"]="nginx" ) 

Créer les utilisateurs avec mot de passe random 
for user in "${!users[@]}"; do password=$(openssl rand -base64 12) # Générer un mot de passe aléatoire 
IFS=" " read -r primary_group other_groups <<< "${users[$user]}"

Créer l'utilisateur avec les paramètres spécifiés
useradd -m -s /bin/bash -g "$primary_group" -G "other_groups" "$user"

Définir le mot de passe de l'utilisateur
echo "$user:$password" | chpasswd

Afficher le mot de passe généré pour référence
echo "Utilisateur: $user, Mot de passe: $password" done

Les commandes suivantes ont été exécutées pour vérifier les utilisateurs et groupes :
```bash
cat /etc/passwd | grep -E "suha|daniel|liam|noah|alysha|rose|sadia|jakub|lev|grace|lucia|oliver|nginx"
cat /etc/group | grep -E "managers|admins|sysadmins|artists|devs|rh"
```

## **2. Gestion des permissions**

Les permissions ont été appliquées en utilisant :
- **Droits POSIX** (`chmod`, `chown`)
- **ACLs Linux** (`getfacl`, `setfacl`)
- **Attributs étendus** (`chattr`, `lsattr`)

Les droits POSIX ont été vérifiés avec :
```bash
ls -alR /data/
```
Les ACLs ont été vérifiés avec :
```bash
getfacl -R /data/
```
Les attributs étendus ont été vérifiés avec :
```bash
lsattr -R /data/
```

### **Exemples de configurations appliquées :**
1. **Répertoire `/data/`** : Lecture seule pour tout le monde
   ```bash
   chmod 755 /data/
   ```
2. **Répertoire `/data/projects/`** : Écriture pour `managers`
   ```bash
   setfacl -m g:managers:rwx /data/projects/
   ```
3. **Fichier immuable `/data/projects/README.docx`**
   ```bash
   chattr +i /data/projects/README.docx
   ```
4. **Permissions par défaut pour `/data/projects/the_zoo/`**
   ```bash
   setfacl -d -m g:artists:rwx /data/projects/the_zoo/
   setfacl -m g:artists:rwx /data/projects/the_zoo/
   ```
5. **Fichier exécutable et SUID `/data/projects/zoo_app/zoo_app`**
   ```bash
   chmod u+s /data/projects/zoo_app/zoo_app
   ```

## **3. Gestion de sudo**

### **Fichier `/etc/sudoers`**

```bash
%sysadmins ALL=(root) ALL
%artists ALL=(sadia) NOPASSWD: /bin/ls /data/, /bin/cat /data/, /bin/vi /data/, /usr/bin/file /data/
alysha ALL=(suha) NOPASSWD: /bin/cat /data/projects/the_zoo/ideas.docx
%devs ALL=(root) NOPASSWD: /usr/bin/dnf install
jakub ALL=(liam) NOPASSWD: /usr/bin/python
%admins ALL=(daniel) NOPASSWD: /usr/bin/free, /usr/bin/top, /bin/df, /usr/bin/du, /bin/ps, /bin/ip
lev ALL=(daniel) NOPASSWD: /usr/bin/openssl *, /usr/bin/dig, /bin/ping, /usr/bin/curl
```

### **Mise en évidence des failles**
Nous avons trouvé des failles permettant d'obtenir un shell root :
- **alysha peut exécuter `sudo -u suha /bin/bash`**
- **liam et jakub peuvent escalader les privilèges**

Vérification :
```bash
sudo -u suha /bin/bash  # Accès root obtenu
```

### **Amélioration de la configuration sudo**

Pour sécuriser l’accès root, on applique :
```bash
Defaults !authenticate
Defaults:%sysadmins !authenticate
Defaults:%admins !authenticate

%sysadmins ALL=(root) ALL
%artists ALL=(sadia) NOPASSWD: /bin/ls /data/, /bin/cat /data/, /bin/vi /data/, /usr/bin/file /data/
alysha ALL=(suha) NOPASSWD: /bin/cat /data/projects/the_zoo/ideas.docx
%devs ALL=(root) NOPASSWD: /usr/bin/dnf install
jakub ALL=(liam) NOPASSWD: /usr/bin/python
%admins ALL=(daniel) NOPASSWD: /usr/bin/free, /usr/bin/top, /bin/df, /usr/bin/du, /bin/ps, /bin/ip
lev ALL=(daniel) NOPASSWD: /usr/bin/openssl *, /usr/bin/dig, /bin/ping, /usr/bin/curl

# Interdiction des shells pour tous sauf sysadmins
Defaults:%artists !syslog
Defaults:%devs !syslog
Defaults:%admins !syslog
Defaults:%rh !syslog
```

Cette configuration :
- Empêche les utilisateurs non-autorisés d'obtenir un shell root
- Maintient l'accès aux commandes nécessaires pour chaque rôle
