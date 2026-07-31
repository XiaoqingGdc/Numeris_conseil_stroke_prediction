## Problématique métier :

Numeris Conseil a transmis un jeu de données contenant les informations médicales et démographiques de 5 110 patients, avec un objectif simple : construire un outil capable de repérer, parmi de nouveaux patients, ceux qui présentent un risque élevé d'AVC, afin d'aider les équipes médicales à prioriser leur attention.

## Question d'analyse 1 : La cible est-elle équilibrée ? Qu'est-ce que ça implique ?

Sur ce dataset, 4 861 patients sont classés 0 (Pas d'AVC / non malade), tandis que seulement 249 patients sont classés 1 (AVC / malade). La variable cible $y$ est donc fortement déséquilibrée, ce qui implique qu'il ne faut pas utiliser uniquement l'exactitude (Accuracy) comme métrique d'évaluation.

![Distribution des classes](images/distribution_classes.png)


## Question d'analyse 2 : Pourquoi le split doit-il précéder tout preprocessing ?

Le train_test_split doit précéder tout preprocessing afin d'éviter le data leakage.

Si l'on applique les transformations sur l'ensemble du dataset avant la division, des informations du jeu de test « fuitent » dans l'entraînement, ce qui fausse l'évaluation et conduit à surestimer artificiellement les performances réelles du modèle.



## Question d'analyse 3 : Pourquoi ces modèles et cette métrique plutôt que l'accuracy brute ?

J'ai choisi 3 modèles afin de trouver le meilleur compromis performance/interprétabilité sur ce dataset :

- **Régression Logistique** : modèle linéaire simple et interprétable, qui sert de baseline.
- **Arbre de Décision** : modèle non-linéaire capable de capturer des interactions plus complexes entre les variables cliniques.
- **Forêt Aléatoire** : modèle d'ensemble robuste qui réduit le surapprentissage par rapport à un arbre unique.

Je n'utilise pas l'accuracy brute (taux de bonne classification global) car ce dataset, dans le domaine de la santé, est extrêmement déséquilibré (les cas malades sont très minoritaires, représentant moins de 5% des données). Un modèle naïf qui prédirait systématiquement « sain » obtiendrait une accuracy de 95%, mais avec un Recall (rappel) de 0%, ce qui est inacceptable sur le plan médical.

C'est pourquoi je privilégie le Recall (pour minimiser le risque critique de faux négatifs / non-diagnostics).



## Question d'analyse 4 : Val et test sont-ils cohérents ? Sinon, que fais-tu ?

Oui, les résultats sur le jeu de validation et le jeu de test sont cohérents. Le Recall passe de 0.84 (validation) à 0.80 (test), soit un écart de seulement 4 points, et le ROC-AUC reste quasiment stable (0.845 contre 0.841). Les matrices de confusion sont également très proches (8 faux négatifs en validation contre 10 en test, sur 50 cas positifs).

![Comparaison Validation vs Test](images/matrice_confusion_val_test.png)

Cette cohérence indique que le modèle généralise correctement à des données non vues.

Si un écart important avait été observé entre validation et test, la bonne pratique n'aurait pas été de revenir modifier les hyperparamètres à partir du score de test — cela aurait constitué une fuite de données et invalidé le caractère indépendant de l'évaluation finale. Il aurait fallu à la place retourner au stade de l'entraînement/validation pour investiguer la cause (par exemple un nombre trop faible de cas positifs rendant l'estimation instable), et documenter honnêtement cette instabilité comme une limite du modèle plutôt que de chercher à artificiellement améliorer le score de test.



## Question d'analyse 5 : Synthèse


### Le problème

Le jeu de données contient les informations médicales et démographiques d'environ 5 110 patients. Parmi eux, seuls 249 (moins de 5%) ont réellement fait un AVC. Cette rareté du cas « à risque » est une contrainte centrale qui a guidé tous nos choix par la suite.

### Le modèle retenu

J'ai testé et comparé trois modèles différents (un modèle simple et transparent, un modèle par arbre de décision, et un modèle plus complexe combinant plusieurs arbres), en essayant systématiquement 140 configurations différentes pour chacun.

C'est le modèle le plus simple — une **régression logistique** — qui a obtenu les meilleurs résultats. C'est une bonne nouvelle : ce modèle est aussi le plus facile à expliquer et à faire valider par une équipe médicale, contrairement à des modèles « boîte noire » plus complexes.

### La performance

Sur des patients que le modèle n'avait jamais vus, il a correctement identifié 8 patients sur 10 parmi ceux qui allaient réellement faire un AVC. C'est le résultat le plus important pour un outil de dépistage médical : on préfère largement « trop alerter » que « manquer un cas ».

En contrepartie, parmi tous les patients signalés comme « à risque » par le modèle, une large majorité ne fera en réalité pas d'AVC. Le modèle a été volontairement réglé pour privilégier la détection des cas réels, quitte à générer davantage de fausses alertes.

J'ai vérifié cette performance à deux reprises, sur deux groupes de patients totalement indépendants et jamais utilisés pendant l'entraînement (validation et test) : les résultats obtenus sont très proches d'un groupe à l'autre, ce qui confirme la fiabilité et la stabilité du modèle.

### Les limites

- **Beaucoup de fausses alertes** : le modèle n'est pas assez précis pour poser un diagnostic seul. Il doit être utilisé comme un outil de pré-repérage, systématiquement suivi d'un examen par un professionnel de santé.
- **Peu de cas réels dans les données** : seulement 249 patients ayant fait un AVC ont servi à entraîner et évaluer le modèle. Ce nombre limité rend les résultats un peu sensibles aux hasards de l'échantillonnage — il faudra confirmer ces performances sur davantage de patients réels avant une utilisation à grande échelle.

### Recommandation

Ce modèle peut être déployé comme outil d'aide à la décision en amont, pour prioriser les patients à examiner en priorité, mais ne doit en aucun cas remplacer le jugement clinique.

Avant un déploiement à plus grande échelle, il est recommandé de :
1. Tester le modèle sur un échantillon plus large de patients réels pour confirmer sa stabilité ;
2. Mettre en place un processus de ré-entraînement périodique à mesure que de nouvelles données sont collectées ;
3. Associer systématiquement une lecture humaine aux alertes générées par le modèle.

