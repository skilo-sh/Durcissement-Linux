Quelques recommandations générales --- essentiellement basé sur le guide de l'ANSSI :

> [!error] Désactiver les services non nécessaires
> - *Objectifs et intérêts* : Lors de l’installation d’une distribution, certains services sont installés par défaut selon le choix des mainteneurs de la distribution. Ces services ne correspondent pas forcément aux besoins propres. Désactiver les services non nécessaires permet de réduire la surface d'attaque.
> - *Commentaires* : N/A
> - *Procédure détaillé* : Il s'agit dans un premier temps de lister les services activés (exemple : `systemctl list -units --type service` sous *systemd*) puis d'itérer sur chaque service pour le désactiver si non utilisés. 
> - *Comparaison avec Lynis* : Lynis ne permet pas de détecter une telle configuration.
> - Référence : [ANSSI_LINUX, R62]

> [!warning] Désactiver les fonctionnalités des services non essentielles
> - *Objectifs et intérêts* : Limiter chaque service au strict nécessaire fonctionnel (protocoles, modules, options activées) afin de réduire la surface d’attaque et les possibilités d’abus en cas de vulnérabilité.
> - *Commentaires* :
>   Les configurations par défaut sont généralement génériques et permissives. C’est particulièrement vrai pour :
>    - SMTP (Exim, Postfix) : risques de relais ouvert, services inutiles (VRFY/EXPN, ports non utilisés, relais non authentifié, etc.).
>    - NTP (ntpd / chronyd) : requêtes non restreintes, fonctions de contrôle à distance, statistiques ou monitoring inutiles.
>    - DNS (Bind) : récursivité ouverte, transferts de zone non restreints, écoute sur toutes les interfaces, modules inutiles.
> - *Procédure détaillé* :
>   1. **Lister les services actifs et exposés** :
>      - Services réseau : `ss -tulpen` ou `ss -lntup`
>      - Services en cours : `systemctl list-units --type=service --state=running`
>   2. **Pour chaque service** :
>      - Identifier les fonctionnalités réellement nécessaires (protocoles, ports, modules, interfaces, commandes de maintenance, APIs, etc.).
>      - Dans les fichiers de configuration, désactiver tout ce qui n’est pas requis :
>        - Services ou modules optionnels.
>        - Ports/protocoles non utilisés.
>        - Interfaces réseau ou adresses de rebouclage inutiles.
>        - Fonctions de debug, d’administration distante ou de statistiques non indispensables.
>   3. **Redémarrer et vérifier** :
>      - Redémarrer les services modifiés (`systemctl restart <service>`).
>      - Vérifier l’exposition effective (`ss -tulpen`, tests fonctionnels) et les journaux.
>   4. **Documenter** les fonctionnalités désactivées et la justification.
> - *Comparaison avec Lynis* : Lynis peut signaler certains services réseau et quelques mauvaises pratiques de configuration, mais il ne sait pas déterminer quelles fonctionnalités sont ou non nécessaires dans un contexte donné. Il ne permet donc pas de valider automatiquement l’application fine de cette règle.
> - Référence : [ANSSI_LINUX, R63]

> [!info] Configurer les privilèges des services
> - *Objectifs et intérêts* : Appliquer le principe de moindre privilège à tous les services et exécutables : chaque composant doit disposer uniquement des droits (UID/GID, permissions, Linux capabilities) strictement nécessaires à son fonctionnement, afin de limiter l’impact d’un compromis.
> - *Commentaires* :
>   - Le noyau Linux décompose les privilèges root en **Linux capabilities** (voir `man 7 capabilities`). Il en existe plus de 40, dont une grande partie permet ou facilite l’élévation à des droits *root* complets.
>   - Des capacités peuvent être associées à des exécutables (fichiers sur disque) et à des processus ; leur usage doit être **exceptionnel** et limité.
>   - Certains services ont besoin de privilèges élevés au démarrage (ou de manière permanente) pour accéder à des ports privilégiés, des périphériques ou des ressources système : ces besoins doivent être identifiés et strictement bornés.
> - *Procédure détaillé* :
>   1. **Inventorier les services privilégiés** :
>      - Lister les services système : `systemctl list-units --type=service --state=running`
>      - Identifier ceux qui tournent en root ou avec des droits élevés (`systemctl status <service>`, `ps aux | grep ^root`).
>   2. **Lister les exécutables dotés de capabilities** :
>      ```bash
>      find / -type f -perm /111 -exec getcap {} \; 2>/dev/null
>      ```
>      - Examiner chaque binaire listé : vérifier si la capability est réellement nécessaire.
>   3. **Réduire les privilèges des services** :
>      - **Comptes dédiés** :
>        - S’assurer que chaque service dispose d’un utilisateur système dédié, non interactif (`/usr/sbin/nologin` ou `/bin/false`).
>      - **Configuration des services (ex. systemd)** : dans les unités `*.service` :
>        - `User=` et `Group=` pour exécuter le service avec un compte dédié.
>        - `CapabilityBoundingSet=` pour limiter le jeu de capabilities accessible.
>        - `AmbientCapabilities=` uniquement pour les capabilities strictement nécessaires.
>        - `NoNewPrivileges=yes` pour empêcher toute élévation de privilèges ultérieure.
>      - Pour les services qui doivent démarrer en root (par ex. ouverture de ports <1024), utiliser leur mécanisme intégré de *drop privileges* (changement d’UID/GID après initialisation) dès que possible.
>   4. **Ajuster ou supprimer les capabilities de fichiers** :
>      - Retirer une capability devenue inutile : `setcap -r /chemin/vers/binaire`
>      - Éviter d’ajouter des capabilities sauf nécessité clairement documentée.
>   5. **Tester et valider** :
>      - Redémarrer les services, vérifier leur bon fonctionnement, les journaux, et l’absence de régression fonctionnelle.
> - *Comparaison avec Lynis* : Lynis peut signaler certains services s’exécutant en root ou des permissions de fichiers trop larges, mais il ne réalise pas une analyse exhaustive des privilèges par service (comptes dédiés, capabilities associées, options systemd comme `CapabilityBoundingSet` ou `NoNewPrivileges`). Le respect complet de cette règle nécessite donc une revue manuelle.
> - Référence : [ANSSI_LINUX, R64]
# Références
- [ANSSI_LINUX] : https://cyber.gouv.fr/publications/recommandations-de-securite-relatives-un-systeme-gnulinux