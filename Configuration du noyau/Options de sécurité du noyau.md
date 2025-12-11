
> [!warning] Paramétrer les options de sécurité du noyau 
> - *Objectifs et intérêts* : Le "noyau" est le cœur du système d'exploitation. Par défaut, il est configuré pour être verbeux (pour aider au débogage) et permissif. Cela aide un attaquant à comprendre l'architecture interne de la machine (adresses mémoire, versions) pour le détourner. Cette règle vise à rendre le système muet sur son fonctionnement interne et à verrouiller des fonctionnalités avancées inutiles pour un usage minimaliste.
> - *Commentaires* : N/A
> - *Procédure détaillée* : Créer un fichier `/etc/sysctl.d/99-hardening.conf`. Copier la configuration ci-dessous qui applique les verrous suivants :
>   1. **Masquage d'informations** : Cache les journaux système et l'organisation de la mémoire aux utilisateurs normaux (`dmesg`, `kptr`).
>   2. **Protection mémoire** : Mélange l'emplacement des programmes en mémoire pour qu'ils soient introuvables par un virus (`randomize_va_space`).
>   3. **Restrictions techniques** : Bloque l'usage d'outils de performance (`perf`, `bpf`) souvent détournés pour espionner le processeur ou exécuter du code malveillant.
>   4. **Stabilité** : Force l'arrêt immédiat du système si le noyau détecte une corruption grave (`panic_on_oops`).
>   
>   ```bash
>   # -- Masquage des informations techniques --
>   kernel.dmesg_restrict = 1
>   kernel.kptr_restrict = 2
>   
>   # -- Renforcement de la mémoire (ASLR) et Processus --
>   kernel.randomize_va_space = 2
>   kernel.pid_max = 65536
>   
>   # -- Blocage des outils d'analyse (souvent utilisés par les attaquants) --
>   kernel.perf_cpu_time_max_percent = 1
>   kernel.perf_event_max_sample_rate = 1
>   kernel.perf_event_paranoid = 2
>   kernel.sysrq = 0
>   
>   # -- Empêcher les utilisateurs normaux de manipuler le cœur du noyau (BPF) --
>   kernel.unprivileged_bpf_disabled = 1
>   
>   # -- Arrêt de sécurité en cas de bug critique --
>   kernel.panic_on_oops = 1
>   
>   ```
>   Appliquer ensuite les changements avec la commande : `sysctl --system`
> - *Comparaison avec Lynis* : Lynis vérifie ces points dans la section *Kernel Hardening*.
> - Référence : [ANSSI_LINUX, R9]

> [!info] Désactiver le chargement des modules noyau
> - *Objectifs et intérêts* : Par défaut, Linux permet de charger des fonctionnalités (pilotes, protocoles) à la volée. Un attaquant peut abuser de ce mécanisme pour charger un module vulnérable et élever ses privilèges, ou installer un "rootkit" (logiciel malveillant invisible) au cœur du système. Cette mesure verrouille le noyau pour interdire tout ajout de code après la phase de démarrage.
> - *Commentaires* : Une fois activée, cette option ne peut être désactivée qu'en redémarrant la VM. Cela signifie qu'il est impossible de charger un nouveau driver à chaud comme une clé USB si celui-ci n'a pas été prévu au démarrage. Cette option est donc à réserver aux VM "figées", c'est-à-dire dont l'installation est totalement terminée et qui n'ont pas vocation à recevoir de nouvelles fonctionnalités ou de nouveaux logiciels.
> - *Procédure détaillée* : La procédure se fait en deux temps : 
>    1. Lister les modules légitimes actuellement utilisés (via `lsmod`) et s'assurer qu'ils sont listés dans `/etc/modules` pour être chargés au boot.
>    2. Ajouter la directive `kernel.modules_disabled = 1` dans le fichier `/etc/sysctl.conf` ou `/etc/sysctl.d/99-hardening.conf` pour activer le verrouillage après le chargement initial.
> - *Comparaison avec Lynis* : Lynis vérifie ce point dans la section *Kernel Hardening*.
> - Référence : [ANSSI_LINUX, R10]

