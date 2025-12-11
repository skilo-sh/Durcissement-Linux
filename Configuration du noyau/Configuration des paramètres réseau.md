
> [!warning] Paramétrer les options de configuration du réseau IPv4
> - *Objectifs et intérêts* : Durcir le comportement réseau par défaut du noyau pour limiter les attaques (spoofing, redirections ICMP, source routing, ARP poisoning, SYN flood, etc.) et s’assurer que la VM se comporte comme un simple hôte terminal (non-routeur). 
> 
> - *Commentaires* :
>   - Pertinent pour une VM Debian minimale non-routeur cependant cette configuration doit être adaptée si un autre usage de type routeur est fait de la VM.
> 
> - *Procédure détaillé* :
>   - Créer un fichier de configuration sysctl `/etc/sysctl.d/99-reseau-ipv4.conf` :
>     ```bash
>     sudo nano /etc/sysctl.d/99-reseau-ipv4.conf
>     ```
>   - Y ajouter la configuration recommandée pour un hôte non-routeur :
>     ```bash
>     net.core.bpf_jit_harden = 2
>     net.ipv4.ip_forward = 0
>     net.ipv4.conf.all.accept_local = 0
>     net.ipv4.conf.all.accept_redirects = 0
>     net.ipv4.conf.default.accept_redirects = 0
>     net.ipv4.conf.all.secure_redirects = 0
>     net.ipv4.conf.default.secure_redirects = 0
>     net.ipv4.conf.all.shared_media = 0
>     net.ipv4.conf.default.shared_media = 0
>     net.ipv4.conf.all.accept_source_route = 0
>     net.ipv4.conf.default.accept_source_route = 0
>     net.ipv4.conf.all.arp_filter = 1
>     net.ipv4.conf.all.arp_ignore = 2
>     net.ipv4.conf.all.route_localnet = 0
>     net.ipv4.conf.all.drop_gratuitous_arp = 1
>     net.ipv4.conf.default.rp_filter = 1
>     net.ipv4.conf.all.rp_filter = 1
>     net.ipv4.conf.default.send_redirects = 0
>     net.ipv4.conf.all.send_redirects = 0
>     net.ipv4.icmp_ignore_bogus_error_responses = 1
>     net.ipv4.ip_local_port_range = 32768 65535
>     net.ipv4.tcp_rfc1337 = 1
>     net.ipv4.tcp_syncookies = 1
>     ```
>   - Appliquer la configuration sans redémarrage :
>     ```bash
>     sudo sysctl --system
>     ```
>   
> - *Comparaison avec Lynis* : Lynis audite une partie de ces paramètres mais certains ne sont pas controllés comme `net.ipv4.ip_forward`. Il est donc intéressant de vérifier la configuration à la main.
> 
> - Référence : [ANSSI_LINUX, R12]

> [!warning] Désactiver le plan IPv6 (si non utilisé)
> - **Objectifs et intérêts** : La majorité des réseaux internes fonctionnent encore exclusivement en IPv4. Si IPv6 est activé par défaut mais non configuré ou surveillé, il constitue une surface d'attaque discrète (contournement des règles de pare-feu IPv4, attaques *Man-in-the-Middle* sur le réseau local, ...). Désactiver la pile IPv6 garantit que tout le trafic passe par les canaux IPv4 contrôlés et filtrés.
> - **Commentaires** :
>     Avant de désactiver IPV6, il est important de vérifier que les services installés sur la VM ne sont pas configurés pour utiliser IPV6 ce qui occasionnerait des dysfonctionnements.
> 	L'ANSSI conseil dans son guide de désactiver IPV6 depuis sysctl mais également depuis la configuration grub lorsque GRUB 2 est utilisé.
> - **Procédure détaillée** :
>     1. **Configuration Sysctl** : Ajouter dans un fichier de configuration (ex: `/etc/sysctl.d/99-reseau-ipv6.conf`) :
>        ```ini
>        net.ipv6.conf.default.disable_ipv6 = 1
>        net.ipv6.conf.all.disable_ipv6 = 1
>        ```
>        Puis appliquer avec `sysctl --system`.
>     2. **Configuration Bootloader (Grub)** : Pour une désactivation complète au niveau noyau, éditer `/etc/default/grub`.
>        Trouver la ligne `GRUB_CMDLINE_LINUX` et ajouter `ipv6.disable=1`.
>        Mettre à jour grub avec la commande `update-grub` et redémarrer la VM.
> - **Comparaison avec Lynis** : Lynis vérifie la configuration IPv6 mais ne préconise pas nécessairement sa désactivation. Les recommandations du NIST sont différentes de celles de l’ANSSI, qui préfère limiter la surface d’attaque en désactivant IPv6 lorsqu’il n’est pas utilisé dans le système d’information. Cette règle doit être adapté au cas d'usage mais la recommandation de l'ANSSI semble plus appropiée à un système minimaliste.
> - **Référence** : [ANSSI_LINUX, R13]

