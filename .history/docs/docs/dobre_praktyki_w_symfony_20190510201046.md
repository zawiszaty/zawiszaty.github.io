# #2 Rozrastajacy sie projekt w Symfony 4
## Dobre praktyki podczas pracy z symfony w większym projekcie
### Kilka słów na początek
W poprzednim wpisie opisywałem jak zacząć tworzenie aplikacji przy użyciu Symfony 4. Zdaje sobie sprawę, że projekt nie był idealny. Kilka rzeczy np postawienie na testy end2end, a nie na jednostkowe. Był to świadomy zabieg. Przy niedużych projektach takie testy dużo ułatwiają i bardzo szybko się je robi. Przy jednostkowych trzeba trochę więcej przygotowań, aby były przydatne i nie robiły problemów.
### Od czego zaczać?
Na pewno rzuciła wam się w oczy struktura katalogów w poprzednim wersji projektu. Nie była ona idealna i przy rozrastaniu się projektu zacząłby się robić burdel.
Ja preferuje strukturę katalogów wzorowaną na DDD. Wszystko jest klarowne i logicznie 
poukładane, wiec wiemy gdzie czego szukać.

Opierać sie bedzie na 4 katalogach głównych a wewnątrz nich podkatalogi z nazwami domen.
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
     * CategoryService constructor.
     *
     * @param CategoryRepositoryInterface $categoryRepository
     */
    public function __construct(CategoryRepositoryInterface $categoryRepository)
    {
        $this->categoryRepository = $categoryRepository;
    }
    /**
     * @param string $name
     */
    public function create(string $name): void
    {
        $this->categoryRepository->save(
            CategoryFactory::create($name)
        );
    }
}
```
CategoryService operia sie na CategoryRepositoryInterface nie na np. MysqlCategoryRepository.
Nic nie szkodzi nam aby nasz test wyglądał tak
```php
<?php
namespace App\Tests\Application\Service;
use App\Application\Service\CategoryService;
use App\Domain\Category\Category;
use App\Infrastructure\Category\Repository\InMemoryCategoryRepository;
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
        $this->service = new CategoryService($this->repository);
    }
    /**
     * @throws ORMException
     * @throws OptimisticLockException
     */
    public function testCreate()
    {
        $this->service->create('test');
        /** @var Category $category */
        $category = $this->repository->findOneByName('test');
        $this->assertSame($category->getName(), 'test');
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
    /** @var string */
    protected $class;
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
     */
    public function __construct(EntityManagerAdapterInterface $entityManager)
    {
        $this->entityManager = $entityManager;
        $this->setRepository($this->class);
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

