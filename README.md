# pipeline-demo

![CI](https://github.com/juniorabakar/pipeline-demo/actions/workflows/ci.yml/badge.svg)

Pipeline d'intégration et de déploiement continus construit avec **GitHub Actions**, sur une application Python minimale.

> **Projet d'apprentissage.** L'application est volontairement triviale : l'objet de ce dépôt n'est pas le code, mais la **chaîne CI/CD** qui l'entoure, et surtout ce qu'on apprend en la cassant.

---

## Objectif

Comprendre en pratique les mécanismes d'une chaîne CI/CD :

- l'enchaînement conditionnel des jobs,
- le transport d'un livrable d'une étape à l'autre,
- l'injection de secrets à l'exécution,
- et la **lecture des logs quand ça échoue** : la compétence réellement utile au quotidien.

---

## Architecture

```mermaid
flowchart LR
    P([push]) --> B[build]
    B -- needs --> T[test]
    T -- needs --> D[deploy]
    B -. artifact .-> D
```

Trois jobs chaînés par `needs`. Le trait plein est la dépendance d'exécution, le pointillé le transport de l'artifact.

| Job | Rôle | Points notables |
|---|---|---|
| **`build`** | Installe Python et les dépendances, prépare le livrable dans `dist/` et le publie comme artifact | Produit un livrable unique, réutilisé plus loin |
| **`test`** | Rejoue l'installation et lance la suite `pytest` | Bloque toute la suite en cas d'échec |
| **`deploy`** | Récupère l'artifact, vérifie la présence du secret, simule la mise en service | **Pas de `checkout`** : il n'a pas besoin des sources, seulement du livrable |

### Détails d'implémentation

**Le chaînage.** Sans `needs`, les trois jobs partiraient en parallèle. C'est `needs` qui garantit qu'un test en échec empêche le déploiement : la barrière est là, pas ailleurs. *(Équivalent des `stages` sous GitLab CI.)*

**L'artifact.** `build` construit une fois et publie ; `deploy` télécharge. On ne reconstruit jamais le livrable entre deux étapes : c'est ce qui garantit que ce qui est testé est bien ce qui est déployé. GitHub vérifie d'ailleurs l'empreinte SHA256 au téléchargement, et refuse un artifact altéré.

**Le secret.** `API_TOKEN` est injecté depuis les *repository secrets*, jamais versionné. Une fois enregistré, il n'est plus lisible : seulement remplaçable.

**La vérification explicite.** Le job `deploy` teste la présence du jeton avant de continuer :

```bash
if [ -z "$API_TOKEN" ]; then
  echo "ERREUR : secret absent"
  exit 1
fi
```

Sans ce garde-fou, un secret manquant ne produirait **aucune erreur** : la variable serait simplement vide et le déploiement passerait au vert. Voir la panne D ci-dessous.

---

# Journal des pannes

Le cœur du projet. Deux séries : celles rencontrées en construisant la chaîne, et celles provoquées volontairement pour observer chaque mode de défaillance.

## Série 1 : pannes rencontrées

Les vraies, celles qui n'étaient pas prévues.

| # | Symptôme | Cause | Leçon |
|---|---|---|---|
| 1 | Le workflow ne démarre pas : *Invalid workflow file* | `-uses:` au lieu de `- uses:` : en YAML, le tiret d'un élément de liste exige un espace | Une erreur de syntaxe n'échoue pas le build, elle **empêche le pipeline d'exister** |
| 2 | *Implicit keys need to be on a single line* | `run:` suivi d'un script multi-lignes sans le block scalar `\|` | Une commande sur une ligne → `run: cmd`. Plusieurs lignes → `run: \|` puis le script indenté |
| 3 | `cp: cannot stat 'app.py': No such file or directory` dans `build`, puis `ModuleNotFoundError: No module named 'App'` dans `test` | Casse incohérente entre le nom du fichier, l'import et le workflow | **Linux distingue `App.py` de `app.py`, Windows non.** Un projet valide en local peut échouer sur le runner pour une seule majuscule |

## Série 2 : pannes provoquées

Quatre modes de défaillance reproduits délibérément, du plus visible au plus discret.

### Panne A : erreur de syntaxe YAML

**Modification :** suppression de l'espace après le tiret d'un élément de liste.

![Erreur de syntaxe signalée dans le fichier de workflow](https://github.com/user-attachments/assets/106d7142-0480-49cc-9b44-09ad1a4dda81)

**Résultat :** aucun job n'est lancé. Le workflow n'est pas seulement en échec, il n'existe pas.

![Aucun job exécuté, le workflow est invalide](https://github.com/user-attachments/assets/24834ecf-96b1-4c31-9841-901bb1a6a67b)

> **La leçon.** Une erreur de syntaxe est détectée **avant toute exécution**. Il n'y a ni log de job, ni étape en rouge, parce qu'il n'y a jamais eu de pipeline. C'est la panne la plus facile à corriger et la plus déroutante quand on cherche des logs qui n'existent pas.

---

### Panne B : version d'outil inexistante

**Modification :** `python-version: '3.98'` dans le job `build`.

![Échec de l'étape d'installation de Python](https://github.com/user-attachments/assets/93e59c36-1029-4b29-a922-035fc891bcc8)

**Localisation précise de l'échec :**

| | |
|---|---|
| **Job** | `build` |
| **Étape** | Installer Python |
| **Message** | `Error: The version '3.98' with architecture 'x64' was not found for Ubuntu 24.04.` |

**Résultat :** `build` échoue, donc `test` et `deploy` ne démarrent jamais.

> **La leçon.** On ne dit pas « le pipeline a échoué ». On dit **quel job, quelle étape, quel message**. Cette précision est ce qui distingue un diagnostic d'un constat, et c'est la première chose qu'on attend de quelqu'un qui opère une chaîne.

---

### Panne C : test en échec

**Modification :** assertion volontairement fausse dans `test_app.py`.

![Échec de la suite de tests](https://github.com/user-attachments/assets/43d0c2fa-97b2-4943-ac16-315341564ae5)

**Résultat :** `build` reste vert, `test` échoue, et `deploy` est marqué *skipped*.

![Le job deploy est ignoré suite à l'échec du test](https://github.com/user-attachments/assets/1a41527c-64dc-4327-bec7-6ee99c0e6cfd)

> **La leçon.** C'est tout l'intérêt de la CI. Le code cassé n'atteint jamais le déploiement. La chaîne de dépendances est une **barrière**, pas une décoration. S'il ne fallait retenir qu'une seule chose de ce projet, ce serait celle-ci.

---

### Panne D : secret manquant

**Modification :** référence à un secret inexistant dans le job `deploy`.

![Échec du job deploy sur le contrôle de présence du secret](https://github.com/user-attachments/assets/01368f82-a37d-4918-8fdc-c88a16fa7ef9)

**Résultat :** aucune erreur de syntaxe, aucun avertissement de GitHub. La variable est simplement **vide**. Seule la vérification explicite écrite dans le job attrape le problème.

> **La leçon.** Un secret manquant ne crie pas. **Les pannes silencieuses sont les plus dangereuses** : sans ce contrôle, le job serait passé au vert en ayant « déployé » sans jeton. D'où le principe : on vérifie ses prérequis au lieu de supposer qu'ils sont là.

---

## Synthèse : quatre familles de défaillance

Les pannes ci-dessus se classent par **le moment où on les détecte**, et c'est aussi l'ordre de dangerosité croissante.

| Famille | Quand elle se manifeste | Exemple ici |
|---|---|---|
| **Syntaxe** | Avant tout démarrage. Le pipeline n'existe pas | Pannes 1, 2, A |
| **Résolution** | Au moment où une ressource demandée est introuvable | Pannes 3, B |
| **Exécution** | Le code tourne et produit un mauvais résultat | Panne C |
| **Silencieuse** | Jamais, sauf si on a prévu de la détecter | Panne D |

Plus une défaillance se manifeste tard, plus elle coûte cher. Une erreur de syntaxe se corrige en trente secondes ; un secret vide peut atteindre la production sans que rien ne s'allume en rouge.

---

## Correspondance avec les autres outils

Les concepts sont transférables ; seule la syntaxe change.

| Concept | GitHub Actions | GitLab CI | Jenkins |
|---|---|---|---|
| Fichier de définition | `.github/workflows/*.yml` | `.gitlab-ci.yml` | `Jenkinsfile` |
| Phase | *(via `needs`)* | `stage` | `stage` |
| Tâche | `job` | `job` | `step` |
| Machine d'exécution | runner | runner | agent / node |
| Livrable transmis | `upload-artifact` | `artifacts:` | `archiveArtifacts` |
| Secret | `secrets.X` | variable masquée | credentials |

---

## Structure

```
.
├── .github/workflows/ci.yml   # définition du pipeline
├── app.py                     # application
├── test_app.py                # tests unitaires
├── requirements.txt           # dépendances
└── README.md
```

**Convention de nommage :** tout en minuscules. Voir la panne 3.

## Reproduire en local

```bash
git clone https://github.com/juniorabakar/pipeline-demo.git
cd pipeline-demo
pip install -r requirements.txt
pytest -v
```

Pour que le job `deploy` aboutisse, créer un secret `API_TOKEN` dans
**Settings → Secrets and variables → Actions**.

---

*Stack : GitHub Actions · Python 3.11 · pytest*
