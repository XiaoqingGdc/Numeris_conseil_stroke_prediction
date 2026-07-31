Problématique métier :

À partir des données cliniques et démographiques des patients, comment identifier et prédire de manière précoce les nouveaux patients présentant un risque élevé d'accident vasculaire cérébral (AVC / stroke), afin d'optimiser le tri, la prévention et la prise en charge médicale ?

---

Question d'analyse 1 : La cible est-elle équilibrée ? Qu'est-ce que ça implique ?



![Distribution des classes](images/distribution_classes.png)

Sur ce dataset, 4 861 patients sont classés 0 (Pas d'AVC/ non malade), tandis que seulement 249 patients sont classés 1 (AVC / malade). La variable cible $y$ est donc fortement déséquilibrée, ce qui implique qu'il ne faut pas utiliser uniquement l'exactitude (Accuracy) comme métrique d'évaluation.

---

Question d'analyse 2 : Pourquoi le split doit-il précéder tout preprocessing ?

Le train_test_split doit impérativement précéder tout preprocessing (comme l'imputation des valeurs manquantes ou la standardisation) afin d'éviter le data leakage (la fuite de données).

Si l'on applique les transformations sur l'ensemble du dataset avant la division, des informations issues de l'ensemble de test « fuitent » dans l'ensemble d'entraînement, ce qui fausse l'évaluation et conduit à surestimer artificiellement les performances réelles du modèle.

---

Question d'analyse 3 : Pourquoi ces modèles et cette métrique plutôt que l'accuracy brute ?

J'ai choisi 3 modèles : Régression Logistique, Random Forest et Decision Tree, et testé différents hyperparamètres afin de trouver le meilleur modèle sur ce dataset et mieux répondre au problème métier.

- **Régression Logistique** : modèle linéaire simple, rapide et interprétable, qui sert de baseline (référence).
- **Arbre de Décision (Decision Tree)** : modèle non-linéaire capable de capturer des interactions plus complexes entre les variables cliniques.
- **Forêt Aléatoire (Random Forest)** : modèle d'ensemble robuste qui réduit le surapprentissage (overfitting) par rapport à un arbre unique, tout en testant différentes profondeurs et différents nombres d'arbres.

Je n'utilise pas l'accuracy brute (taux de bonne classification global) car ce dataset, dans le domaine de la santé, est extrêmement déséquilibré (les cas malades sont très minoritaires, représentant moins de 5% des données). Un modèle naïf qui prédirait systématiquement « sain » obtiendrait une accuracy de 95%, mais avec un Recall (rappel) de 0%, ce qui est inacceptable sur le plan médical.

C'est pourquoi je privilégie le Recall (pour minimiser le risque critique de faux négatifs / non-diagnostics).

---

Question d'analyse 4 : Val et test sont-ils cohérents ? Sinon, que fais-tu ?

Oui, les résultats sur le jeu de validation et le jeu de test sont cohérents. Le Recall passe de 0.84 (validation) à 0.80 (test), soit un écart de seulement 4 points, et le ROC-AUC reste quasiment stable (0.845 contre 0.841). Les matrices de confusion sont également très proches (8 faux négatifs en validation contre 10 en test, sur 50 cas positifs).

![Comparaison Validation vs Test](images/matrice_confusion_val_test.png)

Cette cohérence indique que les hyperparamètres sélectionnés via le GridSearchCV ne sont pas sur-ajustés au jeu de validation, et que le modèle généralise correctement à des données non vues.

Si un écart important avait été observé entre validation et test, la bonne pratique n'aurait pas été de revenir modifier les hyperparamètres à partir du score de test — cela aurait constitué une fuite de données et invalidé le caractère indépendant de l'évaluation finale. Il aurait fallu à la place retourner au stade de l'entraînement/validation pour investiguer la cause (par exemple un nombre trop faible de cas positifs rendant l'estimation instable), et documenter honnêtement cette instabilité comme une limite du modèle plutôt que de chercher à artificiellement améliorer le score de test.

---

Question d'analyse 5 : Quelle limite communiquer avant mise en production ?

Deux limites majeures doivent être communiquées à Numeris Conseil avant toute mise en production :

1. **Une précision (Precision) très faible sur la classe AVC (environ 13-14%)** : sur les patients identifiés par le modèle comme « à risque », plus de 85% ne feront en réalité pas d'AVC. Le modèle ne peut donc pas être utilisé comme outil de diagnostic autonome, mais uniquement comme outil de pré-tri, nécessitant systématiquement une validation clinique par un professionnel de santé.

2. **Un nombre de cas positifs très limité dans les données** (249 patients AVC sur environ 5 100, soit moins de 5%). Cette faible taille d'échantillon rend les estimations de performance (notamment le Recall) sensibles à la variabilité d'échantillonnage — un nouveau tirage aléatoire des données pourrait donner des résultats sensiblement différents. Il serait recommandé de réévaluer régulièrement le modèle sur de nouvelles données réelles avant toute généralisation de son usage.

---

Recommandation :

Ce modèle peut être déployé comme outil d'aide à la décision en amont, pour prioriser les patients à examiner en priorité, mais ne doit en aucun cas remplacer le jugement clinique. Avant un déploiement à plus grande échelle, il est recommandé de : (1) tester le modèle sur un échantillon plus large de patients réels pour confirmer sa stabilité, (2) mettre en place un processus de ré-entraînement périodique à mesure que de nouvelles données sont collectées, (3) associer systématiquement une lecture humaine aux alertes générées par le modèle.


