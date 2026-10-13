# IFT3913 - Tâche 2

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

La commande utilisait le paramètre `-DtestNumber=4` et limitait le nombre de rondes de correction à deux avec `-DmaxRounds=2`. L’exécution a rencontré certaines difficultés, notamment des erreurs de compilation dans les tests générés, des problèmes liés aux dépendances et des limites de mémoire qui ont compliqué la génération et la correction des tests.

### Résultats de la génération des tests IA

Les résultats observés étaient les suivants :

* ChatUniTest a tenté de générer des tests pour les méthodes `supersetOf(Charset)` et `decodeAs(Charset)`.
* Des tentatives de génération et de correction ont été enregistrées dans les fichiers d'historique.
* Aucun fichier de test Java utilisable n'a été créé dans le dossier `tests-generes`.
* Des extraits de tests générés ainsi que des messages d'erreur ont été retrouvés dans les fichiers `error-message`, situés dans le dossier `chatunitest-info`.

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

### Analyse des oracles des tests générés

Les tests générés par ChatUniTest contenaient plusieurs oracles, principalement sous la forme d'assertions `assertEquals()` et `assertNull()`. Ces assertions définissaient les résultats attendus pour les méthodes testées.

Certains oracles correspondaient à des scénarios pertinents. Par exemple, plusieurs tests vérifiaient que `decodeAs()` retourne le surensemble attendu ou `null` lorsque l'entrée est `null`. Un autre test utilisait plusieurs jeux de caractères pour vérifier les résultats de `supersetOf()`.

Cependant, certains résultats proposés devaient être vérifiés par rapport au fonctionnement réel de la classe. De plus, plusieurs tests tentaient de modifier `SUPERSET_MAP`, dont la déclaration est `static final`, ou utilisaient des dépendances absentes de l'environnement de compilation.

Ainsi, même lorsque les assertions étaient pertinentes, les erreurs de compilation empêchaient d'exécuter les tests et de vérifier leurs résultats. Il n'a donc pas été possible de confirmer expérimentalement la validité des oracles générés.

## Analyse de mutation avec PITest

Le score de mutation est calculé comme :

Score de mutation = (Nombre de mutants tués / Nombre total de mutants générés) × 100 %

Un mutant est tué lorsqu'un test détecte la modification introduite. Un mutant non couvert n'est pas tué et contribue donc à diminuer le score de mutation.

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
| Couverture des lignes        | 15/21 (71 %) |

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
| Couverture des lignes            |   17/21 (81 %) |
| Tests exécutés, selon le rapport |              8 |

Les deux mutants qui n'étaient pas couverts lors de la première analyse ont été tués après l'ajout des deux premiers tests manuels. Le troisième test vérifie le comportement de la méthode lorsque l'entrée est `null`.

Le score de mutation est ainsi passé de 60 % à 100 % après l'ajout des tests manuels. L'ajout du troisième test n'a pas modifié ce score, puisque les cinq mutants étaient déjà tués. La couverture des lignesest passée de 71 % à 81 %.

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

| Critère | Tests générés par ChatUniTest | Tests manuels |
|---|---|---|
| Scénarios proposés | Certains scénarios pertinents, notamment le traitement d'une entrée `null` et la recherche d'un surensemble. | Trois scénarios ciblés : un surensemble disponible, l'absence de surensemble et une entrée `null`. |
| Oracles | Plusieurs assertions pertinentes, comme `assertEquals()` et `assertNull()`. Certains résultats attendus devaient toutefois être vérifiés par rapport au comportement réel de la classe. | Résultats attendus définis explicitement et vérifiés par l'exécution des tests. |
| Compilation | Échec pour les tests examinés, notamment à cause de dépendances manquantes et de tentatives de modification de `SUPERSET_MAP`. | Réussie. |
| Exécution | Aucun test généré exploitable n'a pu être exécuté pour confirmer les résultats attendus. | Les trois tests ont réussi. |
| Utilisation des dépendances | Imports Mockito manquants ou inutiles et problèmes liés à certaines fonctionnalités de JUnit. | Aucune dépendance Mockito nécessaire. |
| Résultat de mutation | Aucun score de mutation indépendant n'a pu être mesuré pour les tests générés, puisqu'ils n'étaient pas exécutables. | Avec l'ensemble des tests exécutés, les cinq mutants ont été tués, pour un score de mutation de 100 %. |

Cette comparaison montre que l'IA peut proposer des idées de scénarios utiles, mais que les tests doivent être vérifiés avant d'être utilisés. Dans ce cas, les erreurs de compilation ont empêché l'exécution des tests et la validation de leurs oracles. Les tests manuels ont pu être exécutés avec succès et leurs résultats attendus ont été vérifiés, ce qui souligne l'importance de vérifier les oracles, la compilation des tests et leur exécution.

## Fichiers

* **Classe étudiée :** `tika-core/src/main/java/org/apache/tika/detect/CharsetSupersets.java`
* **Classe de tests :** `tika-core/src/test/java/org/apache/tika/detect/CharsetSupersetsTest.java`
* **Configuration des plugins :** `tika-core/pom.xml`
* **Historique, extraits de tests générés et erreurs de ChatUniTest :** `error-message` et `records.json` dans `chatunitest-info`

## Déclaration de l'usage d'IA

* L'utilisation de ChatUniTest (via Ollama) : modèle `codeqwen:v1.5-chat` a permis de générer de nouveaux tests, comme demandé dans l'énoncé.
* L'utilisation de ChatGPT, en octobre 2026 : comprendre les erreurs rencontrées avec PITest, Maven et ChatUniTest, et d'interpréter les résultats de l'analyse de mutation.
