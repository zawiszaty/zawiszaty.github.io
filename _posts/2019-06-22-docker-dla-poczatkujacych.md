---
layout: post
title:  "#3 Docker dla początkujących"
date:   2019-06-22 21:03:36 +0530
categories: PHP Docker
---

Docker jest oprogramowaniem, które pomaga nam w tworzeniu aplikacji opartych o kontenery. Natywnie działa tylko i wyłącznie na Linux. Na innych systemach działa tylko i wyłącznie dlatego, że Linux jest w pewien sposób wirtualizowany. Właśnie dlatego docker nie zadziała na Windows w wersji Home, ponieważ do wirtualizowania Linuxa używa Hyper-V.
Przez co na innych systemach działa wolniej i mogą być pewne różnice w działaniu.

## Czym są kontenery?
Kontener zachowuje się w podobny sposób jak maszyna wirtualna (ale nią nie jest [czym różni sie kontener od vm](https://www.youtube.com/watch?v=L1ie8negCjc)). Wyobraźmy sobie, że jest on osobnym serwerem, do którego łączymy się po ssh. Nasze kontenery w żaden sposób nie ingerują w nasz natywny system możemy mieć na raz odpalone php7, php5 oraz każda inna wersje (którą znajdziemy na docker hub).

## Kiedy używać?
Mamy wiele projektów, w których jest ryzyko, że będą oddziaływać na siebie (np. kilka projektów ma identyczną nazwę bazy danych itp), kiedy projekty mają diametralnie inną wersję języka albo chcemy mieć pewność, że nikt z naszego zespołu nie zainstaluje (np. przypadkiem) innej wersji jakieś bazy danych, języka itp. W łatwy sposób pozwala odtworzyć środowisko developerskie i automatyzować jego uruchomienie.

## Przykład uruchomienia kontenera
```bash
docker run --name nazwa_kontenera -e MYSQL_ROOT_PASSWORD=haslo -d mysql:5.7.26 -p 3306:3306
```

### --name
Nazwa potrzebna do łączenia sie do kontenera (po nazwie jest łatwiej, ale można też po id kontenera. ID możemy zobaczyć przez polecenie 'docker ps').

### -e
Tutaj podajemy parametry startowe dla naszego kontenera np hasło do naszej bazy danych

### -d
kontener odpali sie w 'tle', bez tej opcji widzielibyśmy logi kontenera i po zamknieciu terminala kontener by sie wyłączył.

### -p
Port, na którym nasza aplikacja ma być dostępna. Pierwszy port to port naszego komputera, czyli publiczny port po którym np nasza aplikacja będzie się łączyć z naszą bazą danych (ten port może być dowolny).
Kolejny port to port kontenera, na którym udostępnia połączenie domyślnie dla mysql jest to port 3306.

### Jak sie połączyć z kontenerem?
```bash
docker exec nazwa_kontenera echo "test" //wykonanie komendy w kontenerze
```

#### -it
```bash
docker exec -it nazwa_kontenera /bin/bash
```
Logowanie interaktywne, czyli łączymy sie do 'serwera' jak po ssh.

## Automatyczne wdrażanie aplikacji z docker-compose

### Co to właściwie daje?
docker-compose pozwala nam automatycznie pobierać kontnery w odpowiedniej wersji (przydatne, wtedy kiedy cała aplikacja chcemy oprzeć o dockera i musimy pobrać np nginx, bazę, php, redis itp). Do tego otacza nasze kontenery siecią oraz serwerem dns i pozwala nam na komunikacje między nimi przez nazwe z konfiguracji np. mysql i automatycznie stworzy nasze wolumeny.

### Wolumeny
Jest to miejsce na naszym dysku, w którym przechowywane są dane z kontenera. Domyślnie, jeżeli mamy bazę danych to bez wolumena po wyłączeniu kontenera wszystkie dane nam znikną. Jeżeli podepniemy pod kontener wolumen dane będą przechowywane w tym miejscu i po restarcie dane wczytają się z wolumena.

#### Przykład bez docker-compose
```bash
docker volume create portainer_data
docker run --restart=always -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```

##### --restart=always
Po restarcie komputera ten kontener uruchomi sie samemu.

##### Portainer
Polecam zapoznać sie z tym softem :P. Jest to fajne UI dla dockera [aby zobaczyć *kliknij tutaj*](https://www.portainer.io/)

### Przykład
```yml
#docker-compose.yml
version: '3.1'

services:
nginx:
image: nginx:1.15.9
ports:
- "80:80"
volumes:
- ./:/var/www/project
- ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
links:
- db
- php

php:
image: zawiszaty/tutorials-tank-php:3.0.3
working_dir: /var/www/project
volumes:
- ./:/var/www/project
links:
- db

composer:
image: composer:1.8
working_dir: /var/www/project
volumes:
- ./:/var/www/project
links:
- php

db:
image: mysql:5.7
environment:
MYSQL_ROOT_PASSWORD: admin
volumes:
- db_data:/var/lib/mysql
- ./docker/mysql/my.cnf:/etc/mysql/my.cnf
ports:
- 3307:3307

phpmyadmin:
image: phpmyadmin/phpmyadmin
environment:
MYSQL_USERNAME: root
MYSQL_ROOT_PASSWORD: admin
PMA_PORT: "3307"
PMA_HOST: db
restart: always
ports:
- 8000:80
volumes:
- /sessions

volumes:
php:
db_data:
sessions:
```

#### version
Poszczególne wersje różnią się od siebie wiec musimy zdefiniować której używamy.

#### services
Tutaj ustalamy jakich kontenerów używamy.

##### links
Pozwalamy, aby kontenery widziały sie przez wewnętrzny dns

##### working_dir
W jakim katalogu znajdziemy sie po zalogowaniu do kontenera.

##### environment
Startowa konfiguracja naszego kontenera

##### volumes
Wewnątrz konfiguracji kontenera definiujemy, jakiego wolumena ma uzywać i co ma być w nim trzymane, ale także możemy 'ładować' jakiś plik przy starcie kontenera np konfiguracje nginx

#### volumes
Tutaj definiujemy jakie volumeny mają się utowrzyć w obrębie tego 'compose'

#### Jak to włączyć?
```bash
docker-compose up //jeżeli chcemy widzieć logi poszczególnych kontenerów, po zamknieciu terminala to konkretne compose sie wyłączy.
docker-compose up -d //uruchomi sie w tle.
```

#### Jak to zatrzymać?

```bash
docker-compose stop //wyłącza ale volumeny i sieć zostaje.
docker-compose down //usuwa wolumeny i sieć oraz kasuje kontener.
```

#### Logowanie sie do kontenera

```bash
docker-compose exec php /bin/bash
```

#### Wykonywanie polecenia w kontenerze bez logowania sie do niego

```bash
docker-compose exec php php bin/console
```

W obu używamy nazwy z services z docker-compose.yml

#### Inna nazwa
Domyślnie docker-compose szuka pliku docker-compose.yml jeżeli chcemy nazwać nasz plik inaczej albo chcemy mieć osobny plik np dla produkcji to dodajemy flage -f i po niej nazwe pliku np:

```bash
docker-compose -f docker-compose.prod.yml up -d
```

Po takiej zmianie przy każdym poleceniu musimy dawać flage -f, jeżeli chcemy korzystać z tego 'compose' np:
```bash
docker-compose -f docker-compose.prod.yml exec php php bin/console d:d:c
```

### Podsumowanie
Jak zwykle wpis jest tylko zbiorem moich luźnych przemyśleń, w których pokazuję jak można używać dockera. Ten wpis nie zrobi z ciebie specjalisty od dockera, ale wystarczy do swobodnego użytkowania przy gotowym projekcie, albo postawienia własnej konfiguracji. Musimy jednak pamiętać o tym, że docker podczas procesu developmentu może nam sporo ułatwić, ale musimy pamiętać, że nie są to prawdzie maszyny wirtualne i powinniśmy sie 2 razy zastanowić co umieść w konfiguracji produkcyjnej. W następnym wpisię poruszę temat pisania własnych obrazów dockera i trzymanie ich w docker hub.

Cały projekt używający docker-compose możecie obejrzeć tutaj
* [nginx i php](https://github.com/zawiszaty/symfony_simple_crud_example)
* [apache i php](https://github.com/ferdyrurka/devBook)

Polecam także, jeżeli korzystacie z Linuxa/Maca pisanie makefile pozwala zaoszczędzić czas i automatyzować rzeczy :P


