---
id: STD-<DOMAINE>
title: <titre du standard>
version: 0.1.0
status: draft            # draft | active | deprecated
owner: myc-public
applies-to: ["<glob>"]
sources: [<dépôt pilote>, <référence externe>]
---

# <Titre du standard>

## 1. Objet et portée

Ce que couvre le standard, ce qu'il ne couvre pas (renvoi vers les autres standards).

## 2. Vocabulaire

DOIT / NE DOIT PAS / DEVRAIT / NE DEVRAIT PAS / PEUT au sens de la RFC 2119 (précisée par la RFC 8174) :
seuls les mots en majuscules ont valeur normative.

## 3. Règles

### STD-<DOMAINE>-001 — <titre court> · **DOIT**

**Règle** : une phrase vérifiable.

**Pourquoi** : 1 à 3 lignes.

**Exemple** :

```text
// conforme
```

**Contre-exemple** :

```text
// non conforme
```

**Contrôle** : automatique (<outil>) | agent `<plugin>-reviewer` | revue humaine.

**Exceptions** : aucune | voir [exception-management.md](../governance/exception-management.md).

## 4. Exceptions

Procédure : [governance/exception-management.md](../governance/exception-management.md) ; exceptions actives :
[governance/compliance-matrix.md](../governance/compliance-matrix.md).

## 5. Références

## 6. Historique

| Version | Date | Changement |
|---|---|---|
| 0.1.0 | AAAA-MM-JJ | Création |
