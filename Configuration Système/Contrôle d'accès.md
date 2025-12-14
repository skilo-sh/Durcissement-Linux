Cette section sera dédiée au contrôle d’accès, qui vise à garantir que chaque compte ne dispose que des droits strictement nécessaires pour accéder aux ressources d'un système, et constitue ainsi un pilier fondamental du durcissement en permettant de limiter l’impact d’une quelconque compromission. Sous GNU/Linux, ce contrôle repose historiquement sur la gestion des utilisateurs et des groupe, complétée par des mécanismes plus avancés tels que les **LSM (SELinux (ce dernier ne sera pas traité dans ce wiki, AppArmor)**, le **filtrage système (seccomp)** et les **mécanismes de cloisonnement**. De ce fait, l’ensemble de ces dispositifs concourt à renforcer l’isolement des processus et à réduire la surface d’attaque globale du système.
## Modèle traditionnel Unix 
Le modèle de sécurité traditionnel d’Unix repose sur le contrôle des accès par **utilisateurs (UID)** et **groupes (GID)**, pour lequel chaque ressource du système (fichier, processus, répertoire…) possède un propriétaire et des droits d’accès définis selon trois niveaux : **propriétaire, groupe et autres**. Ainsi, ce mécanisme correspond à un **contrôle d’accès discrétionnaire (ou DAC)**, dans lequel le propriétaire décide des permissions en lecture, écriture et exécution. Par défaut, ces droits sont influencés par le **UMASK**, souvent trop permissif, ce qui justifie un durcissement systématique, notamment pour les shells utilisateurs et les services, afin de limiter l’exposition du système.

> [!info] Modifier la valeur par défaut de l’UMASK
> - *Objectifs et intérêts* : L’UMASK définit les permissions par défaut appliquées lors de la création de fichiers et de répertoires, où une valeur trop permissive peut provoquer des fuites de données ou des modifications non autorisées. De ce fait, il est primordial de positionner une UMASK restrictive afin d’appliquer automatiquement le principe de moindre privilège dès la création des ressources.
>
> - *Commentaires* :
>   - Pour les **sessions utilisateurs (shells)**, une UMASK à `0077` garantit que seuls les droits du propriétaire sont autorisés (lecture, écriture).
>   - Pour les **services**, une UMASK à `0027` permet :
>     - Une lecture et une écriture pour le propriétaire,
>     - Une lecture pour le groupe,
>     - Aucun accès pour les autres.
>   - De plus, certains services nécessitent une UMASK spécifique selon leur fonctionnement (journaux, partages, sockets), d’où l’étude **au cas par cas** recommandée par le guide de l'ANSSI.
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
>   2. **Vérifier l’UMASK effective pour l'utilisateur** :
>      ```bash
>      umask
>      ```
>      - La valeur attendue est `0077`, comme susmentionné.
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
>      - Vérifier les permissions effectives des fichiers générés par les services sans oublier d'ajuster l’UMASK si un service nécessite un accès groupe plus large.
>
> - *Comparaison avec Lynis* :
>   Lynis vérifie certaines permissions de fichiers sensibles, mais **ne contrôle pas systématiquement les UMASK par défaut des shells ni celles définies dans les unités systemd**, ce qui nécessite une nouvelle fois une manipulation manuelle.
>
> - *Référence* : [ANSSI_LINUX, R6]

> [!info] Utiliser des fonctionnalités de contrôle d'accès obligatoire (MAC)
> - *Objectifs et intérêts* : La sécurisation des systèmes via le contrôle d'accès passe par la complétion du modèle de contrôle d’accès discrétionnaire (DAC) historique des systèmes Unix via un contrôle d’accès obligatoire (MAC) afin d’imposer une politique de sécurité centralisée et non contournable par les utilisateurs. De ce fait, le MAC permet de réduire fortement l’impact d’une compromission applicative, de limiter la surface d’attaque et d’assurer un cloisonnement strict entre processus, même appartenant au même utilisateur.
>
> - *Commentaires* :
>   - Premièrement, le modèle **DAC (Discretionary Access Control)** reste aujourd’hui **majoritaire et appliqué par défaut**, mais présente plusieurs **limitations structurantes** :
>     - Les droits sont laissés à la discrétion du propriétaire, ce qui est incompatible avec des politiques de sécurité fortes.
>     - Le propriétaire peut mal configurer les permissions, entraînant des accès non autorisés à des données sensibles.
>     - L’**isolation entre utilisateurs est grossière**, les droits par défaut étant souvent trop permissifs.
>     - Il n’existe que deux niveaux de privilèges (utilisateur standard et root), l’élévation passant par des binaires à privilèges spéciaux (*setuid root*).
>     - Il est impossible d’adapter finement les privilèges d’un processus selon son contexte d’exécution (un navigateur et un éditeur texte d’un même utilisateur ont les mêmes droits).
>     - La surface d’attaque est élevée, de nombreuses informations système restant accessibles aux utilisateurs standards.
>   - Ainsi, le **MAC (Mandatory Access Control)**, qui impose au contraire des règles **globales, centralisées et non modifiables par l’utilisateur** (au prix d’une **complexité et d’un coût de déploiement plus élevés**) reste absolument vital dans notre cas.
>
> - *Procédure détaillée* :
>   1. **Choisir un mécanisme MAC** :
>      - **AppArmor** (profilage par chemins, plus simple à maintenir).
>      - ou **SELinux** (plus de finesse que AppArmor mais également plus complexe, à savoir que nous ne traiterons pas son utilisation dans ce wiki)
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
>   4. **Appliquer les profils aux services sensibles** :
>      - Ces services peuvent être des serveurs web, des bases de données, des services réseau, des outils d’administration etc.
>   5. **Surveiller les violations MAC** :
>      - Analyser les journaux (`dmesg`, `journalctl`, logs AppArmor).
>
> - *Comparaison avec Lynis* :
>   Lynis détecte la présence d’AppArmor (ou de SELinux le cas échéant) mais n'est pas capable de valider ni la qualité des profils, ni la cohérence de la politique de cloisonnement appliquée, nécessitant encore une fois une analyse humaine afin de garantir l’efficacité réelle du MAC.
>
> - *Référence* : [ANSSI_LINUX, R37]

> [!warning] Créer un groupe dédié à l’usage de sudo
> - *Objectifs et intérêts* : Restreindre strictement l’usage de la commande `sudo` à un groupe d’utilisateurs explicitement autorisés permet de limiter drastiquement les possibilités d’élévation de privilèges, renforçant le principe de moindre privilège en s’assurant que seuls les utilisateurs ayant un besoin opérationnel légitime peuvent exécuter des commandes avec des droits administrateur.
>
> - *Commentaires* :
>   - Il est important de noter que la création d’un groupe dédié spécifique permet une gestion beaucoup plus fine et plus lisible des droits d’élévation. D'autre part,  le nombre de commandes disponibles, l’architecture du système, les services installés et leurs configurations, rendent quasiment impossible le fait de couvrir l'ensemble des situations dans lesquelles sudo peut être utilisé et ainsi l’établissement d’une configuration standard sécurisée.
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
>      - Tester avec un utilisateur membre du groupe et  un utilisateur non membre (l’accès doit bien évidemment être refusé).
>
> - *Comparaison avec Lynis* :
>   Lynis vérifie la présence de `sudo`, la configuration basique de `/etc/sudoers` et certaines permissions, mais toutefois **ne contrôle pas l’existence d’un groupe sudo dédié**, ni la stratégie de restriction par groupe personnalisé.
>
> - *Référence* : [ANSSI_LINUX, R38]

> [!warning] Modifier les directives de configuration sudo
> - *Objectifs et intérêts* : L’outil `sudo` est un point critique de la chaîne de privilèges d'un système GNU/Linux, où ses valeurs par défaut sont historiquement permissives et peuvent faciliter l’évasion de restrictions, l’exploitation via diverses variables d’environnement ou l’exécution de commandes détournées. Par conséquent, le durcissement des directives globales permet de réduire drastiquement les surfaces d’attaque liées à l’escalade de privilèges.
>
> - *Commentaires* :
>   - Comme susmentionné, les valeurs **par défaut de sudo sont insuffisamment restrictives** dans un contexte de durcissement (et il est indispensable de se renseigner avec **man sudoers** en amont)
>   - D'autre part, de nombreuses vulnérabilités historiques de sudo sont directement liées :
>     - à une mauvaise gestion des UID spéciaux (`-1`),
>     - aux échappements de commandes ainsi qu'aux débordements de mémoire.
>
> - *Procédure détaillée* :
>   1. **Éditer le fichier "sudoers" uniquement via visudo** :
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
>   Lynis détecte certaines configurations dangereuses de sudo (droits trop larges, NOPASSWD excessif), mais **ne force pas l’activation systématique de `noexec`, `use_pty`, `ignore_dot` ou `requiretty`**. 
>
> - *Référence* : [ANSSI_LINUX, R39] 

> [!warning] Utiliser des utilisateurs cibles non-privilégiés pour les commandes sudo
> - *Objectifs et intérêts* : L’objectif principal de cette recommandation est de limiter les risques d’escalade de privilèges lors de l’utilisation de `sudo`. Ainsi, lorsque la commande à exécuter ne nécessite pas de droits élevés, elle ne doit pas être exécutée avec les privilèges de `root`. L'application de cette mesure via l’utilisation d’utilisateurs cibles non privilégiés réduit l’impact d’un abus de `sudo`, d’une mauvaise configuration ou de fonctionnalités détournées de certaines commandes.
>
> - *Commentaires* :
>   - De nombreuses actions administratives courantes ne nécessitent pas de privilèges `root` (édition de fichiers appartenant à un autre utilisateur, envoi de signaux à des processus non privilégiés, gestion de services applicatifs locaux, etc.).
>   - D'autre part, l’exécution systématique de commandes via `sudo` en tant que `root` augmente inutilement la surface d’attaque.
>   - Certaines commandes dites *fonctionnellement riches* (éditeurs de texte comme `vi`, `vim`, `less`, interpréteurs, outils interactifs etc.) permettent l’exécution de sous-commandes ou l’ouverture de shells.
>   - Même si `sudo` permet de surcharger certaines fonctions pour limiter ces comportements, ces mécanismes restent **imparfaits** et ne constituent pas une protection robuste contre les contournements.
>
> - *Procédure détaillée* :
>   1. **Identifier les usages de sudo** :
>      - Analyser le fichier `/etc/sudoers` et les fichiers présents dans `/etc/sudoers.d/`.
>      - Puis identifier les règles où la commande est exécutée en tant que `root` sans justification fonctionnelle.
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
>      - Préférer des commandes simples, non interactives, avec des chemins absolus.
>   5. **Tester et valider** :
>      - Vérifier que les actions prévues sont réalisables sans privilèges root.
>      - Tester les règles sudo avec `sudo -l` et en conditions réelles.
>
> - *Comparaison avec Lynis* :
>   Lynis est effectivement capable de signaler des configurations sudo trop permissives ou des usages dangereux évidents.
>
> - *Référence* : [ANSSI_LINUX, R40]

> [!info] Limiter l’utilisation de commandes nécessitant la directive EXEC
> - *Objectifs et intérêts* : Les mécanismes de restriction basés sur des listes de commandes autorisées (ex. sudo, RBAC, wrappers) peuvent être contournés si un processus est autorisé à exécuter librement des appels système comme `execve`. De ce fait, une commande disposant du droit EXEC peut lancer des sous-processus arbitraires (shell, interpréteur, binaire), contournant ainsi les contrôles initiaux. La principale réponse à ces problématiques est de limiter strictement l’usage de la directive EXEC permet de réduire les possibilités de contournement, d’escalade de privilèges et d’exécution de code arbitraire.
>
> - *Commentaires* :
>   - Toute commande autorisée à s’exécuter avec des privilèges élevés doit être considérée comme **potentiellement capable d’exécuter du code arbitraire**. Par conséquent, les commandes capables de lancer un shell (`bash`, `sh`, `python`, `perl`, `awk`, `find -exec`, `vi`, `less`, etc.) doivent être explicitement interdites dans les contextes restreints.
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
>        Vérifier l’absence des :
>        - éditeurs (`vi`, `nano`)
>        - interpréteurs (`python`, `ruby`, `perl`)
>        - commandes avec `-exec`, `-p`, `!`, ou équivalents
>   4. **Restreindre l’environnement d’exécution** :
>      - Forcer un environnement minimal pour sudo :
>        ```
>        Defaults secure_path="/usr/sbin:/usr/bin:/sbin:/bin"
>        Defaults !env_reset
>        Defaults env_delete+="LD_PRELOAD LD_LIBRARY_PATH PYTHONPATH"
>        ```
>   5. **Limiter EXEC via le systemd (si applicable)** :
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
>   Lynis détecte certaines configurations sudo dangereuses (wildcards, permissions trop larges etc.), mais **ne vérifie pas finement les possibilités de contournement via EXEC**, ni les capacités réelles d’une commande autorisée à engendrer des sous-processus. 
>
> - *Référence* : [ANSSI_LINUX, R41] 

> [!warning] Bannir les négations dans les spécifications sudo
> - *Objectifs et intérêts* :  
>   Toujours dans la thématique des règles sudo, ces dernières utilisent des **négations** (approche par liste d’interdiction) qui sont inefficaces et facilement contournables.  
>   En effet, Sudo évalue les droits via une comparaison de chaînes ("globbing shell") sur le chemin de la commande et ses arguments. Néanmoins, toute règle autorisant `ALL` tout en niant un binaire précis permet à un attaquant de contourner la restriction (copie, renommage, appel indirect etc.). Ainsi, l’objectif est d’appliquer une approche par "liste blanche", en n’autorisant explicitement que les commandes nécessaires.
>
> - *Commentaires* :
>   - Comme susmentionné, une règle de type :
>     ```
>     user ALL=(ALL) ALL, !/bin/sh
>     ```
>     est très facilement contournable (copie de `/bin/sh`, lien symbolique, appel via un autre binaire etc.).
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
>      - Et ainsi, il faut restreindre :
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
>   Lynis peut signaler une configuration sudo faible ou trop permissive (nous avons pu l'observer précédemment), mais **ne détecte pas systématiquement les contournements liés aux négations**, ni les problèmes de "globbing" sur les arguments. 
> - *Référence* : [ANSSI_LINUX, R42] 

> [!warning] Préciser strictement les arguments dans les règles sudo
> - *Objectifs et intérêts* : Les arguments passés à une commande peuvent modifier profondément son comportement (lecture, écriture, suppression de fichiers, accès à des ressources sensibles etc), et ainsi, une règle sudo trop permissive permet à un utilisateur de détourner une commande légitime pour exécuter des actions arbitraires avec des privilèges élevés. La réponse première à cette problématique consiste à spécifier strictement les arguments autorisés ce qui permet de réduire fortement les risques d’escalade de privilèges et limite l’impact d’un abus quelconque de sudo.
>
> - *Commentaires* :
>   - Une règle sudo sans arguments spécifiés autorise implicitement tous les arguments, ce qui est dangereux comme mentionné précedemment.
>   - D'autre part, l’usage de wildcards (`*`) doit être évité autant que possible, car il ouvre la porte à des détournements subtils et l'absence d’arguments doit être explicitement indiquée par une chaîne vide (`""`).
>   - De plus, il est important de mentionner que l'autorisation via sudo attribuée à un programme capable d’écrire arbitrairement sur le système (éditeur de texte, shell, utilitaire de copie mal restreint) équivaut dans les faits à donner les privilèges root complets.
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
>      => empêche l’utilisation de `--file` pour lire des fichiers arbitraires.
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
>      - À éviter encore une fois :
>        ```
>        user ALL=(root) /usr/bin/vim /etc/*
>        ```
>      - À préférer de nouveau :
>        - Utiliser un wrapper dédié :
>          ```bash
>          /usr/local/sbin/edit_conf.sh
>          ```
>        - Utiliser un script contrôlant précisément le fichier modifiable :
>          ```bash
>          #!/bin/sh
>          exec /usr/bin/nano /etc/mon_fichier.conf
>          ```
>        - Puis application de la règle sudo :
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
>   6. **Tester et valider le bon fonctionnement de notre mesure** :
>      ```bash
>      sudo -l -U user
>      sudo -u root /bin/dmesg --file /etc/shadow   
>      sudo -u root /bin/dmesg                       
>      ```
> 	  La deuxième commande doit correctement, tandis que la troisième commande est censée ne pas marcher.
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler la présence de sudo et certaines règles dangereuses (encore une fois) mais ne permet aucunement d'assurer la cohérence et la sûreté des arguments autorisés (il est primordial d'effectuer une écriture et une lecture manuelle des règles).
>
> - *Référence* : [ANSSI_LINUX, R43]

## AppArmor
Dans le cadre du durcissement d’un système GNU/Linux, les mécanismes de contrôle d’accès jouent un rôle central. À titre d'exemple, les systèmes Linux modernes s’appuient notamment sur des _Linux Security Modules_ (LSM) pour renforcer le modèle de permissions Unix traditionnel.  
Parmi eux, **AppArmor** constitue un mécanisme de contrôle d’accès obligatoire (Mandatory Access Control ou "MAC") largement déployé, reposant sur des profils associés aux exécutables et définissant précisément les ressources accessibles. De fait, ce modèle permet à une autorité de sécurité de contraindre le comportement des applications indépendamment de leur volonté, tout en restant relativement simple à déployer et à maintenir. En complément du contrôle d’accès aux fichiers, AppArmor offre des possibilités de restriction sur l’usage des capacités POSIX et des accès réseau, contribuant ainsi à la réduction de la surface d’attaque du système, conformément aux principes de sécurité préconisés par l’ANSSI.

> [!info] Activer et faire respecter les profils de sécurité AppArmor
> - *Objectifs et intérêts* : Comme introduit ci-dessus, AppArmor permet de restreindre finement les droits des exécutables via des profils de confinement basés sur des chemins. De fait, activer systématiquement les profils AppArmor en mode "*enforce*" permet de limiter l’impact d’une compromission applicative (lecture/écriture de fichiers, exécution de binaires, accès réseau etc.), y compris pour des services sensibles comme `syslogd`, `ntpd`, ou des démons réseau. Finalement, cette mesure renforce la défense en profondeur en imposant un contrôle d’accès obligatoire (MAC) indépendant des permissions Unix classiques.
>
> - *Commentaires* :
>   - AppArmor applique des profils par exécutable, stockés dans `/etc/apparmor.d/`, avec une gestion explicite des transitions de profils lors de l’appel d’autres exécutables.
>   - Contrairement à SELinux, AppArmor ne repose pas sur des labels de sécurité : il se concentre exclusivement sur le contrôle des accès des exécutables.
>   - De plus, les profils peuvent être chargés via deux modes distincts :
>     - **enforce** : les accès interdits sont bloqués ;
>     - **complain** : les accès interdits sont uniquement journalisés.
>
> - *Procédure détaillée* :
>   1. **Vérifier qu’AppArmor est actif** :
>      ```bash
>      aa-status
>      ```
>      Résultat attendu :
>      - `apparmor module is loaded`
>
>   2. **Lister les profils chargés et leur mode** :
>      ```bash
>      aa-status --verbose
>      ```
>      - Identifier les profils en mode "*complain*" ou les processus **non confinés** disposant d’un profil.
>
>   3. **Activer tous les profils existants en mode "enforce"** :
>      ```bash
>      sudo aa-enforce /etc/apparmor.d/*
>      ```
>
>   4. **Vérifier les processus réellement confinés** :
>      ```bash
>      aa-status | grep "processes are unconfined"
>      ```
>      - Attention: Aucun processus ne doit être en mode "*unconfined*" s’il possède un profil déclaré.
>
>   5. **Analyser les refus AppArmor** :
>      ```bash
>      journalctl | grep apparmor
>      ```
>      ou, si "auditd" est utilisé :
>      ```bash
>      ausearch -m AVC,USER_AVC -ts recent
>      ```
>
>   6. **Ajuster un profil si nécessaire** :
>      - Passer temporairement un profil en mode "*complain*" lorsqu'une analyse est nécessaire:
>        ```bash
>        sudo aa-complain /etc/apparmor.d/<profil>
>        ```
>      - Générer ou affiner les règles à partir des logs :
>        ```bash
>        sudo aa-logprof
>        ```
>      - Rebasculer en mode "*enforce*" une fois les modifications effectuées:
>        ```bash
>        sudo aa-enforce /etc/apparmor.d/<profil>
>        ```
>
> - *Comparaison avec Lynis* :
>   Lynis peut détecter la présence d’AppArmor et signaler des profils absents ou non chargés, mais ne vérifie pas systématiquement que tous les profils existants sont en mode "enforce", ni que les processus actifs sont effectivement confinés. Ainsi, nous pouvons vérifier manuellement que les recommandations sont effectives via `aa-status`.
>
> - *Référence* : [ANSSI_LINUX, R45]
