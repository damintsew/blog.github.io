---
layout: post
title:  "Intellij Idea speed up project indexing"
date:   2022-04-23 15:00:00 +0300
categories: indellij-idea project-shared-index shared-index
---

Большое количество разрабочиков пользуются различными IDE от компании JetBrains. Во всех IDE существуют механизмы
индексации кода и иногда этот процесс становится очень долгим, при этом заблокирована часть функционала, например 
невозможность рефакторинга или части навигации.

Недавно в версии (номер версии) появилась возможность заранее построить индекс, и затем его переиспользовать для
ускорения
индексации проекта. На данный момент существуют индексы, на CDN от JetBrains, для большого количества Java
библиотек (!!!! привести ссылку)

Точно также в IDE появилась возможность создавать проектные индексы (добавить скриншоты). В данной статье я покажу как
это можно создать эти индексы задеплоить их на сервер и сделать так, чтобы код не ушел за пределы интранета.

Для генерации индексов нам потребуется 3 действия:

1. Сгенерить индексы по существующему коду
2. Сгенерить метаинформацию с помощью библиотеки от JetBrains
3. Указать IDE место, где лежат индексы

[Руководство от JetBrains](https://www.jetbrains.com/help/idea/shared-indexes.html#shared-jdk-indexes), где описано как
строить индексы руками.

Делать это каждый раз в ручном режиме - бессмыслено. Поэтому я представляю свое решение для автоматизированной генерации 
и загрузки индексов:
1. запуск [docker контейнер](https://github.com/damintsew/idea-shared-index-dockerfile),
который отвечает за генерацию индексов и их загрузку
2. [сервер](https://github.com/damintsew/idea-shared-index-sever), отвечающий за
хранение индексов.

Давайте рассмотрим из чего состоит докер контейнер и сервер 

#### Docker container

Для запуска контейнера нужно сделать несколько действий. Надо подмонтировать исходные коды. По умолчанию задана директория 
`ENV PROJECT_DIR="/var/project"`и сделать это можно так `-v "path/on/local/machine:/var/project"`. Если необходимо, то можно 
переопределить директорию передав переменную окружения `PROJECT_DIR`.  
Для создания индексоос требуются параметры `PROJECT_ID` и `COMMIT_ID`.
Параметр `INDEXES_CDN_URL` опциональный, и если его указать, то результаты работы контейнера будут загружены на сервер. 
В противном случае результат будет в папке `SHARED_INDEX_BASE="/shared-index/output` и его можно достать 
смапив папку через volume передав параметр `-v "path/on/local/machine:/shared-index/output"`

Основные части контейнера:

- Intellij Idea
  <details>
  <summary>Dockerfile, который описывает</summary>

    ```dockerfile
    ENV IDEA_PROJECT_DIR="/var/project"
    ENV SHARED_INDEX_BASE="/shared-index"
    
    RUN wget https://download-cf.jetbrains.com/idea/${INTELLIJ_IDE_TAR} -nv && \
    tar xzf ${INTELLIJ_IDE_TAR} && \
    tar tzf ${INTELLIJ_IDE_TAR} | head -1 | sed -e 's/\/.*//' | xargs -I{} ln -s {} idea && \
    rm ${INTELLIJ_IDE_TAR} && \
    echo idea.config.path=/etc/idea/config >> /opt/idea/bin/idea.properties && \
    echo idea.log.path=/etc/idea/log >> /opt/idea/bin/idea.properties && \
    echo idea.system.path=/etc/idea/system >> /opt/idea/bin/idea.properties && \
    chmod -R 777 /opt/idea && \
    chmod -R 777 ${SHARED_INDEX_BASE} && \
    chmod -R 777 /etc/idea
    ```
    
  </details>

- Скрипт `run_shared_index.sh`
    - <details>
        <summary>запускает построение индексов</summary>

        ```shell
        CMD /opt/idea/bin/idea.sh dump-shared-index project \
           --project-dir=${IDEA_PROJECT_DIR} \
           --project-id=${PROJECT_ID} \
           --commit-id=${COMMIT_ID} \
           --tmp=${SHARED_INDEX_BASE}/temp \
           --output=${SHARED_INDEX_BASE}/output
        ```
      </details>
    
    - <details>
        <summary>упаковывает результаты в архив</summary>

        ```shell
        # Zip files. Prepare for sending
        echo "Zipping output"
        cd ${SHARED_INDEX_BASE}/output && zip -r /opt/zipped_index.zip ./* && cd -
        ```
      </details>
    - <details>
        <summary>загружает данные на сервер (если указана ссылка `INDEXES_CDN_URL`)</summary>

        ```shell
        echo "Sending output to Server"
        curl -X POST "${INDEXES_CDN_URL}/manage/upload?commitId=${COMMIT_ID}&projectId=${PROJECT_ID}" \
            -H "accept: */*" \
            -H "Content-Type: multipart/form-data" \
            -F "file=@/opt/zipped_index.zip"
        ```
      </details>

В результате выполнения скрипта наш индекс будет загружен на сервер.

Пример запуска:


#### Сервер хранения индексов

Сервер представляет из себя spring-boot приложение, которое хранит индексы в локальной файловой системе.
Хранение индексов в S3 пока еще в разработке. 

Для локального запуска нужно скачать репозиторий, сделать `mvn clean verify` и запустить с профилем `local`. 

Существует докер образ, который можно развернуть. Для этого выполнить команду
```shell
docker run damintsew/shared-index-server:latest \
  -e BASE-URL=http://localhost:8125/shared-index
  -e SPRING_PROFILES_ACTIVE=docker \
  -v "path/to/shared/index/directory:/var/data"
```

Будет запущен сервер, в который отправляются индексы (генерируемые докером) и также принимает запросы из Идеи. 
Теперь нам надо создать конфиг в идее, чтобы указать IDE где искать проектные индексы.

