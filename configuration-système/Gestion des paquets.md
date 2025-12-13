La gestion des paquets constitue une étape critique de la sécurisation d’un système Debian GNU/Linux. Les paquets installés déterminent directement les fichiers présents sur le système ainsi que les services et fonctionnalités exposés. Conformément aux recommandations du guide de l’ANSSI, seuls les paquets strictement nécessaires doivent être installés et maintenus dans le temps, afin de limiter la surface d’attaque et de garantir un système maîtrisé et durablement sécurisé.

> [!error] N’installer que les paquets strictement nécessaires
> - *Objectifs et intérêts* : Réduire la surface d’attaque du système en limitant le nombre de paquets installés au strict nécessaire. Chaque paquet supplémentaire introduit du code, des dépendances, des fichiers et parfois des services actifs, augmentant le risque de vulnérabilités, de mauvaises configurations ou de services exposés inutilement.
>
> - *Commentaires* :
>   - Une installation minimaliste facilite la maintenance, l’audit et la mise à jour du système.
>   - Les environnements serveur n’ont généralement pas besoin d’interface graphique locale (X11, Wayland, gestionnaires de fenêtres).
>   - Les « rôles » ou profils d’installation préconfigurés proposés par certaines distributions sont déconseillés : ils reflètent des choix génériques des mainteneurs, rarement adaptés à un contexte métier précis.
>
> - *Procédure détaillée* :
>   1. **Installer le système en mode minimal** :
>      - Lors de l’installation Debian, sélectionner *« Installation minimale »*.
>      - Ne pas installer d’environnement graphique.
>
>   2. **Lister les paquets installés** :
>      ```bash
>      apt list --installed
>      ```
>
>   3. **Identifier les paquets non nécessaires** :
>      - Paquets graphiques :
>        ```bash
>        dpkg -l | grep -E "xserver|xorg|wayland|gnome|kde|xfce"
>        ```
>      - Paquets serveurs inutiles (exemples) :
>        ```bash
>        dpkg -l | grep -E "avahi|cups|rpcbind"
>        ```
>
>   4. **Supprimer les paquets inutiles** :
>      ```bash
>      apt purge <paquet>
>      ```
>
>   5. **Supprimer les dépendances devenues orphelines** :
>      ```bash
>      apt autoremove --purge
>      ```
>
>   6. **Empêcher l’installation automatique de paquets recommandés** :
>      ```bash
>      echo 'APT::Install-Recommends "false";' > /etc/apt/apt.conf.d/99norecommends
>      echo 'APT::Install-Suggests "false";' >> /etc/apt/apt.conf.d/99norecommends
>      ```
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler la présence de certains paquets à risque ou inutiles (ex. environnement graphique sur serveur), mais il ne dispose pas de la connaissance du besoin métier. La validation de cette règle repose donc sur une analyse fonctionnelle manuelle.
>
> - *Référence* : [ANSSI_LINUX, R58]

> [!error] Utiliser uniquement des dépôts de paquets de confiance
> - *Objectifs et intérêts* : Garantir l’intégrité et l’authenticité des paquets installés sur le système. L’utilisation de dépôts non officiels ou non maîtrisés expose à l’installation de paquets malveillants, modifiés ou insuffisamment maintenus.
>
> - *Commentaires* :
>   - Seuls les dépôts officiels de la distribution ou les dépôts internes à l’organisation doivent être utilisés.
>   - Les dépôts tiers ajoutés sans justification claire constituent un risque majeur.
>
> - *Procédure détaillée* :
>   1. **Lister les dépôts configurés** :
>      ```bash
>      grep -R ^deb /etc/apt/sources.list /etc/apt/sources.list.d/
>      ```
>
>   2. **Supprimer les dépôts non officiels** :
>      - Éditer ou supprimer les fichiers concernés :
>        ```bash
>        rm /etc/apt/sources.list.d/<depot>.list
>        ```
>
>   3. **Vérifier les clés de signature utilisées** :
>      ```bash
>      apt-key list
>      ```
>      *(ou via `/etc/apt/trusted.gpg.d/` pour les versions récentes)*
>
>   4. **Mettre à jour l’index des paquets** :
>      ```bash
>      apt update
>      ```
>
> - *Comparaison avec Lynis* :
>   Lynis détecte la présence de dépôts tiers ou non signés, mais ne valide pas la légitimité organisationnelle d’un dépôt interne. Une revue humaine reste nécessaire.
>
> - *Référence* : [ANSSI_LINUX, R59]

> [!info] Utiliser des dépôts de paquets durcis
> - *Objectifs et intérêts* : Privilégier les paquets intégrant des mécanismes de durcissement supplémentaires (options de compilation sécurisées, correctifs spécifiques, configurations par défaut restrictives) afin de réduire l’impact potentiel d’une vulnérabilité.
>
> - *Commentaires* :
>   - Certaines distributions proposent des variantes de paquets plus durcies ou mieux maintenues.
>   - Les miroirs de dépôts doivent être des copies conformes, à jour, et issus de sources officielles.
>
> - *Procédure détaillée* :
>   1. **Utiliser les dépôts officiels de sécurité** (Debian) :
>      Vérifier la présence de :
>      ```
>      deb http://security.debian.org/ <release>-security main
>      ```
>
>   2. **Préférer les paquets maintenus par la distribution** plutôt que compilés manuellement.
>
>   3. **Comparer les paquets fournissant un même service** :
>      ```bash
>      apt-cache search <service>
>      apt-cache show <paquet>
>      ```
>
>   4. **Éviter les miroirs non officiels ou obsolètes** :
>      - Vérifier la synchronisation du miroir utilisé.
>
>   5. **Maintenir les paquets à jour** :
>      ```bash
>      apt update
>      apt upgrade
>      ```
>
> - *Comparaison avec Lynis* :
>   Lynis ne distingue pas le niveau de durcissement interne des paquets ni leurs options de compilation. Cette recommandation repose sur une connaissance approfondie de la distribution et des paquets utilisés.
>
> - *Référence* : [ANSSI_LINUX, R60]
