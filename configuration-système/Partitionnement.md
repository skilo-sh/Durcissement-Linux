Le partitionnement est la première mesure structurelle de durcissement “système” recommandée par l’ANSSI et permet, entre autres, la séparation logique des zones sensibles permettant de protéger et d'isoler les différents composants du système de fichier, la réduction de la surface d’attaque, la limitation des impacts en cas de compromission. 

> [!danger] Partionnement type
>- **Objectifs et intérêts :** L'objectif principal de cette mesure, à savoir la segmentation du système en plusieurs partitions/volumes logiques, est de réduire la surface d'attaque en isolant les zones critiques, limiter les conséquences d'un incident et empêcher l'exploitation directe de répertoires sensibles. De ce fait, il est nécessaire d'avoir un partitionnement type, comme recommandé par l'ANSSI, permettant d'assurer cet objectif (cf Tableau 1).
>- **Procédure :** 
>	- Après vérification de votre arborescence et des volumes déjà présents, vous pouvez créer un volume logique pour chaque point de montage via l'exemple générique suivant (ici, le nom de ce dernier est "vg0" et les paramètres utilisés sont arbitraires): 
>		- **Création du volume logique:**
>			 ```
>			 lvcreate -L 5G -n lv_var vg0
>			 mkfs.ext4 /dev/vg0/lv_var
>			 ```
>		- **On monte le nouveau volume et on y copie toutes les données associées  :**
>			 ```
>			 mkdir /mnt/var_new
>			 mount /dev/vg0/lv_var /mnt/var_new
>			 rsync -aXS /var/ /mnt/var_new/
>			 ```
>		- **On effectue la migration du point de montage:**
>			 ```
>			 mv /var /var.old
>		   mkdir /var
>		     ```
>		- **On modifie la configuration du fichier fstab:**
>			 ```
>			 /dev/vg0/lv_var /var ext4 defaults,nodev,nosuid,noexec 0  2
>			 ```
>		 - **Finalement, on active le volume logique:**
>			 ```
>			 mount -a 
>			 ```
>	- D'autre part, et ce pour chaque point de montage, il vous sera nécessaire de modifier le fichier de configuration **fstab** afin d'appliquer les diverses options mentionnées dans le tableau. À titre d'exemple, vous trouverez ci-dessous une ligne de configuration pour le point de montage "lv_var": 
>		```
>		 /dev/vg0/lv_var /var ext4 defaults,nodev,nosuid,noexec      0  2
>		 ```
>- **Commentaires :** Cette mesure est effectivement vitale et doit s'appliquer à tous les systèmes GNU/Linux. Néanmoins, il est important de mentionner qu'en fonction des systèmes et distributions (ainsi que les versions de ces derniers), certaines options de montage ne seront pas ou peu applicables, et il sera de fait nécessaire d'adapter le partitionnement et de mettre en place des alternatives. De plus, Lynis n'est ici d'aucune aide, étant donné qu'il ne traite pas de la sécurité des partitions/systèmes de fichiers.
>- **Référence :** Règle R28 du guide de l'ANSSI "RECOMMANDATIONS DE CONFIGURATION D'UN SYSTÈME GNU/LINUX"

**Tableau 1 :**

| Point de montage | Options                                | Description                                                                                                         |
| ---------------- | -------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| /                | <sans option>                          | Partition racine, contient le reste de l’arborescence                                                               |
| /boot            | nosuid,nodev,noexec (noauto optionnel) | Contient le noyau et le chargeur de démarrage. Pas d’accès nécessaire une fois le boot terminé (sauf mise à jour)   |
| /opt             | nosuid,nodev (ro optionnel)            | Packages additionnels au système. Montage en lecture seule si non utilisé                                           |
| /tmp             | nosuid,nodev,noexec                    | Fichiers temporaires. Ne doit contenir que des éléments non exécutables. Nettoyé après redémarrage ou de type tmpfs |
| /srv             | nosuid,nodev (noexec,ro optionnels)    | Contient des fichiers servis par un service type Web, FTP…                                                          |
| /home            | nosuid,nodev,noexec                    | Contient les HOME utilisateurs.                                                                                     |
| /proc            | hidepid=2                              | Contient des informations sur les processus et le système                                                           |
| /usr             | nodev                                  | Contient la majorité des utilitaires et fichiers système                                                            |
| /var             | nosuid,nodev,noexec                    | Fichiers variables (mails, PID, bases de données d’un service…)                                                     |
| /var/log         | nosuid,nodev,noexec                    | Contient les logs du système                                                                                        |
| /var/tmp         | nosuid,nodev,noexec                    | Fichiers temporaires conservés après extinction                                                                     |

> [!error] Restreindre les accès au dossier /boot 
> - *Objectifs et intérêts* : Le répertoire `/boot` contient les éléments les plus sensibles de la chaîne de démarrage : noyau Linux, initramfs, chargeur de démarrage et leurs configurations. Ainsi, restreindre son accès permet de prévenir l’altération du noyau, la mise en place d’un bootkit ou encore la modification de paramètres critiques du démarrage. De plus, la protection du répertoire `/boot` s’inscrit dans le principe de défense en profondeur, en empêchant un attaquant ayant un accès local non-root d’altérer le système.
>
> - *Commentaires* :
>   - Le guide de l'ANSSI recommande, **lorsque cela est compatible avec l’environnement**, de ne **pas monter automatiquement** la partition `/boot`.  
>   - Dans tous les cas, **seul l'utilisateur root** doit pouvoir lire et écrire dans le répertoire `/boot` (`chmod 700 /boot`).  
>   - Sur Debian et dérivées, `dpkg` peut être configuré pour exécuter automatiquement des commandes avant/après installation de paquets affectant `/boot`.
>
> - *Procédure détaillée* :
>   1. **Identifier la partition `/boot`** :
>      ```bash
>      lsblk -f
>      findmnt /boot
>      ```
>   2. **Vérifier les droits actuels du répertoire `/boot`** :
>      ```bash
>      ls -ld /boot
>      ```
>   3. **Restreindre les permissions de `/boot` à l'utilisateur root uniquement ** :
>      ```bash
>      chown root:root /boot
>      chmod 700 /boot
>      ```
>   4. **Configurer les options de montage sécurisées dans `/etc/fstab`** :
>      ```fstab
>      /dev/sdX1  /boot  ext4  nosuid,nodev,noexec,noauto  0  2
>      ```
>      En sachant que chaque option permet les applications suivantes :
>      - `nosuid` : interdit l’escalade via bit SUID.
>      - `nodev`  : interdit les fichiers de périphériques.
>      - `noexec` : interdit l’exécution de binaires.
>      - `noauto` : empêche le montage automatique au démarrage.
>   5. **Appliquer la configuration**:
>      ```bash
>      umount /boot
>      mount -o remount /boot
>      ```
>   6. **Vérification finale de sécurité** :
>      ```bash
>      findmnt /boot
>      ls -ld /boot
>      ```
>
> - *Comparaison avec Lynis* :
>   En ce qui concerne Lynis, ce dernier vérifie les permissions de fichiers sensibles et le paramétrage du chargeur de démarrage, mais **ne détecte pas l’usage de `noauto`**, ni la politique de montage stricte du répertoire `/boot`. De ce fait, une validation manuelle reste indispensable pour atteindre un niveau de durcissement satisfaisant.
>
> - *Référence* : ANSSI_LINUX, R29 (Recommandations de configuration d’un système GNU/Linux, section partitionnement) 
