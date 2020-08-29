---
layout: post
title:  "#4 Autmatyczne budowanie i publikacja obrazów dockera przy uzyciu circleci"
date:   2019-09-23 18:03:36 +0530
categories: PHP Docker
image: thumbnails.jpg
comments: true
---
Niejednokrotnie pewnie musieliście pushować swoje obrazy do jakiegoś rejestru np docker hub. Teraz możecie to zrobić nieco sprytniej. 

# Piszemy swój Dockerfile
Dla przykładu uzyje podstawowego dockerfila dla PHP.
```php
//Dockerfile
FROM php:7.3.6-fpm

RUN docker-php-ext-install pdo_mysql
```
Przy przeciętnej ścieżce budujemy sobie ten kontener, nadajemy tag i pushujemy do docker huba.

My jednak podepniemy CI pod nasz obraz.

# Tworzymy konfiguracje CircleCI

```yml
version: 2.1
executors:
  docker-publisher:
    environment:
      IMAGE_NAME: zawiszaty/php7 #nazwa naszego obrazu w docker hubie
    docker:
      - image: circleci/buildpack-deps:stretch
jobs:
  build:
    executor: docker-publisher
    steps:
      - checkout
      - setup_remote_docker
      - run:
          name: Build Docker image
          command: |
            docker build -t $IMAGE_NAME:latest .
      - run:
          name: Archive Docker image
          command: docker save -o image.tar $IMAGE_NAME
      - persist_to_workspace:
          root: .
          paths:
            - ./image.tar
  publish-latest:
    executor: docker-publisher
    steps:
      - attach_workspace:
          at: /tmp/workspace
      - setup_remote_docker
      - run:
          name: Load archived Docker image
          command: docker load -i /tmp/workspace/image.tar
      - run:
          name: Publish Docker Image to Docker Hub
          command: |
            echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
            IMAGE_TAG="latest"
            docker tag $IMAGE_NAME:latest $IMAGE_NAME:$IMAGE_TAG
            docker push $IMAGE_NAME:latest
            docker push $IMAGE_NAME:$IMAGE_TAG
  publish-tag:
    executor: docker-publisher
    steps:
      - attach_workspace:
          at: /tmp/workspace
      - setup_remote_docker
      - run:
          name: Load archived Docker image
          command: docker load -i /tmp/workspace/image.tar
      - run:
          name: Publish Docker Image to Docker Hub
          command: |
            echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin
            IMAGE_TAG=${CIRCLE_TAG/v/''}
            docker tag $IMAGE_NAME:latest $IMAGE_NAME:$IMAGE_TAG
            docker push $IMAGE_NAME:latest
            docker push $IMAGE_NAME:$IMAGE_TAG
workflows:
  version: 2
  build-master:
    jobs:
      - build:
          filters:
            branches:
              only: master
      - publish-latest:
          requires:
            - build
          filters:
            branches:
              only: master
  build-tags:
    jobs:
      - build:
          filters:
            tags:
              only: /^v.*/
            branches:
              ignore: /.*/
      - publish-tag:
          requires:
            - build
          filters:
            tags:
              only: /^v.*/
            branches:
              ignore: /.*/
```
w niektórych miejscach możemy zaobserwować coś takiego 'echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USERNAME" --password-stdin'

'$DOCKERHUB_PASS' i '$DOCKERHUB_USERNAME' musimy podać nasz login i hasło do dockera, zapisujemy je jednak w zmiennych środowiskowych w circlci. A dokładniej
w zakładce 'Environment Variables' w ustawieniach konkretnego projketu.

Po kazdym pushu do gita autmatycznie zostania opublikowana wersja latest naszego obrazu.

Konkretne wersje np '7.3.6' tworzymy poprzez release np tworzymy release o tagu 'v7.3.6'(w moim configu ważne jest aby release zaczynał się od 'v.' inaczej nie zadziała :P, jeżeli chcecie inaczej, to edytujcie to w build-tags) i automatycznie nasze CI opublikuje nasz obraz pod tagiem '7.3.6'.

# Podsumowanie
Widać, że nie jest to trudna rzecz, a może nam uprzyjemnić prace z naszymi obrazami. Przykładowy projekt możecie zobaczyć pod [linkiem](https://github.com/zawiszaty/php7-docker)