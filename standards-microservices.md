# Référentiel des standards microservices

> Statut : POC, version du 2026-11-15.
> Périmètre : microservices Java / Spring Boot, API REST, événements Kafka.
> Principe : chaque standard a une source officielle et au moins un capteur déterministe en CI.

## 1. Tableau des standards

| # | Domaine | Standard à adopter | Où le trouver | Outil déterministe (capteur) |
|---|---|---|---|---|
| 1 | Socle | **Spring Boot 4.1.x / Spring Framework 7** | [Release notes 4.0](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Release-Notes), [Migration guide](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-4.0-Migration-Guide) | BOM `spring-boot-dependencies`, Maven Enforcer (`requireJavaVersion`, `banDuplicatePomDependencyVersions`), recettes OpenRewrite de migration Boot 4 |
| 1b | Socle | **JSpecify** (null-safety) | [jspecify.dev](https://jspecify.dev) | Error Prone + NullAway, la compilation échoue |
| 2 | Code | **Effective Java, 3e édition** (J. Bloch) | Livre, Addison-Wesley | Error Prone (règles en `ERROR`), Quality Gate SonarQube, Checkstyle / Spotless pour le formatage |
| 3 | API REST | **Zalando RESTful API Guidelines, partie REST uniquement** (chapitres Events exclus) + **OpenAPI 3.1** | [Zalando guidelines](https://opensource.zalando.com/restful-api-guidelines/), [Spécification OpenAPI](https://spec.openapis.org/oas/latest.html) | Zally ou ruleset Spectral Zalando sur les contrats OpenAPI. openapi-generator (`interfaceOnly`) : le build échoue si le code dérive du contrat |
| 3b | API REST | **RFC 9457 Problem Details** | [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457), [Doc Spring Error Responses](https://docs.spring.io/spring/reference/web/webmvc/mvc-ann-rest-exceptions.html) | `spring.mvc.problemdetails.enabled=true` + test d'intégration sur `application/problem+json` |
| 4 | Sécurité | **OWASP API Security Top 10 2023** | [owasp.org/API-Security](https://owasp.org/API-Security/editions/2023/en/0x00-header/) | Spectral OWASP ruleset (contrat), OWASP ZAP (DAST), Trivy ou Dependency-Check (dépendances), tests d'autorisation par endpoint |
| 5 | Observabilité | **OpenTelemetry** via `spring-boot-starter-opentelemetry` | [Blog Spring](https://spring.io/blog/2025/11/18/opentelemetry-with-spring-boot/), [Semantic conventions](https://opentelemetry.io/docs/specs/semconv/) | Enforcer : starter officiel présent, starter communautaire interdit. Test au démarrage qui vérifie `service.name`. ArchUnit : pas de `System.out` |
| 6 | Résilience | **Spring Framework 7 natif** (`@Retryable`, `@ConcurrencyLimit`) + **Resilience4j** pour circuit breaker et rate limiter | [Doc Spring Resilience](https://docs.spring.io/spring-framework/reference/core/resilience.html), [Resilience4j](https://resilience4j.readme.io) | ArchUnit : tout client HTTP sortant porte une politique de résilience. Enforcer : `spring-retry` interdit. Timeouts obligatoires en configuration |
| 7 | Tests | **JUnit 6 + Testcontainers 2.x** (`@ServiceConnection`) | [Doc Boot Testcontainers](https://docs.spring.io/spring-boot/reference/testing/testcontainers.html), [testcontainers.com](https://testcontainers.com) | JaCoCo (seuil de couverture), PIT (mutation testing), Enforcer : JUnit 4 banni |
| 7b | Tests | **Spring Cloud Contract** (HTTP + messaging) | [Spring Cloud Contract](https://spring.io/projects/spring-cloud-contract) | Plugin Maven qui génère les tests fournisseur. Stubs publiés dans Artifactory. Build fournisseur rouge si un contrat casse |
| 8 | EDA | **AsyncAPI 3.0** | [Spécification AsyncAPI 3.0](https://www.asyncapi.com/docs/reference/specification/v3.0.0) | Springwolf (génère le document depuis le code), Spectral AsyncAPI ruleset, diff de contrat en CI |
| 8b | EDA | **CloudEvents 1.0, Kafka Protocol Binding en mode binaire** | [cloudevents.io](https://cloudevents.io), [Binding Kafka](https://github.com/cloudevents/spec/blob/main/cloudevents/bindings/kafka-protocol-binding.md), [SDK Java](https://github.com/cloudevents/sdk-java) | Contrat Spring Cloud Contract qui exige les headers `ce_*` (`ce_id`, `ce_source`, `ce_type`, `ce_specversion`) |
| 8c | EDA | **Avro** + **Apicurio Registry 3.x** (stockage KafkaSQL en POC) | [Apache Avro](https://avro.apache.org/docs/), [Apicurio Registry](https://www.apicur.io/registry/) | `avro-maven-plugin` : le build échoue si un `.avsc` est invalide. Règle de compatibilité `BACKWARD` dans Apicurio. Auto-registration désactivée en prod |

## 2. Décisions prises

| Sujet | Décision | Raison |
|---|---|---|
| Version du socle | Spring Boot 4.1.x / Spring Framework 7 | Branche avec le support OSS le plus long |
| Clean Code / SOLID | Effective Java comme référence | Plus concret et plus spécifique à Java que Clean Code |
| DDD | Aucun standard pour l'instant | Hors périmètre de l'itération |
| Guide REST | Zalando, partie REST uniquement | Référence communautaire, linter disponible |
| Enveloppe d'événement | CloudEvents (et non l'enveloppe Zalando) | Standard CNCF, binding Kafka officiel |
| Mode CloudEvents | Binaire (attributs dans les headers Kafka) | Le payload reste l'objet métier, compatible avec le Schema Registry |
| Format de schéma Kafka | Avro dès le POC | Moins de CPU et de taille que JSON. Évite une migration de sérialiseur avant la prod |
| Schema Registry | Apicurio 3.x, un seul pod, stockage KafkaSQL, sans UI | Pas de Confluent. Aucune infra en plus sur un cluster limité en ressources |
| Contract testing | Spring Cloud Contract | Reste dans l'écosystème Spring et Artifactory |

## 3. Règles à écrire dans les guides

- **Exception Zalando** : les chapitres Events des guidelines Zalando ne s'appliquent pas. Les événements suivent CloudEvents.
- **Java moderne** : Effective Java (2018, Java 9) ne couvre ni records, ni sealed classes, ni pattern matching, ni threads virtuels. Une annexe dédiée est nécessaire.
- **`@ConcurrencyLimit`** : la limite s'applique par JVM, pas par cluster. 10 pods à `@ConcurrencyLimit(80)` peuvent envoyer 800 requêtes simultanées.
- **Testcontainers 2.0** : tous les modules prennent le préfixe `testcontainers-`. Les déclarations de dépendances changent.
- **Observabilité** : ne jamais mélanger le starter OpenTelemetry officiel Spring et le starter communautaire.
- **Évolution Avro** : une évolution compatible = un nouveau champ avec une valeur par défaut.
- **CloudEvents + Avro** : `ce_datacontenttype=application/avro`, et `ce_dataschema` pointe vers l'artefact Apicurio.
- **OWASP API1 (BOLA)** : aucun outil ne le détecte de façon fiable. Des tests d'autorisation par endpoint sont obligatoires.

## 4. Points ouverts

| # | Sujet | Action |
|---|---|---|
| 1 | Spring Cloud Contract messaging avec payloads Avro | Vérifier si c'est pris en charge nativement ou s'il faut un convertisseur maison |
| 2 | Ressources consommées par Apicurio | Mesurer CPU et RAM sur le cluster POC (pas de chiffre officiel trouvé) |
| 3 | Lecture des messages Avro | Choisir un outil Kafka qui désérialise via Apicurio |
| 4 | Stockage Apicurio en prod | Trancher entre KafkaSQL et PostgreSQL |
