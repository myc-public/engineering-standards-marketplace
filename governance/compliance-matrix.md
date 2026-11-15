# Matrice de conformité

Écarts connus entre un dépôt et un standard. Une ligne par couple dépôt / règle ; un écart ouvert est suivi par une
dette du [registre du socle](https://github.com/myc-public/platform-handbook/blob/develop/dettes.md).
Statuts : `écart` (à corriger), `exception` (acceptée, voir [exception-management.md](exception-management.md)),
`corrigé` (avec la preuve).

## inner-donation-api (STD-JAVA 0.1.0, relevé du 2026-11-15)

| Règle | Niveau | Statut | Constat | Dette |
|---|---|---|---|---|
| STD-JAVA-009 | DEVRAIT | écart | `jakarta.transaction.Transactional` dans `DonationServiceImpl` et `DonorServiceImpl` ; aucune lecture en `readOnly` | 31 |
| STD-JAVA-014 | DEVRAIT | écart | Listes non paginées : `DonationController.list`, `listByDonor`, `MyDonationController.list` | 31 |
| STD-JAVA-015 | DEVRAIT | écart | `@Value` dans `DonationServiceImpl`, `DonorServiceImpl` (`spring.application.name`) et `OpenApiConfig` (`myc.docs.version`) | 31 |
| STD-JAVA-020 | DEVRAIT | écart | `BusinessException extends Exception` avec `@Data` ; imports `*` dans 12 fichiers (`api`, `domain/bo`, `mapper`, `outbox`, `audit`, `service`) ; tabulations dans `ErrorHandlingAdvice`, espaces ailleurs | 31 |
