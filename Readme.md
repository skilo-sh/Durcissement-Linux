Cette page décrit les 
# Niveau de sécurité
## Niveau de sécurité ANSSI
La table 1 ci-dessous décrit la signification des différents niveaux de sécurité implémentés par l'ANSSI dans son guide [ANSSI_LINUX] :
![[Pasted image 20251211151219.png]]
Notre wiki traite les niveaux, à partir de minimal (i.e. M) jusqu'à renforcé (i.e. MIR).
## Niveau de sécurité "universelle"
Notre wiki implémente une notion de niveau de sécurité "universelle", c'est à dire que nous associons les niveaux de sécurités des guides usuelles vers un niveau ternaire indexé.

Notre mesure se base sur le guide de l'ANSSI :

| Niveau de sécurité | 1       | 2             | 3        |
| ------------------ | ------- | ------------- | -------- |
| Guide ANSSI        | minimal | intermédiaire | renforcé |
Ensuite, lorsque nous utiliserons comme référence un autre guide ne mettant pas en avant un niveau de sécurité ternaire, nous nous permettrons d'associer la recommandation au niveau nous semblant pertinent.
# Cartouches

Ce wiki s'inspire du système de cartouche des guides de l'ANSSI. Chaque recommandation prendra la forme d'une cartouche, ces dernières peuvent être de trois couleurs différentes :

> [!error] Rouge
> Les cartouches de couleurs rouges correspondent au niveau de sécurité 1

> [!warning] Orange
> Les cartouches de couleurs oranges correspondent au niveau de sécurité 2

> [!info] Bleu
> Les cartouches de couleurs bleus correspondent au niveau de sécurité 3

En plus d'indiquer un niveau de sécurité "universelle", les cartouches traiterons systématiquement les points suivants :
- Objectifs / Intérêts
- Procédure détaillée
- Commentaires
- Comparaison avec Lynis
