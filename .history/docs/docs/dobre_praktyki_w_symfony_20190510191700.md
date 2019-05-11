# #2 Rozrastajacy sie projekt w Symfony 4
## Dobre praktyki podczas pracy z symfony w większym projekcie
### Kilka słów na początek
W poprzednim wpisie opisywałem jak zacząć tworzenie aplikacji przy użyciu Symfony 4. Zdaje sobie sprawę, że projekt nie był idealny. Kilka rzeczy np postawienie na testy end2end, a nie na jednostkowe. Był to świadomy zabieg. Przy niedużych projektach takie testy dużo ułatwiają i bardzo szybko się je robi. Przy jednostkowych trzeba trochę więcej przygotowań, aby były przydatne i nie robiły problemów.
### Od czego zaczać?
Na pewno rzuciła wam się w oczy struktura katalogów w poprzednim wersji projektu. Nie była ona idealna i przy rozrastaniu się projektu zacząłby się robić burdel.
Ja preferuje strukturę katalogów wzorowaną na DDD. Wszystko jest klarowne i logicznie poukładane, wiec wiemy gdzie czego szukać. 