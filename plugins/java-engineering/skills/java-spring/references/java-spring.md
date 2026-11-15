---
id: STD-JAVA
title: Développement Java / Spring Boot
version: 0.1.0
status: draft
owner: myc-public
applies-to: ["**/*.java", "**/pom.xml"]
sources: [inner-donation-api, Google Java Style Guide, Spring Boot reference, RFC 9457]
---

# Développement Java / Spring Boot

## 1. Objet et portée

Règles de code des services Java 21 / Spring Boot du socle : couches, injection, DTO, erreurs, transactions,
sécurité, persistance, configuration, logs, tests, style.

Hors portée : architecture (DDD, clean architecture), conception REST, événements et EDA : standards dédiés.
Les règles propres à un dépôt (classes, permissions, identifiants de test) restent dans son `CLAUDE.md`.

## 2. Vocabulaire

DOIT / NE DOIT PAS / DEVRAIT / NE DEVRAIT PAS / PEUT au sens de la RFC 2119 (précisée par la RFC 8174) :
seuls les mots en majuscules ont valeur normative.

## 3. Règles

### 3.1 Structure et injection

### STD-JAVA-001 — Couches api, service, repository · **DOIT**

**Règle** : un controller délègue à un service et ne contient ni logique métier ni accès à un repository.

**Pourquoi** : la logique métier reste testable sans HTTP, et la transaction et l'autorisation ont un seul lieu.

**Exemple** :

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
@PreAuthorize(Permissions.DONATION_CREATE)
public DonationResponse create(@Valid @RequestBody CreateDonationRequest request) {
    return donationService.create(request);
}
```

**Contre-exemple** :

```java
@PostMapping
public DonationBO create(@RequestBody CreateDonationRequest request) {
    if (!donorRepository.existsById(request.donor().donorId())) { ... }   // règle métier dans le controller
    return donationRepository.save(mapper.toBo(request));                  // repository appelé directement
}
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### STD-JAVA-002 — Injection par constructeur · **DOIT**

**Règle** : les dépendances sont injectées par le constructeur dans des champs `private final` ; jamais
d'`@Autowired` sur un champ ou un setter.

**Pourquoi** : dépendances explicites, objets immuables, tests unitaires sans conteneur Spring.

**Exemple** :

```java
private final DonationRepository donationRepository;

public DonationServiceImpl(DonationRepository donationRepository) {
    this.donationRepository = donationRepository;
}
```

**Contre-exemple** :

```java
@Autowired
private DonationRepository donationRepository;
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### 3.2 API et DTO

### STD-JAVA-003 — Pas d'entité JPA hors du service · **DOIT**

**Règle** : un controller reçoit et renvoie des DTO (`record`) ; une entité JPA n'est jamais sérialisée dans une
réponse ni désérialisée depuis une requête.

**Pourquoi** : le contrat d'API ne dépend pas du schéma, aucun champ interne ne fuit, pas de chargement paresseux
déclenché par la sérialisation.

**Exemple** :

```java
public record DonationResponse(UUID donationId, UUID donorId, DonationCategory category, BigDecimal amount) {}
```

**Contre-exemple** :

```java
@GetMapping("/{id}")
public DonationBO get(@PathVariable UUID id) { ... }   // entité exposée
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### STD-JAVA-004 — Entrées validées par Bean Validation · **DOIT**

**Règle** : chaque corps de requête est annoté `@Valid` et ses contraintes sont déclarées sur le `record`
(`@NotNull`, `@DecimalMin`, `@Valid` sur les objets imbriqués).

**Pourquoi** : une entrée invalide est refusée en 400 avant d'atteindre le service, avec un message par champ.

**Exemple** :

```java
public record CreateDonationRequest(
        @NotNull DonationCategory category,
        @NotNull @DecimalMin(value = "0.01") BigDecimal amount,
        @NotNull @Valid DonorSnapshot donor) {}
```

**Contre-exemple** :

```java
if (request.amount() == null || request.amount().signum() <= 0) {
    throw new IllegalArgumentException("amount");   // validation à la main, dans le service
}
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### STD-JAVA-005 — Conversion par MapStruct · **DEVRAIT**

**Règle** : la conversion entre entité et DTO est faite par une interface MapStruct
(`@Mapper(componentModel = "spring")`), pas par du code de copie dispersé.

**Pourquoi** : conversions générées à la compilation, un champ oublié devient une erreur de build.

**Exemple** :

```java
@Mapper(componentModel = "spring")
public interface DonationMapper {
    @Mapping(target = "donationId", source = "id")
    DonationResponse toResponse(DonationBO bo);
}
```

**Contre-exemple** :

```java
return new DonationResponse(bo.getId(), bo.getDonorId(), bo.getCategory(), bo.getAmount());   // dans chaque méthode
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : DTO de un ou deux champs.

### 3.3 Erreurs

### STD-JAVA-006 — Un seul traducteur d'erreurs, au format RFC 9457 · **DOIT**

**Règle** : les exceptions sont traduites en réponses HTTP par un seul `@RestControllerAdvice`, au format
problem details (`type`, `title`, `status`, `detail`, `instance`) complété par `traceId` ; aucune stack trace ni
message technique dans la réponse.

**Pourquoi** : un format unique pour les clients, aucune fuite d'information interne, corrélation avec les traces.

**Exemple** :

```java
@ExceptionHandler({ TechnicalException.class, RuntimeException.class })
@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
ApiError onTechnicalException(Exception e, HttpServletRequest request) {
    logger.error(e.getMessage(), e);                       // détail dans les logs
    return ApiError.builder().status(500).traceId(traceId) // message générique dans la réponse
            ...build();
}
```

**Contre-exemple** :

```java
catch (Exception e) {
    return ResponseEntity.status(500).body(e.getMessage());   // try/catch dans un controller, message brut
}
```

**Contrôle** : test (un test par famille d'exception de l'advice).

**Exceptions** : aucune.

### STD-JAVA-007 — Même 404 pour une ressource absente ou inaccessible · **DOIT**

**Règle** : une ressource qui n'existe pas et une ressource qui appartient à un autre appelant renvoient la même
réponse 404.

**Pourquoi** : un 403 révélerait l'existence de la ressource (énumération d'identifiants).

**Exemple** :

```java
return donationRepository.findByIdAndDonorId(donationId, donorId)
        .map(donationMapper::toResponse)
        .orElseThrow(() -> new NotFoundException("Donation not found: " + donationId));
```

**Contre-exemple** :

```java
DonationBO d = findDonation(donationId);
if (!d.getDonorId().equals(donorId)) throw new AccessDeniedException("not yours");   // 403 : existence révélée
```

**Contrôle** : test (accès à la ressource d'un autre appelant -> 404).

**Exceptions** : aucune.

### 3.4 Transactions

### STD-JAVA-008 — Transactions dans le service · **DOIT**

**Règle** : les frontières transactionnelles sont déclarées sur les services (`@Transactional` sur la classe ou la
méthode), jamais sur un controller ni un repository appelé directement.

**Pourquoi** : une opération métier, une transaction ; toutes ses écritures sont commitées ensemble ou pas du tout.

**Exemple** :

```java
@Service
@Transactional
public class DonationServiceImpl implements DonationService { ... }
```

**Contre-exemple** :

```java
@RestController
@Transactional
public class DonationController { ... }
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### STD-JAVA-009 — `@Transactional` de Spring, lectures en `readOnly` · **DEVRAIT**

**Règle** : utiliser `org.springframework.transaction.annotation.Transactional` ; les méthodes de lecture portent
`readOnly = true`.

**Pourquoi** : seule l'annotation de Spring porte `readOnly`, `propagation` et `rollbackFor` ; `readOnly` évite le
flush et peut router vers un réplica.

**Exemple** :

```java
import org.springframework.transaction.annotation.Transactional;

@Transactional(readOnly = true)
public DonationResponse get(UUID donationId) { ... }
```

**Contre-exemple** :

```java
import jakarta.transaction.Transactional;   // pas de readOnly possible
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### STD-JAVA-010 — Aucun appel externe dans une transaction métier · **DOIT**

**Règle** : une transaction métier n'appelle ni réseau ni broker ; un effet externe (événement, notification) est
écrit dans une table outbox dans la même transaction et publié ensuite.

**Pourquoi** : pas d'événement sans écriture ni d'écriture sans événement, pas de transaction bloquée par un tiers.

**Exemple** :

```java
DonationBO saved = donationRepository.save(donation);
outboxEventRepository.save(outboxFactory.newEvent(envelope));   // même transaction
```

**Contre-exemple** :

```java
DonationBO saved = donationRepository.save(donation);
kafkaTemplate.send("donations", saved);   // appel au broker dans la transaction
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### 3.5 Sécurité

### STD-JAVA-011 — Autorisation explicite sur chaque endpoint · **DOIT**

**Règle** : chaque méthode d'un `@RestController` porte un `@PreAuthorize` qui référence une constante d'une classe
de permissions ; jamais d'expression littérale dispersée.

**Pourquoi** : aucun endpoint n'est ouvert par oubli ; les permissions sont listées en un seul endroit.

**Exemple** :

```java
@PreAuthorize(Permissions.DONATION_READ)
public DonationResponse get(@PathVariable UUID donationId) { ... }
```

**Contre-exemple** :

```java
@PreAuthorize("hasAuthority('donation:read')")   // littéral
public DonationResponse get(...) { ... }

public List<DonationResponse> list() { ... }     // aucun contrôle
```

**Contrôle** : test qui parcourt les controllers et échoue sur un endpoint sans `@PreAuthorize`.

**Exceptions** : endpoints publics déclarés explicitement dans la configuration de sécurité.

### STD-JAVA-012 — Identité tirée du jeton · **DOIT**

**Règle** : l'identité de l'appelant (identifiant de personne, de compte) est lue dans le jeton validé, jamais dans
un paramètre, un en-tête ou un corps de requête.

**Pourquoi** : un paramètre est contrôlé par le client ; s'y fier permet d'agir au nom d'un autre.

**Exemple** :

```java
@GetMapping("/me")
public DonorResponse me() {
    return donorService.get(currentParty.id());   // party_id du jeton
}
```

**Contre-exemple** :

```java
@GetMapping("/me")
public DonorResponse me(@RequestParam UUID donorId) { ... }
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### 3.6 Persistance

### STD-JAVA-013 — Schéma géré par Flyway · **DOIT**

**Règle** : le schéma évolue par une nouvelle migration `V<n>__<description>.sql` ; une migration déjà appliquée
n'est jamais modifiée ; Hibernate est en `ddl-auto: validate`.

**Pourquoi** : le schéma est reproductible et identique dans tous les environnements ; Flyway refuse une migration
dont la somme de contrôle a changé.

**Exemple** :

```text
db/migration/V6__donation_add_currency.sql   (nouveau fichier)
spring.jpa.hibernate.ddl-auto: validate
```

**Contre-exemple** :

```text
db/migration/V2__init.sql modifié après déploiement
spring.jpa.hibernate.ddl-auto: update
```

**Contrôle** : hook (ESM3) refusant la modification d'une migration existante.

**Exceptions** : aucune.

### STD-JAVA-014 — Listes paginées · **DEVRAIT**

**Règle** : un endpoint qui renvoie une collection non bornée accepte un `Pageable` et renvoie une page.

**Pourquoi** : temps de réponse et mémoire bornés quand le volume croît.

**Exemple** :

```java
public Page<DonationResponse> list(Pageable pageable) {
    return donationRepository.findAll(pageable).map(donationMapper::toResponse);
}
```

**Contre-exemple** :

```java
return donationRepository.findAll().stream().map(donationMapper::toResponse).toList();
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : collections bornées par construction (énumérations, référentiels de quelques lignes).

### 3.7 Configuration et logs

### STD-JAVA-015 — Configuration typée · **DEVRAIT**

**Règle** : la configuration applicative est lue par des classes `@ConfigurationProperties` sous un préfixe dédié,
pas par des `@Value` dispersés.

**Pourquoi** : configuration validée au démarrage, documentée par le processeur de configuration, testable.

**Exemple** :

```java
@ConfigurationProperties(prefix = "myc.security")
public record MycSecurityProps(List<String> issuers, String audience) {}
```

**Contre-exemple** :

```java
public DonationServiceImpl(..., @Value("${spring.application.name:donation-service}") String producerName)
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### STD-JAVA-016 — Logs paramétrés et sans données sensibles · **DOIT**

**Règle** : les logs passent par SLF4J avec des paramètres `{}` au format `clé=valeur` ; ils ne contiennent ni
donnée personnelle (nom, e-mail, téléphone) ni secret (jeton, mot de passe).

**Pourquoi** : logs exploitables par requête, pas de concaténation inutile, conformité RGPD.

**Exemple** :

```java
log.info("Create donation donorId={} category={}", request.donor().donorId(), request.category());
```

**Contre-exemple** :

```java
log.info("Create donation for " + donor.getEmail() + " token=" + jwt.getTokenValue());
```

**Contrôle** : agent `java-reviewer`.

**Exceptions** : aucune.

### 3.8 Tests

### STD-JAVA-017 — Tests unitaires sans contexte Spring · **DOIT**

**Règle** : la logique des services est testée avec JUnit 5, Mockito (`@ExtendWith(MockitoExtension.class)`) et
AssertJ, sans démarrer de contexte Spring ; `mvn verify` passe sans Docker.

**Pourquoi** : tests rapides, déterministes, exécutables partout, y compris en CI sans conteneur.

**Exemple** :

```java
@ExtendWith(MockitoExtension.class)
class DonationServiceImplTest {
    @Mock private DonationRepository donationRepository;
    ...
    assertThatThrownBy(() -> service.get(id)).isInstanceOf(NotFoundException.class);
}
```

**Contre-exemple** :

```java
@SpringBootTest   // contexte complet pour tester une règle métier
class DonationServiceImplTest { ... }
```

**Contrôle** : build (`mvn verify`).

**Exceptions** : tests d'intégration, séparés et nommés `*IT`.

### STD-JAVA-018 — Endpoints testés en 401 et 403 · **DOIT**

**Règle** : chaque endpoint protégé a un test qui vérifie 401 sans jeton et 403 sans la permission requise.

**Pourquoi** : la configuration de sécurité est du code ; une régression ouvre l'API.

**Exemple** :

```java
mockMvc.perform(get("/api/v1/donations/{id}", id)).andExpect(status().isUnauthorized());
mockMvc.perform(get("/api/v1/donations/{id}", id).with(jwt())).andExpect(status().isForbidden());
```

**Contre-exemple** : seuls les cas passants (200, 201) sont testés.

**Contrôle** : test.

**Exceptions** : endpoints publics.

### STD-JAVA-019 — Contrats d'événements vérifiés par test · **DOIT**

**Règle** : chaque événement publié a un JSON Schema versionné dans le dépôt, et un test vérifie que l'événement
produit par le code le respecte.

**Pourquoi** : un consommateur n'est pas cassé par un champ renommé ou retiré sans changement de version.

**Exemple** : `src/main/resources/events/*.schema.json`, vérifiés par `EventSchemasTest`.

**Contre-exemple** : payload construit en `Map<String, Object>` sans schéma.

**Contrôle** : test.

**Exceptions** : services sans événement.

### 3.9 Style

### STD-JAVA-020 — Exceptions non vérifiées et style homogène · **DEVRAIT**

**Règle** : les exceptions métier héritent de `RuntimeException` et ne portent pas `@Data` ; le code suit le Google
Java Style Guide (pas d'import `*`, indentation homogène dans un fichier).

**Pourquoi** : pas de `throws` propagé dans toutes les signatures ; égalité et `toString` d'exception non générés ;
diffs lisibles.

**Exemple** :

```java
public class NotFoundException extends RuntimeException {
    public NotFoundException(String message) { super(message); }
}
```

**Contre-exemple** :

```java
@Data
public class BusinessException extends Exception { ... }
import org.springframework.web.bind.annotation.*;
```

**Contrôle** : hook de formatage (ESM3).

**Exceptions** : aucune.

## 4. Exceptions

Procédure : [governance/exception-management.md](../../governance/exception-management.md) ; exceptions actives :
[governance/compliance-matrix.md](../../governance/compliance-matrix.md).

## 5. Références

- Google Java Style Guide : https://google.github.io/styleguide/javaguide.html
- Spring Boot reference documentation : https://docs.spring.io/spring-boot/
- RFC 9457, Problem Details for HTTP APIs : https://www.rfc-editor.org/rfc/rfc9457
- RFC 2119 et RFC 8174 : vocabulaire normatif
- Pilote : `inner-donation-api` (`docs/architecture.md`)

## 6. Historique

| Version | Date | Changement |
|---|---|---|
| 0.1.0 | 2026-11-15 | Création : 20 règles tirées d'`inner-donation-api` |
