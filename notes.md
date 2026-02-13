Voici une commande simple et complète pour monter un partage Windows :

```bash
mount -t cifs //192.168.1.100/partage /mnt/shared-drive \
  -o username=utilisateur,password=motdepasse,vers=3.0
```

## Décomposition

- `//192.168.1.100/partage` → Votre serveur Windows et nom du partage
- `/mnt/shared-drive` → Point de montage (créez le dossier avant si nécessaire)
- `username=utilisateur` → Votre nom d'utilisateur Windows
- `password=motdepasse` → Votre mot de passe
- `vers=3.0` → Version SMB (3.0 pour Windows récents)

## Commande complète avec création du dossier

```bash
mkdir -p /mnt/shared-drive && \
mount -t cifs //192.168.1.100/partage /mnt/shared-drive \
  -o username=utilisateur,password=motdepasse,vers=3.0
```

## Avec domaine Active Directory

```bash
mount -t cifs //serveur/partage /mnt/shared-drive \
  -o username='DOMAINE\utilisateur',password=motdepasse,vers=3.0
```

## Version robuste (gestion des erreurs)

```bash
mount -t cifs //192.168.1.100/partage /mnt/shared-drive \
  -o username=utilisateur,password=motdepasse,vers=3.0,uid=1000,gid=1000,file_mode=0777,dir_mode=0777
```

Les options `uid`, `gid`, `file_mode`, `dir_mode` assurent que votre application .NET peut lire/écrire les fichiers.

Remplacez simplement l'IP, le nom du partage et les credentials par les vôtres !
