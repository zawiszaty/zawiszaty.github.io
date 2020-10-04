---
layout: post
title:  "#6 PHP finder refactoring"
date:   2020-10-03 18:03:36 +0530
categories: PHP Kata
image: thumbnails.jpg
comments: true
---

Na początku lądujemy w kodzie  w którym kompletnie nie wiadomo o co chodzi. Opisem katy jest, że nowy developer napisał kod w strasznie niezrozumiały sposób. Naszym zadaniem jest to posprzątać.

```php
<?php

declare(strict_types = 1);

namespace CodelyTV\FinderKata\Algorithm;

final class Finder
{
    /** @var Thing[] */
    private $_p;

    public function __construct(array $p)
    {
        $this->_p = $p;
    }

    public function find(int $ft): F
    {
        /** @var F[] $tr */
        $tr = [];

        for ($i = 0; $i < count($this->_p); $i++) {
            for ($j = $i + 1; $j < count($this->_p); $j++) {
                $r = new F();

                if ($this->_p[$i]->birthDate < $this->_p[$j]->birthDate) {
                    $r->p1 = $this->_p[$i];
                    $r->p2 = $this->_p[$j];
                } else {
                    $r->p1 = $this->_p[$j];
                    $r->p2 = $this->_p[$i];
                }

                $r->d = $r->p2->birthDate->getTimestamp()
                    - $r->p1->birthDate->getTimestamp();

                $tr[] = $r;
            }
        }

        if (count($tr) < 1) {
            return new F();
        }

        $answer = $tr[0];

        foreach ($tr as $result) {
            switch ($ft) {
                case FT::ONE:
                    if ($result->d < $answer->d) {
                        $answer = $result;
                    }
                    break;

                case FT::TWO:
                    if ($result->d > $answer->d) {
                        $answer = $result;
                    }
                    break;
            }
        }

        return $answer;
    }
}

``` 
Na pierwszy rzut oka wygląda to na jakiś mechanizm sortowania. Na szczęście mamy testy które są zrobione w lepszy sposób i z nich możemy odnaleźć o co chodzi. 

Po zajrzeniu w testy widzimy, że tworzona jest lista osób i nie ważne ile osób ląduje w finderze zawsze zwracana jest jedna para tych osób w specjalnym dtosie. 

Po chwili analizy pierwszego zbioru pętli można zobaczyć, że łączy ona wszystkie osoby w pary bez żadnego większego sensu. 

```php
 for ($i = 0; $i < count($this->_p); $i++) {
            for ($j = $i + 1; $j < count($this->_p); $j++) {
                $r = new F();

                if ($this->_p[$i]->birthDate < $this->_p[$j]->birthDate) {
                    $r->p1 = $this->_p[$i];
                    $r->p2 = $this->_p[$j];
                } else {
                    $r->p1 = $this->_p[$j];
                    $r->p2 = $this->_p[$i];
                }

                $r->d = $r->p2->birthDate->getTimestamp()
                    - $r->p1->birthDate->getTimestamp();

                $tr[] = $r;
            }
 }
```
Jeżeli podejrzymy co siedzi w zmiennej `$tr` to zobaczymy właśnie taka liste. 

Już teraz możemy zacząć naprawiać zepsute nazewnictwo wewnątrz  tego fora, wiec:

`Thing` możemy zamienić na `Person`

`_p` możemy zamienić na `PersonCollection`

`T` możemy zamienić na `PersonsPair` a `tr` na  `PersonsPairCollection`

żeby móc łatwo połapać się w tym kodzie. 

W międzyczasie pomyślałem, że lepiej będzie wynieść tego fora do osobnej klasy z nazwa która lepiej bedzie oznaczać co ona właściwie robi. 
```php
<?php

declare(strict_types=1);


namespace CodelyTV\FinderKata\Algorithm;


final class PersonsPairBuilder
{
    /**
     * @param Person[] $personsCollection
     *
     * @return PersonPairs[]
     */
    public function buildPersonsPairs(array $personsCollection): array
    {
        $personsRepresentations = [];

        foreach ($personsCollection as $index => $person)
        {
            $count = count($personsCollection);

            for ($j = $index + 1; $j < $count; $j++)
            {
                $personRepresentation = new PersonPairs();
                $nextPerson           = $personsCollection[$j];

                if ($person->isYoungestThan($nextPerson))
                {
                    $personRepresentation = $personRepresentation->withPerson($person)->withOldestPerson($nextPerson);
                }
                else
                {
                    $personRepresentation = $personRepresentation->withPerson($nextPerson)->withOldestPerson($person);
                }
                $personsRepresentations[] = $personRepresentation;
            }
        }

        return $personsRepresentations;
    }
}
```

stworzyłem, więc PersonPairsBuildera który jest odpowiedzialny za połączenie wszystkich osób w pary i wyplucie ich listy. Przeniosłem też powtarzające się ify z obliczaniem różnicy wieku danej pary do środka modeli.  

Tak wygląda teraz klasa person 
```php
<?php

declare(strict_types = 1);

namespace CodelyTV\FinderKata\Algorithm;

use DateTime;

final class Person
{
    /** @var string */
    private $name;

    /** @var DateTime */
    private $birthDate;

    public function __construct(string $name, DateTime $birthDate)
    {
        $this->name      = $name;
        $this->birthDate = $birthDate;
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function setName(string $name)
    {
        $this->name = $name;
    }

    public function getBirthDate(): DateTime
    {
        return $this->birthDate;
    }

    public function setBirthDate(DateTime $birthDate)
    {
        $this->birthDate = $birthDate;
    }

    public function isYoungestThan(Person $person): bool
    {
        return $this->birthDate < $person->birthDate;
    }

    public function ageDiffThan(Person $person): int
    {
        return $this->birthDate->getTimestamp() - $person->getBirthDate()->getTimestamp();
    }
}
```

a tak klasa `PersonsPair` 
```php
<?php

declare(strict_types = 1);

namespace CodelyTV\FinderKata\Algorithm;

final class PersonPairs
{
    /** @var Person|null */
    private $person;

    /** @var Person|null */
    private $olderPerson;

    /** @var int */
    private $ageDiff;

    public function hasSmallerAgeDiffThan(PersonPairs $personRepresentation): bool
    {
        return $this->ageDiff < $personRepresentation->ageDiff;
    }

    private function refreshTimestampDiff(): void
    {
        if (null === $this->olderPerson || null === $this->person)
        {
            $this->ageDiff = 0;

            return;
        }
        $this->ageDiff = $this->olderPerson->ageDiffThan($this->person);
    }

    public function withPerson(Person $person): self
    {
        $clone         = clone $this;
        $clone->person = $person;
        $clone->refreshTimestampDiff();

        return $clone;
    }

    public function withOldestPerson(Person $person): self
    {
        $clone              = clone $this;
        $clone->olderPerson = $person;
        $clone->refreshTimestampDiff();

        return $clone;
    }

    public function getPerson(): ?Person
    {
        return $this->person;
    }

    public function getOlderPerson(): ?Person
    {
        return $this->olderPerson;
    }
}
```

Wydzielając te metody zrobiłem też oczywiście czyszczenie tych klas wiedz wszystkie pola zamieniłem na prywatne i dodałem specjalne metody dostępowe do nich. 
Kodzik zaczyna wyglądać coraz lepiej, ale nadal pozostał nam ogarnięcia jeden foreach.

Wygląda on w następujący sposób: 
```php
   $answer = $tr[0];

        foreach ($tr as $result) {
            switch ($ft) {
                case FT::ONE:
                    if ($result->d < $answer->d) {
                        $answer = $result;
                    }
                    break;

                case FT::TWO:
                    if ($result->d > $answer->d) {
                        $answer = $result;
                    }
                    break;
            }
        }

        return $answer;
    }
```

po chwili zastanowienia widać, że porównuje diff wieku par w zależności od filtra i zwraca pierwsza pare po posortowaniu. 

Zajrzyjmy do interfejsu który jest tu używany
```php
interface FT
{
   const ONE = 1;

   const TWO = 2;
}
```  

kompletnie nie wiadomo co oznaczają te stałe, więc zmieniłem nazwy na bardziej znaczące. 
```php
interface Comparison
{
   public const CLOSEST = 1;

   public const FURTHER = 2;
}
```
Wróćmy do naszej klasy findera. Wydaje mi sie, że nie jest ona najlepszym miejscem do sortowania par. Utworzyłem więc osobną klasę która przejmie te odpowiedzialność. 
```php
<?php

declare(strict_types=1);


namespace CodelyTV\FinderKata\Algorithm;


final class PersonPairsSorter
{
    /**
     * @param PersonPairs[] $personsRepresentations
     *
     * @return PersonPairs[]
     */
    public function sortByAge(array $personsRepresentations, int $comparison): array
    {
        if ($comparison === Comparison::CLOSEST)
        {
            usort($personsRepresentations, static function (PersonPairs $representation1, PersonPairs $representation2) {
                return $representation2->hasSmallerAgeDiffThan($representation1);
            });

            return $personsRepresentations;
        }

        usort($personsRepresentations, static function (PersonPairs $representation1, PersonPairs $representation2) {
            return false === $representation2->hasSmallerAgeDiffThan($representation1);
        });

        return $personsRepresentations;
    }
}
```

zamieniłem też tego foreach na `usort` i użyłem metody z modelu którą sobie wydzieliłem dla lepszego zobrazowania problemu, Widać tutaj jak małym kosztem można skrócić kodzik oraz spowodować, że na pierwszy rzut oka widać co on dokładnie robi. 

Na koniec zobaczymy jak wygląda nasza klasa finder po refactroingu. 
```php
<?php

declare(strict_types=1);

namespace CodelyTV\FinderKata\Algorithm;

final class Finder
{
    /** @var Person[] */
    private $personsCollection;
    /** @var PersonPairsSorter */
    private $personSorter;
    /** @var PersonsPairBuilder */
    private $personsPairBuilder;

    public function __construct(array $personsCollection)
    {
        $this->personsCollection             = $personsCollection;
        $this->personSorter                  = new PersonPairsSorter();
        $this->personsPairBuilder            = new PersonsPairBuilder();
    }

    public function find(int $comparison): PersonPairs
    {
        $personsRepresentations = $this->personsPairBuilder->buildPersonsPairs($this->personsCollection);

        if (count($personsRepresentations) < 1)
        {
            return new PersonPairs();
        }

        return $this->getPersonRepresentationByComparison($personsRepresentations, $comparison);
    }

    /** @param PersonPairs[] $personRepresentations */
    protected function getPersonRepresentationByComparison(array $personRepresentations, int $comparison): PersonPairs
    {
        $personRepresentations = $this->personSorter->sortByAge($personRepresentations, $comparison);

        return $personRepresentations[0];
    }
}
``` 

Aktualnie wszystko jest zamknięte w małe klasy które sama nazwa przekazują co dokładnie robią, więc wdrożenie w ten kod jest o niebo szybsze i przyjemniejsze. 

Postanowiłem też skrócić te testy ponieważ duża ich część była mega powtarzalna. 
```php
<?php

declare(strict_types=1);

namespace CodelyTV\FinderKataTest\Algorithm;

use CodelyTV\FinderKata\Algorithm\Comparison;
use CodelyTV\FinderKata\Algorithm\Finder;
use CodelyTV\FinderKataTest\Algorithm\TestDoubles\PersonMother;
use DateTime;
use PHPUnit\Framework\TestCase;

final class FinderTest extends TestCase
{
    /**
     * @dataProvider person_pair_data_provider
     */
    public function test_person_pair_finder($list, $comparison, $person, $oldestPerson): void
    {
        $finder = new Finder($list);

        $result = $finder->find($comparison);

        self::assertEquals($person, $result->getPerson());
        self::assertEquals($oldestPerson, $result->getOlderPerson());
    }

    public function person_pair_data_provider(): array
    {
        $sue   = PersonMother::create('Sue', new DateTime("1950-01-01"));
        $greg  = PersonMother::create('Greg', new DateTime("1952-05-01"));
        $sarah = PersonMother::create('Sarah', new DateTime("1982-01-01"));
        $mike  = PersonMother::create('Mike', new DateTime("1979-01-01"));

        return [
            'should_return_empty_when_given_empty_list'  => [[], Comparison::CLOSEST, null, null],
            'should_return_empty_when_given_one_person'  => [[$sue,], Comparison::CLOSEST, null, null],
            'should_return_closest_two_for_two_people'   => [[$mike, $greg], Comparison::CLOSEST, $greg, $mike],
            'should_return_furthest_two_for_two_people'  => [[$mike, $greg], Comparison::FURTHER, $greg, $mike],
            'should_return_furthest_two_for_four_people' => [[$sue, $greg, $sarah, $mike], Comparison::FURTHER, $sue, $sarah],
            'should_return_closest_two_for_four_people'  => [[$sue, $greg, $sarah, $mike], Comparison::CLOSEST, $sue, $greg],
        ];
    }
}
```
Teraz dodanie kolejnego testu będzie jedna linijka w dataProviderze. 

### Podsumowanie

Kata jest dostępna na moim githubie [https://github.com/zawiszaty/php-finder_refactoring-kata]()
Oczywiście kodzik da sie jeszcze troche podrasować, ale ten stan kodu wystarczająco mnie satysfakcjonuje. 
Jednak jeżeli chcecie zobaczyć te kate w bardziej `technicznym` wydaniu zapraszam na brancha do CodelyTV od których wziąłem template katy 
[https://github.com/CodelyTV/php-finder_refactoring-kata/tree/04-collections]()

