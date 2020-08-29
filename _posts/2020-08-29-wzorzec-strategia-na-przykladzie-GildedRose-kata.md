---
layout: post
title:  "#5 Wzorzec strategia na przykładzie GildedRose kata"
date:   2020-08-29 18:03:36 +0530
categories: PHP Kata
image: thumbnails.jpg
comments: true
---

GildedRose kata jest idealnym przykładem jak przekombinowana ifologia może przynosić nam problemy ze zrozumieniem istniejącej implementacji. Paroma prostymi krokami może sobie z tym poradzić i znacząco zmniejszyć skomplikowanie implementacji.

Kata jest implenetacja sklepu, w którym sprzedawane są różne typy produktów. Z biegiem czasu niektóre przedmioty tracą na jakości, inne zyskują a w innych w ogóle nic się nie dzieje.
Zacznijmy od sprawdzenia co się dzieje w metodzie 'updateQuality' która odpowiada za zmianę jakości poszczególnych typów produktów.

```php
 public function updateQuality()
    {
        if ($this->name != "Aged Brie" && $this->name != "Backstage passes to a TAFKAL80ETC concert") {
            if ($this->quality > 0) {
                if ($this->name != "Sulfuras, Hand of Ragnaros") {
                    $this->quality = $this->quality - 1;
                }
            }
        } else {
            if ($this->quality < 50) {
                $this->quality = $this->quality + 1;

                if ($this->name == "Backstage passes to a TAFKAL80ETC concert") {
                    if ($this->sellIn < 11) {
                        if ($this->quality < 50) {
                            $this->quality = $this->quality + 1;
                        }
                    }

                    if ($this->sellIn < 6) {
                        if ($this->quality < 50) {
                            $this->quality = $this->quality + 1;
                        }
                    }
                }
            }
        }

        if ($this->name != "Sulfuras, Hand of Ragnaros") {
            $this->sellIn = $this->sellIn - 1;
        }

        if ($this->sellIn < 0) {
            if ($this->name != "Aged Brie") {
                if ($this->name != "Backstage passes to a TAFKAL80ETC concert") {
                    if ($this->quality > 0) {
                        if ($this->name != "Sulfuras, Hand of Ragnaros") {
                            $this->quality = $this->quality - 1;
                        }
                    }
                } else {
                    $this->quality = $this->quality - $this->quality;
                }
            } else {
                if ($this->quality < 50) {
                    $this->quality = $this->quality + 1;
                }
            }
        }
    }
```

Kod jest skomplikowany i całkowicie rozrzucony bez żadnego sensu co sprawia, że bardzo ciężko jest go zrozumieć i trzeba by było poświecić dużą ilość czasu, żeby płynnie się w nim poruszać.
Personalnie zacząłem od wydzielenia nazw produktów stałych z krótsza nazwa, żeby móc się łatwiej poruszać miedzy ifami. Następnie wydzieliłem ifa tylko dla produktu '"Aged Brie"' i starałem sie
wyciągnąć cały kod odpowiedzialny za ten produkt do tego ifa. Co każda najmniejszą zmianę powinno odpalać się testy, ponieważ przy takim kodzie o pomyłkę bardzo łatwo.
Przy wydzieleniu przeszkadzał mi mocno produkt 'Backstage passes to a TAFKAL80ETC concert' wiec od razu wydzieliłem też ifa dla niego.

```php
    public function updateQuality()
    {
        if ($this->name === self::AGED_BRIE)
        {
            if ($this->quality < 50)
            {
                $this->quality = $this->quality + 1;
            }

            if ($this->quality < 50 && $this->sellIn < 0)
            {
                $this->quality = $this->quality + 1;
            }
        }

        if ($this->name === self::BACKSTAGE_PASSES)
        {
            if ($this->quality < 50)
            {
                $this->quality = $this->quality + 1;

                if ($this->sellIn < 11)
                {
                    if ($this->quality < 50)
                    {
                        $this->quality = $this->quality + 1;
                    }
                }

                if ($this->sellIn < 6)
                {
                    if ($this->quality < 50)
                    {
                        $this->quality = $this->quality + 1;
                    }
                }
            }
        }

        if ($this->name != self::AGED_BRIE && $this->name != self::BACKSTAGE_PASSES)
        {
            if ($this->quality > 0)
            {
                if ($this->name != self::SULFURAS)
                {
                    $this->quality = $this->quality - 1;
                }
            }
        }

        if ($this->name != self::SULFURAS)
        {
            $this->sellIn = $this->sellIn - 1;
        }

        if ($this->sellIn < 0)
        {
            if ($this->name != self::AGED_BRIE)
            {
                if ($this->name != self::BACKSTAGE_PASSES)
                {
                    if ($this->quality > 0)
                    {
                        if ($this->name != self::SULFURAS)
                        {
                            $this->quality = $this->quality - 1;
                        }
                    }
                }
                else
                {
                    $this->quality = $this->quality - $this->quality;
                }
            }
        }
    }
```

Nadal kod nie jest idealny, ale już łatwiej nam się połapać co się dzieje w obrębie konkretnego produktu.
W następnym kroku zacząłem optymalizować ify i usuwać zbędne, ponieważ parę z nich jest wyraźnie powtórzonych.
Po optymalizacji zacząłem się przygotowywać do wywalenia ifów z metody 'updateQuality'.
Wydzieliłem interfejs, którego kontrakt to jakość i czas od wystawienia danego produktu.

```php
interface ItemsQualityUpdaterInterface
{
    public function updateQuality($quality, $sellIn): ItemsQualityUpdaterResponse;
}
```

Poszczególny typ produktu będzie miał swój 'quality updater' i metoda 'updateQuality' będzie swojego rodzaju fabryką, która wybierze adapter dla konkretnego produktu.

```php
final class AgedBrie implements ItemsQualityUpdaterInterface
{
    public function updateQuality($quality, $sellIn): ItemsQualityUpdaterResponse
    {
        if ($quality < 50)
        {
            ++$quality;
        }
        --$sellIn;

        if (($sellIn < 0) && $quality < 50)
        {
            ++$quality;
        }

        return new ItemsQualityUpdaterResponse($quality, $sellIn); // value object zeby nie zwracać gołej tablicy
    }
}
```

Przez co nasza 'opasła' metoda znacząco się 'odchudzi'.

```php
    public function __construct($name, $quality, $sellIn)
    {
        $this->name    = $name;
        $this->quality = $quality;
        $this->sellIn  = $sellIn;
        $this->items   = [
            "Aged Brie"                                 => new AgedBrie(),
            "Backstage passes to a TAFKAL80ETC concert" => new BackstagePasses(),
            "+5 Dexterity Vest"                         => new DexterityVest(),
            "Elixir of the Mongoose"                    => new ElixirOfMongoose(),
            "Conjured"                                  => new Conjured(),
        ];
    }

    public function updateQuality()
    {
        if (array_key_exists($this->name, $this->items))
        {
            $response      = $this->items[$this->name]->updateQuality($this->quality, $this->sellIn);
            $this->quality = $response->getQuality();
            $this->sellIn  = $response->getSellIn();
        }
    }
```

Teraz nasz kod sam potrafi podjąć decyzje, który adapter ma wybrać a dodanie kolejnego produktu nie zwiększy skomplikowania naszego kodu i będzie bardzo proste.

Jak widać, czasami bardzo małym nakładem pracy możemy mocno ułatwić prace nad zwariowanym kodem. Najważniejsze uporządkowanie ifów i znalezienie kontraktu, za którym możemy przykryć poszczególne implementacje.
W pojedynczym adapterze kod może być słabiej jakości, ale ważne, żeby jego jakość nie wpłynęła na jakość całej aplikacji.