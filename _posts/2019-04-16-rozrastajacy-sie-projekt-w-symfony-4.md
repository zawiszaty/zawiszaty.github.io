---
layout: post
title:  "#2 Rozrastajacy sie projekt w Symfony 4"
date:   2019-04-16 21:03:36 +0530
categories: PHP Symfony PHPUnit Deptrac PHPStan
image: thumbnails.jpg
comments: true
---


## Dobre praktyki podczas pracy z symfony w większym projekcie
### Kilka słów na początek
W poprzednim wpisie opisywałem jak zacząć tworzenie aplikacji przy użyciu Symfony 4. Zdaje sobie sprawę, że projekt nie był idealny. Kilka osób zwróciło mi uwage na np postawienie na testy end2end, a nie na jednostkowe. Był to świadomy zabieg. Przy niedużych projektach takie testy dużo ułatwiają i bardzo szybko się je robi. Przy jednostkowych trzeba trochę więcej przygotowań, aby były przydatne i nie robiły problemów. Lepiej żeby w projekcie były tylko takie testy niż w ogóle.

### Od czego zaczać?
Na pewno rzuciła wam się w oczy struktura katalogów w poprzedniej wersji projektu. Nie była ona idealna i przy rozrastaniu się projektu zacząłby się robić burdel.
Ja preferuje strukturę katalogów wzorowaną na DDD. Wszystko jest klarowne i logicznie 
poukładane, wiec wiemy gdzie czego szukać.

Opierać sie bedzie na 4 katalogach głównych a wewnątrz nich podkatalogi z nazwami domen lub typem.
#### Application
W tej warstwie (Katalogu) trzymamy naszą logikę aplikacyjną, czyli nasz łącznik pomiędzy UI (Kontrolerem) a encją i repozytoriami. Mianowicie Service i Provider. 
#### Domain
Tutaj trzymamy najważniejsze rzeczy w naszym projekcie, a mianowicie naszą logikę domenową. Co prawda u nas ta logika jest bardzo naiwna, ale mam nadzieje że zrozumiecie, o co chodzi. Powinny tutaj wylądować: Encja, Fabryki Encji, Interfejsy do repozytoriów (każde repozytorium powinno mieć interfejs, o tym później) oraz Asercje. 
#### Infrastructure
Wszelkiego rodzaju implementacje interfejsów oraz rzeczy związane z zewnętrznymi bibliotekami.
#### UI
Wszystko, co bezpośrednio `gada` z użytkownikiem. Kontrolery, Komendy i co tam sobie jeszcze wymyślicie.

Przykład wykorzystania możecie znaleźć na moim [githubie](https://github.com/zawiszaty/symfony_simple_crud_example/tree/master/src) 

### Interfejsy
Jak mówią zasady SOLID nie powinniśmy opierać się na konkretniej implementacji a interfejsie.
Znaczy to tyle ze np repozytorium powinno mieć swój własny interfejs.
```php
<?php

declare(strict_types=1);

namespace App\Domain\Category\Repository;

use App\Domain\Category\Category;

interface CategoryRepositoryInterface
{
    public function findOneByName(?string $name): ?Category;
    public function getAllCategory(int $page, int $limit): array;
    public function save(Category $category): void;
}
```
Klasa nie dostaje konkretnej implementacji a interfejs. Czyli podmiana implementacji na inną to zmiana jednej linijki w naszym kontenerze DI. Kiedy to sie przydaje? Kiedy np zmieniamy bazę danych i mamy nowe repozytoria. Wystarczy taka prosta podmianka, a nasz stary kod nie powinien odczuć róznicy.
Przykład?
```php
<?php

declare(strict_types=1);

namespace App\Application\Service;

use App\Domain\Category\Factory\CategoryFactory;
use App\Domain\Category\Repository\CategoryRepositoryInterface;
use App\Infrastructure\Category\Validator\CategoryValidator;

/**
 * Class CategoryService.
 */
final class CategoryService
{
    /**
     * @var CategoryRepositoryInterface
     */
    private $categoryRepository;
    /**
     * @var CategoryValidator
     */
    private $categoryValidator;

    /**
     * CategoryService constructor.
     *
     * @param CategoryRepositoryInterface $categoryRepository
     * @param CategoryValidator           $categoryValidator
     */
    public function __construct(CategoryRepositoryInterface $categoryRepository, CategoryValidator $categoryValidator)
    {
        $this->categoryRepository = $categoryRepository;
        $this->categoryValidator = $categoryValidator;
    }

    /**
     * @param string $name
     *
     * @throws \App\Domain\Category\Exception\NameExistException
     */
    public function create(string $name): void
    {
        $this->categoryValidator->categoryNameNotExist($name);
        $this->categoryRepository->save(
            CategoryFactory::create($name)
        );
    }
}
```
CategoryService opiera sie na CategoryRepositoryInterface nie na np. MysqlCategoryRepository.
Nic nie szkodzi nam aby nasz test wyglądał tak.(W poprzednim wpisie zapomniałem o Validatorach :P, koncepcja jest taka, że Service powinny być niezależne od UI, czyli każdy ma własna walidację i nie jest uzależniony od warstw UI. Bez zmian w naszym Service powinniśmy dać radę napisać UI w CLI bez zmianny kodu samego Service. Jeżeli nie da się, coś zrobiliśmy źle)
```php
<?php

namespace App\Tests\Application\Service;

use App\Application\Service\CategoryService;
use App\Domain\Category\Category;
use App\Domain\Category\Exception\NameExistException;
use App\Infrastructure\Category\Repository\InMemoryCategoryRepository;
use App\Infrastructure\Category\Validator\CategoryValidator;
use App\Tests\Application\ApplicationTestCase;
use Doctrine\ORM\OptimisticLockException;
use Doctrine\ORM\ORMException;

class CategoryServiceTest extends ApplicationTestCase
{
    /**
     * @var CategoryService|object|null
     */
    private $service;

    /**
     * @var InMemoryCategoryRepository
     */
    private $repository;

    protected function setUp(): void
    {
        parent::setUp();
        $this->repository = new InMemoryCategoryRepository();
        $this->service = new CategoryService($this->repository, new CategoryValidator($this->repository));
    }

    public function test_create()
    {
        $this->service->create('test');
        /** @var Category $category */
        $category = $this->repository->findOneByName('test');
        $this->assertSame($category->getName(), 'test');
    }

    public function test_validate()
    {
        $this->expectException(NameExistException::class);
        $this->service->create('test');
        $this->service->create('test');
    }
}
```
Nie odpalamy nawet kernela symfony, wiec nasz test jest szybszy.
A nasz serwis nie zorientuje sie nawet, że repozytorium które dostał wygląda tak
```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Category\Repository;

use App\Domain\Category\Category;
use App\Domain\Category\Repository\CategoryRepositoryInterface;

class InMemoryCategoryRepository implements CategoryRepositoryInterface
{
    /**
     * @var array
     */
    private $categories = [];

    public function findOneByName(?string $name): ?Category
    {
        /** @var Category $category */
        foreach ($this->categories as $category) {
            if ($category->getName() === $name) {
                return $category;
            }
        }

        return null;
    }

    public function getAllCategory(int $page, int $limit): array
    {
        $data = [
            'data' => \array_slice($this->categories, \count($this->categories) - $limit),
            'count' => \count($this->categories),
        ];

        return $data;
    }

    public function save(Category $category): void
    {
        $this->categories[] = $category;
    }
}
```
Pozwala nam to na łatwe pisanie zaślepek i testowanie jednostkowe.
### Separowanie się od frameworka
#### Repozytorium
Nie korzystam z klas doctrinowych repozytoriów, wiec napisałem sobie swoją abstrakcję.
```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Shared\Repository;

use App\Infrastructure\Shared\Adapter\EntityManagerAdapterInterface;
use Doctrine\ORM\EntityRepository;
use Doctrine\ORM\NonUniqueResultException;
use Doctrine\ORM\QueryBuilder;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;

/**
 * Class MysqlRepository.
 */
abstract class MysqlRepository
{
    /** @var EntityRepository */
    protected $repository;
    /**
     * @var EntityManagerAdapterInterface
     */
    protected $entityManager;

    /**
     * MysqlRepository constructor.
     *
     * @param EntityManagerAdapterInterface $entityManager
     * @param string                        $model
     */
    public function __construct(EntityManagerAdapterInterface $entityManager, string $model)
    {
        $this->entityManager = $entityManager;
        $this->setRepository($model);
    }

    /**
     * @param object $model
     */
    public function register(object $model): void
    {
        $this->entityManager->persist($model);
        $this->apply();
    }

    public function apply(): void
    {
        $this->entityManager->flush();
    }

    /**
     * @param QueryBuilder $queryBuilder
     *
     * @return mixed
     *
     * @throws NonUniqueResultException
     */
    protected function oneOrException(QueryBuilder $queryBuilder)
    {
        $model = $queryBuilder
            ->getQuery()
            ->getOneOrNullResult();
        if (null === $model) {
            throw new NotFoundHttpException();
        }

        return $model;
    }

    private function setRepository(string $model): void
    {
        /** @var EntityRepository $objectRepository */
        $objectRepository = $this->entityManager->getRepository($model);
        $this->repository = $objectRepository;
    }
}
```
Następnie, aby nie uzależnić domeny od doctrina, wywalamy adnotacje orma z encji i przenosimy je do yamla (bądź innego zewnętrznego pliku). 
#### Adaptery
Każda zewnętrzna biblioteka powinna mieć swój adapter. Tak samo, jak w repozytorium nie operujemy na konkretnej implementacji, a zaślepkach. Jeżeli stwierdzimy ze biblioteka była słaba w łatwy sposób ją zmienimy. Bądź zaślepimy ją w testach.
```php
<?php

declare(strict_types=1);

namespace App\Infrastructure\Shared\Adapter;

use Doctrine\ORM\EntityManagerInterface;

class DoctrineEntityManagerAdapter implements EntityManagerAdapterInterface
{
    /**
     * @var EntityManagerInterface
     */
    private $entityManager;

    public function __construct(EntityManagerInterface $entityManager)
    {
        $this->entityManager = $entityManager;
    }

    public function persist(object $model): void
    {
        $this->entityManager->persist($model);
    }

    public function flush(): void
    {
        $this->entityManager->flush();
    }

    public function getRepository(string $model): object
    {
        return $this->entityManager->getRepository($model);
    }
}
```
### Deptrac 
Tutaj właśnie deptrac pokaże swój pazur i będzie dobrze pilnował naszych warstw.

Nowa konfiguracja:
```yaml
paths:
  - ./src
exclude_files:
  - .*test.*
layers:
  - name: Application
    collectors:
      - type: className
        regex: App\\Application\\.*
  - name: Infrastructure
    collectors:
      - type: className
        regex: App\\Infrastructure\\.*
  - name: Domain
    collectors:
      - type: className
        regex: App\\Domain\\.*
  - name: UI
    collectors:
      - type: className
        regex: App\\UI\\.*
ruleset:
  Application:
    - Domain
  Domain:
  Infrastructure:
    - Domain
  UI:
    - Application
- Infrastructure
```
Warstwy powinny sie przenikać w dół `UI -> Application -> Domain`. I tego przenikania bedzie pilnował nam Deptrac
### Kilka słów na zakończenie
Jak poprzednim razem nie jest to jakaś wyrocznia jak to robić, ale zbiór moich luźnych przemyśleń. Jeżeli ktoś ma pomysł jak zrobić coś lepiej albo inaczej, zapraszam do sekcji komentarzy albo do zrobienia pull requesta na githubie.

Cały projekt możecie znaleźć pod linkiem https://github.com/zawiszaty/symfony_simple_crud_example


