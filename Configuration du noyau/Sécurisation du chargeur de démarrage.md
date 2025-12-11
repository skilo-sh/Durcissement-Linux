> [!warning] Configurer un mot de passe pour le chargeur de démarrage
> 
> - _Objectifs et intérêts_ : L'objectif est de bloquer les tentatives de modification des paramètres du noyau au démarrage (ex: passage en mode *single user* ou injection d'un shell via `init=/bin/bash`). Sans cette protection, un attaquant disposant d'un accès physique ou d'un accès à la console virtuelle peut contourner l'authentification du système et obtenir les droits `root`. Le mot de passe GRUB assure que seule une personne autorisée peut modifier la séquence d'amorçage.
>     
> - _Commentaires_ : N/A
>     
> - _Procédure détaillée_ :
>     
>     1. **Générer l'empreinte du mot de passe** :
>         
>         Utiliser l'outil de hachage de GRUB (entrez le mot de passe à l'aveugle) :
>             
>         ```bash
>         grub-mkpasswd-pbkdf2
>         ```
>         _Copiez la chaîne résultante commençant par `grub.pbkdf2...`_
>             
>     2. **Modifier la configuration personnalisée** :
>         
>         Éditer le fichier `/etc/grub.d/99_custom` :
>             
>         ```bash
>         sudo nano /etc/grub.d/99_custom
>         ```
>             
>         Ajouter les lignes suivantes à la fin du fichier (remplacer `<LE_HASH>` par la chaîne copiée) :
>             
>         ```text
>         set superusers="root"
>         password_pbkdf2 root <LE_HASH>
>         ```
>                 
>     3. **Mettre à jour GRUB** :
>         
>         ```bash
>         sudo update-grub
>         ```
>         
>     4. **Vérifier la protection** :
>         
>         Au redémarrage, tenter d'éditer une entrée avec la touche `e`. Le système doit demander un identifiant (`root`) et le mot de passe.
>         
>           
> - _Comparaison avec Lynis_ : Lynis effectue le test `Checking for password protection` dans la section `Démarrage et services` . Il vérifie la présence des directives `password` ou `password_pbkdf2` dans la configuration. L'absence de mot de passe est considérée comme une vulnérabilité.
>     
> - Référence : [ANSSI_LINUX, R5]
