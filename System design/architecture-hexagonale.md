# Architecture Hexagonale (Ports & Adapters)

> Document de référence, construit à partir d'un exemple filé : la règle métier "un étudiant ne peut pas avoir deux inscriptions différentes dans deux filières".

## Sommaire

1. [Origine et définition](#1-origine-et-définition)
2. [Règle métier vs détail technique](#2-règle-métier-vs-détail-technique)
3. [Les ports](#3-les-ports)
4. [Les adapters](#4-les-adapters)
5. [La règle de dépendance](#5-la-règle-de-dépendance)
6. [Conséquence sur les tests](#6-conséquence-sur-les-tests)
7. [Port sortant vs port entrant](#7-port-sortant-vs-port-entrant)
8. [Pourquoi "hexagonale" ?](#8-pourquoi-hexagonale-)
9. [Limites du pattern](#9-limites-du-pattern)
10. [Structure de dossiers](#10-structure-de-dossiers)
11. [Vocabulaire : "Model" vs "objet du domaine"](#11-vocabulaire--model-vs-objet-du-domaine)
12. [Exemple de code](#12-exemple-de-code)
13. [Comparaison avec d'autres architectures](#13-comparaison-avec-dautres-architectures)
14. [Critères de choix](#14-critères-de-choix)
15. [Sources](#15-sources)

---

## 1. Origine et définition

L'architecture hexagonale, aussi appelée *Ports and Adapters*, a été formalisée par Alistair Cockburn dans un article publié en 2005 (source en section 15). Le pattern propose une organisation du code dans laquelle la logique métier est isolée des détails techniques (base de données, framework web, API externes), via des interfaces intermédiaires appelées **ports**, implémentées par des classes appelées **adapters**.

Cette définition est structurelle : elle décrit une contrainte de dépendance entre les composants (section 5), pas un jugement de valeur sur la qualité du code qui en résulte.

---

## 2. Règle métier vs détail technique

On appelle ici **règle métier** une règle dont l'énoncé ne mentionne aucune technologie. Exemple utilisé dans ce document :

> "Un étudiant ne peut pas avoir deux inscriptions différentes dans deux filières."

Cet énoncé ne change pas selon que les données sont stockées en PostgreSQL ou en fichier plat, ni selon que l'application est exposée en REST ou en ligne de commande. C'est ce critère — l'énoncé reste-t-il identique si on change la technologie ? — qui permet de distinguer une règle métier d'un détail d'implémentation.

Dans une base de code, rien n'empêche qu'une règle métier soit écrite dans une classe qui contient aussi du code technique (par exemple un `Service` qui appelle directement un client JPA). L'architecture hexagonale propose une organisation où ce mélange n'apparaît pas, sans que cela garantisse que le mélange ne se produira jamais (voir section 9).

---

## 3. Les ports

Un port est une interface qui répond à deux critères cumulatifs :

1. Elle est définie dans le code du domaine métier (pas dans le code technique).
2. Son vocabulaire (noms de méthodes, types de paramètres et de retour) ne fait référence à aucune technologie.

Une interface peut être syntaxiquement une interface Java sans remplir ces deux critères. Exemple qui remplit le critère 1 mais pas le critère 2 :

```java
public interface InscriptionRepository {
    List<InscriptionEntity> findByEtudiantId(Long id); // InscriptionEntity est un type JPA
    void save(InscriptionEntity entity);
}
```

Exemple qui remplit les deux critères :

```java
public interface InscriptionRepository {
    boolean etudiantDejaInscrit(Long etudiantId);
    void enregistrerInscription(Inscription inscription);
}
```

Le port spécifie un comportement attendu ("vérifier si un étudiant est déjà inscrit"), sans préciser comment ce comportement est réalisé.

---

## 4. Les adapters

Un adapter est une classe qui implémente un port en utilisant une technologie précise.

```java
public class InscriptionRepositoryJpaAdapter implements InscriptionRepository {

    private final SpringDataInscriptionRepository jpaRepo;

    @Override
    public boolean etudiantDejaInscrit(Long etudiantId) {
        return jpaRepo.countByEtudiantId(etudiantId) > 0;
    }

    @Override
    public void enregistrerInscription(Inscription inscription) {
        jpaRepo.save(toEntity(inscription));
    }

    private InscriptionEntity toEntity(Inscription inscription) {
        // conversion entre l'objet du domaine et l'entity JPA
        ...
    }
}
```

Par construction (`implements InscriptionRepository`), c'est l'adapter qui référence le port dans son code, et non l'inverse. Le code du domaine ne contient aucune référence à `InscriptionRepositoryJpaAdapter`.

---

## 5. La règle de dépendance

Le pattern impose que, dans le graphe de dépendances du code source, aucune classe du domaine ne dépende d'une classe technique. Formulé en packages :

- `domain/` ne contient aucun `import` provenant de `infrastructure/`.
- `infrastructure/` contient des `import` provenant de `domain/` (nécessaire pour implémenter les ports).

Cette contrainte peut être vérifiée mécaniquement avec un outil d'analyse statique comme ArchUnit :

```java
@Test
void domain_ne_doit_jamais_dependre_de_infrastructure() {
    noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat().resideInAPackage("..infrastructure..")
        .check(importedClasses);
}
```

Sans un tel test, la contrainte n'est pas imposée par le langage Java lui-même : rien n'empêche syntaxiquement un développeur d'ajouter un import technique dans `domain/`.

---

## 6. Conséquence sur les tests

Si `InscriptionService` dépend du port `InscriptionRepository` (une interface) plutôt que d'une classe concrète, il devient possible d'instancier `InscriptionService` avec une implémentation en mémoire, sans base de données :

```java
@Test
void refuse_double_inscription() {
    InscriptionRepository fauxRepo = new InscriptionRepositoryEnMemoire();
    InscriptionService service = new InscriptionService(fauxRepo);

    fauxRepo.enregistrerInscription(new Inscription(etudiantId, filiereA));

    assertThrows(EtudiantDejaInscritException.class,
        () -> service.inscrire(etudiantId, filiereB));
}
```

Ce test n'a pas besoin d'un environnement d'exécution externe (base de données, serveur web). C'est une conséquence directe et vérifiable de la règle de dépendance : elle ne dépend d'aucune estimation de productivité ou de qualité, seulement du fait que `InscriptionRepositoryEnMemoire` est substituable à `InscriptionRepositoryJpaAdapter` puisque les deux implémentent la même interface.

Ce que ce document ne peut pas affirmer avec la même certitude : que ce gain se traduit systématiquement par un développement plus rapide sur un projet donné. Cela dépend de facteurs non couverts ici (taille de l'équipe, complexité du domaine, maintenance à long terme).

---

## 7. Port sortant vs port entrant

Deux catégories de ports sont généralement distinguées, selon le sens de l'appel :

| | Port sortant | Port entrant |
|---|---|---|
| Sens de l'appel | Le domaine appelle l'extérieur | L'extérieur appelle le domaine |
| Exemples | Persistance, envoi d'email, appel à une API externe | Endpoint REST, commande CLI, consommateur de message |

Le port sortant est nécessaire dès qu'on veut appliquer la règle de dépendance de la section 5 à une opération où le domaine a besoin d'un service externe (par exemple sauvegarder une donnée) : sans interface intermédiaire, le domaine dépendrait directement d'une classe technique.

Le port entrant n'est pas nécessaire au même titre : un `Controller` peut appeler directement une classe concrète `InscriptionService` sans violer la règle de dépendance, puisque c'est `infrastructure/` (où vit le `Controller`) qui dépend de `domain/`, ce qui respecte le sens autorisé. L'introduction d'une interface côté entrant (par exemple `InscrireEtudiantUseCase`) est une décision de conception distincte, parfois justifiée par la lisibilité du code ou par des contraintes d'organisation d'équipe, mais elle n'est pas requise par la définition du pattern.

---

## 8. Pourquoi "hexagonale" ?

Le nombre de côtés de l'hexagone (six) n'a pas de correspondance avec un nombre fixe de catégories de ports ou de règles du pattern. Cockburn explique ce choix par la volonté de représenter plusieurs "portes" d'égale importance autour du domaine, en évitance du schéma en ligne droite (base de données à gauche, interface utilisateur à droite) qui suggère que ces deux éléments ont un statut particulier par rapport aux autres intégrations possibles (file de messages, CLI, API externe).

---

## 9. Limites du pattern

Les points suivants ne dépendent pas d'une évaluation subjective : ils découlent du fait que le pattern est une convention d'organisation du code, non une contrainte imposée par le compilateur.

- Rien dans le langage Java n'empêche un développeur d'écrire une règle métier dans une classe du package `infrastructure/`. Le respect de la séparation domaine/technique dépend de la discipline humaine (revue de code) ou d'un outillage additionnel (ArchUnit ou équivalent).
- Le pattern ne réduit pas nécessairement le nombre de bugs applicatifs indépendants de l'architecture (erreur de logique, appel dupliqué, etc.).
- Le pattern augmente le nombre de fichiers et d'indirections par rapport à une architecture en couches pour une même fonctionnalité, ce qui est vérifiable en comptant les fichiers de l'exemple de la section 12 face à un équivalent en couches classique.

---

## 10. Structure de dossiers

Structure possible pour une seule fonctionnalité :

```
com.ecole
├── domain/
│   ├── Inscription.java                    (objet du domaine)
│   ├── InscriptionRepository.java          (port sortant)
│   ├── NotifierInscription.java            (port sortant)
│   └── InscriptionService.java             (service métier)
│
├── infrastructure/
│   ├── persistence/
│   │   ├── InscriptionEntity.java
│   │   ├── SpringDataInscriptionRepository.java
│   │   └── InscriptionRepositoryJpaAdapter.java   (adapter sortant)
│   ├── notification/
│   │   └── NotifierInscriptionEmailAdapter.java   (adapter sortant)
│   └── web/
│       ├── InscriptionController.java             (adapter entrant)
│       └── InscriptionRequest.java
```

Pour plusieurs fonctionnalités (inscriptions, paiements, notes), un découpage de `domain/` par sous-domaine est une option d'organisation, indépendante de la règle de dépendance :

```
domain/
├── inscription/
├── paiement/
└── notation/
```

Cette question relève de la lisibilité du projet et n'a pas de réponse unique imposée par le pattern.

---

## 11. Vocabulaire : "Model" vs "objet du domaine"

Le terme "Model" désigne des choses différentes selon le contexte :

- En MVC, "Model" désigne généralement une classe de données, parfois directement couplée à la persistance (par exemple `ActiveRecord` en Ruby on Rails, où l'objet Model correspond directement à une ligne de table SQL).
- En JPA, on parle d'`@Entity`.
- Dans le vocabulaire du DDD (Domain-Driven Design) associé à l'architecture hexagonale, on parle d'**objet du domaine** (*Entity* au sens DDD, ou *Value Object*), qui ne référence aucune technologie.

Une classe comme `Inscription.java` dans l'exemple de ce document est un objet du domaine. Elle est distincte d'une éventuelle `InscriptionEntity` JPA : les deux peuvent coexister dans un même projet, avec une conversion entre elles réalisée dans l'adapter.

---

## 12. Exemple de code

**`domain/Inscription.java`**
```java
package com.ecole.domain;

public class Inscription {
    private final Long etudiantId;
    private final Long filiereId;

    public Inscription(Long etudiantId, Long filiereId) {
        this.etudiantId = etudiantId;
        this.filiereId = filiereId;
    }

    public Long getEtudiantId() { return etudiantId; }
    public Long getFiliereId() { return filiereId; }
}
```

**`domain/InscriptionRepository.java`** (port sortant)
```java
package com.ecole.domain;

public interface InscriptionRepository {
    boolean etudiantDejaInscrit(Long etudiantId);
    void enregistrerInscription(Inscription inscription);
}
```

**`domain/EtudiantDejaInscritException.java`**
```java
package com.ecole.domain;

public class EtudiantDejaInscritException extends RuntimeException {
    public EtudiantDejaInscritException(Long etudiantId) {
        super("L'étudiant " + etudiantId + " est déjà inscrit dans une autre filière.");
    }
}
```

**`domain/InscriptionService.java`**
```java
package com.ecole.domain;

public class InscriptionService {
    private final InscriptionRepository repository;

    public InscriptionService(InscriptionRepository repository) {
        this.repository = repository;
    }

    public void inscrire(Long etudiantId, Long filiereId) {
        if (repository.etudiantDejaInscrit(etudiantId)) {
            throw new EtudiantDejaInscritException(etudiantId);
        }
        repository.enregistrerInscription(new Inscription(etudiantId, filiereId));
    }
}
```

**`infrastructure/persistence/InscriptionRepositoryEnMemoire.java`** (adapter sortant utilisé pour les tests)
```java
package com.ecole.infrastructure.persistence;

import com.ecole.domain.Inscription;
import com.ecole.domain.InscriptionRepository;
import java.util.ArrayList;
import java.util.List;

public class InscriptionRepositoryEnMemoire implements InscriptionRepository {
    private final List<Inscription> inscriptions = new ArrayList<>();

    @Override
    public boolean etudiantDejaInscrit(Long etudiantId) {
        return inscriptions.stream().anyMatch(i -> i.getEtudiantId().equals(etudiantId));
    }

    @Override
    public void enregistrerInscription(Inscription inscription) {
        inscriptions.add(inscription);
    }
}
```

Un adapter équivalent utilisant JPA (`InscriptionRepositoryJpaAdapter`) implémenterait la même interface ; `InscriptionService` n'aurait aucun code à modifier pour passer de l'un à l'autre, puisqu'il ne référence que le port.

---

## 13. Comparaison avec d'autres architectures

| Question | Réponses possibles |
|---|---|
| Isoler le domaine de la technique | Hexagonale, Clean Architecture (Robert C. Martin), Onion Architecture (Jeffrey Palermo) |
| Organiser l'affichage et l'interaction utilisateur | MVC |
| Regrouper le code par fonctionnalité plutôt que par couche | Vertical Slice Architecture |
| Séparer les chemins de lecture et d'écriture | CQRS |
| Faire communiquer des composants sans appel direct | Architecture événementielle |
| Déployer une seule application ou plusieurs | Monolithe vs microservices |

Clean Architecture et Onion Architecture partagent avec l'architecture hexagonale la même règle de dépendance (vers le centre), représentée par des cercles concentriques plutôt qu'un hexagone. Clean Architecture introduit une couche nommée *Use Cases*, qui correspond approximativement à un port entrant explicite.

MVC répond à une question distincte (organisation de l'affichage) et ne comporte pas, par définition, de séparation entre logique métier et persistance ; les deux patterns ne sont pas mutuellement exclusifs.

Le choix entre monolithe et microservices porte sur le découpage physique du déploiement, indépendamment de l'organisation interne du code : un microservice peut être organisé en couches, en hexagonale, ou sans organisation particulière.

---

## 14. Critères de choix

Le tableau suivant reflète des recommandations généralement citées dans la littérature sur le sujet (voir section 15), non une règle universelle :

| Contexte | Architecture usuellement recommandée |
|---|---|
| Projet de courte durée, périmètre métier limité, un seul développeur | Architecture en couches |
| Projet destiné à évoluer sur plusieurs années, plusieurs développeurs, règles métier amenées à changer | Architecture hexagonale, Clean ou Onion |

Une pratique documentée consiste à démarrer avec une architecture en couches, et à introduire des ports/adapters seulement sur les parties du code où le besoin se manifeste concrètement (dépendance technique bloquant les tests, changement de fournisseur technique anticipé).

---

## 15. Sources

1. Alistair Cockburn, "Hexagonal Architecture" (2005) — article original :
   https://alistair.cockburn.us/hexagonal-architecture/

2. Juan Manuel Garrido de Paz, co-auteur avec Cockburn d'un ouvrage sur le sujet publié en 2024 — site et exemple de code :
   - https://jmgarridopaz.github.io/
   - https://github.com/jmgarridopaz/bluezone

3. Herberto Graça, "The Software Architecture Chronicles" — série d'articles reliant Hexagonale, Clean, Onion, DDD et CQRS :
   - https://herbertograca.com/2017/07/03/the-software-architecture-chronicles/
   - https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/

4. Robert C. Martin, *Clean Architecture* (2017) — ouvrage de référence sur Clean Architecture, non gratuit.
