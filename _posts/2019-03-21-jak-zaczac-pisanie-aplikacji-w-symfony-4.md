---
layout: post
title:  "#1 Jak zacząć pisanie aplikacji w Symfony 4"
date:   2019-03-21 21:03:36 +0530
image: thumbnails.jpg
categories: PHP Symfony PHPUnit
comments: true
---

## Instalacja Symfony
Zaczynamy od utworzenia nowego projektu poprzez polecenie:
```bash
$ composer create-project symfony/skeleton nazwa
```
Nastepnie przejdzmy do katalogu z nasza aplikacja i instalujemy kilka dodatkowych bibliotek
```bash
$ composer require sensio/framework-extra-bundle symfony/validator symfony/form symfony/orm-pack symfony/translation symfony/monolog-bundle  
$ composer require symfony/maker-bundle phpunit/phpunit ^8 phpstan/phpstan sensiolabs-de/deptrac-shim friendsofphp/php-cs-fixer guzzlehttp/guzzle doctrine/doctrine-fixtures-bundle --dev
``` 

## Konfiguracja
### PHPUnit
#### Biblioteka do testowania naszej aplikacji
Wszystko domyślnie
### Deptrac
#### Narzędzie, które będzie pilnować czy warstwy w naszym projekcie się nie przenikają za bardzo.
depfile.yml
```yml
paths:
  - ./src
exclude_files:
  - .*test.*
layers:
  - name: Controller
    collectors:
      - type: className
        regex: App\\Controller\\.*
  - name: Repository
    collectors:
      - type: className
        regex: App\\Repository\\.*
  - name: Service
    collectors:
      - type: className
        regex: App\\Service\\.*
  - name: Provider
    collectors:
      - type: className
        regex: App\\Provider\\.*
ruleset:
  Controller:
    - Service
    - Provider
  Service:
    - Repository
  Provider:
    - Repository
  Repository:
```
### PHPStan
#### Narzędzie, które będzie pilnować czy kod w naszej aplikacji nie łamie podstawowych zasad bezpiecznego kodu i dobrych praktyk. 
phpstan.neon
```neon
parameters:
	autoload_files:
		- vendor/autoload.php
```
### PHP-cs-fixer
#### Narzędzie, które będzie pilnować czy kod nie łamie standardu formatowania. Jeżeli napotka na błąd to samo narzędzie może go poprawić.
.php_cs.dist
```php
<?php
return PhpCsFixer\Config::create()
    ->setRiskyAllowed(true)
    ->setCacheFile(__DIR__ . '/.php_cs.cache')
    ->setRules([
        'array_syntax' => ['syntax' => 'short'],
        'declare_strict_types' => true,
        'no_useless_return' => true,
        'logical_operators' => true,
        'lowercase_constants' => true,
        'new_with_braces' => false,
        'align_multiline_comment' => true,
        'blank_line_after_opening_tag' => true,
        'blank_line_before_return' => true,
        'blank_line_before_statement' => true,
        'class_attributes_separation' => ['elements' => ['method', 'property']],
        'compact_nullable_typehint' => true,
        'concat_space' => ['spacing' => 'one'],
        'function_typehint_space' => true,
        'is_null' => true,
        'yoda_style' => true,
        'lowercase_keywords' => true,
        'lowercase_static_reference' => true,
        'method_argument_space' => true,
        'modernize_types_casting' => true,
        'native_constant_invocation' => true,
        'native_function_invocation' => true,
        'no_blank_lines_after_class_opening' => true,
        'no_blank_lines_after_phpdoc' => true,
        'no_closing_tag' => true,
        'no_empty_comment' => true,
        'no_leading_import_slash' => true,
        'no_spaces_around_offset' => true,
        'no_trailing_comma_in_singleline_array' => true,
        'no_unused_imports' => true,
        'no_useless_else' => true,
        'no_useless_return' => true,
        'no_whitespace_before_comma_in_array' => true,
        'object_operator_without_whitespace' => true,
        'ordered_class_elements' => [
            'order' => [
                'use_trait',
                'constant_public',
                'constant_protected',
                'constant_private',
                'property_public',
                'property_protected',
                'property_private',
                'construct',
                'destruct',
                'magic',
                'method_public',
                'method_protected',
                'method_private',
            ],
        ],
        'psr0' => true,
        'psr4' => true,
        'return_type_declaration' => ['space_before' => 'none'],
        'short_scalar_cast' => true,
        'single_quote' => true,
        'standardize_not_equals' => true,
        'strict_comparison' => true,
        'ternary_to_null_coalescing' => true,
        'visibility_required' => ['elements' => ['property', 'method', 'const']],
        'void_return' => true,
    ])
    ->setFinder(
        PhpCsFixer\Finder::create()
            ->in(__DIR__ . '/src')
            ->files()
    )
;
```
Wszystkie te narzędzia służą poprawie naszego kodu oraz pozwalają na zaoszczędzenie czasu przy code review. Ponieważ code review, które opiera się na sprawdzaniu poprawności formatowania kodu jest stratą czasu. Sprawdzający powinien sie skupić czy rozwiązanie nie łamie zasad SOLID albo czy nie jest niebezpieczne, a nie sprawdzać ilość spacji albo enterów, jeżeli może to zrobić za nas narzędzie.
## Pierwsza linia kodu
Zaczniemy od wygenerowania pierwszej Encji
```bash
$ php bin/console make:entity Category  
```
Zostaniecie poproszeni o podanie nazwy pola, typu, długości itp. Ja utworzyłem sobie jedynie pole name typu stringi createdAt typu date.
Polecałbym usunąć settery i wymienić to na konstruktor. Jeżeli nie potrzebujesz metody do zmiany obiektu nie rób jej!!! A jeżeli potrzebujesz to staraj sie używać change zamiast set np changeName().
```php
use DateTime;
use Doctrine\ORM\Mapping as ORM;
use Symfony\Bridge\Doctrine\Validator\Constraints\UniqueEntity;

/**
 * @ORM\Entity(repositoryClass="App\Repository\CategoryRepository")
 * @UniqueEntity("name")
 */
class Category
{
    /**
     * @ORM\Id()
     * @ORM\GeneratedValue()
     * @ORM\Column(type="integer")
     */
    private $id;

    /**
     * @ORM\Column(type="string", length=255, unique=true)
     */
    private $name;

    /**
     * @ORM\Column(type="date", length=255)
     */
    private $createdAt;

    /**
     * Category constructor.
     *
     * @param string   $name
     * @param DateTime $createdAt
     */
    public function __construct(string $name, DateTime $createdAt)
    {
        $this->name = $name;
        $this->createdAt = $createdAt;
    }

    /**
     * @return int
     */
    public function getId(): int
    {
        return $this->id;
    }

    /**
     * @return string
     */
    public function getName(): string
    {
        return $this->name;
    }

    /**
     * @return DateTime
     */
    public function getCreatedAt(): DateTime
    {
        return $this->createdAt;
    }

    /**
     * @return array
     */
    public function serialize(): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'createdAt' => $this->createdAt,
        ];
    }
}
```
Po co to? `public function serialize(): array` Pomoże nam to przy odczytywaniu danych.
Następnie wygenerujemy sobie formularz
```bash
$ php bin/console make:form Category 
```
Pierwszą opcje zostawcie pusta.
Jako ze ja robie api wyłączam csrf i dodaje walidacje unikalności.
```php
use App\Repository\CategoryRepository;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;
use Symfony\Component\Validator\Constraints\Callback;
use Symfony\Component\Validator\Constraints\NotNull;
use Symfony\Component\Validator\Context\ExecutionContextInterface;

class CategoryType extends AbstractType
{
    /**
     * @var CategoryRepository
     */
    private $categoryRepository;

    public function __construct(CategoryRepository $categoryRepository)
    {

        $this->categoryRepository = $categoryRepository;
    }

    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('name', TextType::class, ['constraints' => [
                new NotNull(),
            ]]);
    }

    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults([
            'csrf_protection' => false,
            'constraints' => [
                new Callback([
                    'callback' => [$this, 'checkName'],
                ]),
            ]
        ]);
    }

    public function checkName(array $data, ExecutionContextInterface $context): void
    {
        $category = $this->categoryRepository->findOneBy(['name' => $data['name']]);

        if ($category)
        {
            $context->addViolation('Name Exist.');
        }
    }
}
```
Na koniec wygenerujmy sobie kontroler
```bash
$  php bin/console make:controller CategoryController 
```
## Konfiguracja projektu
Musicie podpiąć, jakąś bazę oraz odpalić webserver. Ja do tego celu użyje dockera, jeżeli nie chcecie z niego korzystać odsyłam was do dokumentacji symfony jak zrobić to ręcznie.
## Pierwsza Logika
### Fabryka
Zaczynamy od utworzenia klasy fabryki, która ułatwi nam wypluwanie obiektów z podstawową konfiguracją.
```php
class CategoryFactoryTest extends TestCase
{
    /**
     * @test
     */
    public function it_create_category()
    {
        $category = CategoryFactory::create('test');
        $this->assertInstanceOf(Category::class, $category);
        $this->assertSame($category->getName(), 'test');
    }
}
```
Najpierw testy potem kod
```php
use App\Entity\Category;
use DateTime;

class CategoryFactory
{
    public static function create(string $name): Category
    {
        $category = new Category($name, new DateTime());

        return $category;
    }
}
```
Fabryka sprawdza się najlepiej tam, gdzie albo musimy ustawić jakieś parametry domyślne (które są takie same dla każdego obiektu), albo właśnie datę utworzenia. 
### Warstwa Serwisów
Następnie łączymy to z warstwą serwisów. Kontrolery we wzorcu MVC powinny zarządzać tylko przepływem informacji.
```php
class CategoryServiceTest extends ApplicationTestCase
{
    /**
     * @var CategoryService|object|null
     */
    private $service;

    protected function setUp(): void
    {
        parent::setUp();
        $this->service = $this->container->get(CategoryService::class);

    }

    /**
     * @test
     */
    public function testCreate()
    {
        $this->service->create('test');
        /** @var Category $category */
        $category = $this->entityManager->getRepository(Category::class)->findOneBy(['name' => 'test']);
        $this->assertSame($category->getName(), 'test');
    }
}
```
Przykładowy test. 
Testy, które napisałem to testy funkcjonalne, sprawdzają one czy komunikacja z bazą danych jest poprawna.
A teraz kod
```php
final class CategoryService
{
    /**
     * @var EntityManagerInterface
     */
    private $entityManager;

    public function __construct(EntityManagerInterface $entityManager)
    {
        $this->entityManager = $entityManager;
    }

    public function create(string $name): void
    {
        $category = CategoryFactory::create($name);
        $this->entityManager->persist($category);
        $this->entityManager->flush();
    }
}
```
Odczyt z bazy danych wydzieliłem do osobnych klas. Nazywam je Providerami. Dlaczego po prostu nie wstrzykniemy repozytorium do Kontrolera? Dlatego że po dodaniu paginacji itp. złamiemy zasadę wzorca MVC i kontroler nie będzie zarządzał przepływem danych a wykonywał jakąś logikę. Poza tym odczyt danych ORM nie jest najlepszym pomysłem. Dlaczego? Ponieważ marnotrawimy moc naszego serwera, na operacje których nie będziemy używać. Dużo lepiej jest do tego użyć queryBuildera.
```php
use App\Common\Collection;
use App\Tests\ApplicationTestCase;

class CategoryProviderTest extends ApplicationTestCase
{
    public function test_it_get_all_category()
    {
        /** @var CategoryProvider $provider */
        $provider = $this->container->get(CategoryProvider::class);
        $data = $provider->getAll(1, 10);
        $this->assertInstanceOf(Collection::class, $data);
        $this->assertSame($data->page, 1);
        $this->assertSame($data->limit, 10);
        $this->assertSame($data->total, 30);
        $this->assertSame(count($data->data), 10);
    }
}
```
Test i implementacja
```php
use App\Common\Collection;
use App\Entity\Category;
use App\Repository\CategoryRepository;

class CategoryProvider
{
    /**
     * @var CategoryRepository
     */
    private $categoryRepository;

    public function __construct(CategoryRepository $categoryRepository)
    {
        $this->categoryRepository = $categoryRepository;
    }

    public function getAll(int $page, int $limit): Collection
    {
        $data = $this->categoryRepository->getAllCategory($page, $limit);
        $serializedData = [];
        /** @var Category $datum */
        foreach ($data['data'] as $datum) {
            $serializedData[] = $datum->serialize();
        }
        $collection = new Collection($page, $limit, (int)$data['count'][0][1], $serializedData);

        return $collection;
    }
}
```
Napisałem własną metode w repozytorium, mogłem uzyć findAll ale to metody ORM'owe.
```php
    /**
     * @param int $page
     * @param int $limit
     *
     * @return array
     */
    public function getAllCategory(int $page, int $limit): array
    {
        $qb = $this
            ->createQueryBuilder('category')
            ->setFirstResult(($page - 1) * $limit)
            ->setMaxResults($limit);
        $model = $qb->getQuery()->execute();
        $qbCount = $this
            ->createQueryBuilder('category')
            ->select('count(category.id)');
        $count = $qbCount->getQuery()->execute();
        $data = [
            'data' => $model,
            'count' => $count,
        ];

        return $data;
    }
```
### Kontrolery
No i ostatecznie łączymy to z naszymi Kontrolerami
Tworzymy TestCase do testów e2e.
```php
use App\DataFixtures\AppFixtures;
use App\Kernel;
use Doctrine\Common\DataFixtures\Executor\ORMExecutor;
use Doctrine\Common\DataFixtures\Loader;
use Doctrine\Common\DataFixtures\Purger\ORMPurger;
use GuzzleHttp\Client;
use PHPUnit\Framework\TestCase;

class ApplicationTestCase extends TestCase
{
    /**
     * @var \Doctrine\ORM\EntityManager
     */
    protected $entityManager;

    /**
     * @var \Symfony\Component\DependencyInjection\ContainerInterface|null
     */
    protected $container;

    /**
     * @var Kernel
     */
    private $kernel;

    /**
     * @var Client
     */
    protected $client;

    protected function setUp(): void
    {
        $this->kernel = new Kernel('test', true);
        $this->kernel->boot();
        $this->container = $this->kernel->getContainer();
        $this->entityManager = $this->container->get('doctrine.orm.entity_manager');
        $loader = new Loader();
        $loader->addFixture(new AppFixtures());
        $purger = new ORMPurger($this->entityManager);
        $executor = new ORMExecutor($this->entityManager, $purger);
        $executor->execute($loader->getFixtures());
        $this->client = new Client(['base_uri' => 'http://nginx', 'http_errors' => false]);
    }
}
```
No i piszemy własny test
```php
use App\Entity\Category;
use App\Tests\ApplicationTestCase;
use GuzzleHttp\RequestOptions;

class CategoryControllerTest extends ApplicationTestCase
{
    /**
     * @test
     */
    public function test_create_category(): void
    {
        $response = $this->client->post('api/category', [
            RequestOptions::JSON => ['name' => 'test'],
        ]);
        $this->assertEquals(200, $response->getStatusCode());
        $category = $this->entityManager->getRepository(Category::class)->findOneBy(['name' => 'test']);
        $this->assertNotNull($category);
    }

    /**
     * @test
     */
    public function test_create_category_validation(): void
    {
        $response = $this->client->post('api/category', [
            RequestOptions::JSON => [],
        ]);
        $this->assertEquals(400, $response->getStatusCode());

        $response = $this->client->post('api/category', [
            RequestOptions::JSON => ['name' => 'test category1'],
        ]);
        $this->assertEquals(400, $response->getStatusCode());
    }

    /**
     * @test
     */
    public function test_get_all_category(): void
    {
        $response = $this->client->get('api/category?page=1&limit=10');
        $this->assertEquals(200, $response->getStatusCode());
        $this->assertNotNull($response->getBody()->getContents());
    }
}
```
Następnie piszemy kod Kontrolera:
```php
use App\Form\CategoryType;
use App\FormNormalizer\FormErrorSerializer;
use App\Provider\CategoryProvider;
use App\Service\CategoryService;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Annotation\Route;

final class CategoryController extends AbstractController
{
    /**
     * @var CategoryService
     */
    private $categoryService;

    /**
     * @var CategoryProvider
     */
    private $categoryProvider;

    /**
     * @var FormErrorSerializer
     */
    private $errorSerializer;

    public function __construct(CategoryService $categoryService, CategoryProvider $categoryProvider, FormErrorSerializer $errorSerializer)
    {
        $this->categoryService = $categoryService;
        $this->categoryProvider = $categoryProvider;
        $this->errorSerializer = $errorSerializer;
    }

    /**
     * @Route("/api/category", name="create_category", methods={"POST"})
     *
     * @param Request $request
     *
     * @return Response
     */
    public function createAction(Request $request): Response
    {
        /** @var string $json */
        $json = $request->getContent();
        $data = \json_decode($json, true);
        $form = $this->createForm(CategoryType::class);
        $form->submit($data);

        if ($form->isSubmitted() && $form->isValid()) {
            $this->categoryService->create((string)$data['name']);

            return new JsonResponse('ok');
        }

        return new JsonResponse($this->errorSerializer->convertFormToArray($form), Response::HTTP_BAD_REQUEST);
    }

    /**
     * @Route("/api/category", name="get_all_category", methods={"GET"})
     * @param Request $request
     *
     * @return Response
     */
    public function getAllCategoryAction(Request $request): Response
    {
        $page = (int)$request->get('page') ?? 1;
        $limit = (int)$request->get('limit') ?? 10;
        $data = $this->categoryProvider->getAll($page, $limit);

        return new JsonResponse($data);
    }
}
```
## Continuous Integration
Ja do projektów open source używam Travis.CI
```yml
sudo: required
language: php
php:
  - '7.3'
before_install:
  - sudo service mysql stop
  - sudo service mysql status
  - curl -L https://github.com/docker/compose/releases/download/1.21.1/docker-compose-`uname -s`-`uname -m` > docker-compose
  - chmod +x docker-compose
  - sudo mv docker-compose /usr/local/bin
  - docker-compose -v
  - docker -v

before_script:
  - make start

script:
  - make test
```
Przykładowy konfig uzywajacy dockera.
CI pozwala nam na zobaczenie czy nasz projekt uruchomi sie na świeżym serwerze i czy bedzie działać poprawnie poprzez odpalanie testów. Szczególnie przydatne kiedy nad projektem pracuje kilku devów
### Kilka słów na zakończenie
Na pewno każdy, kto pracuje na co dzień z symfony spotyka sie z mieszaniem warstw, niewydzielaniem logiki poza kontroler, brakiem testów itp. Ten artykuł pokazuje jak małym kosztem można dodać trochę lepszych praktyk do swojego projektu, które czasami mogą uratować nam skórę. Projekt nie jest idealny ani nie jest wyznacznikiem dobrych praktyk w symfony. Ma za zadanie nakłonić innych to większego myślenia nad jakością swojego kodu i udowodnienie że to jednak nie jest takie trudne.
Repo pokazujące te rzeczy w `akcji` jest dostępne pod [*linkiem*](https://github.com/zawiszaty/symfony_simple_crud_example).