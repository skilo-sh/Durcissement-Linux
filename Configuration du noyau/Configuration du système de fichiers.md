
> [!warning] Paramétrer les options de configuration des systèmes de fichiers
> - *Objectifs et intérêts* :  
>   - Réduire les possibilités d’escalade de privilèges via des mécanismes du système de fichiers (coredumps, liens symboliques, liens durs, FIFOs…).  
>   - Limiter certaines classes d’attaques de type TOCTOU (Time Of Check – Time Of Use) exploitant des répertoires partagés (*/tmp*, répertoires sticky, etc.).  
>
> - *Commentaires* : 
>   - Pour une VM debian minimale cette recommandation est pertinente car les paramètres proposés sont génériques, peu intrusifs et adaptés à la majorité des usages.
>   - Les paramètres `fs.protected_fifos =2` et `fs.protected_regular =2` ne sont disponibles que depuis la version 4.19 du noyau linux. Il faut donc adapter la configuration avec sa version.
> - *Procédure détaillé* :  
>   1. Créer (ou éditer) un fichier de configuration sysctl dédié :  
>      - `/etc/sysctl.d/99-hardening-fs.conf`  
>   2.  Vérifier la version du noyau :
>      - `uname -r`
>   3. Y ajouter les options recommandées en fonction de sa version de noyau :
>      ```ini
>      # Désactiver la création de coredump pour les exécutables setuid
>      fs.suid_dumpable = 0
>      
>      # Protéger l'ouverture des FIFOs et fichiers "réguliers" non possédés
>      # par l'utilisateur dans les répertoires sticky en écriture pour tous
>      fs.protected_fifos   = 2
>      fs.protected_regular = 2
>      
>      # Restreindre les liens symboliques (prévention TOCTOU)
>      fs.protected_symlinks = 1
>      
>      # Restreindre les liens durs (prévention TOCTOU et réutilisation de fichiers obsolètes)
>      fs.protected_hardlinks = 1
>      ```
>   4. Appliquer la configuration sans redémarrage (si nécessaire) :  
>      ```bash
>      sudo sysctl --system
>       ```
>   5. Vérifier la prise en compte :
>      ```bash
>      sysctl fs.suid_dumpable fs.protected_fifos fs.protected_regular \
>             fs.protected_symlinks fs.protected_hardlinks
>      ```
>
> - *Comparaison avec Lynis* :  Lynis vérifie ces points dans la section *Kernel Hardening*.
>
> - *Référence* : [ANSSI_LINUX, R14]


