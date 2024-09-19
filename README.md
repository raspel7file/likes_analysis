# Анализ количества лайков под постами
## Задача
Взять страницу Вконтакте, собрать по ней таблицу с датой постов и количеством лайков и ответить на вопрос: что больше всего влияет на количество лайков: время суток публикации, день недели или промежуток между постами?

## Сбор данных
Сначала вытащим из ВК необходимые данные. Моя страница не активна, так что возьмём последние 1000 постов из группы [Северодвинск life](https://vk.com/severodvinsk_life) Процесс можно посмотреть в файле likes_analysis.ipynb.

## Анализ данных в Python
Я считаю, анализ данных также проще делать в Python. В файле likes_analysis.ipynb можно увидеть, что я делаю дальше: привожу данные к удобному виду, рассчитываю необходимые показатели, с помощью дисперсионного анализа оцениваю влияние на число лайков времени суток и дня недели публикации, а с помощью линейной регрессии - влияние промежутка между постами.

## Анализ данных в SQL
### Исходные данные
Данные для анализа лежат в файле posts_data.csv. Будем считать, что у нас есть аналогичная SQL-таблица posts_data со столбцами:
- **post_time** UInt32, время создания поста в UNIX-time;
- **like_count** UInt32, число лайков под постом.
Писать запросы я буду на диалекте Clickhouse.
### Влияние времени суток создания поста на число лайков 
#### Код
```SQL
SELECT
    CASE 
        WHEN 
                EXTRACT (HOUR FROM toTimezone(toDateTime(post_time), 'Europe/Moscow')) >= 6
            AND EXTRACT (HOUR FROM toTimezone(toDateTime(post_time), 'Europe/Moscow')) < 11
        THEN 'Утро'
        WHEN 
                EXTRACT (HOUR FROM toTimezone(toDateTime(post_time), 'Europe/Moscow')) >= 11
            AND EXTRACT (HOUR FROM toTimezone(toDateTime(post_time), 'Europe/Moscow')) < 18
        THEN 'День'
        WHEN 
                EXTRACT (HOUR FROM toTimezone(toDateTime(post_time), 'Europe/Moscow')) >= 18
            AND EXTRACT (HOUR FROM toTimezone(toDateTime(post_time), 'Europe/Moscow')) < 23
        THEN 'Вечер'
        ELSE 'Ночь'
    END AS time_of_day,
    round(AVG(like_count), 2) AS average_like_count
FROM posts_data
GROUP BY time_of_day
```
#### Результат
| time_of_day | average_like_count |
|---|---|
|Утро|159.39|
|День|112.67|
|Ночь|201.17|
|Вечер|168.42|
#### Вывод
Кажется, число лайков под постами, опубликованными утром и вечером слабо отличается. Под дневными постами лайков меньше, под ночными больше. Подтвердить это статистическими тестами можно в Python (см. likes_analysis.ipynb).

### Влияние дня недели создания поста на число лайков
#### Код
```SQL
SELECT
    CASE day_of_week
        WHEN 1 THEN 'Monday'
        WHEN 2 THEN 'Tuesday'
        WHEN 3 THEN 'Wednesday'
        WHEN 4 THEN 'Thursday'
        WHEN 5 THEN 'Friday'
        WHEN 6 THEN 'Saturday'
        ELSE 'Sunday'
    END AS day_of_week,
    average_like_count
FROM (
    SELECT
        toDayOfWeek(toTimezone(toDateTime(post_time), 'Europe/Moscow')) AS day_of_week,
        ROUND(AVG(like_count), 2) AS average_like_count
    FROM posts_data
    GROUP BY day_of_week
)
```
#### Результат
| day_of_week | average_like_count |
|---|---|
|Monday|157.23|
|Tuesday|113.53|
|Wednesday|135.90|
|Thursday|120.52|
|Friday|118.94|
|Saturday|189.69|
|Sunday|203.54|
#### Вывод
Кажется, по выходным лайков больше. Подтвердить это статистическими тестами можно в Python (см. likes_analysis.ipynb).

### Влияние промежутка между постами на число лайков
#### Код
Извините за костыли, но оказалось, что в Clickhouse нет функции LAG()
```SQL
SELECT
    ROUND(corr(time_difference, like_count), 4) AS correlation
FROM (
    SELECT
        post_time,
        like_count,
        post_time - previous_time AS time_difference
    FROM (
        SELECT
            post_time,
            like_count,
            any(post_time) OVER (
                ORDER BY post_time
                ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING
            ) AS previous_time
        FROM 
            posts_data
    )
    WHERE previous_time != 0
)
```
#### Результат
| correlation |
|---|
|-0.0196|
#### Вывод
Кажется, есть слабая обратная зависимость между длительностью промежутка между постами и числом лайков. Подтвердить это статистическими тестами можно в Python (см. likes_analysis.ipynb).

## Итоги работы
Анализ постов в SQL и Python показал:
- статистически значимого влияния времени, прошедшего с предыдущего поста, на число лайков нет;
- статистически значимо меньшее количество лайков получают посты, опубликованные днём;
- статистически значимо большее количество лайков получают посты, опубликованные в выходные дни.
