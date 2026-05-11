# Lab  — Communication Java / C++ via JNI (JNIDemo)

## Présentation

Dans ce laboratoire, l'objectif est de construire une application Android nommée **JNIDemo** capable de communiquer avec du code natif C++ via JNI. L'application appellera plusieurs fonctions natives, enverra des paramètres Java vers C++, récupérera des résultats calculés côté natif, et apprendra à gérer correctement le chargement de la bibliothèque partagée `.so`.

Ce lab permet aussi de comprendre pourquoi JNI est encore utilisé dans les applications modernes : calcul intensif, réutilisation de bibliothèques C/C++, encapsulation d'algorithmes sensibles, traitement temps réel, ou ajout de couches de résistance au reverse engineering.

---

## Objectifs pédagogiques

À la fin de ce laboratoire, il sera possible de :

1. Créer un projet Android avec support C++
2. Comprendre le rôle du NDK, de CMake et de JNI
3. Déclarer et appeler des méthodes natives depuis Java
4. Manipuler des types simples et complexes entre Java et C++
5. Gérer des erreurs fréquentes comme `UnsatisfiedLinkError`
6. Lire les logs natifs dans Logcat
7. Appliquer de bonnes pratiques récentes pour JNI, optimisation et sécurité

---

## Ce que l'application fera

L'application JNIDemo réalisera quatre démonstrations :

| Fonction | Description |
|----------|-------------|
| `helloFromJNI()` | Appel d'une fonction native simple retournant une chaîne |
| `factorial(int n)` | Calcul natif d'un factoriel avec contrôle d'erreur |
| `reverseString(String s)` | Inversion d'une chaîne Java envoyée et retournée depuis C++ |
| `sumArray(int[] values)` | Traitement d'un tableau `int[]` envoyé au natif |

---

## Prérequis

Avant de commencer, vérifier les éléments suivants :

- Android Studio installé
- SDK Android configuré
- NDK, CMake et LLDB disponibles dans le SDK Manager
- Connaissances de base sur Activity, layout XML et Java

---

## Architecture du laboratoire

```
Java / MainActivity
    → appelle une méthode native
        → Android charge libnative-lib.so
            → JNI transmet l'appel au code C++
                → le code C++ exécute le traitement
                    → résultat converti et renvoyé vers Java
                        → l'interface Android affiche le résultat
```

---

## Structure du projet

```
app/
├── src/main/
│   ├── cpp/
│   │   ├── CMakeLists.txt
│   │   └── native-lib.cpp
│   ├── java/com/example/jnidemo/
│   │   └── MainActivity.java
│   └── res/layout/
│       └── activity_main.xml
└── build.gradle
```

---

## Étapes de réalisation

### Étape 1 — Créer le projet Android avec support C++

Ouvrir Android Studio et suivre : `File → New Project → Empty Views Activity`

Configurer ainsi :
- **Name :** `JNIDemo`
- **Language :** `Java`
- **Minimum SDK :** `API 24`
- Cocher **Include C++ support**
- **Build system :** `CMake`

Valider avec **Finish**.

Ce qu'Android Studio génère automatiquement :
- Un dossier `cpp/`
- Un fichier `native-lib.cpp`
- Un fichier `CMakeLists.txt`
- La configuration Gradle pour la compilation native

> **Point de contrôle :** Le projet doit se synchroniser sans erreur. Si Gradle affiche un problème lié au NDK ou à CMake, ouvrir `Tools → SDK Manager → SDK Tools` et vérifier que NDK (Side by side), CMake et LLDB sont bien installés.

---

### Étape 2 — Comprendre les rôles des composants

| Composant | Rôle |
|-----------|------|
| **JNI** | Interface permettant au code Java/Kotlin d'appeler du code natif C/C++ |
| **NDK** | Ensemble d'outils pour utiliser C/C++ dans Android |
| **CMake** | Outil de build pour décrire comment compiler et lier la bibliothèque native |
| **`.so`** | Bibliothèque partagée contenant le code natif compilé |

---

### Étape 3 — Vérifier la configuration Gradle

Ouvrir `app/build.gradle` et vérifier la présence de :

```groovy
externalNativeBuild {
    cmake {
        path file("src/main/cpp/CMakeLists.txt")
    }
}
```

> **Bon réflexe :** Ne pas modifier cette section au hasard. Si le chemin est faux, la bibliothèque `.so` ne sera pas générée.

---

### Étape 4 — Configurer `CMakeLists.txt`

Le fichier doit contenir :

- `cmake_minimum_required(VERSION 3.22.1)` — version minimale de CMake
- `project("jnidemo")` — nom du projet natif
- `add_library(native-lib SHARED native-lib.cpp)` — crée `libnative-lib.so`
- `find_library(log-lib log)` — bibliothèque système Android pour Logcat
- `target_link_libraries(native-lib ${log-lib})` — lie les dépendances

> **Point critique :** Le nom `native-lib` doit correspondre exactement à `System.loadLibrary("native-lib")` dans le code Java. Le préfixe `lib` et le suffixe `.so` sont ajoutés automatiquement.

---

### Étape 5 — Écrire le code natif C++

Ouvrir `app/src/main/cpp/native-lib.cpp` et implémenter les quatre fonctions :

**En-têtes nécessaires :**
- `<jni.h>` — en-tête principal JNI
- `<string>` et `<algorithm>` — manipulation de chaînes et `std::reverse`
- `<climits>` — pour `INT_MAX`
- `<android/log.h>` — pour écrire dans Logcat

**Fonctions à implémenter :**

1. `helloFromJNI()` — retourne une `jstring` via `env->NewStringUTF(...)`
2. `factorial(jint n)` — calcul avec vérification overflow via `INT_MAX`
3. `reverseString(jstring)` — utilise `GetStringUTFChars`, `std::reverse`, `ReleaseStringUTFChars`
4. `sumArray(jintArray)` — utilise `GetIntArrayElements`, boucle, `ReleaseIntArrayElements`

**Convention de nommage JNI obligatoire :**
```
Java_[package]_[Classe]_[méthode]
```
Exemple : `Java_com_example_jnidemo_MainActivity_helloFromJNI`

> Si le package ou le nom de classe change, la signature native doit aussi changer, sinon l'application lèvera une `UnsatisfiedLinkError`.

**Macros de log à définir :**
```cpp
#define LOG_TAG "JNI_DEMO"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)
```

**Points importants sur la gestion mémoire :**
- `GetStringUTFChars` / `ReleaseStringUTFChars` — toujours libérer les ressources
- `GetIntArrayElements` / `ReleaseIntArrayElements` — idem pour les tableaux
- `extern "C"` — empêche le name mangling C++

---

### Étape 6 — Déclarer les méthodes natives côté Java

Dans `MainActivity.java`, déclarer les quatre méthodes avec le mot-clé `native` :

```java
public native String helloFromJNI();
public native int factorial(int n);
public native String reverseString(String s);
public native int sumArray(int[] values);
```

Ajouter le bloc de chargement de la bibliothèque :

```java
static {
    System.loadLibrary("native-lib");
}
```

> **Point de vigilance :** Passer `"native-lib"` et non `"libnative-lib.so"` à `loadLibrary()`.

---

### Étape 7 — Créer le layout XML

Le layout `activity_main.xml` doit contenir quatre `TextView` :
- `tvHello` — affiche le résultat de `helloFromJNI()`
- `tvFact` — affiche le résultat du factoriel
- `tvReverse` — affiche la chaîne inversée
- `tvArray` — affiche la somme du tableau

Utiliser un `ScrollView` pour rendre le layout robuste.

---

### Étape 8 — Compiler et exécuter

Lancer l'application sur un émulateur ou un téléphone réel.

**Résultat attendu :**
- `Hello from C++ via JNI !`
- `Factoriel de 10 = 3628800`
- `Texte inverse : !lufrewop si INJ`
- `Somme du tableau = 150`

---

## Cas où JNI est pertinent

| Cas d'usage | Exemples |
|-------------|----------|
| Calcul intensif | Traitement d'image, vision par ordinateur, chiffrement, moteur de jeu |
| Réutilisation de bibliothèques C/C++ | OpenCV, moteurs audio/vidéo, bibliothèques historiques |
| Protection partielle de logique sensible | Code plus difficile à reverse qu'en Java |
| Interaction bas niveau | Accès à des API natives Android proches du système |

> JNI ne doit pas être utilisé "partout". Il apporte de la complexité et doit être réservé aux cas où il apporte un gain clair.

---

## Erreurs courantes

| Erreur | Cause probable | Solution |
|--------|---------------|----------|
| `UnsatisfiedLinkError` | Signature JNI incorrecte | Vérifier le nom de la fonction C++ |
| `UnsatisfiedLinkError` | Bibliothèque non chargée | Vérifier `System.loadLibrary("native-lib")` |
| Crash sur `GetStringUTFChars` | Chaîne Java null | Vérifier `if (javaString == nullptr)` |
| Build échoue | NDK/CMake manquant | Installer via SDK Manager |

---

## Résumé pédagogique

Dans ce laboratoire, les notions suivantes ont été maîtrisées :

- Création d'un projet Android avec support natif
- Compilation d'une bibliothèque `.so` avec CMake
- Chargement avec `System.loadLibrary`
- Écriture de fonctions JNI exportées
- Passage de `String`, `int` et `int[]` entre Java et C++
- Journalisation avec Logcat
- Gestion d'erreurs et bonnes pratiques
