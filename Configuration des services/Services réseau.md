> [!info] Cloisonner les services réseau
> - *Objectifs et intérêts* : Limiter l’impact de la compromission d’un service réseau exposé (HTTP, SMTP, DNS, VPN, etc.) en l’isolant dans un environnement distinct. Ainsi, un service compromis ne doit pas permettre d’attaquer directement les autres services ou l’ensemble du système.
> - *Commentaires* :
>   - Le terme *service* est à prendre au sens large (tout composant accessible avant authentification robuste).
>   - Plusieurs niveaux de cloisonnement sont possibles : compte système dédié, *chroot*, conteneur (LXC, Docker, Podman), machine virtuelle, serveur physique distinct… Plus le cloisonnement est fort, plus l’intégration et l’administration sont complexes.
> - *Procédure détaillée* :
>   1. Recenser les services réseau exposés (ports ouverts, applications associées).
>   2. Définir pour chaque service un périmètre d’exécution dédié (compte Unix spécifique, répertoires, données).
>   3. Mettre en œuvre un cloisonnement adapté :
>      - a minima : compte Unix dédié + durcissement systemd (namespaces, chroot, `ProtectSystem=`, etc.) ;
>      - idéalement : conteneur ou machine virtuelle séparée pour chaque service critique.
>   4. Vérifier que les partages de ressources (sockets, volumes, fichiers) entre services sont strictement nécessaires.
> - *Comparaison avec Lynis* : Lynis peut détecter la présence de plusieurs services critiques sur un même hôte et signaler certains risques, mais il ne peut pas évaluer de manière exhaustive l’architecture de cloisonnement (VM, conteneurs, séparation physique).
> - Référence : [ANSSI_LINUX, R78]

> [!warning] Durcir et surveiller les services exposés
> - *Objectifs et intérêts* : Réduire le risque d’exploitation des services accessibles depuis des réseaux non maîtrisés (Internet, Wi-Fi publics…) et détecter rapidement tout comportement anormal. Le durcissement doit porter aussi bien sur la configuration que sur la supervision continue.
> - *Commentaires* :
>   - Toute vulnérabilité exploitable *avant* authentification est particulièrement critique.
>   - Un même service peut être attaqué à plusieurs niveaux : matériel, réseau (TCP/IP), protocolaire (TLS), applicatif (HTTP, requêtes, entrées utilisateur…).
> - *Procédure détaillée* :
>   1. Appliquer un guide de durcissement spécifique à chaque service (SSH, serveur Web, base de données, etc.) : désactivation des fonctionnalités inutiles, chiffrement fort, authentification robuste, limitation des commandes ou API exposées.
>   2. Mettre en place une surveillance :
>      - journalisation détaillée et centralisée des accès ;
>      - outils de détection d’anomalies (IDS/IPS, WAF, Fail2ban, etc.) ;
>      - alertes en cas de comportements non conformes au fonctionnement attendu (pics de connexions, erreurs répétées, tentatives de brute force).
>   3. Assurer un processus de mise à jour régulier (correctifs de sécurité) pour tous les composants exposés (service, bibliothèques, stack TLS, etc.).
> - *Comparaison avec Lynis* : Lynis réalise de nombreux tests ciblés sur les services exposés (SSH, HTTP(S), base de données, etc.) : options dangereuses, algorithmes obsolètes, versions vulnérables connues. Il ne remplace toutefois pas une supervision temps réel (SIEM, WAF, IDS).
> - Référence : [ANSSI_LINUX, R79]

> [!error] Réduire la surface d'attaque des services réseau
> - *Objectifs et intérêts* : Empêcher que des services ne soient accessibles sur des interfaces où ils ne sont pas nécessaires (par exemple, base de données écoutant sur toutes les interfaces au lieu de `127.0.0.1`). Moins un service est exposé, plus il est difficile à atteindre et à exploiter.
> - *Commentaires* : La plupart des services écoutent par défaut sur `0.0.0.0` (toutes les interfaces). Il faut restreindre explicitement l’adresse d’écoute aux seules interfaces pertinentes (boucle locale, réseau d’administration, VLAN spécifique…).
> - *Procédure détaillée* :
>   1. Lister les services en écoute sur le réseau :
>      - avec `sockstat` (comme indiqué par l’ANSSI), ou
>      - avec `ss -tulpen` / `netstat -tulpen` sous Linux.
>   2. Pour chaque service, identifier l’usage réel (local, interne, Internet) et adapter la configuration :
>      - serveur Web d’administration : écouter uniquement sur le réseau d’admin ;
>      - base de données locale : écouter uniquement sur `127.0.0.1` ;
>      - services internes : écouter uniquement sur les interfaces internes.
>   3. Mettre en place une politique de filtrage réseau (pare-feu) pour bloquer l’accès aux ports qui ne doivent pas être atteignables depuis certains segments (Internet, Wi-Fi invités, etc.).
>   4. Vérifier après modification que seuls les ports strictement nécessaires sont exposés.
> - *Comparaison avec Lynis* : Lynis inventorie les ports ouverts et les services associés, et signale certains services sensibles en écoute sur toutes les interfaces (par exemple SSH ou bases de données accessibles depuis l’extérieur).
> - Référence : [ANSSI_LINUX, R80]
