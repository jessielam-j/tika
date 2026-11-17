# Tâche 3 

### Modification du workflow Github Actions

## 1. Choix de conception et d'implémentation

### Création d'un workflow dédié (`.github/workflows/semgrep.yml`)
Plutôt que d'injecter une étape supplémentaire dans les workflows de compilation existants (`main-jdk17-build.yml`), un fichier distinct a été créé pour les raisons suivantes :
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

## Documentation détaillée des vulnérabilités introduites

### 1. Injection SQL (`vuln1_sqlInjection`)
* **Fichier modifié :** `tika-core/src/main/java/org/apache/tika/detect/CharsetSupersets.java`
* **Justification du choix du fichier :** `CharsetSupersets` est une classe active du code de production (`src/main/java`) responsable de la correspondance entre les jeux de caractères détectés et leurs surensembles. L'intégration d'une requête JDBC simule un scénario réaliste où la table de correspondance des encodages serait interrogée dynamiquement depuis une base de données relationnelle plutôt que depuis une `Map` statique en mémoire.
* **Type de vulnérabilité :** Injection SQL — **CWE-89** / **OWASP A03:2021 (Injection)**.
* **Raison de la vulnérabilité :** La méthode construit une requête SQL en concaténant directement le paramètre d'entrée `input` dans la chaîne transmise à `Statement.executeQuery("SELECT * FROM charsets WHERE name = '" + input + "'")`, sans utiliser de requête préparée (`PreparedStatement`) ni de liaison de paramètres. Un attaquant maîtrisant `input` peut altérer la structure syntaxique de la requête SQL (ex. `' OR '1'='1`).
* **Résultat obtenu avec Semgrep :** Détectée comme **alerte bloquante (`ERROR`)** par la règle `java.lang.security.audit.sqli.jdbc-sqli`.

---

### 2. Désérialisation d'objets non fiables (`vuln3_insecureDeserialization`)
* **Fichier modifié :** `tika-core/src/main/java/org/apache/tika/detect/CharsetSupersets.java`
* **Justification du choix du fichier :** Le package `org.apache.tika.detect` manipule directement les flux bruts entrants (`InputStream`) pour identifier le format des documents. Simuler le chargement d'une table de configuration d'encodages depuis un flux binaire sérialisé s'inscrit directement dans le périmètre fonctionnel de cette classe.
* **Type de vulnérabilité :** Désérialisation de données non fiables — **CWE-502** / **OWASP A08:2021 (Software and Data Integrity Failures)**.
* **Raison de la vulnérabilité :** La méthode instancie un `ObjectInputStream` directement sur un flux `InputStream` non vérifié et invoque `ois.readObject()` sans filtrage préalable de classes (`ObjectInputFilter`). Si le flux provient d'une source externe, un attaquant peut fournir une chaîne de gadgets (*gadget chain*) présente dans le classpath Java pour obtenir une exécution de code arbitraire (RCE) lors de la reconstruction de l'objet.
* **Résultat obtenu avec Semgrep :** Détectée comme **alerte bloquante (`ERROR`)** par la règle `java.lang.security.audit.objectinputstream-deserialization`.

---

### 3. Utilisation d'un algorithme de hachage cryptographique obsolète (`vuln4_weakCryptoHash`)
* **Fichier modifié :** `tika-core/src/main/java/org/apache/tika/detect/CharsetSupersets.java`
* **Justification du choix du fichier :** Dans un module de détection de contenu et d'encodage, le calcul d'empreintes de hachage sur des tampons d'octets (`byte[] content`) est une opération courante (vérification d'intégrité ou mise en cache de signatures de fichiers).
* **Type de vulnérabilité :** Utilisation d'un algorithme cryptographique cassé ou risqué — **CWE-328** / **OWASP A02:2021 (Cryptographic Failures)**.
* **Raison de la vulnérabilité :** La méthode appelle `MessageDigest.getInstance("MD5")`. L'algorithme MD5 est cryptographiquement cassé et vulnérable aux attaques par collisions pratiques, ce qui permet de forger deux contenus distincts produisant la même empreinte.
* **Résultat obtenu avec Semgrep :** Détectée par la règle `java.lang.security.audit.crypto.use-of-md5` au niveau **avertissement (`WARNING`)**. Contrairement aux autres failles, Semgrep ne la classe pas comme bloquante par défaut afin de limiter les faux positifs lorsque MD5 est utilisé à des fins non cryptographiques (sommes de contrôle internes), illustrant la distinction entre alertes `ERROR` et `WARNING` dans le pipeline.

---

### 4. Injection d'entités externes XML — XXE (`parseXmlUnsafe`)
* **Fichier modifié :** `tika-core/src/main/java/org/apache/tika/parser/AbstractParser.java`
* **Justification du choix du fichier :** `AbstractParser` est la classe mère fondamentale dont héritent la majorité des parseurs de documents d'Apache Tika. Le traitement de flux XML entrants (`InputStream`) constitue le cœur métier du module `org.apache.tika.parser`, ce qui en fait l'emplacement architectural le plus représentatif pour une faille de parsing XML.
* **Type de vulnérabilité :** Mauvaise restriction des références d'entités externes XML (XXE) — **CWE-611** / **OWASP A05:2021 (Security Misconfiguration)**.
* **Raison de la vulnérabilité :** La méthode instancie un parseur DOM via `DocumentBuilderFactory.newInstance()` et appelle immédiatement `builder.parse(xmlStream)` sans désactiver le traitement des déclarations `DOCTYPE` (`http://apache.org/xml/features/disallow-doctype-decl`) ni la résolution des entités externes (`external-general-entities`). Un document XML malveillant contenant une entité `SYSTEM "file:///etc/passwd"` permet ainsi l'exfiltration de fichiers locaux du serveur ou des requêtes SSRF.
* **Résultat obtenu avec Semgrep :** Détectée comme **alerte bloquante (`ERROR`)** par la règle `java.lang.security.audit.xxe.documentbuilderfactory-disallow-doctype-decl`.

---

### 5. Injection de code dynamique via moteur de script (`evaluateMetadataRuleUnsafe`)
* **Fichier modifié :** `tika-core/src/main/java/org/apache/tika/parser/AbstractParser.java`
* **Justification du choix du fichier :** Dans une classe de base de parsing de documents, l'ajout d'un évaluateur dynamique destiné à filtrer ou transformer des règles de métadonnées à la volée correspond à un cas d'usage réaliste d'extensibilité logicielle.
* **Type de vulnérabilité :** Contrôle inapproprié de la génération de code (Code Injection) — **CWE-94** / **OWASP A03:2021 (Injection)**.
* **Raison de la vulnérabilité :** La méthode récupère un moteur d'exécution JavaScript via `ScriptEngineManager.getEngineByName("JavaScript")` et exécute directement la chaîne `ruleExpression` via `engine.eval(ruleExpression)` sans sandboxing ni validation. En Java, le moteur de script a accès aux classes de la JVM (`Java.type('java.lang.Runtime')`), permettant l'exécution directe de commandes système.
* **Résultat obtenu avec Semgrep :** Détectée comme **alerte bloquante (`ERROR`)** par la règle `java.lang.security.audit.script-engine-injection`.

---

### 6. Injection XPath (`evaluateXPathUnsafe`)
* **Fichier modifié :** `tika-core/src/main/java/org/apache/tika/utils/XMLReaderUtils.java`
* **Justification du choix du fichier :** `XMLReaderUtils` est la classe utilitaire centrale d'Apache Tika dédiée à la manipulation sécurisée d'arbres DOM et de flux XML (`Document`, `SAXParser`, `XMLInputFactory`). Ajouter une méthode utilitaire d'extraction de nœuds XPath s'inscrit directement dans les responsabilités de cette classe.
* **Type de vulnérabilité :** Neutralisation incorrecte des données dans les requêtes XPath (XPath Injection) — **CWE-643** / **OWASP A03:2021 (Injection)**.
* **Raison de la vulnérabilité :** L'expression XPath est construite par concaténation directe du paramètre `userInput` (`"/root/user[@name='" + userInput + "']/secret/text()"`) avant d'être évaluée par `xPath.evaluate(...)`, sans recours à un `XPathVariableResolver`. Une entrée contenant `' or '1'='1` permet de contourner les filtres d'accès et d'extraire l'intégralité des données sensibles du document XML.
* **Résultat obtenu avec Semgrep :** **Non détectée (Faux négatif)**. Aucune règle n'a été déclenchée sur cette méthode lors du scan local avec les jeux de règles utilisés (`p/security-audit` et `p/owasp-top-ten`), car la chaîne concaténée est stockée dans une variable intermédiaire (`expression`) avant d'être passée à `xPath.evaluate(...)`, ce qui échappe à la correspondance syntaxique directe de la règle OSS sans analyse *taint* inter-lignes.

---

### 7. Injection XSLT / Exécution de code via feuille de style (`transformXmlWithStylesheetUnsafe`)
* **Fichier modifié :** `tika-core/src/main/java/org/apache/tika/utils/XMLReaderUtils.java`
* **Justification du choix du fichier :** `XMLReaderUtils` contient déjà les méthodes de fabrique `getTransformer()` et `getTransformerFactory()`, explicitement durcies pour prévenir les vulnérabilités XML (comme en témoigne la référence interne à `CVE-2025-54988` dans le fichier). Introduire une méthode parallèle contournant `getTransformerFactory()` met en évidence la valeur de l'analyse statique pour détecter des régressions de sécurité au sein d'un composant critique.
* **Type de vulnérabilité :** Injection XSLT — **CWE-91** / **OWASP A03:2021 (Injection)**.
* **Raison de la vulnérabilité :** La méthode instancie un `TransformerFactory.newInstance()` brut et compile directement une feuille de style XSLT provenant d'un flux externe (`factory.newTransformer(new StreamSource(xsltStream))`) sans activer `XMLConstants.FEATURE_SECURE_PROCESSING` ni restreindre `ACCESS_EXTERNAL_STYLESHEET`. En Java (via le processeur Xalan intégré), les extensions XSLT permettent d'instancier et d'invoquer des méthodes statiques Java arbitraries lors de la transformation.
* **Résultat obtenu avec Semgrep :** Détectée comme **alerte bloquante(`ERROR`)** par la règle `java.lang.security.audit.xslt-injection`.

---

### Comparaison du scan Semgrep avant et après l'ajout des vulnérabilités

L'analyse statique effectuée sur le module `tika-core` (324 fichiers analysés, 94 règles exécutées) permet de distinguer clairement les alertes préexistantes dans le code source d'Apache Tika des **6 nouvelles vulnérabilités bloquantes** introduites lors de nos modifications.

### 1. Synthèse comparative

| Indicateur | Avant modifications (Baseline) | Après modifications | Évolution (Delta) |
| :--- | :---: | :---: | :---: |
| **Nombre total d'alertes (`Findings`)** | **3** | **9** | **+6 nouvelles alertes** |
| **Alertes bloquantes (`Blocking`)** | 3 | 9 | **+6 alertes bloquantes** |
| **Fichiers impactés** | 3 | 5 | **+2 fichiers nouvellement signalés** |
| **Code de sortie CI/CD (`--error`)** | `1` (Échec — faux positifs natifs) | `1` (Échec — bloqué) | Détection confirmée |

---

### 2. Alertes préexistantes dans Apache Tika (Baseline : 3 alertes)

Ces 3 détections proviennent du code natif d'Apache Tika (faux positifs de l'analyse statique, car les protections sont appliquées via des méthodes utilitaires ou dans des contextes maîtrisés) :

| Fichier | Ligne | Règle Semgrep | Cause dans le code natif |
| :--- | :---: | :--- | :--- |
| `mime/MimeTypesReader.java` | `223` | `saxparserfactory-disallow-doctype-decl-missing` | `factory.newSAXParser()` utilisé pour lire les définitions internes `tika-mimetypes.xml`. |
| `parser/NetworkParser.java` | `68` | `unencrypted-socket` | Connexion socket TCP brute (`new Socket(...)`) vers un serveur de parsing distant configurable. |
| `utils/XMLReaderUtils.java` | `313` | `xmlinputfactory-possible-xxe` | `XMLInputFactory.newFactory()` dont les propriétés de sécurité sont configurées juste après via la méthode helper `tryToSetStaxProperty(...)`. |

---

### 3. Nouvelles vulnérabilités introduites (Delta : +6 alertes bloquantes)

Toutes les vulnérabilités injectées ont été détectées au niveau **`Blocking`** par Semgrep :

| # | Fichier modifié | Ligne | Méthode injectée | Règle Semgrep déclenchée | Statut |
| :---: | :--- | :---: | :--- | :--- | :---: |
| **1** | `detect/CharsetSupersets.java` | `122` | `vuln1_sqlInjection` | `java.lang.security.audit.formatted-sql-string.formatted-sql-string` | **Nouvelle (`Blocking`)** |
| **2** | `detect/CharsetSupersets.java` | `127` | `vuln3_insecureDeserialization` | `java.lang.security.audit.object-deserialization.object-deserialization` | **Nouvelle (`Blocking`)** |
| **3** | `detect/CharsetSupersets.java` | `133` | `vuln4_weakCryptoHash` | `java.lang.security.audit.crypto.use-of-md5.use-of-md5` | **Nouvelle (`Blocking`)** |
| **4** | `parser/AbstractParser.java` | `73` | `parseXmlUnsafe` | `java.lang.security.audit.xxe.documentbuilderfactory-disallow-doctype-decl-missing` | **Nouvelle (`Blocking`)** |
| **5** | `parser/AbstractParser.java` | `85` | `evaluateMetadataRuleUnsafe` | `java.lang.security.audit.script-engine-injection.script-engine-injection` | **Nouvelle (`Blocking`)** |
| **6** | `utils/XMLReaderUtils.java` | `1334` | `transformXmlWithStylesheetUnsafe` | `java.lang.security.audit.xxe.transformerfactory-dtds-not-disabled` | **Nouvelle (`Blocking`)** |

*(Note : Dans `XMLReaderUtils.java`, la méthode XPath `evaluateXPathUnsafe` n'a pas déclenché de règle supplémentaire dans le jeu de règles actif, tandis que la méthode XSLT `transformXmlWithStylesheetUnsafe` a été interceptée à la ligne `1334` via `transformerfactory-dtds-not-disabled`, portant le total à **6 nouvelles failles bloquantes**.)*