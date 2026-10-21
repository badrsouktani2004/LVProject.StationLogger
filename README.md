# 📝 LVProject.StationLogger

![C#](https://img.shields.io/badge/C%23-.NET-blue)
![NI TestStand](https://img.shields.io/badge/NI-TestStand-yellow)
![XML](https://img.shields.io/badge/Data-XML-green)
![LabVIEW Migration](https://img.shields.io/badge/Migration-LabVIEW-red)

> **StationLogger** est une DLL développée en C# .NET conçue pour intercepter, formater et sauvegarder les erreurs d'exécution des bancs de test automatisés sous **NI TestStand**. Ce composant remplace une ancienne architecture graphique LabVIEW tout en garantissant une iso-fonctionnalité absolue et une robustesse accrue grâce à une approche orientée objet.

---

## 🏗️ Architecture et Organisation du Projet

Le code source est structuré de manière modulaire au sein de l'espace de noms principal `LVProject.StationLogger`, réparti en sous-dossiers thématiques :

### 📂 Core : `XmlLogger.cs` (Chef d'Orchestre)
Ce fichier représente le cœur du système. Il contient la classe statique `XmlLogger`, qui agit comme le point d'entrée unique de la DLL appelé directement par TestStand. Son rôle est d'orchestrer la génération simultanée du rapport XML, de sa version HTML et de la mise à jour du fichier d'historique CSV. Il est l'équivalent direct du "VI Global" de l'ancienne architecture LabVIEW.

**Méthodes principales :**
* **`Execute(...)`** : La méthode principale (*Main*) de l'application.
  * *Étape 1 (Initialisation) :* Localise dynamiquement la DLL en cours d'exécution pour charger la feuille de style `StyleSheet.xsl`.
  * *Étape 2 (Horodatage) :* Génère un nom de fichier unique basé sur la date (`ddMMyyyy`), l'heure précise (`HHmmssfff`), et le code d'erreur.
  * *Étape 3 (Flux XML) :* Appelle séquentiellement les services pour créer le fichier, y injecter l'horodatage, construire le contexte d'erreur, et récupérer les variables TestStand.
  * *Étape 4 (Flux HTML) :* Convertit immédiatement le flux XML généré en un rapport HTML interactif via `XmlToHtmlTransformer`.
  * *Étape 5 (Flux CSV) :* Parallèlement, écrit une ligne de log dans le fichier d'historique global via `AppendLogInSequenceErrors`.

* **`GetPropertyNumberValue(...)`** : 
  * Équivalent du sous-VI *"TestStand - Get Property Value (Number)"*. Utilise l'interopérabilité COM (`dynamic`) pour extraire le code de l'erreur, avec une libération stricte (`Marshal.ReleaseComObject`) pour éviter les fuites mémoire.

* **`MergeErrors(...)`** : 
  * Équivalent de la fonction *"Merge Errors"*. Reçoit deux flux d'erreurs et retourne la première rencontrée (paradigme Dataflow).

---

### 📦 DataStructures : Modèles de Données (Équivalents "Clusters")

En C#, une méthode ne pouvant retourner qu'une seule valeur par défaut, ces classes encapsulent plusieurs variables pour imiter les Clusters LabVIEW.

* **`CreateXmlDocumentResult.cs`** : Conteneur (DTO) regroupant les références nécessaires après l'initialisation du XML (`DocRef`, `RootNode`, `FileNamePath`, `ErrorOut`).
* **`GetCurrentDateAndTimeResult.cs`** : Encapsule les différentes chaînes de caractères formatées pour l'horodatage (`Date`, `Time`, `TagName`, `DateTimeString`).

---

### 🎨 Resources : `StyleSheet.xsl` (Moteur de Rendu Visuel)

Feuille de style **XSLT** injectée par la DLL pour transformer automatiquement les données XML brutes en un **rapport HTML lisible, mis en page et interactif**. 

* **CSS Intégré :** Embarque son propre style (autonome), incluant un **Logo Base64** (affichage garanti même par e-mail) et des arborescences interactives.
* **Boucle sur la Call Stack :** Parcourt récursivement les niveaux d'appels (`Caller0`, `Caller1`...) via des blocs dépliables affichant `Locals`, `Parameters` et `Globals`.
* **Règles de Modèles XSLT :** Gère l'affichage coloré des nœuds simples, des tableaux (avec limites `LBound`, `HBound`), et des énumérations.

---

### 🛠️ Services : Extraction, Injection et Historisation

* **`AddDataToXmlDoc.cs`** : Parcourt la pile d'appels de TestStand de bas en haut pour extraire 5 requêtes consécutives (`StationGlobals`, `FileGlobals`, `Parameters`, `Locals`, `Step`). Isole le nom de la séquence avec le moteur `Regex` natif de C#.
* **`AddNodeWithStringValue.cs`** : Valide le flux d'erreur entrant et utilise `InnerXml` (plutôt que `InnerText`) pour conserver les balises XML sans caractères d'échappement HTML.
* **`AppendLogInSequenceErrors.cs`** : Gère la sauvegarde chronologique dans `SequenceErrors.csv`. Utilise l'instruction `using` avec `StreamWriter` pour garantir une fermeture immédiate du fichier (*Flush & Dispose*), évitant tout verrouillage sous Excel.
* **`CreateXmlDocument.cs`** : Initialise le fichier physique avec la déclaration XML, le lien XSLT et la balise `<Root/>` via un flux sécurisé.
* **`ErrorContextDetailsBuildUp.cs`** : Rassemble les métadonnées de l'erreur (`StepIndex`, `CallStackDepth`) et utilise la méthode COM `GetXML(4, 1, "", "")` pour sérialiser rapidement des conteneurs TestStand entiers.
* **`XmlToHtmlTransformer.cs`** : Utilise le moteur ultra-rapide `XslCompiledTransform` de .NET pour fusionner les données XML et la feuille XSLT.

---

## ⚡ Flux d'Exécution et Orchestration (`XmlLogger.cs`)

La méthode principale `Execute` coordonne le flux selon une séquence rigoureuse :

1. **Initialisation :** Chargement XSLT et nommage unique.
2. **Création XML :** Instanciation du fichier et ossature de base.
3. **Horodatage :** Ajout des nœuds de date et d'heure.
4. **Audit :** Remontée de la *Call Stack* de TestStand pour identifier l'origine du crash.
5. **Extraction :** Capture des variables de l'étape (`Locals`, `Parameters`, etc.).
6. **Transformation :** Compilation XSLT vers HTML interactif.
7. **Traçabilité :** Enregistrement de l'historique dans le fichier CSV.

---

## 🔍 Détail des Composants Techniques Clés

### 1️⃣ La gestion du Dataflow et des erreurs (`ErrorCluster.cs`)

Pour imiter le comportement sécurisé du "fil d'erreur" de LabVIEW, la structure `ErrorCluster` est transmise à chaque étape :

```csharp
public struct ErrorCluster
{
    public bool Status { get; set; }
    public int Code { get; set; }
    public string Source { get; set; }

    public static ErrorCluster NoError => new()
    {
        Status = false,
        Code = 0,
        Source = string.Empty
    };
}
