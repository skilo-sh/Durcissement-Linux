Le contrôle d’accès vise à garantir que chaque compte ne dispose que des **droits strictement nécessaires** pour accéder aux ressources du système. Il constitue un pilier fondamental du durcissement, en permettant de **limiter l’impact d’une compromission** par le principe du moindre privilège.  
Sous GNU/Linux, ce contrôle repose historiquement sur la **gestion des utilisateurs et des groupes**, complétée par des mécanismes plus avancés tels que les **LSM (SELinux, AppArmor)**, le **filtrage système (seccomp)** et les **mécanismes de cloisonnement** (conteneurs, compatibilité UNIX). L’ensemble de ces dispositifs concourt à renforcer l’isolement des processus et à réduire la surface d’attaque globale du système.

## Modèle traditionnel Unix 
Le modèle de sécurité traditionnel d’Unix repose sur le contrôle des accès par **utilisateurs (UID)** et **groupes (GID)**. Chaque ressource du système (fichier, processus, répertoire…) possède un propriétaire et des droits d’accès définis selon trois niveaux : **propriétaire, groupe et autres**. Ce mécanisme correspond à un **contrôle d’accès discrétionnaire (DAC)**, dans lequel le propriétaire décide des permissions en lecture, écriture et exécution. Par défaut, ces droits sont influencés par le **UMASK**, souvent trop permissif (0022), ce qui justifie un durcissement systématique, notamment pour les shells utilisateurs et les services, afin de limiter l’exposition du système.

> [!info] Modifier la valeur par défaut de l’UMASK
> - *Objectifs et intérêts* : L’UMASK définit les permissions par défaut appliquées lors de la création de fichiers et de répertoires. Une valeur trop permissive peut provoquer des fuites de données ou des modifications non autorisées. Positionner une UMASK restrictive permet d’appliquer automatiquement le principe de moindre privilège dès la création des ressources.
>
> - *Commentaires* :
>   - Pour les **sessions utilisateurs (shells)**, une UMASK à `0077` garantit que seuls les droits du propriétaire sont autorisés (lecture, écriture).
>   - Pour les **services**, une UMASK à `0027` permet :
>     - Lecture + écriture pour le propriétaire,
>     - Lecture pour le groupe,
>     - Aucun accès pour les autres.
>   - Certains services nécessitent une UMASK spécifique selon leur fonctionnement (journaux, partages, sockets), d’où l’étude **au cas par cas** recommandée.
>   - Sous systemd, la valeur de l’UMASK peut être forcée par service via la directive `UMask=`.
>
> - *Procédure détaillée* :
>   1. **Configurer l’UMASK globale des shells** :
>      - Éditer le fichier `/etc/profile` :
>        ```bash
>        umask 0077
>        ```
>      - Vérifier également :
>        - `/etc/bash.bashrc`
>        - `/etc/login.defs` (`UMASK 077`)
>   2. **Vérifier l’UMASK effective utilisateur** :
>      ```bash
>      umask
>      ```
>      - La valeur attendue est `0077`.
>   3. **Configurer l’UMASK des services systemd** :
>      - Éditer ou surcharger une unité :
>        ```ini
>        [Service]
>        UMask=0027
>        ```
>      - Puis recharger :
>        ```bash
>        systemctl daemon-reexec
>        systemctl restart <service>
>        ```
>   4. **Contrôler les fichiers créés** :
>      - Vérifier les permissions effectives des fichiers générés par les services.
>      - Ajuster l’UMASK si un service nécessite un accès groupe plus large.
>
> - *Comparaison avec Lynis* :
>   Lynis vérifie certaines permissions de fichiers sensibles, mais **ne contrôle pas systématiquement les UMASK par défaut des shells ni celles définies dans les unités systemd**. La conformité complète à cette règle nécessite donc une vérification manuelle.
>
> - *Référence* : [ANSSI_LINUX, R?] – Gestion des droits par défaut (UMASK) :contentReference[oaicite:0]{index=0}

> [!info] Utiliser des fonctionnalités de contrôle d'accès obligatoire (MAC)
> - *Objectifs et intérêts* : Compléter le modèle de contrôle d’accès discrétionnaire (DAC) historique des systèmes Unix par un contrôle d’accès obligatoire (MAC) afin d’imposer une politique de sécurité centralisée, non contournable par les utilisateurs. Le MAC permet de réduire fortement l’impact d’une compromission applicative, de limiter la surface d’attaque et d’assurer un cloisonnement strict entre processus, même appartenant au même utilisateur.
>
> - *Commentaires* :
>   - Le modèle **DAC (Discretionary Access Control)** reste aujourd’hui **majoritaire et appliqué par défaut**, mais présente plusieurs **limitations structurantes** :
>     - Les droits sont laissés **à la discrétion du propriétaire**, ce qui est incompatible avec des politiques de sécurité fortes.
>     - Le propriétaire peut **mal configurer les permissions**, entraînant des accès non autorisés à des données sensibles.
>     - L’**isolation entre utilisateurs est grossière**, les droits par défaut étant souvent trop permissifs.
>     - Il n’existe que **deux niveaux de privilèges** (utilisateur standard et root), l’élévation passant par des binaires à privilèges spéciaux (*setuid root*).
>     - Il est **impossible d’adapter finement les privilèges d’un processus selon son contexte d’exécution** (un navigateur et un éditeur texte d’un même utilisateur ont les mêmes droits).
>     - La **surface d’attaque est élevée**, de nombreuses informations système restant accessibles aux utilisateurs standards.
>   - Le **MAC (Mandatory Access Control)** impose au contraire des règles **globales, centralisées et non modifiables par l’utilisateur**, au prix d’une **complexité et d’un coût de déploiement plus élevés**.
>
> - *Procédure détaillée* :
>   1. **Choisir un mécanisme MAC** :
>      - **AppArmor** (profilage par chemins, plus simple à maintenir).
>      - **SELinux** (contrôles par labels, plus fin mais plus complexe).
>   2. **Vérifier l’activation du LSM** :
>      ```bash
>      cat /sys/kernel/security/lsm
>      ```
>   3. **Activation d’AppArmor (Debian/Ubuntu)** :
>      ```bash
>      apt install apparmor apparmor-utils
>      systemctl enable apparmor
>      systemctl start apparmor
>      aa-status
>      ```
>   4. **Activation de SELinux (si utilisé)** :
>      - Installer les outils SELinux.
>      - Configurer `/etc/selinux/config`.
>      - Redémarrer le système.
>   5. **Appliquer les profils aux services sensibles** :
>      - Serveur web, bases de données, services réseau, outils d’administration.
>      - Vérifier les profils fournis et basculer progressivement en mode enforce.
>   6. **Surveiller les violations MAC** :
>      - Analyse des journaux (`dmesg`, `journalctl`, logs AppArmor/SELinux).
>      - Ajustement progressif des profils.
>
> - *Comparaison avec Lynis* :
>   Lynis détecte la présence d’AppArmor ou de SELinux, ainsi que leur état (enforcing ou non), mais **ne valide ni la qualité des profils, ni la cohérence de la politique de cloisonnement appliquée**. Une analyse humaine reste indispensable pour garantir l’efficacité réelle du MAC.
>
> - *Référence* : [ANSSI_LINUX, R? – Contrôle d’accès obligatoire] :contentReference[oaicite:0]{index=0}

> [!warning] Créer un groupe dédié à l’usage de sudo
> - *Objectifs et intérêts* : Restreindre strictement l’usage de la commande `sudo` à un groupe d’utilisateurs explicitement autorisés permet de limiter drastiquement les possibilités d’élévation de privilèges. Cette approche renforce le principe de moindre privilège en s’assurant que seuls les utilisateurs ayant un besoin opérationnel légitime peuvent exécuter des commandes avec des droits administrateur.
>
> - *Commentaires* :
>   - Par défaut, certaines distributions autorisent l’usage de `sudo` via des groupes génériques (`sudo`, `wheel`, etc.), parfois avec des politiques trop permissives.
>   - La création d’un **groupe dédié spécifique** permet une gestion plus fine et plus lisible des droits d’élévation.
>   - Le binaire `/usr/bin/sudo` étant **setuid root**, toute mauvaise configuration de ses permissions constitue un risque critique.
>   - Les permissions du binaire `sudo` peuvent être **écrasées lors des mises à jour du paquet**, ce qui impose une vérification régulière ou une automatisation.
>
> - *Procédure détaillée* :
>   1. **Créer un groupe dédié à sudo** :
>      ```bash
>      groupadd sudogrp
>      ```
>   2. **Ajouter les utilisateurs autorisés dans ce groupe** :
>      ```bash
>      usermod -aG sudogrp <utilisateur>
>      ```
>   3. **Modifier les droits du binaire sudo** :
>      ```bash
>      chown root:sudogrp /usr/bin/sudo
>      chmod 4750 /usr/bin/sudo
>      ```
>      Ce qui donne :
>      ```
>      -rwsr-x--- 2 root sudogrp [...] /usr/bin/sudo
>      ```
>   4. **Configurer proprement sudo via visudo** :
>      ```bash
>      visudo
>      ```
>      Puis autoriser uniquement le groupe dédié :
>      ```
>      %sudogrp ALL=(ALL:ALL) ALL
>      ```
>   5. **Tester le bon fonctionnement** :
>      - Tester avec un utilisateur membre du groupe.
>      - Tester avec un utilisateur non membre (l’accès doit être refusé).
>
> - *Automatisation post-mise à jour* :
>   - Il est recommandé d’ajouter un script `dpkg` de post-installation pour réappliquer automatiquement :
>     ```bash
>     chown root:sudogrp /usr/bin/sudo
>     chmod 4750 /usr/bin/sudo
>     ```
>
> - *Comparaison avec Lynis* :
>   Lynis vérifie la présence de `sudo`, la configuration basique de `/etc/sudoers` et certaines permissions, mais **ne contrôle pas l’existence d’un groupe sudo dédié**, ni la stratégie de restriction par groupe personnalisé. Cette mesure nécessite donc une validation manuelle.
>
> - *Référence* : [ANSSI_LINUX, R?]

> [!warning] Modifier les directives de configuration sudo
> - *Objectifs et intérêts* : L’outil `sudo` est un point critique de la chaîne de privilèges. Ses valeurs par défaut sont historiquement permissives et peuvent faciliter l’évasion de restrictions, l’exploitation via variables d’environnement ou l’exécution de commandes détournées. Le durcissement des directives globales permet de réduire drastiquement les surfaces d’attaque liées à l’escalade de privilèges.
>
> - *Commentaires* :
>   - Les valeurs **par défaut de sudo sont insuffisamment restrictives** dans un contexte de durcissement.
>   - Toute **surcharge locale** (par utilisateur ou groupe) doit être **exceptionnelle, documentée et analysée en terme de risque**.
>   - La lecture complète de `man sudoers` est **indispensable**, notamment la section sur les **échappements shell**.
>   - Plusieurs vulnérabilités historiques de sudo sont directement liées :
>     - à une mauvaise gestion des UID spéciaux (`-1`),
>     - aux échappements de commandes et débordements mémoire.
>
> - *Procédure détaillée* :
>   1. **Éditer le fichier sudoers uniquement via visudo** :
>      ```bash
>      visudo
>      ```
>   2. **Activer les directives de durcissement globales** :
>      ```bash
>      Defaults noexec,requiretty,use_pty,umask=0027
>      Defaults ignore_dot,env_reset
>      ```
>   3. **Effet des directives** :
>      - `noexec` : empêche l’exécution de sous-commandes via `exec`, `bash`, `sh`, etc.
>      - `requiretty` : impose un terminal interactif pour l’exécution via sudo.
>      - `use_pty` : force l’usage d’un pseudo-terminal pour tracer précisément les commandes.
>      - `umask=0027` : restreint fortement les permissions des fichiers créés via sudo.
>      - `ignore_dot` : empêche l’usage du répertoire courant (`.`) dans `$PATH`.
>      - `env_reset` : purge les variables d’environnement dangereuses.
>   4. **Surcharges contrôlées (si nécessaire)** :
>      - Pour un utilisateur :
>        ```
>        myuser ALL=EXEC: /usr/bin/mycommand
>        ```
>      - Pour un groupe :
>        ```
>        Defaults:%admins !noexec
>        ```
>   5. **Vérifier le bon fonctionnement** :
>      - Tester l’exécution de commandes simples.
>      - Vérifier l’impossibilité d’ouvrir un shell via sudo (`sudo bash`, `sudo sh`).
>      - Contrôler les permissions des fichiers créés via sudo.
>
> - *Comparaison avec Lynis* :
>   Lynis détecte certaines configurations dangereuses de sudo (droits trop larges, NOPASSWD excessif), mais **ne force pas l’activation systématique de `noexec`, `use_pty`, `ignore_dot` ou `requiretty`**. La mise en conformité complète avec cette règle nécessite donc une **intervention manuelle experte**.
>
> - *Référence* : [ANSSI_LINUX, R39] – Configuration de sudo :contentReference[oaicite:0]{index=0}

> [!warning] Utiliser des utilisateurs cibles non-privilégiés pour les commandes sudo
> - *Objectifs et intérêts* : L’objectif est de limiter les risques d’escalade de privilèges lors de l’utilisation de `sudo`. Lorsque la commande à exécuter ne nécessite pas de droits élevés, elle ne doit pas être exécutée avec les privilèges de `root`. L’utilisation d’utilisateurs cibles non privilégiés réduit l’impact d’un abus de `sudo`, d’une mauvaise configuration ou de fonctionnalités détournées de certaines commandes.
>
> - *Commentaires* :
>   - De nombreuses actions administratives courantes ne nécessitent pas de privilèges `root` (édition de fichiers appartenant à un autre utilisateur, envoi de signaux à des processus non privilégiés, gestion de services applicatifs locaux, etc.).
>   - L’exécution systématique de commandes via `sudo` en tant que `root` augmente inutilement la surface d’attaque.
>   - Certaines commandes dites *fonctionnellement riches* (éditeurs de texte comme `vi`, `vim`, `less`, interpréteurs, outils interactifs) permettent l’exécution de sous-commandes ou l’ouverture de shells.
>   - Même si `sudo` permet de surcharger certaines fonctions pour limiter ces comportements, ces mécanismes restent **imparfaits** et ne constituent pas une protection robuste contre les contournements.
>
> - *Procédure détaillée* :
>   1. **Identifier les usages de sudo** :
>      - Analyser le fichier `/etc/sudoers` et les fichiers présents dans `/etc/sudoers.d/`.
>      - Identifier les règles où la commande est exécutée en tant que `root` sans justification fonctionnelle.
>   2. **Créer des utilisateurs cibles dédiés non privilégiés** :
>      - Créer des comptes techniques spécifiques aux tâches concernées (ex. : `backup`, `deploy`, `logreader`).
>      - S’assurer qu’ils ne disposent pas de shell interactif inutile (`/usr/sbin/nologin` si applicable).
>   3. **Adapter les règles sudo** :
>      - Privilégier les formes :
>        ```
>        utilisateur ALL=(utilisateur_cible) /chemin/commande
>        ```
>        plutôt que :
>        ```
>        utilisateur ALL=(root) /chemin/commande
>        ```
>   4. **Éviter les commandes à fort potentiel d’abus** :
>      - Éviter autant que possible l’usage de `sudo` avec des éditeurs (`vi`, `vim`, `nano`), interpréteurs (`python`, `bash`, `perl`) ou outils interactifs.
>      - Préférer des commandes simples, non interactives, avec chemins absolus.
>   5. **Tester et valider** :
>      - Vérifier que les actions prévues sont réalisables sans privilèges root.
>      - Tester les règles sudo avec `sudo -l` et en conditions réelles.
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler des configurations sudo trop permissives ou des usages dangereux évidents, mais il **ne vérifie pas systématiquement la pertinence du compte cible** (`root` vs utilisateur non privilégié). Une revue manuelle des règles sudo reste indispensable pour appliquer correctement cette recommandation.
>
> - *Référence* : [ANSSI_LINUX, R40] :contentReference[oaicite:0]{index=0}

> [!info] Limiter l’utilisation de commandes nécessitant la directive EXEC
> - *Objectifs et intérêts* : Les mécanismes de restriction basés sur des listes de commandes autorisées (ex. sudo, RBAC, wrappers) peuvent être contournés si un processus est autorisé à exécuter librement des appels système comme `execve`. Une commande disposant du droit EXEC peut lancer des sous-processus arbitraires (shell, interpréteur, binaire), contournant ainsi les contrôles initiaux. Limiter strictement l’usage de la directive EXEC permet de réduire les possibilités de contournement, d’escalade de privilèges et d’exécution de code non prévu.
>
> - *Commentaires* :
>   - Toute commande autorisée à s’exécuter avec des privilèges élevés doit être considérée comme **potentiellement capable d’exécuter du code arbitraire**.
>   - Les commandes capables de lancer un shell (`bash`, `sh`, `python`, `perl`, `awk`, `find -exec`, `vi`, `less`, etc.) doivent être **explicitement interdites** dans les contextes restreints.
>   - Les restrictions doivent être appliquées **au plus près du mécanisme de contrôle** (sudo, systemd, RBAC), et non uniquement au niveau applicatif.
>
> - *Procédure détaillée* :
>   1. **Identifier les commandes autorisées avec EXEC** :
>      - Lister les règles sudo existantes :
>        ```bash
>        sudo -l
>        ```
>      - Examiner les fichiers :
>        ```bash
>        /etc/sudoers
>        /etc/sudoers.d/*
>        ```
>   2. **Réduire explicitement la liste des commandes autorisées** :
>      - Autoriser uniquement les binaires strictement nécessaires, avec **chemin absolu** :
>        ```
>        utilisateur ALL=(root) /usr/bin/systemctl restart nginx
>        ```
>      - Interdire toute commande générique :
>        ```
>        ALL, !/bin/sh, !/bin/bash, !/usr/bin/python*, !/usr/bin/perl, !/usr/bin/find
>        ```
>   3. **Supprimer toute possibilité de shell implicite** :
>      - Refuser les commandes interprétées :
>        ```bash
>        visudo
>        ```
>        Vérifier l’absence de :
>        - éditeurs (`vi`, `nano`)
>        - interpréteurs (`python`, `ruby`, `perl`)
>        - commandes avec `-exec`, `-p`, `!`, ou équivalents
>   4. **Restreindre l’environnement d’exécution** :
>      - Forcer un environnement minimal dans sudo :
>        ```
>        Defaults secure_path="/usr/sbin:/usr/bin:/sbin:/bin"
>        Defaults !env_reset
>        Defaults env_delete+="LD_PRELOAD LD_LIBRARY_PATH PYTHONPATH"
>        ```
>   5. **Limiter EXEC via systemd (si applicable)** :
>      - Dans les unités `*.service` :
>        ```
>        NoNewPrivileges=yes
>        RestrictSUIDSGID=yes
>        MemoryDenyWriteExecute=yes
>        SystemCallFilter=@system-service
>        ```
>      - Interdire explicitement `execve` si possible :
>        ```
>        SystemCallFilter=~execve
>        ```
>   6. **Tester les restrictions** :
>      - Vérifier qu’aucune commande autorisée ne permet :
>        ```bash
>        sudo /commande_autorisée
>        ```
>        d’ouvrir un shell ou d’exécuter un binaire arbitraire.
>
> - *Comparaison avec Lynis* :
>   Lynis détecte certaines configurations sudo dangereuses (wildcards, permissions trop larges), mais **ne vérifie pas finement les possibilités de contournement via EXEC**, ni les capacités réelles d’une commande autorisée à engendrer des sous-processus. Une analyse manuelle approfondie est donc indispensable pour cette règle.
>
> - *Référence* : [ANSSI_LINUX, R41] – Recommandations de configuration d’un système GNU/Linux :contentReference[oaicite:0]{index=0}

> [!warning] Bannir les négations dans les spécifications sudo
> - *Objectifs et intérêts* :  
>   Les règles sudo utilisant des **négations** (approche par liste d’interdiction) sont inefficaces et facilement contournables.  
>   Sudo évalue les droits via une **comparaison de chaînes** (globbing shell) sur le chemin de la commande et ses arguments. Toute règle autorisant `ALL` tout en niant un binaire précis permet à un attaquant de contourner la restriction (copie, renommage, appel indirect).  
>   L’objectif est d’appliquer une **approche par liste blanche stricte**, en n’autorisant explicitement que les commandes nécessaires.
>
> - *Commentaires* :
>   - Une règle de type :
>     ```
>     user ALL=(ALL) ALL, !/bin/sh
>     ```
>     est **trivialement contournable** (copie de `/bin/sh`, lien symbolique, appel via un autre binaire).
>   - Les arguments font partie de l’évaluation sudo : une commande autorisée avec arguments non restreints peut être détournée.
>   - Toute utilisation de `ALL` dans les spécifications de commandes est à considérer comme **dangereuse** hors contexte maîtrisé.
>
> - *Procédure détaillée* :
>   1. **Auditer les règles sudo existantes** :
>      ```bash
>      sudo cat /etc/sudoers
>      sudo ls /etc/sudoers.d/
>      sudo grep -R "!" /etc/sudoers /etc/sudoers.d/
>      ```
>      - Identifier toute règle contenant `!commande`.
>
>   2. **Supprimer les négations** :
>      - Éditer les fichiers via `visudo` uniquement :
>        ```bash
>        sudo visudo
>        ```
>        ou :
>        ```bash
>        sudo visudo -f /etc/sudoers.d/<fichier>
>        ```
>      - Supprimer toute syntaxe utilisant `!`.
>
>   3. **Mettre en place une liste blanche stricte** :
>      - Exemple **correct** :
>        ```
>        user ALL=(root) /usr/bin/systemctl restart apache2
>        ```
>      - Restreindre :
>        - le **chemin absolu**
>        - l’**utilisateur cible**
>        - les **arguments autorisés**
>
>   4. **Restreindre les alias dangereux** :
>      - Éviter les alias de commandes trop larges :
>        ```
>        Cmnd_Alias SERVICES = /usr/bin/systemctl *
>        ```
>      - Préférer :
>        ```
>        Cmnd_Alias APACHE = /usr/bin/systemctl restart apache2
>        ```
>
>   5. **Tester les règles** :
>      ```bash
>      sudo -l -U <utilisateur>
>      ```
>      - Vérifier que seules les commandes explicitement autorisées sont exécutables.
>
> - *Comparaison avec Lynis* :  
>   Lynis peut signaler une configuration sudo faible ou trop permissive, mais **ne détecte pas systématiquement les contournements liés aux négations**, ni les problèmes de globbing sur les arguments. Une revue manuelle des règles sudo est indispensable.
>
> - *Référence* : [ANSSI_LINUX, R42] :contentReference[oaicite:0]{index=0}

> [!warning] Préciser strictement les arguments dans les règles sudo
> - *Objectifs et intérêts* : Les arguments passés à une commande peuvent modifier profondément son comportement (lecture, écriture, suppression de fichiers, accès à des ressources sensibles). Une règle sudo trop permissive permet à un utilisateur de détourner une commande légitime pour exécuter des actions arbitraires avec des privilèges élevés. Spécifier strictement les arguments autorisés réduit fortement les risques d’escalade de privilèges et limite l’impact d’un abus de sudo.
>
> - *Commentaires* :
>   - Une règle sudo **sans arguments spécifiés** autorise implicitement **tous les arguments**, ce qui est dangereux.
>   - L’usage de jokers (`*`) doit être évité autant que possible, car il ouvre la porte à des détournements subtils.
>   - L’absence d’arguments doit être **explicitement indiquée** par une chaîne vide (`""`).
>   - Autoriser via sudo un programme capable d’écrire arbitrairement sur le système (éditeur de texte, shell, utilitaire de copie mal restreint) équivaut dans les faits à donner les privilèges root complets.
>
> - *Procédure détaillée* :
>   1. **Auditer les règles sudo existantes** :
>      ```bash
>      sudo visudo -c
>      sudo grep -R "" /etc/sudoers /etc/sudoers.d/
>      ```
>      - Identifier les règles :
>        - sans arguments,
>        - avec jokers (`*`),
>        - appelant des outils polyvalents (`cp`, `mv`, `tar`, `vim`, `less`, `awk`, `perl`, `python`, `sh`, etc.).
>
>   2. **Restreindre strictement les arguments autorisés** :
>      - Mauvais exemple (trop permissif) :
>        ```
>        user ALL=(root) /bin/dmesg
>        ```
>      - Bon exemple (arguments explicitement interdits) :
>        ```
>        user ALL=(root) /bin/dmesg ""
>        ```
>      → empêche l’utilisation de `--file` pour lire des fichiers arbitraires.
>
>   3. **Éviter les jokers (`*`)** :
>      - À éviter :
>        ```
>        user ALL=(root) /bin/cat *
>        ```
>      - À préférer :
>        ```
>        user ALL=(root) /bin/cat /var/log/syslog
>        ```
>
>   4. **Encadrer strictement les commandes d’édition** :
>      - ❌ Interdit :
>        ```
>        user ALL=(root) /usr/bin/vim /etc/*
>        ```
>      - ✅ Alternative sûre :
>        - Utiliser un wrapper dédié :
>          ```bash
>          /usr/local/sbin/edit_conf.sh
>          ```
>        - Script contrôlant précisément le fichier modifiable :
>          ```bash
>          #!/bin/sh
>          exec /usr/bin/nano /etc/mon_fichier.conf
>          ```
>        - Puis règle sudo :
>          ```
>          user ALL=(root) /usr/local/sbin/edit_conf.sh ""
>          ```
>
>   5. **Toujours utiliser des chemins absolus** :
>      - Interdire toute ambiguïté sur la commande exécutée :
>        ```
>        user ALL=(root) /bin/systemctl restart nginx.service
>        ```
>
>   6. **Tester et valider** :
>      ```bash
>      sudo -l -U user
>      sudo -u root /bin/dmesg --file /etc/shadow   # doit échouer
>      sudo -u root /bin/dmesg                       # doit fonctionner
>      ```
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler la présence de sudo et certaines règles dangereuses, mais **ne valide pas finement la cohérence et la sûreté des arguments autorisés**. La conformité à cette recommandation nécessite une revue manuelle détaillée des fichiers sudoers.
>
> - *Référence* : [ANSSI_LINUX, R43]

## AppArmor
Dans le cadre du durcissement d’un système GNU/Linux selon les recommandations de l’ANSSI, les mécanismes de contrôle d’accès jouent un rôle central. Les systèmes Linux modernes s’appuient notamment sur des _Linux Security Modules_ (LSM) pour renforcer le modèle de permissions Unix traditionnel.  
Parmi eux, **AppArmor** constitue un mécanisme de contrôle d’accès obligatoire (_Mandatory Access Control – MAC_) largement déployé, reposant sur des profils associés aux exécutables et définissant précisément les ressources accessibles. Ce modèle permet à une autorité de sécurité de contraindre le comportement des applications indépendamment de leur volonté, tout en restant relativement simple à déployer et à maintenir.  
En complément du contrôle d’accès aux fichiers, AppArmor offre des possibilités de restriction sur l’usage des capacités POSIX et des accès réseau, contribuant ainsi à la réduction de la surface d’attaque du système, conformément aux principes de sécurité préconisés par l’ANSSI.

> [!info] Activer et faire respecter les profils de sécurité AppArmor
> - *Objectifs et intérêts* : AppArmor permet de restreindre finement les droits des exécutables via des profils de confinement basés sur des chemins. Activer systématiquement les profils AppArmor en mode *enforce* permet de limiter l’impact d’une compromission applicative (lecture/écriture de fichiers, exécution de binaires, accès réseau), y compris pour des services sensibles comme `syslogd`, `ntpd`, ou des démons réseau. Cette mesure renforce la défense en profondeur en imposant un contrôle d’accès obligatoire (MAC) indépendant des permissions Unix classiques.
>
> - *Commentaires* :
>   - AppArmor applique des profils **par exécutable**, stockés dans `/etc/apparmor.d/`, avec une gestion explicite des **transitions de profils** lors de l’appel d’autres exécutables (héritage, changement de profil, sortie du confinement).
>   - Contrairement à SELinux, AppArmor ne repose pas sur des labels de sécurité : il se concentre exclusivement sur le contrôle des accès des exécutables.
>   - Les profils peuvent être chargés en :
>     - **enforce** : les accès interdits sont bloqués ;
>     - **complain** : les accès interdits sont uniquement journalisés.
>   - Les refus et violations AppArmor sont journalisés via `auditd` ou le journal système (`journald`).
>
> - *Procédure détaillée* :
>   1. **Vérifier qu’AppArmor est actif** :
>      ```bash
>      aa-status
>      ```
>      Résultat attendu :
>      - `apparmor module is loaded`
>      - Présence de profils en *enforce mode*
>
>   2. **Lister les profils chargés et leur mode** :
>      ```bash
>      aa-status --verbose
>      ```
>      - Identifier les profils en *complain* ou les processus **non confinés** disposant d’un profil.
>
>   3. **Activer tous les profils existants en mode enforce** :
>      ```bash
>      sudo aa-enforce /etc/apparmor.d/*
>      ```
>      - Cette commande bascule automatiquement tous les profils disponibles en mode *enforce*.
>
>   4. **Activer AppArmor au démarrage (si nécessaire)** :
>      ```bash
>      systemctl enable apparmor
>      systemctl start apparmor
>      ```
>
>   5. **Vérifier les processus réellement confinés** :
>      ```bash
>      aa-status | grep "processes are unconfined"
>      ```
>      - Aucun processus ne doit être *unconfined* s’il possède un profil déclaré.
>
>   6. **Analyser les refus AppArmor** :
>      ```bash
>      journalctl | grep apparmor
>      ```
>      ou, si auditd est utilisé :
>      ```bash
>      ausearch -m AVC,USER_AVC -ts recent
>      ```
>
>   7. **Ajuster un profil si nécessaire** :
>      - Passer temporairement un profil en mode *complain* pour analyse :
>        ```bash
>        sudo aa-complain /etc/apparmor.d/<profil>
>        ```
>      - Générer ou affiner les règles à partir des logs :
>        ```bash
>        sudo aa-logprof
>        ```
>      - Rebasculer en *enforce* une fois validé :
>        ```bash
>        sudo aa-enforce /etc/apparmor.d/<profil>
>        ```
>
> - *Comparaison avec Lynis* :
>   Lynis peut détecter la présence d’AppArmor et signaler des profils absents ou non chargés, mais **ne vérifie pas systématiquement que tous les profils existants sont en mode enforce**, ni que les processus actifs sont effectivement confinés. Une vérification manuelle via `aa-status` reste indispensable pour garantir l’application effective de cette mesure.
>
> - *Référence* : [ANSSI_LINUX, R45]
