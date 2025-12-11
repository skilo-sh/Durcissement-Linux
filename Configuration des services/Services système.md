# `Pluggable Authentication Module` ou module d'authentification enfichable

> [!warning] Sécuriser les authentifications distantes par PAM
> - *Objectifs et intérêts* : Lorsque l'authentification repose sur un annuaire distant (LDAP, Kerberos, etc.), les échanges (mots de passe, tickets) ne doivent pas circuler en clair sur le réseau pour éviter l'interception. Et lorsqu'ils sont chiffrés, les mécanismes sous-jacent doivent être conforme à [ANSSI_CRYPTO].
> - *Commentaires* : Certains modules PAM ne chiffrent pas par défaut. Par exemple, `pam_ldap` ou `pam_mysql` doivent être explicitement configurés pour utiliser TLS/SSL.
> - *Procédure détaillée* : Configurer les fichiers clients (ex: `/etc/nslcd.conf`, `/etc/sssd/sssd.conf` ou `/etc/ldap/ldap.conf`) pour utiliser `uri ldaps://` ou forcer `ssl start_tls`. Vérifier la validité des certificats du serveur (`tls_reqcert demand`).
> - *Comparaison avec Lynis* : Lynis vérifie généralement la configuration LDAP pour s'assurer que le chiffrement (TLS/SSL) est activé. Mais Lynis ne pourra pas vérifier la conformité des mécanismes custom présents dans le PAM, une analyse manuelle est requise.
> - Référence : [ANSSI_LINUX, R67]

> [!error] Protéger les mots de passe stockés
> - *Objectifs et intérêts* : Empêcher la réutilisation des mots de passe en cas de vol de la base locale (`/etc/shadow`). Le stockage doit utiliser des fonctions de hachage robustes et lentes pour contrer les attaques par force brute.
> - *Commentaires* : L'ANSSI recommande l'algorithme **yescrypt** (rounds=11) ou à défaut **SHA-512** avec un coût élevé (rounds=65536). MD5 et SHA-256 sont à proscrire.
> - *Procédure détaillée* :
>   1. Modifier `/etc/pam.d/common-password` (ou équivalent selon la distribution).
>   2. Ajouter ou modifier la ligne `password required pam_unix.so` pour inclure `obscure yescrypt rounds=11` ou `sha512 rounds=65536`.
> - *Comparaison avec Lynis* : Lynis analyse `/etc/login.defs` et la configuration PAM pour identifier l'algorithme de hachage utilisé et alerte si MD5 ou DES sont détectés.
> - Référence : [ANSSI_LINUX, R68]

À noter que des directives existent afin de renforcer la résilience du PAM, par exemple :
- `pam_faillock` permet de bloquer temporairement un compte après un certain nombre d’échecs ;
- `pam_time` permet de restreindre les accès à une plage horaire ;
- `pam_passwdqc` permet d’appliquer des contraintes suivant une politique de complexité de mots de passe ;
- `pam_pwquality` permet de tester la robustesse des mots de passe ;
- `pam_wheel` permet de restreindre un accès aux utilisateurs membres d’un groupe particulier (wheel par défaut) ;
- `pam_mktemp` permet de fournir des répertoires temporaires privés par utilisateur sous /tmp ;
- `pam_namespace` permet de créer un espace de noms privés par utilisateur.
Il est également nécessaire de vérifier les permissions associées au fichiers de configuration du PAM — seul root doit y avoir accès en écriture.
# Name Service Switch ou service de gestion de noms

> [!warning] Sécuriser les accès aux bases utilisateur distantes
> - *Objectifs et intérêts* : NSS (Name Service Switch) permet au système de "voir" les utilisateurs distants (LDAP/AD). Comme pour l'authentification, ce canal doit être chiffré et le serveur authentifié pour éviter l'usurpation ou l'énumération de comptes par un attaquant sur le réseau.
> - *Commentaires* : Concerne les bases `passwd`, `group`, `shadow` configurées dans `/etc/nsswitch.conf`.
> - *Procédure détaillée* : Configurer le service client (SSSD, NSLCD) pour exiger une connexion chiffrée (LDAPS port 636 ou StartTLS port 389) et valider le certificat de l'autorité de certification (CA).
> - *Comparaison avec Lynis* : Lynis détecte la présence de configuration LDAP et vérifie si des paramètres de sécurité basiques sont présents.
> - Référence : [ANSSI_LINUX, R69]

> [!warning] Séparer les comptes système et d'administrateur de l'annuaire
> - *Objectifs et intérêts* : Le compte utilisé par le système Linux pour se connecter à l'annuaire (bind DN) ne doit pas être un compte administrateur de l'annuaire. En cas de compromission du serveur Linux, l'annuaire entier serait en danger.
> - *Commentaires* : Ce compte de service doit avoir les droits de lecture stricts nécessaires et rien de plus.
> - *Procédure détaillée* : Créer un utilisateur dédié dans l'annuaire (ex: `svc_linux_bind`) avec des droits limités. Configurer ce compte dans `/etc/sssd/sssd.conf` ou `/etc/nslcd.conf`.
> - *Comparaison avec Lynis* : Lynis ne peut pas vérifier les privilèges du compte distant, c'est une vérification organisationnelle.
> - Référence : [ANSSI_LINUX, R70]

# Journalisation

> [!info] Mettre en place un système de journalisation
> - *Objectifs et intérêts* : La centralisation et la gestion des logs sont essentielles pour la détection d'incidents et l'analyse post-mortem (forensic).
> - *Commentaires* : Utiliser `rsyslog` ou `syslog-ng`. Il est recommandé d'envoyer les logs vers un serveur centralisé pour éviter leur altération locale par un attaquant.
> - *Procédure détaillée* : Installer le paquet rsyslog. S'assurer que le service est actif : `systemctl enable --now rsyslog`. Configurer `/etc/rsyslog.conf` pour définir les règles de collecte.
> - *Comparaison avec Lynis* : Lynis vérifie la présence d'un démon de journalisation, son état actif, et si l'envoi de logs distants est configuré.
> - Référence : [ANSSI_LINUX, R71]

> [!info] Mettre en place des journaux d'activité de service dédiés
> - *Objectifs et intérêts* : Séparer les logs par service facilite l'analyse et protège l'intégrité des journaux (un service compromis ne doit pas pouvoir effacer les traces des autres).
> - *Commentaires* : Les services ne doivent pas écrire directement dans des fichiers (droits d'écriture dangereux), mais passer par l'API syslog (ex: `/dev/log`).
> - *Procédure détaillée* : Configurer les services (Apache, Nginx, SSH, etc.) pour utiliser les facilités Syslog (`local0` à `local7`). Configurer rsyslog pour rediriger ces facilités vers des fichiers distincts dans `/var/log/`.
> - *Comparaison avec Lynis* : Lynis vérifie les permissions des fichiers dans `/var/log` pour s'assurer qu'ils ne sont pas inscriptibles par tout le monde.
> - Référence : [ANSSI_LINUX, R72]

> [!info] Journaliser l'activité système avec auditd
> - *Objectifs et intérêts* : `auditd` permet une surveillance beaucoup plus fine que syslog (appels système, accès fichiers, chargement de modules kernel). C'est indispensable pour détecter des activités suspectes complexes.
> - *Commentaires* : Attention, une configuration trop verbeuse peut impacter les performances. Les règles doivent être verrouillées (`-e 2`) pour empêcher leur modification sans redémarrage.
> - *Procédure détaillée* : 
>   1. Installer `auditd` et `audispd-plugins`.
>   2. Configurer `/etc/audit/audit.rules` (surveiller `execve`, `mount`, modification de `/etc/`, chargement de modules, etc.).
>   3. Activer le service : `systemctl enable --now auditd`.
> - *Comparaison avec Lynis* : Lynis effectue un contrôle approfondi d'auditd : présence, statut, et qualité des règles définies (si elles sont vides ou insuffisantes).
> - Référence : [ANSSI_LINUX, R73]

Afin d'aller plus loin, le lecteur peut se référer à [ANSSI_LOG] afin d'avoir d'avantage de détail tant qu'a l'architecture d'un système de journalisation.
# Messagerie

> [!warning] Durcir le service de messagerie locale
> - *Objectifs et intérêts* : Empêcher qu’un MTA local (Postfix, Exim, etc.) ne soit exploitable comme relais de spam ou comme point d’entrée réseau, tout en conservant la capacité d’envoyer les notifications système (cron, jobs divers) vers les comptes locaux.
> - *Commentaires* : Le service de messagerie ne doit servir qu’à la distribution **locale** des mails générés par le système. Si un relais sortant est nécessaire (SMTP d’un FAI ou d’une passerelle), privilégier un outil simple de redirection (ssmtp, msmtp, nullmailer…) plutôt qu’un MTA complet.
> - *Procédure détaillée* :
>   - Identifier le MTA installé (`postfix`, `exim4`, `sendmail`, etc.).
>   - Le configurer pour :
>     - n’accepter que les messages **destinés à des comptes locaux** ;
>     - n’écouter que sur l’interface **loopback** (127.0.0.1).
>   - Exemple Postfix (fichier `/etc/postfix/main.cf`) :
>     - `inet_interfaces = loopback-only`
>     - `mydestination = $myhostname, localhost.$mydomain, localhost`
>   - Redémarrer le service (`systemctl restart postfix` ou équivalent) et vérifier qu’il n’écoute pas sur les interfaces réseau externes (`ss -tulpn | grep :25`).
> - *Comparaison avec Lynis* : Lynis vérifie la présence d’un MTA, tente de détecter un relais ouvert et signale les services d’écoute SMTP accessibles depuis le réseau.
> - *Référence* : [ANSSI_LINUX, R74]

> [!warning] Configurer un alias de messagerie des comptes de service
> - *Objectifs et intérêts* : S’assurer que tous les messages envoyés aux comptes techniques (en particulier `root`) sont effectivement reçus et lus par un ou plusieurs administrateurs identifiés, même si ces comptes ne sont jamais utilisés pour se connecter.
> - *Commentaires* : Conformément à R33, le compte `root` ne doit pas être utilisé pour les connexions interactives. Sans alias, les notifications critiques (cron, rapports, alertes de sécurité) risquent de ne jamais être consultées.
> - *Procédure détaillée* :
>   - Éditer le fichier d’alias du MTA (généralement `/etc/aliases` ou `/etc/postfix/aliases`).
>   - Créer des alias pour chaque compte de service pertinent, notamment :
>     - `root: admin1@example.org, admin2@example.org`
>     - `www-data: admin-web@example.org`
>   - Mettre à jour la base des alias (par exemple `newaliases` pour Postfix).
>   - Vérifier le bon routage en envoyant un mail de test : `echo test | mail -s "test root" root`.
> - *Comparaison avec Lynis* : Lynis contrôle généralement la présence d’un alias pour `root` et signale l’absence de redirection des mails de ce compte.
> - *Référence* : [ANSSI_LINUX, R75]
# Références
- [ANSSI_LINUX] : 
- [ANSSI_CRYPTO] :
- [ANSSI_LOG] :