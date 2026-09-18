# Migration vers un poste local — notes de session

Ce document résume le travail mené sur ce repo depuis une instance AWS (Windows Server) en vue de
la migration vers un poste local et de la résiliation de cette instance.

## Résumé de la session

**Contexte** : produire des captures d'écran illustrant le fonctionnement du pipeline de détection
de logos (image et vidéo annotées, métriques d'entraînement, interface Label Studio), pour le
dossier CIR/CII 2025 (section "détection de marques par reconnaissance de logo").

**Ce qui a été fait** :

1. **Exploration du repo** : identification du modèle déjà entraîné
   (`models/team_chambe_3L_fine_tune_v2/weights/best.pt`, classes `escoffier` / `mizuno` /
   `teamchambe`, mAP@0.5 ≈ 0.915, precision ≈ 0.85, recall ≈ 0.91) et des données déjà présentes
   (`media_detection_60/`, `images utiles/`).

2. **Mise en place de l'environnement** : aucun venv n'existait sur l'instance. Création d'un venv
   et installation des dépendances de `requirements.txt`.
   - **Blocage rencontré** : `torch` en version courante (`2.13.0+cpu`) plantait au chargement
     (`OSError: DLL initialization routine failed` sur `c10.dll`) sur cette instance AWS.
     Résolu en épinglant `torch==2.4.1` + `torchvision==0.19.1` (roues CPU,
     `--index-url https://download.pytorch.org/whl/cpu`). Ce pin a été reporté dans
     `requirements.txt`.
   - **Dépendance manquante** : `image_detection.py` et `video_detection_with_tracker_and_db_insert.py`
     importent un module `dc_utils` (utilitaire interne pour l'insertion MySQL) absent de ce repo
     et jamais installé dans l'environnement. Ces deux scripts ne sont donc pas exécutables tels
     quels sans ce module. Pour produire les captures, la logique de détection a été rejouée dans
     des scripts temporaires (hors repo), en laissant de côté l'étape d'insertion en base.

3. **Production des captures** (déposées à l'origine dans `C:\tmp\CIR_2025_logo_detection_captures\`
   sur l'instance AWS — **à récupérer avant résiliation, voir plus bas**) :
   - Détection sur images déjà téléchargées de l'étude 60 (dont des exemples avec l'ancien jeu de
     classes `lidl` / `caisse epargne`, issus d'un entraînement antérieur du même modèle).
   - Détection + tracking (SORT) sur vidéo : les liens AWS S3 de `videos_60/aws_links.txt` étaient
     tous morts (403, bucket devenu inaccessible) — bascule sur les liens Twitter de secours
     (`videos_60/twitter_links.txt`, toujours valides), qui ont permis d'obtenir une vidéo annotée
     et une frame avec double détection nette (deux logos, confiances 0.74 et 0.39, boîtes bien
     ajustées sur des petits badges).
   - Courbes et matrice de confusion issues de l'entraînement déjà réalisé
     (`models/team_chambe_3L_fine_tune_v2/`).
   - Captures Label Studio déjà présentes dans `images utiles/` (vue projet, export au format YOLO).

4. **Question de la base de données** (dans le cadre de la migration) : recherche de `dc_utils` /
   `env_db.py` sur l'instance — confirmation que **ce repo n'a jamais eu de connexion MySQL
   fonctionnelle** (pas de `env_db.py` propre au projet, module `dc_utils` absent). Aucun des
   scripts effectivement exécutés pendant cette session ne dépend de la base de données.
   **Conclusion : aucun dump MySQL n'est nécessaire pour migrer ce repo.** Si les scripts
   dépendant de la DB (`link_construct.py`, insertion des détections) doivent être utilisés un
   jour, cela nécessitera une configuration séparée (module `dc_utils` + `env_db.py` propre au
   projet), à traiter indépendamment de cette migration.

## À récupérer manuellement avant de couper l'instance AWS

Ces éléments ne sont pas versionnés dans git et seront perdus sinon :

| Chemin (sur l'instance AWS) | Contenu | Nécessaire ? |
|---|---|---|
| `C:\tmp\CIR_2025_logo_detection_captures\` | Captures finales pour le dossier CIR (images, vidéo annotée, métriques) | **Oui, prioritaire** |
| `media_detection_60\videos\*.mp4` | Les 2 vidéos Twitter téléchargées pour la démo | Optionnel — retéléchargeables via `videos_60/twitter_links.txt` |

Tout le reste (code, `models/`, `media_detection_60/images` + `detection_jsons` + `detections_images`
déjà existants, `images utiles/`) est déjà versionné dans git et sera récupéré par un simple clone.

## Installation sur le poste local

```powershell
git clone <url-du-repo-logo-detection>
cd logo-detection
python -m venv venv
.\venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt --extra-index-url https://download.pytorch.org/whl/cpu
```

Vérification rapide (le modèle doit se charger et afficher ses 3 classes) :

```powershell
python -c "from ultralytics import YOLO; m = YOLO('models/team_chambe_3L_fine_tune_v2/weights/best.pt'); print(m.names)"
```

Sortie attendue : `{0: 'escoffier', 1: 'mizuno', 2: 'teamchambe'}`.

Si le poste local dispose d'un GPU NVIDIA, la version CPU de torch/torchvision peut être remplacée
par la version CUDA correspondante pour de meilleures performances — la version CPU ci-dessus est
celle dont le bon fonctionnement a été vérifié.
