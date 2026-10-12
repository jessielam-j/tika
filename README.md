# IFT3913 — Tâche 2

## Classe sélectionnée

La classe choisie est `org.apache.tika.detect.CharsetSupersets`, située dans le module `tika-core` du projet Apache Tika.

**Fichier source :**

`tika-core/src/main/java/org/apache/tika/detect/CharsetSupersets.java`

**Fichier de tests existant :**

`tika-core/src/test/java/org/apache/tika/detect/CharsetSupersetsTest.java`

Deux méthodes ont été étudiées :

* `supersetOf(Charset detected)` : retourne le surensemble associé au jeu de caractères fourni, s'il existe. Sinon, elle retourne `null`.
* `decodeAs(Charset detected)` : retourne le surensemble lorsqu'il existe, ou conserve le jeu de caractères original dans le cas contraire.

Cette classe a été choisie parce qu'elle possède déjà des tests, mais que la couverture initiale était incomplète, notamment pour la méthode `decodeAs(Charset)`.

## Génération des tests avec ChatUniTest

### Configuration

ChatUniTest a été exécuté avec Maven et un modèle local fourni par Ollama. Le modèle utilisé était `codeqwen:v1.5-chat`.

La configuration ajoutée dans `tika-core/pom.xml` était :

```xml
<plugin>
  <groupId>io.github.zju-aces-ise</groupId>
  <artifactId>chatunitest-maven-plugin</artifactId>
  <version>2.1.1</version>
  <configuration>
    <apiKeys>ollama</apiKeys>
    <model>codeqwen:v1.5-chat</model>
    <url>http://localhost:11434/v1/chat/completions</url>
  </configuration>
</plugin>
```

### Commande utilisée

La commande suivante a été utilisée pour lancer la génération :

```powershell
mvn -f tika-core/pom.xml chatunitest:class "-DselectClass=org.apache.tika.detect.CharsetSupersets" "-DtestOutput=C:\Users\sarah\Desktop\uni\IFT3913\tika\tests-generes" "-DtmpOutput=C:\Users\sarah\Desktop\uni\IFT3913\tika\chatunitest-info" "-DtestNumber=4" "-DmaxRounds=2" "-Dthread=false"
```

La configuration demandait quatre tests par méthode, avec un maximum de deux rondes de correction.

### Résultats de la génération des tests IA

Les résultats observés étaient les suivants :

* ChatUniTest a traité les méthodes `supersetOf(Charset)` et `decodeAs(Charset)`
* Des tentatives de génération et de correction ont été enregistrées dans les fichiers d'historique
* Aucun fichier de test Java utilisable n'a été retrouvé dans le dossier `tests-generes`
* Plusieurs messages d'erreur de compilation ont été conservés dans `chatunitest-info`

La génération a donc été exécutée, mais aucun test généré n'a pu être intégré directement à la suite de tests.

### Erreurs observées

Les fichiers `records.json` et les messages d'erreur ont permis d'identifier plusieurs problèmes.

**Dépendances Mockito manquantes**

Les tests générés contenaient des imports comme :

```java
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;
```

Ces classes n'étaient pas disponibles dans l'environnement de compilation utilisé par ChatUniTest. De plus, les méthodes testées étant statiques, Mockito n'était pas nécessaire pour les scénarios proposés.

**Utilisation du constructeur privé**

Certains tests contenaient :

```java
charsetSupersets = new CharsetSupersets();
```

Cette instruction ne compile pas, car le constructeur est privé. Il n'est pas nécessaire de créer une instance pour utiliser les méthodes statiques.

**Modification de `SUPERSET_MAP`**

Plusieurs tests essayaient de remplacer la variable `SUPERSET_MAP` par une nouvelle carte :

```java
CharsetSupersets.SUPERSET_MAP = Collections.unmodifiableMap(supersetMap);
```

Certains tests tentaient aussi de modifier cette variable indirectement, ce qui n'était pas approprié dans ce cas.

**Import manquant pour `Method`**

Un test utilisait la classe `Method` pour appeler une méthode par réflexion, sans importer :

```java
import java.lang.reflect.Method;
```

Cela provoquait une erreur de compilation.

**Utilisation incorrecte de jeux de caractères inconnus**

Certains tests utilisaient :

```java
Charset.forName("UNKNOWN")
```

En supposant que la méthode testée retournerait `null`. Pourtant, `Charset.forName()` peut lancer une exception avant même l'appel à `decodeAs()`.

**Corrections automatiques inefficaces**

ChatUniTest a essayé de corriger les erreurs, mais les problèmes principaux, comme le constructeur privé et la modification de `SUPERSET_MAP`, sont restés.

### Évaluation des tests générés

ChatUniTest a proposé quelques scénarios pertinents, comme les entrées null, la présence d'un surensemble et son absence. 
Cependant, plusieurs tests contenaient des erreurs de compilation ou des résultats attendus incorrects. Certains tentaient aussi de modifier la configuration interne de la classe.

## Analyse de mutation avec PITest

Le score de mutation est calculé comme :

Score de mutation = (Nombre de mutants tués / Nombre de mutants évalués) × 100 %

Les mutants non-couverts ne sont pas comptés parmi les mutants évalués dans ce calcul.

### Configuration et commande

PITest a été configuré dans `tika-core/pom.xml`, en ciblant la classe `CharsetSupersets` et sa classe de tests `CharsetSupersetsTest`.

La commande utilisée pour lancer l'analyse était :

```powershell
mvn -f tika-core/pom.xml org.pitest:pitest-maven:mutationCoverage
```

### Analyse

Avant l'ajout des tests manuels, les résultats étaient :

| Mesure                       |     Résultat |
| ---------------------------- | -----------: |
| Mutants générés              |            5 |
| Mutants tués                 |            3 |
| Mutants sans couverture      |            2 |
| Score de mutation            |         60 % |
| Couverture des lignes mutées | 15/21 (71 %) |

Les trois premiers mutants ont été tués par les tests existants. Ils concernaient des conditions et le remplacement d'une valeur de retour par `null`.

Les deux autres mutants concernaient `decodeAs(Charset)`. Ils n'étaient pas couverts par les tests exécutés lors de cette première analyse.

Ces résultats montraient qu'il était nécessaire d'ajouter des tests visant directement les comportements de cette méthode.

### Tests ajoutés

Trois tests manuels ont été ajoutés à `CharsetSupersetsTest` pour vérifier les principaux comportements de `decodeAs(Charset)` :

* le retour du surensemble lorsqu'il existe
* le retour du jeu de caractères original lorsqu'aucun surensemble n'existe
* le retour de `null` lorsque l'entrée est `null`

Après l'ajout de ces tests, PITest a été exécuté de nouveau.

Les résultats finaux étaient :

| Mesure                           | Résultat final |
| -------------------------------- | -------------: |
| Mutants générés                  |              5 |
| Mutants tués                     |              5 |
| Mutants survivants               |              0 |
| Mutants sans couverture          |              0 |
| Score de mutation                |          100 % |
| Couverture des lignes mutées     |   17/21 (81 %) |
| Tests exécutés, selon le rapport |              8 |

Les deux mutants qui n'étaient pas couverts lors de la première analyse ont été tués après l'ajout des deux premiers tests manuels. Le troisième test vérifie le comportement de la méthode lorsque l'entrée est `null`.

Le score de mutation est ainsi passé de 60 % à 100 % après l'ajout des tests manuels. L'ajout du troisième test n'a pas modifié ce score, puisque les cinq mutants étaient déjà tués. La couverture des lignes mutées est restée à 81 %.

## Tests ajoutés manuellement

Les trois tests suivants ont été ajoutés dans :

`tika-core/src/test/java/org/apache/tika/detect/CharsetSupersetsTest.java`

### Test d'un jeu de caractères possédant un surensemble

**Nom :** `decodeAsReturnsSupersetWhenAvailable`

**Objectif :** vérifier que `decodeAs()` retourne le surensemble lorsqu'il en existe un

**Donnée choisie :** `GB2312`

**Résultat attendu :** `GB18030`

**Oracle :** l'assertion compare le résultat obtenu au jeu de caractères attendu

```java
@Test
void decodeAsReturnsSupersetWhenAvailable() {
    Charset detected = Charset.forName("GB2312");
    Charset result = CharsetSupersets.decodeAs(detected);

    // On vérifie que la méthode retourne le surensemble GB18030.
    assertEquals(Charset.forName("GB18030"), result);
}
```

Ce test vérifie que la méthode retourne le surensemble attendu pour `GB2312`.

### Test d'un jeu de caractères sans surensemble

**Nom :** `decodeAsReturnsOriginalWhenNoSupersetExists`

**Objectif :** vérifier que la méthode conserve le jeu de caractères initial lorsqu'aucun surensemble n'est disponible

**Donnée choisie :** `StandardCharsets.UTF_8`

**Résultat attendu :** `UTF-8`

**Oracle :** le résultat doit être égal au jeu de caractères fourni en entrée

```java
@Test
void decodeAsReturnsOriginalWhenNoSupersetExists() {
    Charset detected = StandardCharsets.UTF_8;
    Charset result = CharsetSupersets.decodeAs(detected);

    // Sans surensemble disponible, la méthode doit retourner le jeu de caractères original
    assertEquals(detected, result);
}
```

Ce test vérifie que la méthode retourne le jeu de caractères original lorsqu'aucun surensemble n'est trouvé.

### Test avec une entrée `null`

**Nom :** `decodeAsReturnsNullWhenInputIsNull`

**Objectif :** vérifier que la méthode retourne `null` lorsque le jeu de caractères fourni en entrée est `null`

**Donnée choisie :** `null`

**Résultat attendu :** `null`

**Oracle :** l'assertion `assertNull(result)` vérifie que le résultat obtenu est `null`

```java
@Test
public void decodeAsReturnsNullWhenInputIsNull() {
    Charset detected = null;
    Charset result = CharsetSupersets.decodeAs(detected);

    // La méthode doit retourner null lorsque l'entrée est null
    assertNull(result);
}
```

Ce test vérifie un cas limite important. Comme `decodeAs()` appelle `supersetOf()` avec la valeur reçue, une entrée `null` doit entraîner un résultat `null`.

## Comparaison entre les tests générés et les tests manuels

| Critère                     | Tests générés par ChatUniTest                                           | Tests manuels                                                                |
| --------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Scénarios proposés          | Certains scénarios pertinents                                           | Trois cas vérifiés explicitement                                             |
| Compilation                 | Échec pour les tentatives examinées                                     | Réussie                                                                      |
| Exécution                   | Aucun test généré utilisable                                            | Les trois tests ont réussi                                                   |
| Assertions                  | Certaines assertions pertinentes, mais plusieurs hypothèses incorrectes | Résultats attendus explicites                                                |
| Utilisation des dépendances | Imports Mockito inutiles ou manquants                                   | Pas de dépendance Mockito nécessaire                                         |
| Résultat de mutation        | Aucun score indépendant mesuré pour des tests générés exécutables       | Les cinq mutants sont tués avec l'ensemble des tests, dont les tests manuels |

Cette comparaison montre que l'IA peut proposer des idées de scénarios utiles, mais que les tests doivent être vérifiés avant d'être utilisés. 
Dans ce cas, les erreurs de compilation et les hypothèses incorrectes ont empêché une évaluation directe des tests générés.
Ainsi, les tests manuels ont été plus faciles à valider parce qu'ils s'appuient directement sur le fonctionnement réel de la classe et sur des résultats attendus précis.

## Fichiers importants

* **Classe étudiée :** `tika-core/src/main/java/org/apache/tika/detect/CharsetSupersets.java`
* **Classe de tests :** `tika-core/src/test/java/org/apache/tika/detect/CharsetSupersetsTest.java`
* **Configuration des plugins :** `tika-core/pom.xml`
* **Sortie prévue des tests générés :** `tests-generes`
* **Historique et erreurs de ChatUniTest :** `chatunitest-info`

Welcome to Apache Tika  <https://tika.apache.org/>
=================================================

[![license](https://img.shields.io/github/license/apache/tika.svg?maxAge=2592000)](http://www.apache.org/licenses/LICENSE-2.0)
[![Jenkins](https://img.shields.io/jenkins/s/https/ci-builds.apache.org/job/Tika/job/tika-main-jdk17.svg?maxAge=3600)](https://ci-builds.apache.org/job/Tika/job/tika-main-jdk17/)
[![Jenkins tests](https://img.shields.io/jenkins/t/https/ci-builds.apache.org/job/Tika/job/tika-main-jdk17.svg?maxAge=3600)](https://ci-builds.apache.org/job/Tika/job/tika-main-jdk17/lastBuild/testReport/)
[![Maven Central](https://img.shields.io/maven-central/v/org.apache.tika/tika.svg?maxAge=86400)](http://search.maven.org/#search|ga|1|g%3A%22org.apache.tika%22)

Apache Tika(TM) is a toolkit for detecting and extracting metadata and structured text content from various documents using existing parser libraries.

Tika is a project of the [Apache Software Foundation](https://www.apache.org).

Apache Tika, Tika, Apache, the Apache feather logo, and the Apache Tika project logo are trademarks of The Apache Software Foundation.

Quick Start
===========

**Parse a file in Java:**

```java
import org.apache.tika.Tika;

Tika tika = new Tika();
String text = tika.parseToString(new File("document.pdf"));
System.out.println(text);
```

**From the command line:**

```bash
java -jar tika-app-*.jar --text document.pdf
```

**Maven dependency:**

```xml
<dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-parsers-standard-package</artifactId>
    <version>4.x.y</version>
    <type>pom</type>
</dependency>
```

Getting Started
===============
Pre-built binaries of Apache Tika standalone applications are available
from https://tika.apache.org/download.html . Pre-built binaries of all the
Tika jars can be fetched from Maven Central or your favourite Maven mirror.

**Tika 2.X and support for Java 8 reached End of Life (EOL) in April, 2025. 
See [Tika Roadmap 2.x, 3.x and beyond](https://cwiki.apache.org/confluence/display/TIKA/Tika+Roadmap+--+2.x%2C+3.x+and+Beyond).** 

Tika is based on **Java 17** and uses the [Maven 3](https://maven.apache.org) build system.
**N.B.** [Docker](https://www.docker.com/products/personal) is used for tests in tika-integration-tests. If Docker is not installed, those tests are skipped.

To build Tika from source, use the following command in the main directory:

    ./mvnw clean install

The Maven wrapper (`mvnw`) is included in the repository and will automatically download
the correct Maven version if needed. On Windows, use `mvnw.cmd` instead.

The build consists of a number of components, including a standalone runnable jar that you can use to try out Tika features. You can run it like this:

    java -jar tika-app/target/tika-app-*.jar --help


To build a specific project (for example, tika-server-standard):

    ./mvnw clean install -am -pl :tika-server-standard

If the ossindex-maven-plugin is causing the build to fail because a dependency
has now been discovered to have a vulnerability:

    ./mvnw clean install -Dossindex.skip


Faster Builds
=============

**Fast profile** - Use `-Pfast` to skip tests, checkstyle, and spotless:

    ./mvnw clean install -Pfast

**Parallel builds** - Add `-T1C` to build with 1 thread per CPU core:

    ./mvnw clean install -Pfast -T1C

**Maven Daemon (mvnd)** - Keeps a warm JVM running for 2-3x faster rebuilds:

```bash
# Install: https://github.com/apache/maven-mvnd
# macOS: brew install mvndaemon/tap/mvnd

# Use exactly like mvn
mvnd clean install -Pfast
mvnd test -pl :tika-core
```

**Combine both** for maximum speed during development:

    mvnd clean install -Pfast -T1C


Reproducible Builds
===================

Apache Tika supports [reproducible builds](https://reproducible-builds.org/). This means
that building the same source code with the same JDK version should produce
byte-for-byte identical artifacts, regardless of the build machine or time.

Key configuration:
- `project.build.outputTimestamp` is set in `tika-parent/pom.xml`
- All Maven plugins are configured to produce deterministic output

To verify the build plan supports reproducibility:

    ./mvnw artifact:check-buildplan

To verify two builds produce identical artifacts:

    ./mvnw clean install -DskipTests
    mv ~/.m2/repository/org/apache/tika tika-build-1
    ./mvnw clean install -DskipTests
    diff -r tika-build-1 ~/.m2/repository/org/apache/tika


Maven Dependencies
==================

Apache Tika provides *Bill of Material* (BOM) artifact to align Tika module versions and simplify version management. 
To avoid convergence errors in your own project, import this
bom or Tika's parent pom.xml in your dependency management section.

If you use Apache Maven:

```xml
<project>
  <dependencyManagement>
    <dependencies>
      <dependency>
       <groupId>org.apache.tika</groupId>
       <artifactId>tika-bom</artifactId>
       <version>4.x.y</version>
       <type>pom</type>
       <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.apache.tika</groupId>
      <artifactId>tika-parsers-standard-package</artifactId>
      <type>pom</type>
      <!-- version not required since BOM included -->
    </dependency>
  </dependencies>
</project>
```

For Gradle:

```kotlin
dependencies {
  implementation(platform("org.apache.tika:tika-bom:4.x.y"))

  // version not required since bom (platform in Gradle terms)
  implementation("org.apache.tika:tika-parsers-standard-package@pom")
}
```

Migrating to 4.x
================
TBD

Contributing
============
See [CONTRIBUTING.md](CONTRIBUTING.md) and https://tika.apache.org/contribute.html

[![contributors](https://contributors-img.web.app/image?repo=apache/tika)](https://github.com/apache/tika/graphs/contributors)

Building from a Specific Tag
============================
Let's assume that you want to build the 3.0.1 tag:
```
0. Download and install hub.github.com
1. git clone https://github.com/apache/tika.git
2. cd tika
3. git checkout 3.0.1
4. ./mvnw clean install
```

If a new vulnerability has been discovered between the date of the
tag and the date you are building the tag, you may need to build with:

```
4. ./mvnw clean install -Dossindex.skip
```

If a local test is not working in your environment, please notify
 the project at dev@tika.apache.org. As an immediate workaround,
 you can turn off individual tests with e.g.:

```
4. ./mvnw clean install -Dossindex.skip -Dtest=\!UnpackerResourceTest#testPDFImages
```

License (see also LICENSE.txt)
==============================

Collective work: Copyright 2011 The Apache Software Foundation.

Licensed to the Apache Software Foundation (ASF) under one or more contributor license agreements.  See the NOTICE file distributed with this work for additional information regarding copyright ownership.  The ASF licenses this file to You under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License.  You may obtain a copy of the License at

<https://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  See the License for the specific language governing permissions and limitations under the License.

Apache Tika includes a number of subcomponents with separate copyright notices and license terms. Your use of these subcomponents is subject to the terms and conditions of the licenses listed in the LICENSE.txt file.

Export Control
==============

This distribution includes cryptographic software.  The country in which you currently reside may have restrictions on the import, possession, use, and/or re-export to another country, of encryption software.  BEFORE using any encryption software, please  check your country's laws, regulations and policies concerning the import, possession, or use, and re-export of encryption software, to  see if this is permitted.  See <http://www.wassenaar.org/> for more information.

The U.S. Government Department of Commerce, Bureau of Industry and Security (BIS), has classified this software as Export Commodity Control Number (ECCN) 5D002.C.1, which includes information security software using or performing cryptographic functions with asymmetric algorithms.  The form and manner of this Apache Software Foundation distribution makes it eligible for export under the License Exception ENC Technology Software Unrestricted (TSU) exception (see the BIS Export Administration Regulations, Section 740.13) for both object code and source code.

The following provides more details on the included cryptographic software:

Apache Tika uses the Bouncy Castle generic encryption libraries for extracting text content and metadata from encrypted PDF files.  See <http://www.bouncycastle.org/> for more details on Bouncy Castle.  

Mailing Lists
=============

* user@tika.apache.org - About using Tika
* dev@tika.apache.org - About developing Tika

Subscribe by sending a message to `{list}-subscribe@tika.apache.org`.

Issue Tracker
=============

https://issues.apache.org/jira/browse/TIKA

Security
========

See [SECURITY.md](SECURITY.md) and https://tika.apache.org/security.html
