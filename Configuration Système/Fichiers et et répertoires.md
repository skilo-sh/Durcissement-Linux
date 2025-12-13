La sécurisation d’un système GNU/Linux repose en grande partie sur une gestion rigoureuse des fichiers et des répertoires. Certains d’entre eux présentent des enjeux de sécurité particuliers, notamment ceux contenant des informations sensibles, les mécanismes de communication inter-processus, les espaces de stockage temporaires partagés, ainsi que les exécutables disposant de privilèges élevés. Une configuration stricte et cohérente des droits d’accès est indispensable afin de limiter les abus, prévenir les escalades de privilèges et réduire la surface d’attaque du système, conformément aux recommandations du guide de l’ANSSI.

## Fichiers et répertoires sensibles
Les fichiers et répertoires sensibles regroupent l’ensemble des éléments du système contenant des informations critiques telles que des mots de passe, des empreintes cryptographiques, des clés privées ou certaines données de journalisation. Une mauvaise gestion de leurs droits d’accès peut compromettre directement la sécurité du système, en facilitant l’élévation de privilèges, l’usurpation d’identité ou la fuite d’informations confidentielles.

Conformément aux recommandations de l’ANSSI, ces fichiers doivent faire l’objet de restrictions strictes : les fichiers contenant des secrets système (mots de passe ou empreintes associées) ne doivent être accessibles qu’à l’utilisateur **root**, tandis que les fichiers publics nécessaires au fonctionnement du système (comme la liste des utilisateurs) peuvent être lisibles par tous mais restent modifiables exclusivement par **root**. La maîtrise fine des permissions sur ces ressources constitue un fondement essentiel du principe de moindre privilège et du durcissement global du système GNU/Linux.

> [!warning] Restreindre les droits d'accès aux fichiers et aux répertoires sensibles
> - *Objectifs et intérêts* : Même lorsque leur contenu est protégé par des mécanismes cryptographiques (hachage des mots de passe, chiffrement, signatures…), les fichiers et répertoires sensibles doivent voir leurs **droits d’accès strictement restreints**. Cette mesure relève du principe de **défense en profondeur** : elle limite l’exploitation d’une mauvaise configuration, d’une vulnérabilité applicative ou d’une élévation de privilèges partielle en empêchant l’accès direct aux données critiques.
>
> - *Commentaires* :
>   - Les fichiers sensibles incluent notamment : bases de mots de passe, clés privées, secrets applicatifs, fichiers de configuration critiques, journaux sensibles.
>   - Le guide ANSSI distingue trois cas :
>     1. **Fichiers système sensibles** : propriété `root`, accès restreint à `root` uniquement.
>     2. **Fichiers sensibles applicatifs** : propriété de l’utilisateur du service, groupe dédié, accès en lecture seule.
>     3. **Autres utilisateurs** : aucun droit d’accès.
>   - Toute permission excessive (lecture, écriture ou exécution inutile) constitue une surface d’attaque supplémentaire.
>
> - *Procédure détaillée* :
>   1. **Identifier les fichiers et répertoires sensibles** :
>      - Exemples courants :
>        - `/etc/shadow`, `/etc/gshadow`
>        - `/etc/ssl/private/*`
>        - `/home/*/.ssh/*`
>        - fichiers de configuration contenant des secrets applicatifs
>
>   2. **Fichiers sensibles système (accès root uniquement)** :
>      ```bash
>      chown root:root /chemin/vers/fichier
>      chmod 600 /chemin/vers/fichier
>      ```
>      Pour un répertoire :
>      ```bash
>      chown root:root /chemin/vers/repertoire
>      chmod 700 /chemin/vers/repertoire
>      ```
>
>   3. **Fichiers sensibles accessibles à un service applicatif** :
>      - Créer un groupe dédié si nécessaire :
>        ```bash
>        groupadd www-group
>        ```
>      - Associer l’utilisateur du service au groupe :
>        ```bash
>        usermod -aG www-group www-data
>        ```
>      - Appliquer les droits minimaux :
>        ```bash
>        chown www-data:www-group /chemin/vers/fichier
>        chmod 440 /chemin/vers/fichier
>        ```
>      - Pour un répertoire :
>        ```bash
>        chown -R www-data:www-group /chemin/vers/repertoire
>        chmod 750 /chemin/vers/repertoire
>        ```
>
>   4. **Vérifier l’absence de droits pour les autres utilisateurs** :
>      ```bash
>      ls -l /chemin/vers/fichier
>      ```
>      - Aucun droit ne doit apparaître pour `other`.
>
>   5. **Audit global des permissions sensibles (optionnel)** :
>      ```bash
>      find /etc -type f -perm /022 -ls
>      find / -type f -perm /002 -ls 2>/dev/null
>      ```
>      - Examiner et corriger toute écriture inutile pour le groupe ou les autres.
>
> - *Comparaison avec Lynis* :
>   Lynis détecte certaines permissions faibles (fichiers monde-lisibles ou accessibles en écriture), mais **ne distingue pas le contexte fonctionnel** (fichiers applicatifs légitimement accessibles à un service via un groupe dédié). La conformité complète à cette recommandation nécessite donc une **analyse manuelle fine**, basée sur le besoin réel d’accès.
>
> - *Référence* : [ANSSI_LINUX, R50] – *Recommandations de configuration d’un système GNU/Linux* :contentReference[oaicite:0]{index=0}

> [!warning] Changer les secrets et droits d'accès dès l'installation
> - *Objectifs et intérêts* :  
>   De nombreux fichiers sensibles et éléments d’authentification (clés cryptographiques, certificats, mots de passe, secrets applicatifs) sont générés ou installés automatiquement lors de l’installation du système ou des services. Conserver des secrets par défaut ou mal protégés expose le système à des compromissions immédiates. Cette règle vise à garantir que tous les secrets sont **uniques**, **maîtrisés** et **correctement protégés dès l’origine**, réduisant fortement les risques d’élévation de privilèges, d’usurpation d’identité ou de compromission persistante.
>
> - *Commentaires* :
>   - Certains paquets installent des **clés, certificats ou secrets par défaut**, parfois identiques sur toutes les machines.
>   - Les mécanismes d’authentification (SSH, TLS, comptes système, services réseau) doivent être considérés comme **critiques**.
>   - Toute régénération de secret doit s’accompagner d’une **revue des permissions** associées.
>   - Cette règle doit être appliquée **pendant ou immédiatement après l’installation**, avant toute mise en production ou exposition réseau.
>
> - *Procédure détaillée* :
>   1. **Identifier les fichiers sensibles existants** :
>      ```bash
>      find /etc /root /var /home -type f \( \
>        -name "*key*" -o -name "*secret*" -o -name "*.pem" -o -name "*.crt" -o -name "*.der" \
>      \) 2>/dev/null
>      ```
>
>   2. **SSH – régénérer les clés hôte** :
>      ```bash
>      rm -f /etc/ssh/ssh_host_*
>      dpkg-reconfigure openssh-server
>      ```
>      Vérifier les permissions :
>      ```bash
>      chown root:root /etc/ssh/ssh_host_*
>      chmod 600 /etc/ssh/ssh_host_*
>      ```
>
>   3. **Mots de passe des comptes système et administrateur** :
>      - Définir ou modifier le mot de passe root :
>        ```bash
>        passwd root
>        ```
>      - Vérifier qu’aucun compte système n’a de mot de passe actif :
>        ```bash
>        awk -F: '($2 != "!" && $2 != "*") {print $1}' /etc/shadow
>        ```
>
>   4. **Certificats et autorités de certification locales** :
>      - Régénérer toute CA ou certificat auto-signé utilisé localement.
>      - Exemple OpenSSL (CA minimale) :
>        ```bash
>        openssl genrsa -out ca.key 4096
>        openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt
>        ```
>      - Restreindre les droits :
>        ```bash
>        chmod 600 ca.key
>        chmod 644 ca.crt
>        ```
>
>   5. **Secrets applicatifs (bases de données, services)** :
>      - Modifier les mots de passe par défaut (ex. MySQL, PostgreSQL, Redis, etc.).
>      - Vérifier l’absence de secrets en clair :
>        ```bash
>        grep -R "password\|secret\|token" /etc 2>/dev/null
>        ```
>
>   6. **Vérifier et corriger les permissions des fichiers sensibles** :
>      ```bash
>      chmod 600 <fichier_sensible>
>      chown root:root <fichier_sensible>
>      ```
>
>   7. **Tracer et documenter** :
>      - Documenter chaque secret généré (date, service, emplacement).
>      - Prévoir une procédure de rotation régulière.
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler certains fichiers sensibles avec des permissions trop larges ou la présence de clés SSH faibles, mais **ne détecte pas les secrets par défaut spécifiques aux services**, ni la nécessité de régénérer des certificats ou clés lors de l’installation. L’application complète de cette règle repose donc sur une **revue manuelle et contextuelle**.
>
> - *Référence* : [ANSSI_LINUX, R51] :contentReference[oaicite:0]{index=0}

## Fichiers IPC nommés, sockets ou pipes
Les mécanismes de communication inter-processus (IPC) tels que les **pipes nommés** et les **sockets locales Unix** sont couramment utilisés pour permettre des échanges de données entre processus locaux, en alternative aux communications réseau classiques.  
Cependant, ces mécanismes présentent des **enjeux de sécurité spécifiques**, notamment en matière de **contrôle d’accès** : selon les systèmes et les implémentations, les droits appliqués à une socket locale peuvent dépendre du **répertoire qui la contient**, et non uniquement de la socket elle-même. Cette ambiguïté, non strictement définie par la norme POSIX, peut conduire à des expositions involontaires si les permissions des répertoires ne sont pas correctement maîtrisées.  
Il est donc essentiel d’**encadrer strictement les droits d’accès** aux emplacements hébergeant ces fichiers IPC afin de limiter les interactions non autorisées entre processus.
> [!warning] Restreindre les accès aux sockets et aux pipes nommées
> - *Objectifs et intérêts* :  
>   Les sockets Unix et les pipes nommées (FIFO) sont des mécanismes d’IPC (Inter-Process Communication) permettant l’échange de données entre processus locaux. Une mauvaise gestion de leurs emplacements ou de leurs permissions peut permettre à un utilisateur non autorisé d’intercepter, d’injecter ou de perturber des communications inter-processus, menant à des fuites d’information ou à des élévations de privilèges. Cette règle vise à garantir que seuls les processus légitimes puissent accéder à ces canaux IPC.
>
> - *Commentaires* :
>   - Les **droits effectifs d’une socket locale** peuvent dépendre **du répertoire qui la contient**, et non uniquement de la socket elle-même (comportement non strictement défini par POSIX).
>   - Les sockets ou pipes **ne doivent jamais être créées dans des répertoires temporaires accessibles en écriture à tous** (ex. `/tmp`, `/var/tmp`) sans protections supplémentaires.
>   - Les répertoires hébergeant des IPC doivent être **dédiés**, avec des permissions strictes et un propriétaire approprié.
>   - Les outils d’inspection IPC peuvent ne pas fournir une vue exhaustive selon les modules de sécurité chargés (AppArmor, SELinux, namespaces).
>
> - *Procédure détaillée* :
>   1. **Identifier les sockets locales en cours d’utilisation** :
>      ```bash
>      ss -xp
>      ```
>      ou :
>      ```bash
>      sockstat
>      ```
>      - Vérifier le chemin des sockets (`/run`, `/var/run`, `/var/lib/<service>`, etc.).
>
>   2. **Lister les ressources IPC System V** :
>      ```bash
>      ipcs
>      ```
>      - Examiner les permissions des segments de mémoire partagée, files de messages et sémaphores.
>
>   3. **Inspecter la mémoire partagée POSIX** :
>      ```bash
>      ls -ld /dev/shm
>      ls -l /dev/shm
>      ```
>      - Identifier les objets accessibles en écriture à des utilisateurs non autorisés.
>
>   4. **Identifier les processus utilisant des IPC** :
>      ```bash
>      lsof
>      ```
>      ou de manière ciblée :
>      ```bash
>      lsof | grep -E 'sock|FIFO|/dev/shm'
>      ```
>
>   5. **Vérifier et corriger les permissions des répertoires hébergeant les IPC** :
>      - Exemple pour un service dédié :
>        ```bash
>        chown root:root /run/<service>
>        chmod 700 /run/<service>
>        ```
>      - Pour un service non-root :
>        ```bash
>        chown <user>:<group> /run/<service>
>        chmod 750 /run/<service>
>        ```
>
>   6. **Corriger les permissions des sockets ou pipes existants si nécessaire** :
>      ```bash
>      chmod 600 /chemin/socket
>      chown root:root /chemin/socket
>      ```
>      (ou utilisateur dédié du service)
>
>   7. **Empêcher l’usage dangereux des répertoires temporaires** :
>      - Vérifier qu’aucune socket ou FIFO n’est créée directement dans `/tmp` ou `/var/tmp`.
>      - Préférer `/run/<service>` ou `/var/lib/<service>/run` avec permissions strictes.
>
>   8. **Valider après redémarrage** :
>      - Redémarrer les services concernés.
>      - Vérifier à nouveau la création des sockets et leurs permissions.
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler certaines permissions dangereuses sur des fichiers IPC ou répertoires temporaires, mais **ne vérifie pas systématiquement la cohérence entre permissions du répertoire et celles des sockets qu’il contient**, ni leur emplacement logique. Une revue manuelle reste indispensable pour cette règle.
>
> - *Référence* : [ANSSI_LINUX, R52]

## Droits d'accès
> [!error] Éviter les fichiers ou répertoires sans utilisateur ou sans groupe connu
> - *Objectifs et intérêts* :  
>   Les fichiers ou répertoires dont l’UID ou le GID ne correspondent plus à un utilisateur ou un groupe existant (fichiers *orphelins*) constituent un risque de sécurité. Lors de la création ultérieure d’un nouvel utilisateur ou groupe réutilisant ces identifiants numériques, ces fichiers peuvent être automatiquement réattribués, entraînant une élévation de privilèges involontaire ou un accès non autorisé à des données sensibles.
>
> - *Commentaires* :
>   - Cette situation survient fréquemment après la suppression d’un compte utilisateur ou groupe sans nettoyage préalable de ses fichiers.
>   - Les UID/GID sont des identifiants numériques : le système ne vérifie pas l’intention lors de leur réutilisation.
>   - Les fichiers orphelins peuvent concerner aussi bien des données applicatives que des fichiers systèmes ou de logs.
>
> - *Procédure détaillée* :
>   1. **Lister les fichiers et répertoires orphelins** :
>      ```bash
>      find / \( -nouser -o -nogroup \) -ls 2>/dev/null
>      ```
>      - `-nouser` : fichiers dont l’UID ne correspond à aucun utilisateur connu
>      - `-nogroup` : fichiers dont le GID ne correspond à aucun groupe connu
>
>   2. **Analyser chaque occurrence** :
>      - Identifier l’origine du fichier (service, ancien compte, application supprimée).
>      - Vérifier son utilité fonctionnelle avant toute correction.
>
>   3. **Corriger la propriété si le fichier est légitime** :
>      - Réattribuer à `root` si aucun autre propriétaire n’est pertinent :
>        ```bash
>        chown root:root /chemin/vers/fichier
>        ```
>      - Ou réattribuer à un utilisateur/groupe système dédié :
>        ```bash
>        chown utilisateur:groupe /chemin/vers/fichier
>        ```
>
>   4. **Supprimer les fichiers inutiles ou obsolètes** :
>      ```bash
>      rm -f /chemin/vers/fichier
>      ```
>      ou pour un répertoire :
>      ```bash
>      rm -rf /chemin/vers/repertoire
>      ```
>
>   5. **Prévenir la réapparition** :
>      - Avant toute suppression de compte utilisateur ou groupe, identifier et traiter les fichiers associés :
>        ```bash
>        find / -uid <UID> -ls 2>/dev/null
>        find / -gid <GID> -ls 2>/dev/null
>        ```
>      - Mettre en place une vérification périodique (audit manuel ou tâche planifiée).
>
> - *Comparaison avec Lynis* :
>   Lynis peut détecter la présence de fichiers sans utilisateur ou groupe valide, mais il ne fournit pas d’analyse contextuelle ni de stratégie de correction adaptée. Une revue manuelle reste nécessaire pour éviter des corrections inappropriées.
>
> - *Référence* : [ANSSI_LINUX, R53] :contentReference[oaicite:0]{index=0}

> [!error] Activer le sticky bit sur les répertoires accessibles en écriture par tous
> - *Objectifs et intérêts* :  
>   Les répertoires accessibles en écriture par tous (world-writable) sont majoritairement utilisés comme zones de stockage temporaire (ex. `/tmp`, `/var/tmp`). Lorsqu’ils sont mal configurés, ils peuvent être détournés pour des attaques locales, notamment des escalades de privilèges (remplacement de fichiers, attaques TOCTOU, liens symboliques malveillants).  
>   L’activation du **sticky bit** empêche un utilisateur ou un processus de supprimer ou remplacer les fichiers créés par un autre, limitant ainsi les abus et renforçant l’isolation entre utilisateurs et applications.
>
> - *Commentaires* :
>   - Le **sticky bit** (`+t`) garantit que seul le **propriétaire du fichier**, le **propriétaire du répertoire** ou **root** peut supprimer ou renommer un fichier dans un répertoire world-writable.
>   - Le sticky bit **ne remplace pas** une gestion correcte des permissions :  
>     le **propriétaire du répertoire doit impérativement être `root`**, sans quoi un utilisateur propriétaire pourrait contourner la protection.
>   - Les répertoires typiquement concernés sont `/tmp`, `/var/tmp` et tout autre répertoire accessible en écriture par tous.
>
> - *Procédure détaillée* :
>   1. **Identifier les répertoires world-writable sans sticky bit** :
>      ```bash
>      find / -type d \( -perm -0002 -a \! -perm -1000 \) -ls 2>/dev/null
>      ```
>   2. **Activer le sticky bit sur les répertoires concernés** :
>      ```bash
>      chmod +t /chemin/du/repertoire
>      ```
>      Exemple :
>      ```bash
>      chmod +t /tmp
>      chmod +t /var/tmp
>      ```
>   3. **Vérifier que le propriétaire est root** :
>      ```bash
>      chown root:root /chemin/du/repertoire
>      ```
>   4. **Lister les répertoires world-writable dont le propriétaire n’est pas root** :
>      ```bash
>      find / -type d -perm -0002 -a \! -uid 0 -ls 2>/dev/null
>      ```
>      - Corriger chaque cas identifié en ajustant le propriétaire ou en réévaluant la nécessité du répertoire.
>   5. **Vérification finale** :
>      ```bash
>      ls -ld /chemin/du/repertoire
>      ```
>      - La permission doit apparaître sous la forme `drwxrwxrwt`.
>
> - *Comparaison avec Lynis* :
>   Lynis détecte généralement l’absence de sticky bit sur certains répertoires critiques comme `/tmp`, mais **ne garantit pas l’exhaustivité** pour l’ensemble des répertoires world-writable ni la vérification systématique du propriétaire (`root`). Une revue manuelle reste nécessaire pour satisfaire pleinement cette recommandation.
>
> - *Référence* : [ANSSI_LINUX, R54] – Recommandations de configuration d’un système GNU/Linux

> [!warning] Séparer les répertoires temporaires des utilisateurs
> - *Objectifs et intérêts* : Le sticky bit (`+t`) appliqué à des répertoires partagés comme `/tmp` empêche la suppression de fichiers appartenant à d’autres utilisateurs, mais **n’élimine pas les conditions de compétition (race conditions)** entre processus s’exécutant sous le même compte. Séparer les répertoires temporaires par utilisateur ou par application permet d’éviter les attaques de type TOCTOU, les collisions de noms de fichiers et les manipulations croisées de fichiers temporaires, en particulier dans des contextes multi-processus ou multi-services.
>
> - *Commentaires* :
>   - Les répertoires temporaires partagés (`/tmp`, `/var/tmp`) restent nécessaires, mais **ne doivent pas être utilisés comme espace temporaire principal par des applications sensibles**.
>   - Chaque utilisateur ou application doit disposer de **son propre répertoire temporaire**, avec des permissions exclusives.
>   - Les distributions GNU/Linux récentes permettent cette séparation de manière transparente via **PAM**, notamment avec `pam_mktemp` ou `pam_namespace`.
>   - Lorsqu’un fichier ou un répertoire doit être modifiable par plusieurs utilisateurs, **un groupe dédié** doit être créé, et **l’écriture ne doit être autorisée qu’à ce groupe**.
>
> - *Procédure détaillée* :
>   1. **Vérifier l’état actuel des répertoires temporaires globaux** :
>      ```bash
>      ls -ld /tmp /var/tmp
>      ```
>      (permissions attendues : `drwxrwxrwt`)
>
>   2. **Créer des répertoires temporaires dédiés par utilisateur (exemple manuel)** :
>      ```bash
>      mkdir /tmp/user_$USER
>      chown $USER:$USER /tmp/user_$USER
>      chmod 700 /tmp/user_$USER
>      ```
>      - Configurer les applications pour utiliser ce chemin via les variables d’environnement :
>        ```bash
>        export TMPDIR=/tmp/user_$USER
>        ```
>
>   3. **Automatiser la création via PAM (méthode recommandée)** :
>      - Installer les modules nécessaires :
>        ```bash
>        apt install libpam-mktemp
>        ```
>      - Activer `pam_mktemp` (ex. pour les sessions locales) :
>        ```bash
>        echo "session optional pam_mktemp.so" >> /etc/pam.d/common-session
>        ```
>      - Chaque session utilisateur disposera alors d’un répertoire temporaire privé, supprimé à la fermeture de session.
>
>   4. **Alternative via pam_namespace (isolation renforcée)** :
>      - Activer le module :
>        ```bash
>        apt install libpam-namespace
>        ```
>      - Configurer `/etc/security/namespace.conf` pour isoler `/tmp` par utilisateur.
>
>   5. **Gérer les fichiers temporaires partagés via des groupes dédiés** :
>      ```bash
>      groupadd tmp_shared
>      chown root:tmp_shared /chemin/fichier_temporaire
>      chmod 660 /chemin/fichier_temporaire
>      ```
>
>   6. **Identifier les fichiers modifiables par tous (audit)** :
>      ```bash
>      find / -type f -perm -0002 -ls 2>/dev/null
>      ```
>      - Examiner chaque fichier listé et corriger les permissions ou l’appartenance au groupe si nécessaire.
>
> - *Comparaison avec Lynis* :
>   Lynis détecte la présence de répertoires world-writable et l’usage du sticky bit, mais **ne vérifie pas la séparation effective des espaces temporaires par utilisateur ou application**. La mise en œuvre de cette recommandation nécessite donc une configuration manuelle et une validation fonctionnelle.
>
> - *Référence* : [ANSSI_LINUX, R55]

> [!error] Éviter l’usage d’exécutables avec les droits spéciaux setuid et setgid
> - *Objectifs et intérêts* :  
>   Les exécutables disposant des bits spéciaux **setuid** ou **setgid** s’exécutent avec les privilèges du propriétaire du fichier (souvent `root`) ou de son groupe, et non avec ceux de l’utilisateur appelant. Ils constituent donc une cible privilégiée pour l’élévation de privilèges. Limiter strictement leur usage réduit fortement la surface d’attaque locale et empêche l’exploitation de vulnérabilités liées à un changement de contexte d’exécution mal maîtrisé.
>
> - *Commentaires* :
>   - Les exécutables setuid/setgid doivent être **explicitement conçus** pour gérer un changement d’identité sécurisé (nettoyage de l’environnement, descripteurs de fichiers, signaux, variables héritées).
>   - La majorité des binaires **ne sont pas conçus** pour fonctionner correctement avec ces droits et deviennent dangereux si ces bits leur sont ajoutés.
>   - Seuls des logiciels **de confiance**, issus de la distribution ou de dépôts officiels, et **documentés comme nécessitant setuid/setgid**, doivent conserver ces permissions.
>   - Toute présence non justifiée de ces bits constitue un **risque critique d’escalade de privilèges**.
>
> - *Procédure détaillée* :
>   1. **Lister l’ensemble des exécutables setuid/setgid présents sur le système** :
>      ```bash
>      find / -type f -perm /6000 -ls 2>/dev/null
>      ```
>      - Identifier chaque binaire, son propriétaire, son groupe et son origine (paquet Debian, binaire tiers, script).
>
>   2. **Identifier l’origine des binaires** :
>      - Vérifier s’ils appartiennent à un paquet officiel :
>        ```bash
>        dpkg -S /chemin/vers/binaire
>        ```
>      - Tout binaire non rattaché à un paquet officiel doit être considéré comme **suspect par défaut**.
>
>   3. **Évaluer la légitimité des droits spéciaux** :
>      - Vérifier la documentation du paquet.
>      - Vérifier si une alternative existe (Linux capabilities, service systemd, wrapper).
>      - Justifier explicitement chaque binaire conservant setuid/setgid.
>
>   4. **Supprimer les droits spéciaux inutiles** :
>      - Retirer le bit setuid :
>        ```bash
>        chmod u-s /chemin/vers/binaire
>        ```
>      - Retirer le bit setgid :
>        ```bash
>        chmod g-s /chemin/vers/binaire
>        ```
>
>   5. **Vérifier les permissions après modification** :
>      ```bash
>      ls -l /chemin/vers/binaire
>      ```
>
>   6. **Tester le fonctionnement applicatif** :
>      - Vérifier que la suppression des bits spéciaux n’introduit pas de régression fonctionnelle.
>      - Surveiller les journaux système (`journalctl`, `/var/log/syslog`).
>
> - *Comparaison avec Lynis* :
>   Lynis détecte la présence de fichiers setuid/setgid et peut signaler certains binaires à risque, mais **ne juge pas la légitimité fonctionnelle** de ces droits. Une analyse manuelle est indispensable pour décider de leur suppression ou conservation.
>
> - *Référence* : [ANSSI_LINUX, R56] :contentReference[oaicite:0]{index=0}

> [!info] Éviter l’usage d’exécutables avec les droits spéciaux setuid root et setgid root
> - *Objectifs et intérêts* :  
>   Les exécutables dotés des bits spéciaux **setuid root** ou **setgid root** s’exécutent avec les privilèges du propriétaire (souvent root) et non ceux de l’utilisateur appelant. Ils constituent donc un **vecteur privilégié d’élévation de privilèges**. En présence de vulnérabilités (débordements mémoire, injections, erreurs logiques), ces binaires sont fréquemment exploités afin d’obtenir un shell root ou de détourner leur usage légitime. Réduire leur nombre limite fortement l’impact d’une compromission locale.
>
> - *Commentaires* :
>   - Les binaires setuid/setgid sont **légitimes dans certains cas précis** (ex. `passwd`, `mount`, `su`), mais doivent être **justifiés, documentés et audités**.
>   - Lorsqu’un exécutable est destiné uniquement aux administrateurs, il est préférable de **supprimer les bits setuid/setgid** et de passer par `sudo` ou `su`, qui offrent une **traçabilité**, une **journalisation** et un **contrôle fin** des droits.
>   - Les mises à jour de paquets peuvent **restaurer automatiquement** les bits setuid/setgid ou en introduire de nouveaux : une vérification régulière est indispensable.
>
> - *Procédure détaillée* :
>   1. **Identifier les exécutables setuid et setgid** :
>      ```bash
>      find / -xdev -type f \( -perm -4000 -o -perm -2000 \) 2>/dev/null
>      ```
>      - `-4000` : setuid
>      - `-2000` : setgid
>      - `-xdev` : évite de traverser les autres systèmes de fichiers
>
>   2. **Analyser chaque exécutable identifié** :
>      - Identifier le paquet propriétaire :
>        ```bash
>        dpkg -S /chemin/vers/binaire
>        ```
>      - Vérifier l’usage réel et la nécessité fonctionnelle du bit setuid/setgid.
>
>   3. **Supprimer les bits setuid/setgid non nécessaires** :
>      - Retirer le setuid :
>        ```bash
>        chmod u-s /chemin/vers/binaire
>        ```
>      - Retirer le setgid :
>        ```bash
>        chmod g-s /chemin/vers/binaire
>        ```
>
>   4. **Remplacer par sudo lorsque pertinent** :
>      - Restreindre l’exécution via `/etc/sudoers` ou `/etc/sudoers.d/` :
>        ```bash
>        utilisateur ALL=(root) /chemin/vers/commande
>        ```
>      - Vérifier la journalisation des usages sudo (`/var/log/auth.log`).
>
>   5. **Surveiller les régressions après mise à jour** :
>      - Comparer périodiquement la liste des binaires setuid/setgid :
>        ```bash
>        find / -xdev -type f \( -perm -4000 -o -perm -2000 \) -exec ls -l {} \;
>        ```
>      - Intégrer cette vérification dans un audit régulier ou un script de contrôle.
>
> - *Comparaison avec Lynis* :  
>   Lynis détecte la présence d’exécutables setuid/setgid et signale ceux considérés comme sensibles, mais **ne peut pas déterminer leur légitimité fonctionnelle** ni décider lesquels doivent être supprimés. Une **analyse manuelle au cas par cas** reste indispensable pour respecter pleinement cette recommandation.
>
> - *Référence* : [ANSSI_LINUX, R57]
