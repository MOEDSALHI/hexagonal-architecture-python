# Hexagonal Architecture and Python

## Introduction
L'architecture hexagonale, également appelée "Ports and Adapters", est une approche visant à structurer les applications pour améliorer leur maintenabilité, leur testabilité et leur évolutivité. Ce guide se concentre sur une implémentation pratique en Python avec FastAPI, en illustrant un cas d'utilisation : **la gestion des articles dans une plateforme de blogging.**

---

## Cas d'utilisation : Gestion des articles
Dans cette application, nous voulons permettre les fonctionnalités suivantes :
- Ajouter des articles avec un titre et un score initial.
- Mettre à jour le score d'un article via un "upvote".
- Consulter les articles via une API REST.

Ce cas d'utilisation servira de fil conducteur pour expliquer les différentes étapes de l'implémentation de l'architecture hexagonale.

---

## Étape 1 : Comprendre les bases
L'architecture hexagonale se décompose en plusieurs couches clés :

- **Domain Layer** :
  - Cette couche contient la logique métier pure.
  - Exemple : Calculer le score d'un article après un "upvote".
  - **Pourquoi ?** Cela permet d'avoir une logique métier indépendante des frameworks ou des outils techniques.

- **Ports** :
  - Les ports sont des interfaces qui définissent comment les différentes parties de l'application interagissent.
  - Exemple : Un port définit une méthode pour sauvegarder ou récupérer un article, sans préciser comment cela est implémenté.
  - **Pourquoi ?** Les ports assurent que la logique métier reste découplée des implémentations techniques.

- **Adapters** :
  - Les adaptateurs implémentent les ports pour intégrer des technologies concrètes.
  - Exemple : Un adaptateur peut utiliser une base de données en mémoire pour sauvegarder des données définies dans le port.

- **Application Layer** :
  - Cette couche orchestre les cas d'utilisation en combinant des composants du domaine et des adaptateurs via des ports.
  - Exemple : Orchestrer la récupération d'un article, appliquer un "upvote", et sauvegarder les changements.

Les dépendances sont toujours dirigées de l'extérieur vers le centre (adapters -> ports -> domain).

---

## Étape 2 : Structurer le projet
Une bonne structure est essentielle pour maintenir la séparation des préoccupations.

### Exemple de structure
```
project_name/
├── src/
│   ├── domain/          # Logique métier (entités, services métiers)
│   ├── application/     # Cas d'utilisation (services applicatifs)
│   ├── ports/           # Interfaces API et SPI (définition des contrats)
│   ├── adapters/        # Implémentations concrètes des ports
│   │   ├── api/         # Interfaces exposées (API REST)
│   │   ├── spi/         # Interfaces système (base de données, fichiers, etc.)
│   ├── config/          # Configuration et démarrage
│   └── tests/           # Tests unitaires et d'intégration
```

**Pourquoi cette structure ?**
- **Séparation des responsabilités :** Chaque composant a un rôle bien défini.
- **Flexibilité :** Vous pouvez facilement changer un composant (par exemple, une base de données).
- **Testabilité :** Les composants peuvent être testés isolément.

---

## Étape 3 : Implémentation du domaine

### Qu'est-ce qu'une `dataclass` ?
Une `dataclass` en Python est une structure légère utilisée pour représenter des données. Elle génère automatiquement des méthodes comme `__init__`, `__repr__`, et `__eq__`, ce qui simplifie le code.

### Pourquoi l'utiliser dans le domaine ?
Les entités du domaine représentent des concepts métier. Une `dataclass` est idéale pour :
- Stocker des informations simples (comme un article avec un titre et un score).
- Réduire le code répétitif.

### Exemple d'entité : Article
```python
from dataclasses import dataclass

@dataclass
class Article:
    id: int
    title: str
    rating: int
```

### Service métier : Upvoter un article
```python
class ArticleService:
    def upvote(self, article):
        article.rating += 1
        return article
```
Ce service applique la règle métier d'augmenter le score d'un article.

---

## Étape 4 : Définir les ports

### Qu'est-ce qu'un SPI ?
Un **SPI (Service Provider Interface)** est une interface qui définit des contrats pour les services externes (comme une base de données). Dans notre cas, nous avons besoin d'un SPI pour gérer les articles.

### Pourquoi ?
- Découpler la logique métier des technologies concrètes.
- Faciliter le remplacement ou la simulation des services externes (par exemple, utiliser une base en mémoire pour les tests).

### Exemple de port SPI : ArticleRepository
```python
from abc import ABC, abstractmethod

class ArticleRepository(ABC):
    @abstractmethod
    def find_by_id(self, article_id: int):
        pass

    @abstractmethod
    def save(self, article):
        pass
```
Ce port définit les actions possibles sur les articles : récupération et sauvegarde.

---

## Étape 5 : Implémentation des adapters

### Qu'est-ce qu'un adapter ?
Un adapter est une implémentation concrète d'un port. Il traduit les appels entre la logique métier et les systèmes externes.

### Exemple : Repository en mémoire
```python
from domain.models import Article
from ports.spi import ArticleRepository

class InMemoryArticleRepository(ArticleRepository):
    def __init__(self):
        self.articles = {}

    def find_by_id(self, article_id):
        return self.articles.get(article_id)

    def save(self, article):
        self.articles[article.id] = article
```
Ce repository utilise un dictionnaire Python pour stocker les articles, ce qui est pratique pour les tests ou les prototypes.

---

## Étape 6 : Cas d'utilisation

### Qu'est-ce qu'un cas d'utilisation ?
Un cas d'utilisation orchestre les interactions entre les différentes couches pour implémenter une fonctionnalité métier.

### Exemple : Service pour "upvoter" un article
```python
from domain.services import ArticleService

class UpvoteArticleService:
    def __init__(self, article_repo):
        self.article_repo = article_repo

    def upvote_article(self, article_id):
        article = self.article_repo.find_by_id(article_id)
        if not article:
            raise ValueError("Article not found")
        service = ArticleService()
        updated_article = service.upvote(article)
        self.article_repo.save(updated_article)
        return updated_article
```
Ce service :
- Récupère l'article via le repository.
- Applique la règle métier (upvote).
- Sauvegarde les changements.

---

## Étape 7 : Exposer l'API

### Pourquoi une API ?
L'API agit comme un adaptateur pour exposer les fonctionnalités de l'application à des utilisateurs ou d'autres systèmes via HTTP.

### Exemple avec FastAPI
```python
from fastapi import FastAPI, HTTPException
from application.services import UpvoteArticleService
from adapters.spi.memory_repository import InMemoryArticleRepository

app = FastAPI()
article_repo = InMemoryArticleRepository()
upvote_service = UpvoteArticleService(article_repo)

# Articles initiaux
article_repo.save(Article(id=1, title="Article 1", rating=0))
article_repo.save(Article(id=2, title="Article 2", rating=0))

@app.post("/articles/{article_id}/upvote")
def upvote_article(article_id: int):
    try:
        updated_article = upvote_service.upvote_article(article_id)
        return {"id": updated_article.id, "rating": updated_article.rating}
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))
```
Cette API permet d'"upvoter" un article en envoyant une requête POST.

---

## Étape 8 : Tests unitaires
### Exemple de test
```python
from unittest.mock import MagicMock
from application.services import UpvoteArticleService
from domain.models import Article

def test_upvote_article():
    repo_mock = MagicMock()
    repo_mock.find_by_id.return_value = Article(id=1, title="Test", rating=0)
    repo_mock.save = MagicMock()

    service = UpvoteArticleService(repo_mock)
    updated_article = service.upvote_article(1)

    assert updated_article.rating == 1
    repo_mock.save.assert_called_once()
```
Ce test vérifie que le service applique correctement la règle métier et sauvegarde les changements.

---

## Étape 9 : Conclusion
En utilisant un cas d'utilisation unique (gestion des articles), cette implémentation illustre les concepts de l'architecture hexagonale. L'approche rend l'application maintenable, testable et extensible. Vous pouvez adapter ce modèle pour d'autres domaines métier ou technologies.
