
> [!warning] Activer et configurer le LSM Yama 
> - *Objectifs et intérêts* : Limiter l’usage de l’appel système `ptrace`, qui permet à un processus de lire/modifier la mémoire d’un autre processus (débogage). Par défaut, tout utilisateur peut tracer tous ses propres processus, ce qui permet à un attaquant ayant compromis un simple processus utilisateur de récupérer des secrets en mémoire (mots de passe de navigateur, clés dans `ssh-agent`, jetons de session, etc.). Restreindre `ptrace` réduit fortement la surface d’attaque entre processus d’un même utilisateur.
> 
> - *Commentaires* :  
>   - Impact possible sur les outils de débogage/profiling comme `gdb`.  
>   - Valeurs possibles de `kernel.yama.ptrace_scope` :  
>     - `0` : comportement classique, tous les processus d’un même UID sont traçables.  
>     - `1` (recommandé a minima) : un processus non privilégié ne peut tracer que ses descendants, sauf autorisation explicite (via `prctl`).  
>     - `2` : seul un processus privilégié (avec `CAP_SYS_PTRACE`) peut utiliser `ptrace`.  
>     - `3` : interdiction totale de `ptrace`.  
>   - Pour une VM Debian minimale, une valeur `1` est généralement un bon compromis ; les valeurs 2 ou 3 sont réservées pour des VMs plus sensibles ne nécessitant pas de débogage.
> 
> - *Procédure détaillée* :  
> 
>   1. **Activer Yama au démarrage (si non actif)**  
> 	 -Vérifier si Yama est actif :
> 	```bash
> 	 cat /sys/kernel/security/lsm
> 	 ```
>      - Si Yama est innactif éditer `/etc/default/grub` et ajouter la directive au noyau :  
>        ```bash
>        GRUB_CMDLINE_LINUX_DEFAULT="quiet lsm=yama"
>        ```  
>        (adapter en fonction des autres LSM utilisés, pour ne pas désactiver AppArmor/SELinux si présents).  
>      - Mettre à jour la configuration de GRUB :  
>        ```bash
>        sudo update-grub
>        ```  
>      - Redémarrer la VM et vérifier que `yama` est bien présent dans `cat /sys/kernel/security/lsm`.
> 
>   3. **Configurer `kernel.yama.ptrace_scope`**  
>      - Créer le fichier de configuration `/etc/sysctl.d/99-yama.conf` et rajouter :  
>        ```bash
> 	   kernel.yama.ptrace_scope = 1
>        ```  
>        Adapter la valeur à 2 ou 3 si la VM est très sensible et qu’aucun débogage n’est nécessaire.  
>      - Appliquer la configuration :  
>        ```bash
>        sudo sysctl --system
>        ```  
>      - Vérifier :  
>        ```bash
>        sysctl kernel.yama.ptrace_scope
>        ```
> 
> - *Comparaison avec Lynis* : Lynis vérifie la valeur de `kernel.yama.ptrace_scope` dans sa section *Kernel Hardening*. Si la valeur est `0` (ou absente), il recommande de la durcir (au moins `1`). En revanche, il ne vérifie pas que le LSM Yama est explicitement sélectionné dans la configuration de grub, ce point reste à contrôler manuellement.
> 
> - *Référence* : [ANSSI_LINUX, R11]

