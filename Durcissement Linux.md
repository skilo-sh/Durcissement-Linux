<p align="center">
  <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/35/Tux.svg/265px-Tux.svg.png" width="200">
</p>

Ce guide de durcissement Linux a été écrit dans le cadre de l'UE "Durcissement OS" encadrée par M. Breyault. Les auteurs de ce guide sont respectivement :
- RICHARD Étienne
- CIZDZIEL Matthieu
- DAVID--SIMON Malo

Ce dernier s'inscrit dans le contexte du durcissement d'une machine virtuelle équipé d'un système d'exploitation "Debian 12", qui a été configurée au préalable de la manière suivante :
- Import de l'ISO Debian 12 sur VirtualBox ;
- Activation du vTPM (TPM virtualisé) ;
- Chiffrement du disque avec LVM ;
- Configuration du SSH pour utiliser l'OS depuis l'hôte ;
- Configuration des utilisateurs avec des mots de passes forts (i.e. au moins 100 bits d'entropie).

D'autre part, la structure de nos sections reprends celle utilisée dans le guide de l'ANSSI (cf. [ANSSI_LINUX]) :
- [Configuration matérielle](Configuration%20matérielle/Configuration%20matérielle.md) ;
- [Configuration du noyau Linux](Configuration%20du%20noyau/Configuration%20du%20noyau) ;
- [Configuration système](Configuration%20Système/Configuration%20Système.md) ;
- [Configuration des services](Configuration%20des%20services/Configuration%20des%20services.md).

Avant de lire le guide, il est primordial de lire la page suivante : [Readme](Readme.md). En effet, cette dernière détaille les points suivants :
- Le niveau de sécurité traité par notre guide ;
- La signification des différents style de cartouche.

Bonne lecture :)