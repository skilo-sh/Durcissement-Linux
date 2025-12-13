> [!error] Effectuer des mises à jour de sécurité régulières
> - *Objectifs et intérêts* : Tout système doit être maintenu en conditions de sécurité afin de limiter son exposition aux vulnérabilités connues. Les mises à jour corrigent des failles de sécurité, peuvent améliorer les mécanismes de protection existants et réduire la surface d’attaque exploitable. Une gestion proactive des correctifs permet de limiter les fenêtres d’exposition entre la divulgation d’une vulnérabilité et l’application de son correctif.
>
> - *Commentaires* :
>   - Les correctifs de sécurité sont publiés en continu par les éditeurs et les distributions ; un système non maintenu devient rapidement vulnérable.
>   - Une **veille sécurité** est indispensable afin d’identifier les vulnérabilités affectant le système et de prioriser l’application des correctifs.
>   - Sur Debian, les mises à jour de sécurité sont distribuées via les dépôts officiels (`security.debian.org`) et peuvent être automatisées de manière contrôlée.
>
> - *Procédure détaillée* :
>   1. **Vérifier la configuration des dépôts de sécurité** :
>      - S’assurer que le dépôt sécurité est présent dans `/etc/apt/sources.list` ou `/etc/apt/sources.list.d/*.list` :
>        ```
>        deb http://security.debian.org/debian-security bookworm-security main contrib non-free-firmware
>        ```
>   2. **Mettre à jour la liste des paquets** :
>      ```bash
>      apt update
>      ```
>   3. **Appliquer les mises à jour de sécurité** :
>      - Mise à jour standard :
>        ```bash
>        apt upgrade
>        ```
>      - Mise à jour complète (si nécessaire, avec gestion des dépendances) :
>        ```bash
>        apt full-upgrade
>        ```
>   4. **Automatiser les mises à jour de sécurité (recommandé)** :
>      - Installer l’outil dédié :
>        ```bash
>        apt install unattended-upgrades apt-listchanges
>        ```
>      - Activer les mises à jour automatiques de sécurité :
>        ```bash
>        dpkg-reconfigure unattended-upgrades
>        ```
>      - Vérifier la configuration dans :
>        - `/etc/apt/apt.conf.d/50unattended-upgrades`
>        - `/etc/apt/apt.conf.d/20auto-upgrades`
>   5. **Vérifier les correctifs appliqués** :
>      - Lister les paquets pouvant être mis à jour :
>        ```bash
>        apt list --upgradable
>        ```
>      - Vérifier l’historique des mises à jour :
>        ```bash
>        grep -i upgrade /var/log/dpkg.log
>        ```
>   6. **Redémarrages et services** :
>      - Identifier si un redémarrage est requis (noyau, libc…) :
>        ```bash
>        needrestart
>        ```
>      - Redémarrer les services impactés ou le système si nécessaire.
>   7. **Mettre en place une veille sécurité** :
>      - S’abonner aux bulletins de sécurité Debian :
>        - https://www.debian.org/security/
>      - Suivre les alertes du CERT-FR :
>        - https://www.cert.ssi.gouv.fr/
>      - Surveiller les CVE affectant les composants critiques du système.
>
> - *Comparaison avec Lynis* :
>   Lynis peut signaler l’absence de mises à jour récentes ou un système obsolète, mais **ne garantit pas** la mise en place d’une procédure régulière, réactive et documentée ni l’existence d’une veille sécurité. Le respect complet de cette règle repose sur une organisation opérationnelle et un suivi humain.
>
> - *Référence* : [ANSSI_LINUX, R61] – Recommandations de configuration d’un système GNU/Linux, section *Veille et maintenance* :contentReference[oaicite:0]{index=0}


