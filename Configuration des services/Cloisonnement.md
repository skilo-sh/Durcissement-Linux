Le cloisonnement est une technique qui vise à isoler les processus s’exécutant sur un même système. Historiquement, il était réalisé au travers de l’utilitaire *chroot* qui permet de restreindre la vue du système de fichiers d’un processus à un répertoire donné qui devient sa racine. Une fois confiné ou *chrooté*, le processus ne peut alors plus accéder aux répertoires autres que celui pris comme racine ainsi que ses sous-répertoires. Il n’a donc qu’une vision partielle du système de fichiers.

> [!info] Cloisonner les services
> - *Objectifs et intérêts* : Cloisonner les services afin de limiter leur visibilité sur le système (système de fichiers, réseau, autres processus, ressources) et de contenir l’impact d’un éventuel compromis à un périmètre réduit, en appliquant le principe de minimisation.
> - *Commentaires* :
>   - Le cloisonnement historique via **chroot(2)** est désormais insuffisant sous GNU/Linux :
>     - root peut en sortir ;
>     - certains systèmes permettent à un utilisateur non privilégié d’en sortir avec l’aide d’un autre processus ;
>     - il ne cloisonne que le système de fichiers (accès aux processus, au réseau, etc. non bloqués) ;
>     - il nécessite les privilèges initiaux de root.
>   - Les mécanismes de cloisonnement modernes du noyau Linux sont à privilégier :
>     - **Linux namespaces** : partitionnent diverses ressources (PID, montage, réseau, UTS, IPC, users…) ;
>     - **seccomp / seccomp‑BPF** : filtrent les appels systèmes des processus ;
>     - **cgroups** (control groups) : limitent et isolent l’usage des ressources (CPU, mémoire, I/O…).
>   - Ces briques sont exposées par les systèmes d’initialisation (systemd, OpenRC, …) et par des solutions de plus haut niveau :
>     - Conteneurs : LXC, Docker, podman… ;
>     - Émulation complète : QEMU, VirtualBox… ;
>     - Hyperviseurs : Xen (bare‑metal), KVM (hyperviseur noyau)…
>   - Un compromis du noyau ou de l’hyperviseur implique en général le compromis de tous les services cloisonnés au‑dessus.
> - *Procédure détaillé* :
>   1. **Identifier les services à cloisonner en priorité** :
>      - Services exposés sur Internet ou sur des réseaux peu maîtrisés ;
>      - Services traitant des données non fiables (entrées utilisateur, flux externes) ;
>      - Services s’exécutant avec des privilèges élevés ou manipulant des données sensibles.
>   2. **Choisir le niveau de cloisonnement approprié** :
>      - Cloisonnement léger dans le même OS via les mécanismes systemd (namespaces, cgroups, seccomp) ;
>      - Cloisonnement plus fort via **conteneur dédié** (LXC, Docker, podman…) ;
>      - Cloisonnement maximal via **machine virtuelle** ou **hyperviseur** distinct.
>   3. **Mettre en œuvre le cloisonnement avec systemd** (recommandé pour la granularité) :
>      - Dans l’unité `*.service`, utiliser notamment :
>        - `User=` / `Group=` pour exécuter le service sous un compte dédié ;
>        - `PrivateTmp=yes`, `PrivateDevices=yes` ;
>        - `ProtectSystem=full` ou `strict`, `ProtectHome=yes` ;
>        - `ProtectKernelModules=yes`, `ProtectKernelTunables=yes`, `ProtectControlGroups=yes` ;
>        - `NoNewPrivileges=yes` ;
>        - `RestrictAddressFamilies=`, `RestrictNamespaces=`, `RestrictSUIDSGID=yes` ;
>        - `MemoryDenyWriteExecute=yes` lorsque compatible avec l’application ;
>        - Limitation des ressources via cgroups : `CPUQuota=`, `MemoryMax=`, etc.
>      - Activer le filtrage d’appels systèmes via seccomp :
>        - `SystemCallFilter=` et directives associées si supportées par la version de systemd.
>   4. **Mettre en œuvre le cloisonnement par conteneur** (si pertinent) :
>      - Utiliser des conteneurs **rootless** dès que possible ;
>      - Employer des images minimales, idéalement avec système de fichiers racine en lecture seule ;
>      - Restreindre les Linux capabilities, limiter les montages de volumes, réduire l’exposition réseau ;
>      - Compléter avec des profils SELinux/AppArmor et seccomp.
>   5. **Mettre en œuvre le cloisonnement par virtualisation** :
>      - Isoler dans des VM dédiées les services les plus exposés ou les plus critiques ;
>      - Séparer les réseaux (production, administration, stockage, sauvegarde) ;
>      - Restreindre les périphériques passés en direct (PCI passthrough, USB, dossiers partagés).
>   6. **Tester et documenter** :
>      - Vérifier que chaque service fonctionne correctement dans son cloisonnement ;
>      - Documenter pour chaque service : mécanisme de cloisonnement retenu, options activées, justification.
> - *Comparaison avec Lynis* :
>   - Lynis détecte la présence de virtualisation ou de conteneurs et signale certains durcissements systemd possibles.
>   - Il ne conçoit pas l’architecture de cloisonnement (choix conteneur/VM/hyperviseur, niveau d’isolation par service) et ne peut pas valider automatiquement que chaque service est cloisonné de manière adéquate.
> - Référence : [ANSSI_LINUX, R65]

> [!info] Durcir les composants de cloisonnement
> - *Objectifs et intérêts* : Renforcer la sécurité de tous les composants utilisés pour cloisonner (noyau, namespaces, seccomp, cgroups, moteurs de conteneurs, hyperviseurs, solutions de virtualisation), afin qu’ils ne deviennent pas une cible privilégiée permettant de contourner le cloisonnement.
> - *Commentaires* :
>   - Les composants de cloisonnement constituent une **surface d’attaque critique** :
>     - moteur de conteneurs (Docker, containerd, podman, LXC…),
>     - hyperviseur / stack de virtualisation (KVM, Xen, QEMU, libvirt, VirtualBox…),
>     - orchestrateurs (Kubernetes, OpenStack…),
>     - ainsi que le noyau lui‑même et ses interfaces (namespaces, cgroups, seccomp).
>   - Un attaquant qui compromet l’hyperviseur, le noyau ou le moteur de conteneurs peut souvent compromettre l’ensemble des services ou des environnements s’appuyant sur ce composant.
>   - Ces éléments doivent donc être durcis au même titre – voire davantage – que les services applicatifs qu’ils hébergent.
> - *Procédure détaillé* :
>   1. **Recenser tous les composants de cloisonnement** :
>      - Mécanismes noyau utilisés (namespaces, seccomp, cgroups, capabilities) ;
>      - Moteurs de conteneurs : Docker, containerd, podman, LXC/LXD, CRI‑O, … ;
>      - Stack de virtualisation : QEMU/KVM, Xen, libvirt, VirtualBox, VMware, … ;
>      - Éventuels orchestrateurs : Kubernetes, OpenShift, OpenStack, etc.
>   2. **Durcir l’hôte support de cloisonnement** :
>      - Installer un système minimal, sans services superflus ;
>      - Appliquer des paramètres noyau de durcissement (sysctl, LSM comme AppArmor/SELinux, protections mémoire, etc.) ;
>      - Maintenir le noyau, l’hyperviseur et les moteurs de conteneurs **strictement à jour** (prioriser les correctifs de sécurité).
>   3. **Restreindre l’accès aux interfaces d’administration** :
>      - Limiter l’accès aux sockets locaux (ex. `/var/run/docker.sock`, `libvirtd`) à un groupe d’administrateurs spécifique ;
>      - Désactiver les APIs réseau inutiles, ou les protéger (filtrage, TLS, authentification forte) ;
>      - Séparer les réseaux d’administration de ceux de production.
>   4. **Configurer les composants de cloisonnement de manière restrictive** :
>      - **Pour les conteneurs** :
>        - Interdire ou encadrer strictement l’usage de `--privileged` ;
>        - Limiter les montages de volumes et exclure les parties sensibles de l’hôte (`/`, `/proc`, `/sys`, périphériques bruts) ;
>        - Appliquer des profils **seccomp** adaptés, des politiques **AppArmor/SELinux**, des **user namespaces** et une réduction des capabilities ;
>        - Privilégier les déploiements **rootless** lorsque possible.
>      - **Pour la virtualisation / hyperviseurs** :
>        - Activer les mécanismes de confinement entre hôte et invités (sVirt, SELinux/AppArmor pour libvirt, protections mémoire invité) ;
>        - Restreindre les canaux de communication invité↔hôte (dossiers partagés, clipboard, USB, PCI passthrough) au strict nécessaire ;
>        - Durcir les consoles et interfaces de gestion (accès restreint, journalisé).
>   5. **Journaliser et superviser** :
>      - Activer une journalisation détaillée des composants (logs Docker/podman, libvirt, hyperviseur, orchestrateur, auditd) ;
>      - Mettre en place une supervision pour détecter :
>        - la création de conteneurs ou de VM inhabituels,
>        - l’usage d’options trop permissives,
>        - des erreurs ou tentatives d’accès répétées.
>   6. **Réaliser des revues régulières** :
>      - Auditer périodiquement les configurations et les permissions des composants de cloisonnement ;
>      - Appliquer les guides de durcissement spécifiques fournis par les éditeurs / communautés (Docker, Kubernetes, hyperviseurs, etc.).
> - *Comparaison avec Lynis* :
>   - Lynis détecte généralement les environnements virtualisés ou l’usage de conteneurs et fournit des recommandations de durcissement générales de l’hôte.
>   - En revanche, il ne couvre pas en détail le durcissement spécifique de chaque composant de cloisonnement (configuration fine de Docker/podman, paramètres d’hyperviseur, profils seccomp, politiques LSM, etc.). Une revue manuelle ciblée est nécessaire pour satisfaire pleinement cette règle.
> - Référence : [ANSSI_LINUX, R66]