# Часть 1. API Umbrellio-blog

## Развёртывание приложения

### Подготовка

Для развёртывания в эмуляторе терминала введите:

```bash
  $ git clone https://github.com/Medvedu/umbrellio-blog
  $ cd $PROJECT_DIR
  $ docker-compose build
  $ docker-compose run backend sh -c "rake db:migrate"
```

где **$PROJECT_DIR** - путь, куда был склонирован данный репозиторий.

### Посев базы данных

```bash
  $ docker-compose run backend sh -c "rake db:seed"
```

**Внимание**, запись ведётся напрямую в базу, минуя dry-транзакции. Так сделано из-за существенного прироста в скорости посева базы.

_Параметры посева (см. ```db/seeds.rb```)_

```ruby
  USERS_COUNT = 400
  POSTS_COUNT = 600_000
  RATES_COUNT = 25_000
  IPS_COUNT = 75
  POSTS_WITH_RATES_COUNT = 15_000
```

### Запуск

```bash
  $ docker-compose up
```

После окончания загрузки приложение ```Umbrellio-blog``` будет доступно по адресу: ```http://localhost:3001```.

## API

### 1.1 Создание поста 

```bash
  $ curl -X POST \
    http://localhost:3001/api/v1/posts \
    -H 'Content-Type: application/json' \
    -H 'Host: localhost:3001' \
    -d '{
      "author_login" : "my_login",
      "title" : "my_title",
      "body" : "my_body"
  }'
```

_Вернёт:_ **```{"post":{"title":"my_title","body":"my_body"}}```** **10-12 ms**

Передавать все аттрибуты поста посчитал некорректным, т.к. с точки зрения безопасности приложения хорошо чтобы пользователь не видел свой id. Тем не менее, отображение аттрибутов настраивается в ```Api::V1::Post#viewable```. 

### 1.2 Создание поста (с ошибками валидаций)

```bash
  $ curl -X POST \
    http://localhost:3001/api/v1/posts \
    -H 'Content-Type: application/json' \
    -H 'Host: localhost:3001' \
    -d '{
      "author_login" : "my_login",
      "title" : ""
    }'
```

_Вернёт_:  **```{"errors":{"title":["must be filled"],"body":["is missing"]}}```** **10 ms**

### 2.1 Поставить оценку посту

```bash
  $ curl -X POST \
   http://localhost:3001/api/v1/rates \
   -H 'Content-Type: application/json' \
   -H 'Host: localhost:3001' \
   -d '{
     "rate" : 4,
     "post_id" : 25
   }'
```

_Вернёт_:  **```{"rate":4.5}```** **20 ms**

В виду примечания о конкурентных запросах обращение к базе данных выполняется на 4 уровне изоляции транзакций. 

### 2.2 Поставить оценку посту (с ошибками валидаций)

```bash
  $ curl -X POST \
    http://localhost:3001/api/v1/rates \
     -H 'Content-Type: application/json' \
     -H 'Host: localhost:3001' \
     -d '{
       "rate" : 3.5,
       "post_id" : 9870000123
     }'
```

Вернёт:  **```{"errors":{"rate":["must be an integer"],"post_id":["post not exist"]}}```**  **8 ms**

### 3. Получить топ постов по среднему рейтингу

```bash
  $ curl -X GET http://localhost:3001/api/v1/posts/top_by_avg_rate
```

Вернёт: Топ **555** постов по рейтингу. **88 ms** 

Запрос достаточно быстрый, но для больших ```N```, возможно, потребуется оптимизировать скорость генерации JSON-объекта (например, добавив гем [oj](https://github.com/ohler55/oj)).

### 4. Получить список авторов которые постили с одного айпи.

```bash
  $ curl -X GET http://localhost:3001/api/v1/posts/find_authors_with_same_ip
```

Вернёт: json на ~300 кбайт. **21ms** 

* Для ускорения запроса использовал [материализованные представления](https://postgrespro.ru/docs/postgrespro/11/rules-materializedviews). Представление обновляется асинхронно при помощи гема clockwork (см. ```config/clockwork```) раз в 45 секунд. 

* Для быстрого рендера JSON передаю массив логинов в строке.

## Тесты

### Перед запуском тестов выполните

```bash
  $ docker-compose run backend sh -c "RAILS_ENV=test rake db:migrate"
  $ docker-compose run backend sh -c "RAILS_ENV=test rake db:seed"
```

### Запуск

```bash
  $ docker-compose run backend sh -c "rspec"
```

## Задание

Желательно использовать версии: Ruby 2.5+, RoR 5+, PostgreSQL 10+
Результат лучше всего опубликовать на github.

Задание на знания Ruby on Rails:
У нас имеется некий блог со следующими сущностями:

1. Юзер. Имеет только логин.
2. Пост, принадлежит юзеру. Имеет заголовок, содержание, айпи автора (сохраняется
отдельно для каждого поста).
3. Оценка, принадлежит посту. Принимает значение от 1 до 5.

Задача: создать JSON API на RoR со следующими экшенами:

1. Создать пост. Принимает заголовок и содержание поста (не могут быть пустыми), а также
логин и айпи автора. Если автора с таким логином еще нет, необходимо его создать.
Возвращает либо атрибуты поста со статусом 200, либо ошибки валидации со статусом 422.

2. Поставить оценку посту. Принимает айди поста и значение, возвращает новый средний
рейтинг поста. Важно: экшен должен корректно отрабатывать при любом количестве
конкурентных запросов на оценку одного и того же поста.

3. Получить топ N постов по среднему рейтингу. Просто массив объектов с заголовками и
содержанием.

4. Получить список айпи, с которых постило несколько разных авторов. Массив объектов с
полями: айпи и массив логинов авторов.

Базу данных используем PostgreSQL. Для девелопмента написать скрипт в db/seeds.rb,
который генерирует тестовые данные. Часть постов должна получить оценки. Скрипт должен
использовать тот же код, что и контроллеры, можно вообще дергать непосредственно
сервер курлом или еще чем-нибудь.

Постов в базе должно быть хотя бы 200к, авторов лучше сделать в районе 100 штук,
айпишников использовать штук 50 разных. Экшены должны на стандартном железе
работать достаточно быстро как для указанного объема данных (быстрее 100 мс), так и для
намного большего, то есть нужен хороший запас в плане оптимизации запросов. Для этого
можно использовать денормализацию данных и любые другие средства БД. Можно
использовать любые нужные гемы, обязательно наличие спеков, хорошо покрывающих
разные кейсы. В коде желательно не использовать рельсовых антипаттернов типа колбеков
и валидаций в моделях, сервис-классы наше все. Также желательно не использовать
генераторов и вообще обойтись без лишних мусорных файлов в репозитории.

# Часть 2. Составить SQL запрос.

```sql
  WITH cte AS (
    SELECT id, 
           group_id, 
           (id - ROW_NUMBER() OVER (PARTITION BY group_id ORDER BY id)) AS column_num_in_partition
    FROM users
  )
  SELECT MIN(id) AS min_id, 
         group_id, 
         COUNT(column_num_in_partition)
  FROM cte
  GROUP by group_id, column_num_in_partition
  ORDER BY min_id ASC
```

Проверить работу запроса можно [здесь](http://sqlfiddle.com/#!15/8c1b8/1/0). 
 
## Задание

Дана таблица users вида - id, group_id

create temp table users(id bigserial, group_id bigint);
insert into users(group_id) values (1), (1), (1), (2), (1), (3);

В этой таблице, упорядоченной по ID необходимо:

Выделить непрерывные группы по group_id с учетом указанного порядка записей (их 4).
Посчитать количество записей в каждой группе.
Вычислить минимальный ID записи в группе.
