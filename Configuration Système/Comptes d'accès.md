La gestion des comptes d'accès (et permissions associées) est la deuxième mesure structurelle de la partie durcissement “système” recommandée par l’ANSSI, reposant principalement sur la notion de comptes d'accès à deux niveaux: les comptes d'utilisateur "classique" et le compte d'administrateur système "root". Si cette dernière n'est pas suffisamment rigoureuse, l'utilisateur s'expose à des fuites d'information, des vols de comptes ou d'identifiants, etc.
### Comptes utilisateur
Dans un premier temps, il est primordial d'avoir une gestion extrêmement rigoureuse des divers comptes utilisateur, faisant partie des principes essentiels de la sécurisation d'un système.
> [!error] Désactiver les comptes utilisateur inutilisés
> - *Objectifs et intérêts* : Les comptes inutilisés constituent une porte d’entrée rêvée pour un attaquant (mots de passe faibles, comptes oubliés, anciens utilisateurs etc.). De ce fait, leur désactivation réduit directement la surface d’attaque et limite les risques d’usurpation d’identité locale (et les exploitations qui peuvent en découler)
>
> - *Commentaires* :
>   - N/A
>
> - *Procédure détaillée* :
>   1. Lister les comptes utilisateurs :
>      ```bash
>      awk -F: '$3 >= 1000 {print $1}' /etc/passwd
>      ```
>   2. Verrouiller un compte inutilisé :
>      ```bash
>      usermod -L <utilisateur>
>      ```
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler l’existence de comptes inactifs, mais ne décide pas du caractère légitime ou non de leur usage, impliquant une vérification manuelle.
>
> - *Référence* : [ANSSI_LINUX, R30]

> [!error] Utiliser des mots de passe robustes
> - *Objectifs et intérêts* : De manière assez triviale, un mot de passe faible expose directement le système aux attaques par dictionnaire ou brute force, obligeant ainsi l’usage de mots de passe robustes, uniques et conformes aux recommandations actuelles, constituant une mesure fondamentale de protection des accès.
>
> - *Commentaires* :
>   - Voir le guide de l'ANSSI ci-après concernant l'authentification multifacteur et les mots de passe: [Guide_ANSSI_Authentification](https://messervices.cyber.gouv.fr/guides/recommandations-relatives-lauthentification-multifacteur-et-aux-mots-de-passe)
>
> - *Procédure détaillée* :
>   1. Modifier un mot de passe :
>      ```bash
>      passwd <utilisateur>
>      ```
>   2. Activer les règles de complexité via PAM (cf ) (ex. Debian) :
>      - Vérifier la présence de `pam_pwquality` dans :
>        ```
>        /etc/pam.d/common-password
>        ```
>
> - *Comparaison avec Lynis* :
>   Lynis détecte l’absence de politique de complexité des mots de passe sans pour autant vérifier la conformité de ceux utilisés sur un système à un moment T.
>
> - *Référence* : [ANSSI_LINUX, R31]

> [!warning] Expirer les sessions utilisateur locales
> - *Objectifs et intérêts* : En lien direct avec les deux recommandations citées précédemment, une session laissée ouverte permet une compromission immédiate en cas d’accès physique ou distant. De ce fait, le verrouillage automatique protège contre l’usurpation de session et complète la sécurité des accès locaux.
>
> - *Commentaires* :
>   - Les sessions utilisateur locales concernent :
>     - les consoles TTY,
>     - les sessions graphiques locales.
>
> - *Procédure détaillée* :
>   1. Activer l’expiration automatique des sessions TTY via `TMOUT` :
>      ```bash
>      echo "export TMOUT=600" >> /etc/profile
>      ```
>   2. Pour les environnements graphiques : activer le verrouillage automatique dans les paramètres de session.
>
> - *Comparaison avec Lynis* :
>  L'utilisation de l'analyse de Lynis a permis d'observer que ce dernier est capable de détecter l'absence de durée maximale d'inactivité avant un verrouillage automatique d'un système.
>
> - *Référence* : [ANSSI_LINUX, R_32]

### Comptes administrateur
Dans un second temps, la sécurisation de notre système repose également sur une gestion rigoureuse des comptes administrateur. De ce fait, l’usage du compte "root", qui dispose de tous les privilèges, ne permet ni une traçabilité fiable des actions ni une séparation fine des responsabilités. Afin d’assurer l’imputabilité, de limiter l’impact d’un compromis quelconque et d’appliquer le principe de moindre privilège, les rôles doivent être strictement dissociés. Ainsi, chaque administrateur doit disposer de comptes distincts selon ses missions (utilisateur, administration système, services, bases de données, etc.), avec des secrets d’authentification uniques pour chaque usage.

> [!warning] Assurer l'imputabilité des actions d'administration
> - *Objectifs et intérêts* : Les actions d’administration impliquant des privilèges élevés doivent être traçables afin de pouvoir identifier précisément l’administrateur à l’origine d’une action, notamment en cas d’incident de sécurité. Ainsi, l’imputabilité permet de dissuader les comportements abusifs, de faciliter les investigations en aval des incidents et de garantir la responsabilité individuelle des administrateurs.
>
> - *Commentaires* :
>   - L’usage du compte générique `root` empêche toute traçabilité fiable : toutes les actions sont indifférenciées.
>   - Chaque administrateur doit disposer d’un compte nominatif dédié et utiliser **sudo** pour toute élévation de privilèges.
>   - Les secrets d’authentification doivent être bien évidemment distincts entre les comptes utilisateur et les comptes d’administration.
>   - La traçabilité peut être renforcée via **auditd**, en journalisant la création de tous les processus.
>   - Le volume de logs généré par `auditd` peut être **très important** : cette mesure doit être combinée avec un **système de centralisation de journaux** (cf [Recommandations de sécurité pour l'achitecture d'un système de journalisation](https://messervices.cyber.gouv.fr/guides/recommandations-de-securite-pour-larchitecture-dun-systeme-de-journalisation).
>
> - *Procédure détaillée* :
>   1. **Créer des comptes d’administration nominatifs** :
>      - Un compte par administrateur (local ou distant).
>
>   2. **Interdire l’accès direct au compte root** :
>      - Verrouillage du compte :
>        ```bash
>        usermod -L -e 1 root
>        ```
>      - Désactivation du shell de connexion :
>        ```bash
>        usermod -s /bin/false root
>        ```
>
>   3. **Forcer l’usage de sudo pour toute action privilégiée** :
>      - Ajouter les administrateurs au groupe `sudo` :
>        ```bash
>        usermod -aG sudo <compte>
>        ```
>      - Vérifier la journalisation dans `/var/log/auth.log`.
>
>   4. **Activer la journalisation complète des créations de processus via auditd** :
>      - Ajouter les règles suivantes :
>        ```bash
>        -a exit,always -F arch=b64 -S execve,execveat
>        -a exit,always -F arch=b32 -S execve,execveat
>        ```
>      - Redémarrer le service :
>        ```bash
>        systemctl restart auditd
>        ```
>
>   5. **Exporter et centraliser les journaux** :
>      - Idéalement, il faudrait mettre en place un export régulier des logs vers un serveur de journalisation.
>
> - *Comparaison avec Lynis* :
>   Lynis vérifie bien la présence de `sudo`, l’état du compte `root` et certains paramètres de journalisation, mais **ne garantit pas une imputabilité complète** : il ne contrôle ni la traçabilité fine des processus via `auditd`, ni l’unicité réelle des identités administrateur.
>
> - *Référence* : [ANSSI_LINUX, R33]

### Comptes de service
Cette troisième sous-partie est dédiée aux services et applications qui ne s’exécutent généralement pas avec les privilèges d’un utilisateur classique, mais sous des comptes de service dédiés. De ce fait, cette séparation permet d’isoler chaque service (serveur web, DNS, base de données, etc.) et de limiter l’impact d’une compromission à un périmètre restreint. Toutefois, ces comptes n’ayant pas vocation à être utilisés pour des connexions interactives, leur accès doit être strictement désactivé afin d’empêcher toute tentative d’authentification abusive, conformément aux recommandations de l'ANSSI.

> [!warning] Désactiver les comptes de service
> - *Objectifs et intérêts* : Les comptes de service ne sont pas destinés à l’ouverture de sessions interactives, justifiant ainsi leur désactivation qui empêche toute connexion directe et limite les risques d’exploitation en cas de compromission d’un service. D'autre part, cela réduit fortement les possibilités de rebond, de persistance et d’escalade de privilèges depuis un service compromis.
>
> - *Commentaires* :
>   - Certains services peuvent s’exécuter avec le compte générique `nobody` (à noter que si plusieurs services partagent ce même compte, ils partagent **les mêmes UID, GID et privilèges système**, et donc cela ahnilé totalement l'isolation entre services).
>   - Un serveur Web et un annuaire exécutés sous `nobody` peuvent alors :
>     - se contrôler mutuellement,
>     - altérer leurs fichiers respectifs,
>     - s’envoyer des signaux (`kill`),
>     - utiliser `ptrace` entre eux,
>     - modifier leurs configurations respectives.
>
> - *Procédure détaillée* :
>   1. **Lister les comptes de service** :
>      ```bash
>      grep -E 'nologin|false' /etc/passwd
>      ```
>   2. **Vérifier les shells associés** :
>      - Chaque compte de service doit utiliser :
>        - `/usr/sbin/nologin`  
>        - ou `/bin/false`
>   3. **Désactiver les accès interactifs** si nécessaire :
>      ```bash
>      usermod -s /usr/sbin/nologin <service>
>      ```
>   4. **Vérifier qu’aucun service critique n’utilise `nobody`** :
>      ```bash
>      ps aux | grep nobody
>      ```
>   5. **Redémarrer les services après correction**.
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler la présence de comptes système avec un shell valide, mais il ne détecte néanmoins pas automatiquement les conflits d’usage du compte `nobody` entre plusieurs services.
>
> - *Référence* : [ANSSI_LINUX, R34]

> [!warning] Utiliser des comptes de service uniques et exclusifs
> - *Objectifs et intérêts* : Chaque service métier doit disposer de son propre compte système dédié afin de garantir une isolation stricte entre les composants applicatifs. Cette séparation empêche qu’un service compromis puisse interagir avec les ressources système d’un autre service.
>
> - *Commentaires* :
>   - Les services tels que `nginx`, `apache`, `mysql`, `postgres`, `php-fpm`, etc. doivent chacun avoir :
>     - un **UID dédié**,
>     - un **GID dédié**,
>     - des **répertoires propres**.
>   - Une nouvelle fois, l’utilisation d’un compte générique (`nobody`, `www`, etc.) pour plusieurs services est donc formellement déconseillée.
>
> - *Procédure détaillée* :
>   1. **Lister les services actifs** :
>      ```bash
>      systemctl list-units --type=service --state=running
>      ```
>   2. **Identifier les utilisateurs associés** :
>      ```bash
>      ps aux | awk '{print $1, $11}'
>      ```
>   3. **Créer un compte dédié pour chaque service métier** :
>      ```bash
>      useradd -r -s /usr/sbin/nologin -d /nonexistent <service>
>      ```
>   4. **Affecter le compte au service (systemd)** :
>      - Dans le fichier `*.service` :
>        ```
>        User=<service>
>        Group=<service>
>        ```
>   5. **Adapter les permissions des fichiers** :
>      ```bash
>      chown -R <service>:<service> /var/lib/<service>
>      ```
>   6. **Redémarrer et tester le bon fonctionnement du service**.
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler des services exécutés avec des comptes dangereux (`root`, `nobody`), mais il ne vérifie pas que chaque service dispose bien d’un compte strictement exclusif. 
>
> - *Référence* : [ANSSI_LINUX, R35]
