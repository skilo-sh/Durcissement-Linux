L'objectif de la configuration matérielle est de paramétrer certaines options matérielles afin de durcir le socle qui va servir à l’installation. Ce paramétrage doit être fait de préférence avant l’installation pour que le système soit capable d’en tenir compte le plus tôt possible.

Étant dans le contexte d'une machine virtuelle, nous n'allons parler que des méthodes suivantes :
- Démarrage sécurisé (de l'anglais *Secure Boot*) : mécanismes permettant de vérifier (par signature cryptographique) le code chargé par l'UEFI,
- Démarrage de confiance (de l'anglais *Measured Boot*) : complémentaire au *Secure Boot*, permet vérifier que le système démarré est dans un état précis (par vérification des PCR du TPM).

Ces deux mécanismes ne sont pas supportés sur tout les superviseurs, les recommandations ci-dessous ne seront donc pas applicables dans certains cas. De plus, pour le démarrage de confiance, il est nécessaire que le superviseurs supporte la virtualisation de TPM.

Un lecteur voulant creuser le sujet de la sécurité matériel est invité à lire [ANSSI_LINUX] et [ANSSI_MAT] ; nous n'entrons pas d'avantage dans les détails dans le cadre de ce wiki.

> [!info] Démarrage sécurisé
> - *Objectifs et intérêts* : Garantir que seuls des composants de démarrage (firmware UEFI, chargeur, noyau EFI) **signés et autorisés** sont exécutés, afin de limiter les bootkits/rootkits et les compromissions avant le chargement du système d’exploitation.
> - *Commentaires* : Le démarrage sécurisé UEFI repose sur des bases de signatures (`db`, `dbx`) et des clés (`KEK`, `PK`). Sur la plupart des plates-formes x86, des clés Microsoft sont préchargées et un **SHIM** signé est utilisé par les distributions pour chaîner la vérification jusqu’au noyau. En environnement virtualisé, cette fonctionnalité n’est disponible que si l’hyperviseur expose un firmware UEFI avec Secure Boot (ex. OVMF avec Secure Boot, Hyper‑V Gen2, etc.).
> - *Procédure détaillé* :
>   1. Créer/convertir la VM en mode **UEFI** et activer l’option *Secure Boot* dans la configuration de la VM (si disponible sur l’hyperviseur).
>   2. Installer une distribution Linux dont le **SHIM** et le noyau sont signés et reconnus par les clés préchargées de l’UEFI (ou par vos propres clés, si vous avez mis en place une PKI).
>   3. (Option avancée, cf. R4 ANSSI) Remplacer les clés préchargées (PK/KEK/db/dbx) par vos propres clés, puis signer vous‑même le chargeur/noyau.
>   4. Vérifier côté invité que le Secure Boot est bien actif (par ex. via `mokutil --sb-state` ou les messages du noyau).
> - *Comparaison avec Lynis* : Lynis peut détecter la présence d’un firmware UEFI et, selon la plate-forme, indiquer si le Secure Boot semble activé ou non. Il ne gère pas la configuration des clés (PK/KEK/db/dbx) et ne remplace pas un audit spécifique du firmware/hyperviseur.
> - Référence : [ANSSI_LINUX]

> [!info] Démarrage de confiance
> - *Objectifs et intérêts* : S’assurer que le système démarré se trouve dans un **état précisément attendu**, vérifiable via les registres de configuration de plate-forme (PCR) du TPM. Cela permet, par exemple, de ne libérer une clé de chiffrement (disque, secret applicatif) **que si** la chaîne de démarrage n’a pas été altérée.
> - *Commentaires* : Le démarrage de confiance complète le Secure Boot : celui-ci vérifie des signatures, mais peut accepter plusieurs versions signées ; le Measured Boot enregistre dans les PCR l’état réel de la chaîne de démarrage (firmware, binaires UEFI, état du Secure Boot, etc.). Toute mise à jour d’un composant mesuré modifie les PCR, ce qui impose de **mettre à jour la politique de descellement** des secrets. En VM, cette fonctionnalité nécessite un **TPM (ou vTPM) exposé par l’hyperviseur**.
> - *Procédure détaillé* :
>   1. Sur l’hyperviseur, activer et attacher un **TPM virtuel** (TPM 2.0 de préférence) à la VM.
>   2. Dans l’invité, vérifier la présence du TPM (ex. `dmesg | grep -i tpm`, `ls /dev/tpm*`, outils `tpm2-tools`).
>   3. Configurer le système pour utiliser le TPM et ses PCR comme condition d’accès à des secrets (ex. chiffrement de disque ou de partitions, clés applicatives) : scellement d’un secret avec une politique basée sur un ensemble de PCR.
>   4. Documenter et tester la procédure de **mise à jour de la politique PCR** lors des mises à jour de firmware/UEFI/chargeur/noyau, afin d’éviter de bloquer l’accès légitime aux secrets après maintenance.
> - *Comparaison avec Lynis* : Lynis peut vérifier la présence d’un TPM et signaler certains paramètres de base, mais ne valide pas la configuration détaillée du Measured Boot ni des politiques de scellement/dessellement associées. Un outillage spécifique TPM est nécessaire pour cela.
> - Référence : [ANSSI_LINUX]
# Références
- [ANSSI_LINUX] : https://cyber.gouv.fr/publications/recommandations-de-securite-relatives-un-systeme-gnulinux
- [ANSSI_MAT] : https://cyber.gouv.fr/publications/recommandations-de-configuration-materielle-de-postes-clients-et-serveurs-x86