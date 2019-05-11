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


