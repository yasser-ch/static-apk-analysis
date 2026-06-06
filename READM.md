# 🔬 Lab 4 : Analyse Statique d'un APK avec JADX GUI + dex2jar + JD-GUI

## Objectif

Découvrir les techniques d'**analyse statique** d'applications Android (APK) : extraire le contenu d'un APK, analyser son manifeste, explorer son code source décompilé et identifier des vulnérabilités potentielles **sans exécuter l'application**.

---

## Objectifs Pédagogiques

- Comprendre la structure interne d'un APK (code, ressources, manifeste)
- Analyser `AndroidManifest.xml` pour identifier permissions et composants exposés
- Explorer le code source décompilé avec **JADX GUI**
- Convertir des fichiers DEX en JAR avec **dex2jar** et analyser avec **JD-GUI**
- Identifier des vulnérabilités courantes (secrets en clair, logs sensibles, debug)
- Produire un mini-rapport d'audit professionnel

---

## ⚠️ Règles de Sécurité (Obligatoires)

- Analyser **uniquement** des APK autorisés (fournis par l'enseignant ou générés soi-même)
- Ne pas utiliser d'applications tierces du Play Store sans autorisation
- Ne pas tenter d'exploiter les vulnérabilités découvertes
- Travailler uniquement avec des données fictives
- Lab strictement pédagogique

---

## Prérequis

| Outil | Source |
|-------|--------|
| JADX GUI | https://github.com/skylot/jadx/releases |
| dex2jar | https://github.com/pxb1988/dex2jar/releases |
| JD-GUI | https://github.com/java-decompiler/jd-gui/releases |
| APK autorisé | Projet de cours ou OWASP UnCrackable L1 |

---

## Glossaire

| Terme | Définition |
|-------|-----------|
| APK | Android Package — archive contenant code, ressources, manifeste |
| DEX | Dalvik Executable — bytecode de la VM Android |
| Manifest | Fichier XML déclarant composants, permissions et configurations |
| Décompilation | Conversion du bytecode en code source lisible (approximatif) |
| Obfuscation | Technique rendant le code difficile à comprendre |
| Exported | Composant accessible depuis d'autres applications |

---

## Architecture du Processus d'Analyse

```
APK (archive ZIP)
      │
      ├── AndroidManifest.xml  → JADX GUI (analyse directe)
      ├── resources/           → JADX GUI
      ├── classes.dex          → dex2jar → app.jar → JD-GUI
      └── classes2.dex         → dex2jar → app2.jar → JD-GUI
```

---

## Tâche 1 — Préparer le Workspace

```bash
# Windows
mkdir C:\APK-Analysis
cd C:\APK-Analysis

# Linux/macOS
mkdir ~/APK-Analysis
cd ~/APK-Analysis
```

**Vérifier que l'APK est une archive ZIP valide :**
```bash
# Linux/macOS
hexdump -n 4 app-debug.apk
# Doit commencer par "PK" (50 4B)
```

**Calculer le hash SHA-256 pour traçabilité :**
```bash
# Windows
Get-FileHash -Algorithm SHA256 .\app-debug.apk

# Linux/macOS
sha256sum app-debug.apk
```

**Checkpoints :**
- [ ] Dossier de travail créé
- [ ] APK vérifié comme archive valide
- [ ] Hash SHA-256 noté
- [ ] Structure de base identifiée

---

## Tâche 2 — Obtenir l'APK

**Option A — APK fourni par l'enseignant**
- Documenter la provenance dans le rapport

**Option B — Générer depuis Android Studio**
```
Build > Build Bundle(s) / APK(s) > Build APK(s)
Sortie : app/build/outputs/apk/debug/app-debug.apk
```

**Option C — OWASP UnCrackable L1 (recommandé pour ce lab)**
```
https://mas.owasp.org/crackmes/Android/
```

---

## Tâche 3 — Analyse avec JADX GUI

### Éléments à analyser dans AndroidManifest.xml

| Élément | Ce qu'on cherche |
|---------|-----------------|
| `package`, `versionName`, `minSdk` | Identification de l'app |
| `uses-permission` | Toutes les permissions demandées |
| `android:exported="true"` | Composants accessibles de l'extérieur |
| `android:debuggable="true"` | Mode debug activé en production |
| `android:usesCleartextTraffic="true"` | Trafic HTTP non chiffré autorisé |
| `android:allowBackup="true"` | Sauvegarde des données autorisée |

### Ressources importantes à explorer
- `strings.xml` → chaînes de caractères en clair
- `network_security_config.xml` → politique réseau
- Fichiers de configuration XML

> ⚠️ `android:exported="true"` élargit la surface d'attaque — ces composants peuvent être démarrés par des applications malveillantes

**Checkpoints :**
- [ ] Package principal et version identifiés
- [ ] Liste des permissions établie
- [ ] Composants exportés identifiés
- [ ] Configurations sensibles notées

---

## Tâche 4 — Recherche de Chaînes Sensibles

### Patterns à rechercher (Ctrl+F dans JADX GUI)

| Catégorie | Patterns |
|-----------|---------|
| URLs/Endpoints | `http://`, `https://`, `api`, `endpoint`, `server` |
| Authentification | `token`, `api_key`, `secret`, `password`, `bearer`, `jwt` |
| Mode développement | `DEBUG`, `test`, `staging`, `dev`, `firebase` |
| Cryptographie | `AES`, `RSA`, `MD5`, `SHA`, `encrypt` |

### Grille de Sévérité

| Niveau | Description | Exemple |
|--------|-------------|---------|
| 🟢 Faible | Information non sensible ou publique | URL de documentation |
| 🟡 Moyen | Information sensible à impact limité | URL de test, clé de dev |
| 🔴 Élevé | Secret critique | Clé de production, mot de passe |

**Checkpoints :**
- [ ] Recherches effectuées sur tous les patterns critiques
- [ ] Au moins 5 observations documentées
- [ ] Niveau de risque évalué pour chaque observation
- [ ] Emplacements précis notés

---

## Tâche 5 — Convertir DEX → JAR avec dex2jar

### Extraire les fichiers DEX
```bash
# Linux/macOS
mkdir -p dex_out
unzip -j app-debug.apk "classes*.dex" -d dex_out
```

### Convertir en JAR
```bash
# Linux/macOS
./d2j-dex2jar.sh ~/APK-Analysis/dex_out/classes.dex -o ~/APK-Analysis/app.jar

# Script multi-DEX
for file in dex_out/classes*.dex; do
    ./d2j-dex2jar.sh "$file" -o "${file%.dex}.jar"
done
```

**Checkpoints :**
- [ ] Fichiers DEX extraits avec succès
- [ ] Conversion en JAR réussie
- [ ] Gestion du multi-DEX si nécessaire

---

## Tâche 6 — Comparaison JADX GUI vs JD-GUI

| Aspect | JADX GUI | JD-GUI |
|--------|----------|--------|
| Navigation | Structure Android complète (Manifest, res, code) | Structure Java uniquement |
| Kotlin | Bonne gestion | Syntaxe parfois illisible |
| Obfuscation | Tente de reconstruire les noms | Conserve les noms obfusqués |
| Ressources | Accès direct (XML, assets) | Pas d'accès aux ressources Android |
| Annotations Android | Bien préservées | Peut perdre certaines annotations |
| Recommandation | **Analyse complète Android** | **Analyse Java pure / comparaison** |

**Checkpoints :**
- [ ] Même classe analysée dans les deux outils
- [ ] Au moins 3 différences documentées
- [ ] Conclusion sur l'outil le plus adapté

---

## Tâche 7 — Mini-Rapport d'Audit

### Structure du Rapport

```markdown
# Rapport d'Analyse Statique — [Nom de l'application]

## Informations Générales
- Date d'analyse : [date]
- Analyste : [nom]
- APK analysé : [nom_fichier.apk]
- Version : [version]
- Provenance : [source]
- Outils : JADX GUI vX.X, dex2jar vX.X, JD-GUI vX.X

## Résumé Exécutif
Cette analyse a révélé [N] vulnérabilités potentielles.
Principales préoccupations : [résumé 2-3 problèmes majeurs]
Niveau de risque global : [Faible/Moyen/Élevé]

## Constats Détaillés

### Constat #1 : [titre]
**Sévérité :** [Faible/Moyenne/Élevée]
**Description :** [description factuelle]
**Localisation :** [fichier/classe/méthode]
**Impact potentiel :** [conséquences]
**Remédiation :** [solution proposée]

## Annexes
- Liste des permissions
- Composants exportés
```

**Checkpoints :**
- [ ] Toutes les sections complétées
- [ ] Au moins 3 constats documentés
- [ ] Remédiations proposées
- [ ] Format professionnel

---

## Vulnérabilités Courantes à Rechercher

| Vulnérabilité | Indicateur | Risque |
|--------------|-----------|--------|
| Clés en dur | Strings avec `key`, `secret`, `token` | 🔴 Élevé |
| Mode debug activé | `android:debuggable="true"` | 🔴 Élevé |
| Trafic HTTP clair | `usesCleartextTraffic="true"` | 🟡 Moyen |
| Backup autorisé | `android:allowBackup="true"` | 🟡 Moyen |
| Composants exportés | `exported="true"` sans protection | 🟡 Moyen |
| Logs sensibles | `Log.d`, `Log.e` avec données | 🟡 Moyen |
| URL en clair | `http://` dans le code | 🟢 Faible |

---

## Troubleshooting

| Problème | Solution |
|---------|---------|
| JADX ne peut pas ouvrir l'APK | Vérifier que l'APK est une archive ZIP valide |
| dex2jar échoue | Utiliser la bonne commande selon la version |
| Code illisible | Obfuscation normale — se concentrer sur le manifeste |
| Erreur Out of Memory | `java -Xmx4g -jar jadx-gui.jar` |
| Scripts non exécutables | `chmod +x d2j-dex2jar.sh` sur Linux/macOS |
| Multiples DEX non traités | Traiter classes2.dex, classes3.dex séparément |

---

## Questions de Réflexion

1. Quelles permissions semblent excessives par rapport à la fonction principale de l'app ?
2. Comment un composant exporté pourrait-il être exploité par une app malveillante ?
3. Comment sécuriser une URL trouvée en clair dans le code ?
4. Comment l'obfuscation complique-t-elle l'analyse statique ?
5. Quel est le risque de `android:allowBackup="true"` ?
6. Quelle est la différence de risque entre `exported="true"` explicite et un intent-filter sans attribut exported ?
7. Comment évaluer la sécurité d'une app utilisant `WebView.setJavaScriptEnabled(true)` ?

---

## Checklist Finale (Deliverables)

- [ ] Mini-rapport d'analyse (1-2 pages)
- [ ] Liste des permissions et composants exportés
- [ ] Au moins 3 constats de sécurité documentés
- [ ] Remédiations proposées pour chaque constat
- [ ] Captures d'écran des éléments critiques
- [ ] JAR décompilé pour référence future
- [ ] Workspace nettoyé

---

## Référence du Lab

- **Numéro du lab :** 4 (Sécurité Android)
- **Titre :** Analyse Statique d'un APK avec JADX GUI + dex2jar + JD-GUI
- **Outils :** JADX GUI, dex2jar, JD-GUI
- **Type d'analyse :** Statique (sans exécution)
- **Concepts clés :** APK, DEX, décompilation, manifeste, vulnérabilités statiques
