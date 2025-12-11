
> [!loggin] Activer l'IOMMU
> 
> - _Objectifs et intérêts_ : L'objectif lié à l’[activation de l’IOMMU](https://www.sstic.org/2010/presentation/Analyse_de_l_efficacite_du_service_fourni_par_une_IOMMU/) repose sur le cloisonnement strict des accès mémoire réalisés par les périphériques. Dans un système sans IOMMU, tout composant capable de faire du DMA peut théoriquement lire ou écrire n’importe où dans la RAM, ce qui crée un risque majeur en matière d’intégrité et de confidentialité des données. L’IOMMU introduit un mécanisme de traduction et de contrôle des adresses DMA, permettant d’isoler chaque périphérique dans un domaine mémoire dédié. Ce principe limite fortement l’impact d’un périphérique compromis ou défaillant, prévient les attaques par accès direct mémoire et renforce la sécurité globale.
>     
> - _Commentaires_ : Dans le cas de l'utilisation d'une VM basique, cette règle est peu pertinente. Les périphériques sont émulés, ce qui rend ce type d'attaque peu réaliste. Cependant, si la VM est directement liée à un port PCI via passthrough, l'activation de l'IOMMU devient nécessaire.
>     
> - _Procédure détaillé_ :
>     
>     1. **Identifier la plateforme physique** :
>         
>         - Vérifier si l’hyperviseur tourne sur un CPU Intel ou AMD :
>             
>             ```bash
>             lscpu
>             ```
>             
>     2. **Modifier la configuration GRUB** :
>         
>         - Éditer le fichier `/etc/default/grub`
>             
>         - Ajouter le paramètre selon la plateforme :
>             
>             - Intel : `iommu=force intel_iommu=on`
>                 
>             - AMD : `iommu=force amd_iommu=on`
>                 
>     3. **Mettre à jour GRUB** :
>         
>         ```bash
>         sudo update-grub
>         ```
>         
>     4. **Redémarrer la VM** :
>         
>         ```bash
>         sudo reboot
>         ```
>         
>     5. **Vérifier l’activation de l’IOMMU** :
>         
>         ```bash
>         dmesg | grep -e IOMMU -e DMAR
>         ```
>         
>           
> - _Comparaison avec Lynis_ : Lynis ne vérifie pas l'activation de l'IOMMU, il faut donc vérifier son activation manuellement.
>     
> - Référence : [ANSSI_LINUX, R7] 


> [!warning] Paramétrer les options de configuration de la mémoire
> 
> - _Objectifs et intérêts_ : L’ajout de ces options au noyau vise à renforcer l’isolation mémoire, limiter les fuites d’informations, compliquer l’exploitation de vulnérabilités (heap/rowhammer, etc.) et activer/forcer diverses contre‑mesures matérielles/CPU (Spectre, Meltdown, MDS…). 
> 
> - _Commentaires_ :  
>   - **Contexte VM vs hyperviseur** :
>     - Les options liées aux vulnérabilités CPU/micro‑archi pour renforcer la sécurité contre les attaques telles que meltdown, spectre et autres sont extrêmement importantes sur l'hôte. En effet, ces attaques permettent de casser l'isolation entre deux VM si elles partagent des ressources physiques communes. Cependant, il est à noter qu'elles sont moins importantes dans une VM. En effet, ces attaques permettent à différents processus de s'attaquer entre eux dans la VM, mais cela ne casse pas la protection d'isolation de la VM.
>
>   - **Pertinence par option (dans une VM)** :
>     - `l1tf=full,force` / `mds=full,nosmt` / `spectre_v2=on` / `spec_store_bypass_disable=seccomp` / `pti=on` : surtout cruciaux sur l’hyperviseur, mais pertinents dans la VM si on veut se protéger d’attaques d’escalade intra‑système via bugs noyau + side‑channels.  
>     - `page_poison=on`, `slab_nomerge=yes`, `slub_debug=FZP`, `page_alloc.shuffle=1` : durcissement mémoire “générique”, utile même en VM (complique les exploitations de vulnérabilités).  
>     - `mce=0` : pertinent partout
>     - `rng_core.default_quality=500` : améliore l’initialisation du générateur aléatoire si TPM/HWRNG est disponible sur la VM, cela est donc pertinent uniquement si la VM possède un TPM.
> 
> - _Procédure détaillée_ :
> 
>     1. **Identifier les options à appliquer**  
>        Pour un durcissement fort sur VM Debian, l'ensemble minimal recommandé par l'ANSSI est le suivant :
>        ```text
>        l1tf=full,force pti=on page_poison=on slab_nomerge=yes \
>        slub_debug=FZP spec_store_bypass_disable=seccomp spectre_v2=on \
>        mds=full,nosmt mce=0 page_alloc.shuffle=1 rng_core.default_quality=500
>        ```
> 
>     2. **Modifier la configuration GRUB**  
>        - Éditer `/etc/default/grub` :
>          ```bash
>          sudo nano /etc/default/grub
>          ```
>        - Repérer la ligne :
>          ```bash
>          GRUB_CMDLINE_LINUX_DEFAULT="quiet"
>          ```
>        - Ajouter les paramètres noyau souhaités à cette ligne, par exemple :
>          ```bash
>          GRUB_CMDLINE_LINUX_DEFAULT="quiet l1tf=full,force pti=on page_poison=on slab_nomerge=yes slub_debug=FZP spec_store_bypass_disable=seccomp spectre_v2=on mds=full,nosmt mce=0 page_alloc.shuffle=1 rng_core.default_quality=500"
>          ```
>        - Sauvegarder et quitter.
> 
>     3. **Mettre à jour GRUB**  
>        ```bash
>        sudo update-grub
>        ```
> 
>     4. **Redémarrer la VM**  
>        ```bash
>        sudo reboot
>        ```
> 
>     5. **Vérifier la prise en compte des options**  
>        Après redémarrage :
>        ```bash
>        cat /proc/cmdline
>        ```
>        Vérifier que toutes les options ajoutées apparaissent bien dans la ligne de commande du noyau.
> 
> - _Comparaison avec Lynis_ : Ces points ne sont pas abordés dans les tests de Lynis, il faut donc les vérifier à la main. Lynis ne vérifie que la présence de mot de passe pour grub mais pas ses paramètres.
> 
> - Référence : [ANSSI LINUX, R8]




