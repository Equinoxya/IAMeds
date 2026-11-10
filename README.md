# IAMeds

Assistant d'aide à l'interprétation de bilans sanguins. L'objectif : entraîner un
modèle sur des données d'analyses biologiques pour proposer une **orientation
diagnostique multi-pathologies** (un classement d'hypothèses), accompagnée des
marqueurs qui justifient chaque hypothèse.

> ⚠️ **Aide à l'orientation, pas diagnostic.** Un outil qui pose un diagnostic
> relève du dispositif médical (règlement UE 2017/745) et de l'IA à haut risque
> (AI Act). Le projet se positionne volontairement comme un outil
> pédagogique / d'aide à la lecture, avec une sortie toujours explicable et
> jamais présentée comme une conclusion médicale.

---

## 1. Objectif

| | |
|---|---|
| **Entrée** | Un bilan sanguin (NFS, ionogramme, bilan hépatique, rénal, lipidique, thyroïdien…) + âge et sexe |
| **Sortie** | Probabilité pour chaque pathologie du référentiel + marqueurs contributifs (SHAP) |
| **Type de problème** | Classification **multi-label** (un patient peut cumuler plusieurs anomalies) |

Pathologies visées en v1 (choisies pour être détectables sur un bilan courant) :
anémie ferriprive / inflammatoire / macrocytaire, thrombopénie, syndrome
infectieux bactérien vs viral, insuffisance rénale, cytolyse hépatique,
cholestase, diabète, dyslipidémie, hypo- et hyperthyroïdie.

## 2. Approche technique

Sur des données **tabulaires** (ce qu'est un bilan sanguin), le gradient boosting
surpasse quasi systématiquement le deep learning. Pas de réseau de neurones ici.

1. **Normalisation** — mapping des libellés de labo vers un schéma d'analytes
   commun via les codes **LOINC**, conversion des unités (pièges classiques :
   créatinine µmol/L vs mg/dL, glycémie mmol/L vs g/L).
2. **Nettoyage** — bornes physiologiques, doublons, valeurs aberrantes. Les
   valeurs manquantes sont conservées telles quelles : l'absence d'un examen est
   une information clinique (LightGBM gère nativement les `NaN`).
3. **Features** — valeurs brutes + **écart aux bornes de référence selon
   âge/sexe** + ratios cliniques (neutrophiles/lymphocytes, ASAT/ALAT (De Ritis),
   urée/créatinine, DFG estimé via CKD-EPI, TG/HDL).
4. **Modèle** — baseline par règles de seuils → **LightGBM** one-vs-rest sur les
   labels → calibration des probabilités (isotonic / Platt).
5. **Évaluation** — AUC-ROC et AUC-PR par pathologie, sensibilité à spécificité
   fixée, matrice de confusion, et **SHAP** pour expliquer chaque prédiction
   individuelle.

La baseline par règles n'est pas décorative : si le modèle ne la bat pas, il n'a
rien appris d'utile.

## 3. Données

Aucune donnée propre pour l'instant. Le pipeline est conçu pour démarrer sur du
public, puis accueillir des données réelles sans changer de code.

### Recommandation : NHANES (CDC) pour commencer

**NHANES** — <https://wwwn.cdc.gov/nchs/nhanes/>

- Domaine public, **aucune démarche d'accès**, téléchargement immédiat (fichiers `.XPT`).
- Les fichiers *Laboratory* couvrent NFS, bilan lipidique, glycémie + HbA1c,
  fonction rénale, bilan hépatique, ferritine, B12, TSH — exactement le panel visé.
- Les labels se dérivent des fichiers *Examination* et *Questionnaire*
  (diabète diagnostiqué, maladie rénale, pathologie thyroïdienne…) ou de seuils
  biologiques standards (HbA1c ≥ 6,5 % → diabète ; DFG < 60 → IRC).
- Plusieurs dizaines de milliers de participants sur l'ensemble des cycles.
- Limite : population **générale**, donc pathologies peu fréquentes et peu de cas
  sévères. Bon pour le dépistage, insuffisant pour des tableaux aigus.

### Autres sources utiles

| Dataset | Contenu | Accès | Usage ici |
|---|---|---|---|
| [MIMIC-IV](https://physionet.org/content/mimiciv/) | Réanimation (Beth Israel) : labos en série temporelle + diagnostics ICD | Accréditation PhysioNet : formation CITI + DUA signée (compter quelques semaines) | La référence pour des labels diagnostiques réels et des cas sévères |
| [MIMIC-IV Demo](https://physionet.org/content/mimic-iv-demo/) | ~100 patients, même schéma | **Libre, sans accréditation** | Idéal pour écrire et tester le pipeline avant d'obtenir l'accès complet |
| [eICU](https://physionet.org/content/eicu-crd/) | Réanimation multicentrique US | Même accréditation que MIMIC | Validation externe (généralisation inter-hôpitaux) |
| [UCI HCV data](https://archive.ics.uci.edu/dataset/571/hcv+data) | 615 patients, bilan hépatique → donneur sain / hépatite / fibrose / cirrhose | Libre | Petit mais propre : parfait pour valider le bloc hépatique |
| [UCI Thyroid Disease](https://archive.ics.uci.edu/dataset/102/thyroid+disease) | TSH, T3, T4 → hypo/hyperthyroïdie | Libre | Bloc thyroïde |
| [Anemia Disease (Kaggle)](https://www.kaggle.com/datasets/serhathoca/anemia-disease) | 15 300 NFS sur 5 ans | Libre (vérifier la licence) | Bloc anémies |

### Stratégie de données retenue

1. **Pipeline d'abord** : construction et tests sur un **générateur synthétique**
   (profils biologiques dérivés de la physiopathologie connue) + MIMIC-IV Demo.
   Permet d'avancer sans attendre et de fournir des données de test versionnables.
2. **Entraînement v1** : NHANES, disponible immédiatement.
3. **Entraînement v2** : MIMIC-IV une fois l'accréditation obtenue — demander
   l'accès **maintenant**, c'est le chemin critique du projet.
4. ⚠️ Un modèle entraîné sur du synthétique n'apprend que les règles qui ont servi
   à le générer. Le synthétique sert à valider le **code**, jamais à mesurer une
   performance clinique.

## 4. Pièges identifiés

- **Fuite de données** : séparer les splits **par patient**, pas par bilan — sinon
  deux prélèvements du même patient se retrouvent en train et en test.
- **Labels dérivés de seuils** : si un label est calculé à partir d'un analyte, ce
  dernier doit sortir des features, sinon le modèle apprend la règle, pas la maladie.
- **Déséquilibre des classes** : AUC-PR et sensibilité à spécificité fixée, pas
  l'accuracy.
- **Décalage inter-laboratoires** : bornes de référence et méthodes de dosage
  varient d'un labo à l'autre — d'où la normalisation LOINC + unités dès l'ingestion.
- **RGPD** : toute donnée réelle de patient exige une base légale, une
  pseudonymisation et une AIPD avant le moindre entraînement.

## 5. État d'avancement

- [x] Cadrage de l'objectif et du type de modèle
- [x] Identification des sources de données
- [ ] Demande d'accréditation PhysioNet (MIMIC-IV) — **à lancer en priorité**
- [ ] Schéma d'analytes (LOINC, unités, bornes de référence)
- [ ] Générateur de bilans synthétiques
- [ ] Ingestion NHANES
- [ ] Features + baseline par règles
- [ ] LightGBM multi-label + calibration
- [ ] Évaluation + explications SHAP
- [ ] Interface de démonstration

## 6. Stack prévue

Python 3.13 · pandas · scikit-learn · LightGBM · SHAP
