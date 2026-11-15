# Tâche 3 

### Modification du workflow Github Actions

## 1. Choix de conception et d'implémentation

### Création d'un workflow dédié (`.github/workflows/semgrep.yml`)
Plutôt que d'injecter une étape supplémentaire dans les workflows de compilation existants (`main-jdk17-build.yml`)[cite: 1], un fichier distinct a été créé pour les raisons suivantes :
* **Séparation des responsabilités :** Découplage clair entre les tests fonctionnels/unitaires Java et l'audit de sécurité.
* **Rapidité et retour d'information :** L'analyse statique s'exécute en quelques secondes indépendamment de la compilation Maven complète, offrant un diagnostic immédiat aux développeurs.
* **Maintenabilité :** Facilite la mise à jour des règles de sécurité et des versions de l'analyseur sans risquer de perturber le build applicatif.

### Configuration technique du pipeline
Les directives retenues dans `.github/workflows/semgrep.yml` répondent aux exigences suivantes :

* **Déclencheurs (`on: pull_request`, `on: push`) :** Le workflow s'exécute systématiquement sur chaque Pull Request ciblant `main` afin d'intercepter les vulnérabilités avant leur fusion, ainsi que sur les commits poussés directement sur la branche.
* **Analyse différentielle (`fetch-depth: 0`) :** L'option `fetch-depth: 0` lors de l'étape `actions/checkout` permet de récupérer l'historique Git complet. Semgrep peut ainsi calculer le différentiel (`diff`) par rapport à la branche cible et **ne bloquer que sur les nouvelles failles introduites**, évitant de paralyser le développement à cause de la dette technique préexistante dans le projet.
* **Exclusion du dossier de configuration CI (`--exclude .github`) :** L'ajout du drapeau `--exclude .github` permet de restreindre le périmètre d'analyse au code applicatif du projet. Cela évite que Semgrep ne signale des alertes ou faux positifs liés aux fichiers de configuration internes des workflows GitHub Actions (notamment l'utilisation d'actions tierces par tag de version plutôt que par hash SHA dans les pipelines existants d'Apache Tika).
* **Jeux de règles sélectionnés (`p/security-audit` et `p/owasp-top-ten`) :** Ces catalogues officiels offrent une couverture robuste des vulnérabilités applicatives critiques en Java (injections, cryptographie faible, désérialisation non sécurisée, etc.) tout en limitant les faux positifs stylistiques.
* **Politique de blocage strict (Quality Gate) :** Aucune directive `continue-on-error: true` n'est présente. Dès qu'une vulnérabilité est détectée dans le périmètre modifié, Semgrep retourne un code de sortie non nul (`exit code 1`), faisant échouer le job GitHub Actions.
* **Principe du moindre privilège (`permissions`) :** Les droits du jeton `GITHUB_TOKEN` sont restreints à `contents: read` (lecture du code) et `security-events: write` (remontée des alertes).

---

## 2. Protocole de validation

La validation du bon fonctionnement de cette modification s'est déroulée en trois étapes :

### A. Validation syntaxique et d'intégration (Cas nominal)
* **Procédure :** Ajout du fichier `.github/workflows/semgrep.yml` seul sur une branche dédiée et ouverture d'une Pull Request vers `main`.
* **Résultat :** Le déclencheur `pull_request` s'est activé automatiquement, l'environnement Ubuntu a exécuté le scan sans erreur de configuration (en ignorant le dossier `.github` grâce à `--exclude .github`), et le workflow s'est terminé avec succès (`exit code 0`, statut vert), confirmant l'absence de faux positifs sur la base de code saine.

### B. Validation du blocage (Test négatif)
* **Procédure :** Introduction volontaire de vulnérabilités Java reconnues (telles que l'utilisation de l'algorithme obsolète MD5 ou une injection de commande via `Runtime.getRuntime().exec()`) dans un commit de test sur la branche de la Pull Request.
* **Résultat :** L'analyse Semgrep a identifié les failles introduites et a immédiatement interrompu le pipeline (`Process completed with exit code 1`). Le check GitHub est passé au rouge, démontrant l'efficacité de la barrière de sécurité.

### C. Validation du retour à la normale (Correction)
* **Procédure :** Suppression des fragments de code vulnérables et publication d'un nouveau commit sur la même Pull Request.
* **Résultat :** Le workflow s'est relancé automatiquement et le statut est repassé au vert, validant la réactivité du mécanisme lors des cycles de correction.

--- 

### Ajout de vulnérabilités

## Vue d'ensemble des vulnérabilités

| \# | Type de vulnérabilité | Identifiant CWE | Catégorie OWASP 2021 | Fichier cible (`src/main/java/...`) | Règle Semgrep ciblée | Sévérité |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | **Injection SQL (SQLi)** | CWE-89 | A03: Injection | `org/apache/tika/detect/CharsetSupersets.java` | `java.lang.security.audit.sqli.jdbc-sqli` | `ERROR` (Bloquant) |
| **2** | **Désérialisation non sécurisée** | CWE-502 | A08: Software & Data Integrity | `org/apache/tika/detect/CharsetSupersets.java` | `java.lang.security.audit.objectinputstream-deserialization` | `ERROR` (Bloquant) |
| **3** | **Hachage cryptographique faible (MD5)** | CWE-328 | A02: Cryptographic Failures | `org/apache/tika/detect/CharsetSupersets.java` | `java.lang.security.audit.crypto.use-of-md5` | `WARNING` (Avertissement) |
| **4** | **XML External Entity (XXE)** | CWE-611 | A05: Security Misconfiguration | `org/apache/tika/parser/AbstractParser.java` | `java.lang.security.audit.xxe.documentbuilderfactory-disallow-doctype-decl` | `ERROR` (Bloquant) |
| **5** | **Injection de Script (`ScriptEngine`)** | CWE-94 | A03: Injection | `org/apache/tika/parser/AbstractParser.java` | `java.lang.security.audit.script-engine-injection` | `ERROR` (Bloquant) |
| **6** | **Injection XPath** | CWE-643 | A03: Injection | `org/apache/tika/utils/XMLReaderUtils.java` | `java.lang.security.audit.xpath-injection` | `ERROR` (Bloquant) |
| **7** | **Injection XSLT (Exécution de code)** | CWE-91 | A03: Injection | `org/apache/tika/utils/XMLReaderUtils.java` | `java.lang.security.audit.xslt-injection` | `ERROR` (Bloquant) |

---

1. Injection SQL (vuln1_sqlInjection)

Choix du fichier modifié : tika-core/src/main/java/org/apache/tika/detect/CharsetSupersets.java

Justification : CharsetSupersets est une classe du code source principal (src/main/java) chargée d'associer les jeux de caractères détectés à leurs surensembles d'encodage. L'ajout d'une requête JDBC simule un cas d'usage réaliste où cette table de correspondance serait interrogée dynamiquement depuis une base de données relationnelle plutôt que depuis une structure statique en mémoire.

Type de vulnérabilité : Injection SQL — CWE-89 / OWASP A03:2021 (Injection).

Raison pour laquelle le code est vulnérable : La requête SQL est assemblée par concaténation directe du paramètre input dans la chaîne transmise à Statement.executeQuery("SELECT * FROM charsets WHERE name = '" + input + "'"), sans recours à une requête préparée (PreparedStatement). Un attaquant contrôlant la valeur d'input peut injecter des commandes SQL arbitraires (ex. ' OR '1'='1) pour manipuler la requête ou extraire des données non autorisées.

Résultat obtenu avec Semgrep : Détectée comme alerte bloquante (ERROR) par la règle java.lang.security.audit.sqli.jdbc-sqli.

2. Désérialisation de données non fiables (vuln3_insecureDeserialization)

Choix du fichier modifié : tika-core/src/main/java/org/apache/tika/detect/CharsetSupersets.java

Justification : Le package org.apache.tika.detect traite directement les flux binaires entrants (InputStream) afin d'identifier l'encodage et le format des documents. Charger un dictionnaire d'encodages sérialisé depuis un flux brut s'inscrit directement dans le périmètre fonctionnel de cette classe.

Type de vulnérabilité : Désérialisation de données non fiables — CWE-502 / OWASP A08:2021 (Software and Data Integrity Failures).

Raison pour laquelle le code est vulnérable : La méthode instancie un ObjectInputStream directement sur un flux InputStream externe et invoque ois.readObject() sans validation préalable ni filtrage de classes (ObjectInputFilter). Si le flux provient d'une source non fiable, un attaquant peut fournir une chaîne de gadgets (gadget chain) présente dans le classpath Java pour déclencher une exécution de code à distance (RCE) lors de la reconstruction de l'objet.

Résultat obtenu avec Semgrep : Détectée comme alerte bloquante (ERROR) par la règle java.lang.security.audit.objectinputstream-deserialization.

3. Utilisation d'un algorithme de hachage cryptographique faible (vuln4_weakCryptoHash)

Choix du fichier modifié : tika-core/src/main/java/org/apache/tika/detect/CharsetSupersets.java

Justification : Dans un module de détection de contenu, le calcul d'empreintes de hachage sur des tableaux d'octets (byte[] content) est une opération courante pour identifier ou mettre en cache des signatures de fichiers.

Type de vulnérabilité : Utilisation d'un algorithme cryptographique cassé ou risqué — CWE-328 / OWASP A02:2021 (Cryptographic Failures).

Raison pour laquelle le code est vulnérable : La méthode utilise MessageDigest.getInstance("MD5"). L'algorithme MD5 est cryptographiquement obsolète et vulnérable aux attaques par collisions pratiques, permettant à un attaquant de générer deux contenus distincts produisant exactement la même empreinte.

Résultat obtenu avec Semgrep : Détectée au niveau avertissement (WARNING) par la règle java.lang.security.audit.crypto.use-of-md5. Semgrep ne la classe pas en ERROR par défaut afin d'éviter les faux positifs lorsque MD5 est utilisé pour de simples sommes de contrôle non sécuritaires, ce qui permet d'observer la distinction entre alertes bloquantes et non bloquantes.

4. Injection d'entités externes XML — XXE (parseXmlUnsafe)

Choix du fichier modifié : tika-core/src/main/java/org/apache/tika/parser/AbstractParser.java

Justification : AbstractParser est la classe de base dont hérite la majorité des parseurs de documents d'Apache Tika. Le traitement de flux XML entrants (InputStream) étant au cœur du module org.apache.tika.parser, cet emplacement est particulièrement représentatif d'une faille de parsing XML.

Type de vulnérabilité : Mauvaise restriction des références d'entités externes XML (XXE) — CWE-611 / OWASP A05:2021 (Security Misconfiguration).

Raison pour laquelle le code est vulnérable : La méthode crée un parseur DOM via DocumentBuilderFactory.newInstance() et appelle directement builder.parse(xmlStream) sans désactiver le traitement des déclarations DOCTYPE (http://apache.org/xml/features/disallow-doctype-decl) ni la résolution des entités externes (external-general-entities). Un fichier XML malveillant contenant une référence SYSTEM "file:///..." permet d'exfiltrer des fichiers locaux du serveur ou d'effectuer des requêtes SSRF.

Résultat obtenu avec Semgrep : Détectée comme alerte bloquante (ERROR) par la règle java.lang.security.audit.xxe.documentbuilderfactory-disallow-doctype-decl.

5. Injection de code dynamique via moteur de script (evaluateMetadataRuleUnsafe)

Choix du fichier modifié : tika-core/src/main/java/org/apache/tika/parser/AbstractParser.java

Justification : Dans une classe parente de parsing, l'ajout d'un évaluateur dynamique permettant d'appliquer des règles de filtrage ou de normalisation sur les métadonnées extraites correspond à un besoin réaliste d'extensibilité logicielle.

Type de vulnérabilité : Contrôle inapproprié de la génération de code (Code Injection) — CWE-94 / OWASP A03:2021 (Injection).

Raison pour laquelle le code est vulnérable : La méthode récupère un moteur JavaScript via ScriptEngineManager.getEngineByName("JavaScript") et exécute directement la chaîne ruleExpression avec engine.eval(ruleExpression) sans sandboxing ni filtrage. Dans la JVM, ce moteur dispose d'un accès aux classes Java (Java.type('java.lang.Runtime')), permettant à une entrée malveillante d'exécuter des commandes système arbitraires.

Résultat obtenu avec Semgrep : Détectée comme alerte bloquante (ERROR) par la règle java.lang.security.audit.script-engine-injection.

6. Injection XPath (evaluateXPathUnsafe)

Choix du fichier modifié : tika-core/src/main/java/org/apache/tika/utils/XMLReaderUtils.java

Justification : XMLReaderUtils est la classe utilitaire centrale dédiée à toutes les manipulations sécurisées d'arbres DOM et de flux XML (Document, SAXParser, XMLInputFactory) au sein de tika-core. L'ajout d'une méthode utilitaire d'interrogation XPath s'intègre naturellement aux responsabilités de ce fichier.

Type de vulnérabilité : Neutralisation incorrecte des données dans les requêtes XPath (XPath Injection) — CWE-643 / OWASP A03:2021 (Injection).

Raison pour laquelle le code est vulnérable : L'expression XPath est construite par concaténation directe du paramètre userInput ("/root/user[@name='" + userInput + "']/secret/text()") avant son évaluation par xPath.evaluate(...), sans utiliser un XPathVariableResolver. Un attaquant peut injecter des apostrophes et des opérateurs logiques (' or '1'='1) pour contourner le filtrage et extraire l'intégralité des nœuds sensibles du document XML.

Résultat obtenu avec Semgrep : Détectée comme alerte bloquante (ERROR) par la règle java.lang.security.audit.xpath-injection.

7. Injection XSLT / Exécution de code via feuille de style (transformXmlWithStylesheetUnsafe)

Choix du fichier modifié : tika-core/src/main/java/org/apache/tika/utils/XMLReaderUtils.java

Justification : XMLReaderUtils définit déjà les méthodes de fabrique getTransformer() et getTransformerFactory(), explicitement sécurisées contre les attaques XML (notamment en référence à CVE-2025-54988). Introduire une méthode parallèle qui contourne getTransformerFactory() démontre l'intérêt d'un outil SAST pour détecter une régression de sécurité au sein d'un composant critique.

Type de vulnérabilité : Injection XSLT — CWE-91 / OWASP A03:2021 (Injection).

Raison pour laquelle le code est vulnérable : La méthode instancie un TransformerFactory.newInstance() brut et compile directement une feuille de style XSLT provenant d'un flux externe (factory.newTransformer(new StreamSource(xsltStream))) sans activer XMLConstants.FEATURE_SECURE_PROCESSING ni bloquer ACCESS_EXTERNAL_STYLESHEET. Avec le processeur XSLT intégré à Java (Xalan), une feuille de style malveillante peut invoquer des méthodes statiques Java et exécuter du code arbitraire sur le serveur.

Résultat obtenu avec Semgrep : Détectée comme alerte bloquante (ERROR) par la règle java.lang.security.audit.xslt-injection.
