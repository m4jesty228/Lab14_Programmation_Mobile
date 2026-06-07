# Lab 14 — SecureStorage : Persistance locale sécurisée sous Android (Java)

**Auteur :** DOSSAH Yao Landry  
**Filière :** Génie Cloud, Données, Sécurité et Technologies Embarquées (GCDSTE)  
**Établissement :** ENSA Marrakech

---

## Contexte pédagogique

Ce laboratoire porte sur la gestion complète de la persistance locale sous Android en Java, en appliquant des règles strictes de sécurité. L'application **SecureStorageLabJava** couvre cinq mécanismes de stockage : SharedPreferences, EncryptedSharedPreferences, fichiers internes (texte UTF-8 et JSON), cache temporaire, et stockage externe app-specific. Le tout sans connexion Internet, avec une architecture modulaire par package.

---

## Environnement

| Composant | Détail |
|-----------|--------|
| IDE | Android Studio |
| Langage | Java |
| Min SDK | API 24 |
| Dépendance sécurité | `androidx.security:security-crypto:1.1.0-alpha06` |
| Chiffrement | AES256-GCM (valeurs) + AES256-SIV (clés) via Keystore |
| Format de données | JSON (org.json), texte UTF-8 |

---

## Architecture du projet

```
SecureStorageLabJava/
├── app/src/main/java/com/example/securestoragejava/
│   ├── ui/
│   │   └── MainActivity.java          ← écran unique, orchestration des actions
│   ├── prefs/
│   │   ├── AppPrefs.java              ← SharedPreferences (non sensibles)
│   │   └── SecurePrefs.java           ← EncryptedSharedPreferences (token)
│   ├── files/
│   │   ├── InternalTextStore.java     ← lecture/écriture texte UTF-8 interne
│   │   └── StudentsJsonStore.java     ← sérialisation/désérialisation JSON
│   ├── cache/
│   │   └── CacheStore.java            ← écriture/lecture/purge du cacheDir
│   ├── external/
│   │   └── ExternalAppFilesStore.java ← export vers stockage externe app-specific
│   └── model/
│       └── Student.java               ← modèle de données (id, name, age)
└── res/layout/activity_main.xml
```

---

## Flux de données

```
Utilisateur (UI)
      |
      ├── savePrefs()   → AppPrefs (SharedPreferences, MODE_PRIVATE)
      │                 → SecurePrefs (EncryptedSharedPreferences, AES256-GCM)
      │                 → CacheStore (cacheDir, last_ui.txt)
      |
      ├── loadPrefs()   → AppPrefs.load()  → restaure nom/langue/thème
      │                 → SecurePrefs.loadToken() → affiche longueur uniquement
      |
      ├── saveJson()    → StudentsJsonStore (students.json, stockage interne)
      │                 → InternalTextStore (note.txt, UTF-8)
      |
      ├── loadJson()    → StudentsJsonStore.load() → liste de Student
      │                 → InternalTextStore.readUtf8() → note.txt
      |
      └── clearAll()    → AppPrefs.clear()
                        → SecurePrefs.clear()
                        → StudentsJsonStore.delete()
                        → InternalTextStore.delete(note.txt)
                        → CacheStore.purge() → N fichiers supprimés
```

---

## Tâche 1 — Dépendance Gradle

La dépendance AndroidX Security Crypto est ajoutée dans `build.gradle` (module app) :

```gradle
dependencies {
    implementation "androidx.security:security-crypto:1.1.0-alpha06"
}
```

Après synchronisation Gradle, le build compile sans erreur et l'application se lance sur émulateur API 24+.

---

## Tâche 2 — SharedPreferences : préférences non sensibles

`AppPrefs.java` expose trois opérations : `save`, `load`, `clear`. Les clés stockées sont le nom utilisateur, la langue et le thème.

Point clé : `apply()` est utilisé par défaut (asynchrone, non bloquant pour l'UI). `commit()` est disponible via le paramètre `sync` lorsqu'une confirmation synchrone est nécessaire — par exemple avant un redémarrage forcé de l'activité.

**Résultat observé :** après sauvegarde et redémarrage de l'application, `loadPrefsToUi()` restaure correctement le nom, la langue sélectionnée dans le Spinner, et l'état du Switch thème. Le Logcat confirme :

```
D/SecureStorageJava: Prefs chargées name=Landry, lang=fr, theme=dark, tokenLength=12
```

Aucune valeur sensible n'apparaît dans les logs.

---

## Tâche 3 — EncryptedSharedPreferences : stockage chiffré du token

`SecurePrefs.java` utilise un `MasterKey` AES256-GCM adossé au Keystore Android. Les clés sont chiffrées en AES256-SIV, les valeurs en AES256-GCM.

Règle stricte appliquée : **le token n'est jamais loggé**. Seule sa longueur est affichée dans le `TextView` et dans Logcat.

```java
// Ce que le Logcat affiche — jamais le token brut
Log.d(TAG, "tokenLength=" + tokenLen);
```

**Résultat observé :** un token saisi (`aBcXyZ123!@#` — 12 caractères) est sauvegardé, l'application est redémarrée, et `loadPrefsToUi()` affiche `tokenLength=12` sans jamais exposer la valeur. Le fichier `secure_prefs` dans `/data/data/<package>/` contient du contenu chiffré illisible en clair via Device File Explorer.

---

## Tâche 4 — Fichiers internes : texte UTF-8 et JSON

### Modèle Student

```java
public class Student {
    public final int id;
    public final String name;
    public final int age;
}
```

### Résultats observés

`saveJsonFile()` écrit deux fichiers dans le stockage interne :

- `students.json` — tableau JSON de 3 étudiants
- `note.txt` — confirmation UTF-8

`loadJsonFile()` recharge les données et les affiche dans le `TextView` :

```
Chargement fichier JSON terminé.
note=Sauvegarde JSON effectuée (UTF-8).
students=3
 - id=1, name=Amina, age=20
 - id=2, name=Omar, age=21
 - id=3, name=Sara, age=19
```

Via Device File Explorer, les deux fichiers sont visibles sous `/data/data/com.example.securestoragejava/files/` :

```
files/
├── note.txt
└── students.json
```

Le contenu de `students.json` est conforme :

```json
[{"id":1,"name":"Amina","age":20},{"id":2,"name":"Omar","age":21},{"id":3,"name":"Sara","age":19}]
```

Si le fichier est absent ou corrompu, `StudentsJsonStore.load()` retourne une liste vide — aucun crash.

---

## Tâche 5 — Cache temporaire (cacheDir)

`CacheStore.java` écrit `last_ui.txt` dans `getCacheDir()` à chaque sauvegarde de préférences. Ce fichier contient un résumé de l'état courant de l'UI (nom, langue, thème) — aucune donnée sensible.

`purge()` supprime tous les fichiers du cache et retourne le nombre de fichiers effacés.

**Résultat observé :** après `savePrefs()`, le fichier `last_ui.txt` est visible dans `/data/data/<package>/cache/`. Après `clearAll()`, le `TextView` confirme :

```
cache purgé: 1 fichier(s)
```

---

## Tâche 6 — Stockage externe app-specific

`ExternalAppFilesStore.java` écrit dans `getExternalFilesDir(null)` — répertoire propre à l'application sur le stockage externe, sans nécessiter de permission `WRITE_EXTERNAL_STORAGE` (API 29+). Le chemin absolu est retourné et affiché.

Aucune donnée sensible n'est exportée vers ce répertoire — il est réservé aux exports explicitement demandés par l'utilisateur (ex. rapport, fichier partageable).

---

## Action "Effacer" — nettoyage complet

`clearAll()` exécute la séquence complète dans l'ordre :

1. `AppPrefs.clear()` — efface les préférences non sensibles
2. `SecurePrefs.clear()` — efface le token chiffré
3. `StudentsJsonStore.delete()` — supprime `students.json`
4. `InternalTextStore.delete("note.txt")` — supprime `note.txt`
5. `CacheStore.purge()` — supprime tous les fichiers du cache

**Résultat affiché :**

```
Nettoyage terminé.
prefs: clear()
secure_prefs: clear()
students.json: delete
note.txt: delete
cache purgé: 1 fichier(s)
```

Aucune donnée sensible n'est loggée durant cette opération.

---

## Checklist sécurité

| # | Règle | Statut |
|---|-------|--------|
| 1 | Aucun token/mot de passe dans Logcat | ✅ |
| 2 | EncryptedSharedPreferences pour les secrets | ✅ |
| 3 | MODE_PRIVATE pour fichiers internes et prefs claires | ✅ |
| 4 | Token masqué à l'écran (`inputType="textPassword"`) | ✅ |
| 5 | Nettoyage complet disponible (prefs + secure + fichiers + cache) | ✅ |
| 6 | Cache réservé au temporaire régénérable | ✅ |
| 7 | Export externe limité à app-specific (pas de stockage public) | ✅ |
| 8 | Exceptions gérées sans fuite d'informations | ✅ |
| 9 | Encodage UTF-8 imposé pour tous les fichiers texte | ✅ |
| 10 | Token non affiché en clair (longueur uniquement) | ✅ |
| 11 | Vérification via Device File Explorer effectuée | ✅ |
| 12 | Concept d'expiration de token prévu (date de création + invalidation locale) | ✅ |

---

## Erreurs fréquentes rencontrées

| Erreur | Cause | Solution |
|--------|-------|----------|
| Crash sur `EncryptedSharedPreferences` | API < 24 ou contexte invalide | Vérifier minSdk=24 et passer un contexte Activity valide |
| Fichiers internes invisibles dans Device File Explorer | Application pas encore lancée | Lancer l'app et effectuer au moins une sauvegarde |
| JSON vide au chargement | Fichier absent ou corrompu | Appuyer sur "Effacer" puis re-sauvegarder |
| Token non récupéré après redémarrage | `apply()` pas encore flushé avant kill | Utiliser `commit()` pour les opérations critiques |
| Logs sensibles détectés | Token loggé par erreur | Remplacer par `tokenLength` ou flag `hasToken` |

---

## Points clés retenus

- `apply()` est non bloquant et suffisant pour les préférences UI ; `commit()` est réservé aux cas où la confirmation synchrone est nécessaire.
- `EncryptedSharedPreferences` + `MasterKey` délèguent la gestion de la clé maîtresse au Keystore Android — la clé n'est jamais exposée en mémoire applicative.
- Le stockage interne (`openFileOutput` avec `MODE_PRIVATE`) est inaccessible aux autres applications sans root.
- `cacheDir` peut être vidé par le système à tout moment — n'y stocker que des données régénérables.
- La gestion des exceptions dans `StudentsJsonStore.load()` (retour liste vide) évite tout crash sur fichier absent ou corrompu.
- Un nettoyage explicite et complet (`clearAll`) est indispensable pour les applications manipulant des secrets — la désinstallation seule ne garantit pas l'effacement immédiat du Keystore sur tous les appareils.

---

*Lab réalisé dans le cadre du cours Développement Mobile — ENSA Marrakech, Filière GCDSTE*
