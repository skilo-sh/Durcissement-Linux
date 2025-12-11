# SSH
## Etat de l'art cryptographique
### Authentification
L'authentification repose sur l'usage de signature digitale, l'état de l'art (tant pour la taille des clés, la sécurité et l'efficacité) est : EdDSA ou à défaut ECDSA (voir : https://www.ssl.com/article/comparing-ecdsa-vs-rsa-a-simple-guide/).

> Nous préférons l'usage d'EdDSA plutôt qu'ECDSA car ce dernier ne repose pas sur des preuves de sécurité réalisées dans des modèles raisonnables. 

Il suffira donc d'utiliser des courbes avec une taille de groupe suffisante, l'ANSSI suggère l'utilisation d'une courbe engendrant un sous-groupe ayant un ordre multiple d'un nombre premier d'au moins 250 bits \[CRYPTO, RegleECp\]. A noter d'ailleurs que pour luter contre les attaques par courbes invalides (voir : https://safecurves.cr.yp.to/twist.html) un ordre premier est recommandé \[CRYPTO, RecommendationECp\].

Les courbes suivantes respectent ces critères :
- ed25519
- X448
- Nist P-256
- NIST P-384
- NIST P-521

La signature numérique n'étant pas la cible du *store now, decrypt later* la considération d'algoritme post-quantique n'est pas nécessaire - ils ne sont de toute facon pas encore supportés par OpenSSH.
### Échange de clé
Quant aux échanges de clé, l'état de l'art est représenté a nouveau par l'usage des courbes elliptique, cette fois-ci couplées à l'algorithme Diffie-Hellman (voir : https://security.stackexchange.com/questions/25122/ecdh-is-better-than-plain-rsa).

À l'aide d'un raisonnement similaire à celui énoncé dans la section `Authentification`, voici les courbes recommandées :
- ed25519
- Nist P-256
- X448
- NIST P-384
- NIST P-521

Les mécanismes d'échange de clé sont eux la cible du *store now, decrypt later*. De plus, la BSI (homologue allemand de l'ANSSI) estime conservatirement l'arrivé d'un CRQC (Cryptographically Relevant Quantum Computer) (voir : https://www.bsi.bund.de/EN/Themen/Unternehmen-und-Organisationen/Informationen-und-Empfehlungen/Quantentechnologien-und-Post-Quanten-Kryptografie/Entwicklungsstand-Quantencomputer/entwicklungsstand-quantencomputer_node.html) d'ici 16 ans.

Il est donc bienvenu d'intégrer par défaut une suite mettant en oeuvre une hybridation entre ECDH et un KEM post-quantique, comme recommandé par l'ANSSI (voir : https://cyber.gouv.fr/publications/anssi-views-post-quantum-cryptography-transition). Nous hybridons dans le but d'éviter la régression, en effet, les KEM post-quantique ne sont pas encore éprouvés.
### Chiffrement des données
Les algorithmes sous-jacent respectent [CRYPTO, 2.1 Cryptographie symétrique] par défaut ; rien à signaler. Il est cependant recommandé de désactiver les mécanismes utilisant la construction *mac-then-encrypt* (i.e. n'activer que les suites *-etm* ou basé sur un AEAD, comme ChaCha20Poly1305)
## Audit de la configuration SSH par défaut
### Lynis
Lynis suggère des améliorations non cryptographiques.
```console
[+] Prise en charge SSH
------------------------------------
- Checking running SSH daemon                             [ TROUVÉ ]
- Searching SSH configuration                             [ TROUVÉ ]
- OpenSSH option: AllowTcpForwarding                      [ SUGGESTION ]
- OpenSSH option: ClientAliveCountMax                     [ SUGGESTION ]
- OpenSSH option: ClientAliveInterval                     [ OK ]
- OpenSSH option: FingerprintHash                         [ OK ]
- OpenSSH option: GatewayPorts                            [ OK ]
- OpenSSH option: IgnoreRhosts                            [ OK ]
- OpenSSH option: LoginGraceTime                          [ OK ]
- OpenSSH option: LogLevel                                [ SUGGESTION ]
- OpenSSH option: MaxAuthTries                            [ SUGGESTION ]
- OpenSSH option: MaxSessions                             [ SUGGESTION ]
- OpenSSH option: PermitRootLogin                         [ SUGGESTION ]
- OpenSSH option: PermitUserEnvironment                   [ OK ]
- OpenSSH option: PermitTunnel                            [ OK ]
- OpenSSH option: Port                                    [ SUGGESTION ]
- OpenSSH option: PrintLastLog                            [ OK ]
- OpenSSH option: StrictModes                             [ OK ]
- OpenSSH option: TCPKeepAlive                            [ SUGGESTION ]
- OpenSSH option: UseDNS                                  [ OK ]
- OpenSSH option: X11Forwarding                           [ SUGGESTION ]
- OpenSSH option: AllowAgentForwarding                    [ SUGGESTION ]
- OpenSSH option: AllowUsers                              [ NON TROUVÉ ]
- OpenSSH option: AllowGroups                             [ NON TROUVÉ ]
```
### ssh-audit
Cet outil est complémentaire à Lynis, en effet, il émet des conseils cryptographiques.

Voici ce que retourne l'outil `ssh-audit` (voir : https://github.com/jtesta/ssh-audit) pour une configuration par défaut d'OpenSSH sur Debian :
```console
# general
(gen) banner: SSH-2.0-OpenSSH_10.0p2 Debian-7
(gen) software: OpenSSH 10.0p2
(gen) compatibility: OpenSSH 9.9+, Dropbear SSH 2020.79+
(gen) compression: enabled (zlib@openssh.com)

# key exchange algorithms
(kex) mlkem768x25519-sha256               -- [info] available since OpenSSH 9.9
                                          `- [info] default key exchange since OpenSSH 10.0
                                          `- [info] hybrid key exchange based on post-quantum resistant algorithm and proven conventional X25519 algorithm
(kex) sntrup761x25519-sha512              -- [info] available since OpenSSH 9.9
                                          `- [info] default key exchange in OpenSSH 9.9
                                          `- [info] hybrid key exchange based on post-quantum resistant algorithm and proven conventional X25519 algorithm
(kex) sntrup761x25519-sha512@openssh.com  -- [info] available since OpenSSH 8.5
                                          `- [info] default key exchange from OpenSSH 9.0 to 9.8
                                          `- [info] hybrid key exchange based on post-quantum resistant algorithm and proven conventional X25519 algorithm
(kex) curve25519-sha256                   -- [warn] does not provide protection against post-quantum attacks
                                          `- [info] available since OpenSSH 7.4, Dropbear SSH 2018.76
                                          `- [info] default key exchange from OpenSSH 7.4 to 8.9
(kex) curve25519-sha256@libssh.org        -- [warn] does not provide protection against post-quantum attacks
                                          `- [info] available since OpenSSH 6.4, Dropbear SSH 2013.62
                                          `- [info] default key exchange from OpenSSH 6.5 to 7.3
(kex) ecdh-sha2-nistp256                  -- [fail] using elliptic curves that are suspected as being backdoored by the U.S. National Security Agency
                                          `- [warn] does not provide protection against post-quantum attacks
                                          `- [info] available since OpenSSH 5.7, Dropbear SSH 2013.62
(kex) ecdh-sha2-nistp384                  -- [fail] using elliptic curves that are suspected as being backdoored by the U.S. National Security Agency
                                          `- [warn] does not provide protection against post-quantum attacks
                
```
## Configuration durcie
Voici les options à ajouter à la fin du fichier afin de durcir au point de mettre en pratique les recommandations décrite ci-dessus :
```bash
echo "\
# Debut durcissement
# Directives lynis
AllowTcpForwarding no\
ClientAliveCountMax 2\
ClientAliveInterval 300\
FingerprintHash SHA256\
GatewayPorts no\
IgnoreRhosts yes\
LoginGraceTime 120\
LogLevel DEBUG\
MaxAuthTries 3\
MaxSessions 2\
PermitRootLogin PROHIBIT-PASSWORD\
PermitUserEnvironment no\
PermitTunnel no\
Port 1000\
PrintLastLog yes\
StrictModes yes\
TCPKeepAlive no\
UseDNS no\
X11Forwarding no\
AllowAgentForwarding no\

# Directives cryptographiques
KexAlgorithms sntrup761x25519-sha512,sntrup761x25519-sha512@openssh.com,curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
HostKeyAlgorithms sk-ssh-ed25519-cert-v01@openssh.com,ssh-ed25519-cert-v01@openssh.com,sk-ssh-ed25519@openssh.com,ssh-ed25519
RequiredRSASize 3072
CASignatureAlgorithms sk-ssh-ed25519@openssh.com,ssh-ed25519
GSSAPIKexAlgorithms gss-curve25519-sha256-
HostbasedAcceptedAlgorithms sk-ssh-ed25519-cert-v01@openssh.com,ssh-ed25519-cert-v01@openssh.com,sk-ssh-ed25519@openssh.com,ssh-ed25519
PubkeyAcceptedAlgorithms sk-ssh-ed25519-cert-v01@openssh.com,ssh-ed25519-cert-v01@openssh.com,sk-ssh-ed25519@openssh.com,ssh-ed25519
" >> /etc/ssh/
```
> Warning : SSH tournera dorénavant sur le port `1000`.
# Bibliographie
- [CRYPTO] : https://cyber.gouv.fr/sites/default/files/2021/03/anssi-guide-mecanismes_crypto-2.04.pdf 