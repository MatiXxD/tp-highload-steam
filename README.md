# Проектирование высоконагруженного сервиса Steam

## Содержание
- [1. Тема и целевая аудитория](#1-тема-и-целевая-аудитория)
  - [Целевая аудитория](#целевая-аудитория)
  - [Функционал](#функционал)
- [2. Расчет нагрузки](#2-расчет-нагрузки)
  - [Продуктовые метрики](#продуктовые-метрики)
    - [Средний размер хранилища](#средний-размер-хранилища)
    - [Действия пользователей](#действия-пользователей)
  - [Технические метрики](#технические-метрики)
    - [Хранилище](#хранилище)
    - [Сетевой трафик](#сетевой-трафик)
    - [RPS](#rps)
- [3. Глобальная балансировка нагрузки](#3-глобальная-балансировка-нагрузки)
  - [Функциональное разбиение по доменам](#функциональное-разбиение-по-доменам)
  - [Расположение ДЦ](#расположение-дц)
  - [Расчет распределение запросов по ДЦ](#расчет-распределение-запросов-из-секции-расчет-нагрузки-по-типам-запросов-по-датацентрам)
  - [Схема балансировки](#схема-балансировки)
- [4. Локальная балансировка нагрузки](#4-локальная-балансировка-нагрузки)
  - [Схемы балансировки для входящих и межсервисных запросов](#схемы-балансировки-для-входящих-и-межсервисных-запросов)
  - [Схема отказоустойчивости](#схема-отказоустойчивости)
  - [Нагрузка по терминации SSL](#нагрузка-по-терминации-ssl)
- [5. Логическая схема БД](#5-логическая-схема-бд)
  - [Схема](#схема)
  - [Расчет размеров](#расчет-размеров)
  - [Требования к консистентности](#требования-к-консистентности)
  - [Шардирование](#шардирование)
- [6. Физическая схема БД](#6-физическая-схема-бд)
  - [Схема после денормализации](#схема-после-денормализации)
  - [Индексы](#индексы)
  - [Выбор СУБД](#выбор-субд)
  - [Шардирование и резервирование СУБД](#шардирование-и-резервирование-субд)
  - [Клиентские библиотеки / интеграции](#клиентские-библиотеки--интеграции)
- [7. Алгоритмы](#7-алгоритмы)
  - [Коллаборативная фильтрация](#коллаборативная-фильтрация)
    - [Подробнее про алгоритм (Item-based)](#подробнее-про-алгоритм-item-based)
  - [Кэширование и дедупликация для VPK (Valve Package System)](#кэширование-и-дедупликация-для-vpk-valve-package-system)
- [8. Технологии](#8-технологии)
- [Список источников](#список-источников)

## 1. Тема и целевая аудитория
Steam — это крупнейшая платформа для PC-гейминга, где можно покупать игры, играть в бесплатные проекты, торговать внутриигровыми предметами и общаться с другими игроками.

### Целевая аудитория
Согласно сайту Steamworks MAU Steam состовляет 132M[^1], также на сайте Steam можно посмотреть количество пользователей, которые находятся online[^2].

#### Распределение по странам
Также Steam предоставляет статистику по количеству пользователей в зависимости от страны[^3].
![](imgs/usermap.png)

**Количество пользователей Steam в зависимости от страны[^4]:**
| Страна            | Пользователи Steam |
|-------------------|---------------------|
| Соединенные Штаты | 13.7M              |
| Китай             | 11.4M              |
| Россия            | 9.5M               |
| Бразилия          | 4.9M               |
| Германия          | 3.6M               |
| Канада            | 3M                 |
| Турция            | 2.8M               |
| Франция           | 2.8M               |
| Великобритания    | 2.6M               |
| Польша            | 2.4M               |

### Функционал 
Основной функционал сервиса **заключается в покупке игр**.

**Функционал MVP**:
- Регистрация и авторизация
- Корзина для покупки игр
- Каталог игр для покупки: фильтры по жанрам, поиск, рекомендации
- Библиотека пользователя, которая позволяет запускать/устанавливать игры
- Торговая площадка для игровых предметов
- Система профилей: можно добавлять друзей, чтобы играть с ними по сети
- Интеграция Steam в игры: статистика по играм для пользователя, ачивки, мультиплеер, игровые предметы

**Ключевые продуктовые решения**:
- **Steamworks API**: используется для интеграции функций Steam в игры, реализован на основе HTTP и gRPC[^1];
- **Steam Guard (двухфакторная аутентификация)**: использует мобильное приложение Steam для генерации кодов[^5];

## 2. Расчет нагрузки

### Продуктовые метрики
- Месячная аудитория (MAU): 132M[^1]
- Дневная аудитория (DAU): 69[^6]

#### Средний размер хранилища
**Профиль пользователя:**
| Хранимые данные         | Средний размер |
|-------------------------|----------------|
| Аватарка                | 15 кб          |
| Скриншоты               | 7.5 мб         |
| Информация профиля      | 2 мб           |

- при загрузке автатарка пользователя автоматически рескейлится до 128x128 px, средний размер получается ~15 кб, максимальный возможный размер -- 1 мб; 
- размеры скриншотов варируются от 50 кб до 1.5 мб, в среднем получается где-то 250 кб на один скриншот, в среднем у пользователя ~30 скриншотов; 
- информация пользователя: описание, ачивки, библиотека игр, список друзей, комментарии профиля...

**Игровой предмет:**
| Хранимые данные         | Средний размер |
|-------------------------|----------------|
| Картинка                | 70 кб          |
| Информация о предмете   | 100 кб           |

- размер картинки может меняться в зависимости от игры, в среднем получается 70 кб;
- информация о предмете: описание, редкость, из какой игры...

**Игра:**
| Хранимые данные         | Средний размер |
|-------------------------|----------------|
| Скриншоты и трейлеры    | 15 мб          |
| Информация об игре      | 1 мб           |
| Файлы игры              | 25 гб          |

- в среднем у игры 3 трейлера (средний размер трейлера 3.5 мб) и 15 скриншотов (средний размер 250кб), если округлять получится ~15 мб на все; 
- информация об игре: описание, тэги, системные требования...;

#### Действия пользователей
| Действие                      | Количество (раз в день)  |
|-------------------------------|--------------------------|
| Регистрация/авторизация       | 0.0323                   |
| Просмотр игр [^7]             | 2.8000                   |
| Покупка игр [^8]              | 0.0075                   |
| Комментарий к игре [^9]       | 0.0002                   |
| Скачивание игр                | 0.0301                   |
| Обновление игр                | 1.5714                   |
| Совершить трейд [^10]         | 0.3623                   |
| Просмотр профилей             | 2.5000                   |
| Просмотр игровых предметов    | 5.0000                   |
| Обновление информации профиля | 0.0323                   |
| Загрузка скриншотов/видео     | 0.1612                   |
| Обновление статуса            | 6.0000                   |
| Мультиплеер                   | 0.8775                   |

- пользователь в среднем авторизируется в Steam 1 раз в месяц;
- согласно SteamSpy[^8] (статистика за 2017) на пользователя, в среднем, приходится ~11 игр, а средний возраст аккаунтов ~4 лет;
- согласно [^9] медианное значение множителя для игр ~40, т.е. на один отзыв приходится 40 проданных игр;
- если учесть, что у пользователя в среднем появляется 11 игр за год, а также, то что в steam игры обычно обновляются раз в неделю, получим, что в среднем пользователь обновляет 11 игр в неделю;
- согласно [^10] за последний месяц было в среднем 2.5M трейдов в день => количество трейдов = 2.5 / DAU;
- согласно [^15], в среднем, пользователи играют 3 часа в день, одна игровая сессия длиться ~1 час, получаем, что пользователь запускает игру 3 раза в день. Для игры по сети Steam использует `ISteamFriends` из steamworks api [^14], когда пользователь запускает игру этот интерфейс позволяет обновить его статус (показать что он играет в игру), после этого его друзья могут отправить запрос на совместную игру, статус также обновляется при выходе из игры;
- для того, чтобы играть по сети надо добавить пользователя в группу, одна мультиплеер сессия длится ~40 минут, согласно [^16] отношение времени, проведенному в онлайн играх, ко времени в одиночных играх $595 / 1509 \approx 0.39$, если учитывать, что 50% игроков играют в онлайн игры с друзьями получаем: $180 / 40 * 0.39 * 0.5 \approx $ для оценки мультиплеера


### Технические метрики

#### Хранилище
- на данный момент в Steam ~130K игр [^11] => для хранения игр понадобится: 
$130000 \times (25гб + 16мб) \approx 3.25208пб$
- согласно [^12] на 2020 год было ~667M аккаунтов, пририост начиная с 2017 ежигодны прирост пользователей ~63M в год [^8], получаем 919M пользователей на конец 2024 года => на хранение данных пользователей понадобится:
$919 \times 10^6 \times (9.5мб + 15кб) \approx 8.744285пб$
- количество уникальных предметов в Steam Market ~$10^5$ на хранение надо:
$10^5 \times (100кб + 70кб) \approx 17гб$

**Итог: 11.996382 пб** 

#### Сетевой трафик
**DAU = 69M**

Расчет общей нагрузки (в Гбит/с):
- Просмотр игр:
$2.8DAU \times 16мб \div 86400 с \approx 35.8 гб/c = 286.4 гбит/с$
- Просмотр профиля:
$2.5DAU \times (2мб + 15кб) \div 86400 с \approx 4.0 гб/c = 32.0 гбит/с$
- Просмотр предметов:
$5DAU \times 170кб \div 86400 с \approx 0.679 гб/c = 5.432 гбит/с$
- Скачивание игр:
$0.0301DAU \times 25гб \div 86400 с \approx 600.95 гб/c = 4807.6 гбит/с$
- Обновление игр:
$1.5714DAU \times 0.05 \times 25гб \div 86400 с \approx 1568.67 гб/c = 12549.36 гбит/с$
- Покупка игры:
$0.0075DAU \times 1мб \div 86400 с \approx 0.006 гб/c = 0.048 гбит/с$

Для пика был выбран множитель x2 согласно [^13].

| Действие             | Общая нагрузка<br>[Гбит/с] | Общая нагрузка (пик)<br>[Гбит/с] | Суточная нагрузка<br>[Гбайт/сутки] |
|----------------------|----------------------------|----------------------------------|------------------------------------|
| Просмотр игр         | 286.4                      |  572.8                           | 3 093 120                          |
| Просмотр профиля     | 32.0                       |  64.0                            | 345 600                            |
| Просмотр предметов   | 5.432                      |  10.864                          | 58 604.4                           |
| Скачивание игр       | 4807.6                     |  9615.2                          | 51 922 080                         |
| Обновление игр       | 12549.36                   |  25098.72                        | 135 533 088                        |
| Покупка игры         | 0.048                      |  0.096                           | 0.5184                             |

#### RPS
Использую таблицу с действиями пользователя можно посчитать RPS: 
$RPS = actionCount \times DAU \div 86400$

| Действие                      | RPS        | RPS (пик)       |
|-------------------------------|------------|-----------------|
| Регистрация/авторизация       | 25.80      | 51.60           |
| Просмотр игр                  | 2236.11    | 4472.22         |
| Покупка игр                   | 5.99       | 11.98           |
| Комментарий к игре            | 0.16       | 0.32            |
| Скачивание игр                | 24.04      | 48.08           |
| Обновление игр                | 1254.94    | 2509.88         |
| Совершить трейд               | 289.34     | 578.68          |
| Просмотр профилей             | 1996.53    | 3993.06         |
| Просмотр игровых предметов    | 3993.06    | 7986.12         |
| Обновление информации профиля | 25.80      | 51.60           |
| Загрузка скриншотов/видео     | 128.74     | 257.47          |
| Обновление статуса            | 4791.67    | 9583.34         |
| Мультиплеер                   | 700.78     | 1401.56         |


## 3. Глобальная балансировка нагрузки

### Функциональное разбиение по доменам
Будем считать, что основным доменом будет `steam.com`, имеет смысл сделать отдельный домен для раздачи игр `content.steam.com`.

### Расположение ДЦ
Steam отрыто предоставляет статистику скачиваний в зависимости от региона [^13]:
![bandwidth-used](imgs/bandwidth-used.png)

**Пример расположения серверов:**
Для обслуживания `steam.com`:
- *Северная Америка:* Ашберн, Даллас, Чикаго, Лос-Анджелес, Торонто, Ванкувер, Майами;
- *Европа:* Лондон, Париж, Стокгольм, Амстердам;
- *Азия:* Токио, Сингапур, Мумбаи;
- *Россия:* Москва;
- *Южная Америка:* Сан-Паулу, Богота;
- *Океания:* Сидней;
- *Ближний Восток:* Дубай.

Для обслуживания `content.steam.com`:
- *Северная Америка:* Сиэтл;
- *Европа:* Франкфурт.

![dc-map](imgs/dc-map.png)
*Синий цвет ДЦ, которые обслуживают `steam.com`, красный -- `content.steam.com`.*

### Расчет распределение запросов из секции "Расчет нагрузки" по типам запросов по датацентрам

| Регион                  | Использование (%) |
|-------------------------|-------------------|
| **Северная Америка**    | 39.67             |
| **Европа**              | 23.55             |
| **Азия**                | 13.64             |
| **Россия**              | 3.61              |
| **Южная Америка**       | 11.98             |
| **Океания**             | 2.20              |
| **Ближний Восток**      | 4.13              |
| **Африка**              | 0.43              |
| **Центральная Америка** | 0.60              |

Используя эту таблицу и данные из [расчета нагрузки](#2-расчет-нагрузки), получаем:

| Регион               | Общий RPS  |
|----------------------|------------|
| Северная Америка     | 12275.81   |
| Европа               | 7287.84    |
| Азия                 | 4221.45    |
| Россия               | 1116.84    |
| Южная Америка        | 3707.15    |
| Океания              | 680.82     |
| Ближний Восток       | 1277.31    |
| Африка               | 132.91     |
| Центральная Америка  | 183.67     |

### Схема балансировки
Т.к. Steam-ом пользуются во всем мире имеет смысл применять Geo-based DNS.

Можно выделить следующие регионы с большим трафиком: **Северная Америка**, **Европа**, **Азия**. Для этих регионов имеет смысл использовать bgp anycast для равномерного распределения пользователей по ДЦ. В рамках каждого региона все дата-центры будут использовать один общий IP-адрес в глобальной сети.


## 4. Локальная балансировка нагрузки

### Схемы балансировки для входящих и межсервисных запросов
После того, как запрос приходит в ДЦ начинается локальная балансировка. Локальная балансировка будет включать два уровня:
- **L4-балансировка:** будет реализована с помощью IPVS (Linux Virtual Server)
- **L7-балансировка:** будет использовать ingress-контроллер (ingerss-nginx)

Для каждого региона будет использоваться свой kubernetes кластер, когда запрос приходит в ДЦ он поподает на L4 балансировщик, который знает только про ingress-nginx, которые находятся в данном ДЦ. L4 балансировщик выбирает (round-robin) одну из нод в кластере kubernetes, где запущен ingress-nginx. Ingress-nginx балансирует запросы на уровне L7, отправляя на нужный service, который выбирает конкретный под.

### Схема отказоустойчивости
Для того, чтобы обеспечить отказоустойчивость на L4 слое будем использовать keepalived. В случае L7 балансировки будем использовать специальный endpoint (`/health`) для проверки статуса сервера.

Кроме этого для обеспечения отказоустойчивости будут использоваться средства kubernetes: автоматический перезапуск упавших контейнеров, автоматическое масштабирование подов при росте нагрузки, добавление новых узлов в кластер при нехватке ресурсов.

### Нагрузка по терминации SSL
SSL терминацию будем делать на уровне L7 (ingress-nginx). Чтобы снизить количество round-tripов будет использовать session tickets.

Для того, чтобы установить TLS соединение надо ~120мс, пиковое значение rps -- 30946. При использование session tickets подключение будет занимать ~60мс. Пусть для 5% запросов не сохранили сессию, тогда для терминации по SSL каждую секунду надо:
$30946 \times (0.95 \times 0.06 + 0.05 \times 0.12) \approx 1949 с$.
Тут имеется ввиду, что надо 1949с процессороного времени => 1949 ядер.


## 5. Логическая схема БД

### Схема
![db-scheme](imgs/db-scheme.svg)

### Расчет размеров
Можно рассчитать размер одной записи для каждой таблицы:
| Таблица                   | Размер одной записи (байт) |
|---------------------------|----------------------------|
| User                      | 633                        |
| MultiplayerSession        | 78                         |
| MultiplayerSessionUser    | 148                        |
| Session                   | 148                        |
| Game                      | 445                        |
| GamePrice                 | 50                         |
| Developer                 | 217                        |
| UserGame                  | 56                         |
| Achievement               | 333                        |
| UserAchievement           | 40                         |
| Screenshot                | 257                        |
| UserRelationship          | 68                         |
| GameFile                  | 233                        |
| Item                      | 433                        |
| UserItem                  | 48                         |
| Trade                     | 68                         |
| TradeItem                 | 48                         |
| Review                    | 269                        |
| OrderItem                 | 364                        |
| ItemPrice                 | 54                         |
| OrderGame                 | 248                        |
| OrderGameCart             | 40                         |
| Wallet                    | 58                         |

Для расчетов размер полей типа `TEXT` = 100 байт.

Теперь можно рассчитать размер всех данных:
| Таблица               | Суммарный размер (байт)               |
|------------------------|----------------------------------------|
| User                  | 919 * 10^6 * 633 = 541.78 ГБ           |
| MultiplayerSession    | 7 * 60 * 10^6 * 78 = 30.51 ГБ          |
| MultiplayerSessionUser| 3 * 7 * 60 * 10^6 * 148 = 173.67 ГБ      |
| Session               | 132 * 10^6 * 148 = 18.19 ГБ            |
| Game                  | 130000 * 445 = 55.17 МБ                |
| GamePrice             | 10 * 130000 * 50 = 61.99 МБ            |
| Developer             | 44000 * 217 = 9.11 МБ                   |
| UserGame              | 2 * 11 * 919 * 10^6 * 56 = 1.03 ТБ     |
| Achievement           | 130000 * 25 * 333 = 1.01 ГБ          |
| UserAchievement       | 919 * 10^6 * 2 * 11 * 25 * 0.2 * 40 = 3.66 ТБ |
| Screenshot            | 919 * 10^6 * 200 * 257 = 42.96 ТБ      |
| UserRelationship      | 919 * 10^6 * 10 * 68 = 582.00 ГБ       |
| GameFile              | 130000 * 233 = 28.89 МБ                |
| Item                  | 10000 * 433 = 4.13 МБ                  |
| UserItem              | 919 * 10^6 * 100 * 48 = 4.01 ТБ        |
| Trade                 | 2.5 * 10^6 * 31 * 24 * 68 = 117.79 ГБ   |
| TradeItem             | 8 * 2.5 * 10^6 * 31 * 24 * 48 = 665.19 ГБ |
| Review                | 130000 * 100 * 269 = 3.26 ГБ           |
| OrderItem             | 132 * 10^6 * 100 = 43.7 ГБ             |
| ItemPrice             | 10000 * 10 * 54 = 5.15 МБ              |
| OrderGame             | 11 * 130000 * 248 = 338.21 МБ          |
| OrderGameCart         | 2 * 11 * 130000 * 40 = 109.10 МБ       |
| Wallet                | 919 * 10^6 * 58 = 49.64 ГБ             |

### Требования к консистентности
- Пользователи: При удалении пользователя необходимо удалить все связанные данные (отзывы, скриншоты, ...);
- Финансовые операции: Покупки, транзакции и пополнения кошелька должны быть атомарными и неизменяемыми, чтобы избежать дублирования или потери данных;
- Игры: При удаление игры надо удалять все связанные с ней данные (отзывы, скриншоты, ...)
- Отзывы и рейтинг игр: Когда пользователь добавляет отзыв, надо пересчитывать рейтинг игры;
- Обмен предметами: Нельзя обменять предмет, если он уже был передан другому пользователю. Надо изменять инвентари пользователей после обмена;
- Достижения: Игрок не может получить достижение, если у него нет соответствующей игры;

### Шардирование
- Пользователи: Поскольку количество пользователей будет увеличиваться имеет смысл использовать шардирование по user_id;
- Игры: Аналогично пользователям, количество игр увеличивается, поэтому будем использовать шардирование по game_id.


## 6. Физическая схема БД

### Схема после денормализации
![db-scheme-denorm](imgs/db-scheme-denorm.svg)

### Индексы
Для того, чтобы ускорить выполнение запросов используем индексы:
- `GameInfo`:
  - индекс на `release_date` для быстрого поиска по времени релиза (`b-tree index`)
  - индекс на `is_deleted` чтобы не показывать удаленные игры (`b-tree index`)
- `Game`:
  - индекс на `checksum` часто используется для проверки обновлений (`hash index`)
- `User`:
  - индекс на `status` для быстрого обновления статуса пользователя (`b-tree index`)
- `Item`:
  - индекс на `game_id` т.к. часто фильтруем по играм (`b-tree index`)
- `MultiplayerSession`:
  - индекс на `game_id` для фильтрации по играм (`b-tree index`)
  - индекс на `host_user_id` для того, чтобы быстро находить все сесси, где юзер был хостом (`b-tree index`)

### Выбор СУБД
- Основной базой данных будет MongoDB, в ней будут храниться таблицы: `User`, `Game`, `UserGame`, `Achievement`, `Screenshot`, `Item`, `Wallet`, `GameInfo`, `Review`, `Trade`, `TradeItem`;
- Для `OrderGame`, `OrderItem` будем использовать Tarantool, т.к. обеспечивает высокую производительность и дает ACID-гарантии;
- Для `Session`, `MultiplayerSession`, `UserMultiplayerSession` будем использовать Redis;
- Таблицы `UserSearch`, `GameSearch`, `ItemSearch` специально созданы для обеспечения быстрого поиска с помощью Elasticsearch.

### Шардирование и резервирование СУБД
**Шардирование:**
Шардировать имеет смысл, те таблицы, для которых предпологаемы размер будет большим + будет большая нагрузка на запись.
- **Таблица `user`**. На предыдущем этапе (логическая схема), оценка размера данной таблицы -- 295 Гб. На этапе проектирования физической схемы БД, в данную таблицу добавились поля, содержащие ачивки, друзей и предметы пользователя, поэтому размер одной записи увеличился: `345 + (5 * 40) + (10 * 60) + (10 * 60) = 1745`, размер всех записей: `~1.6 ТБ`. Размер индексов для этой таблицы будет `(16 (UUID index) + 60 (b-tree index)) * 919 * 10^6 = 69.844 ГБ`, поэтому идексы можно поместить в оперативку => **шардирование не нужно**;

- **Таблица `UserGame`.** Не смотря на большой размер записей в этой таблице (1.03 ТБ), нагрузка на запись в эту таблицу маленькая (< 12 RPS в пике), поэтому **шардировать не нужно**;

- **Таблица `UserAchievement`.** Размер записей в этой таблице равен 3.66 ТБ, для того, чтобы оценить RPS возьмем, что активный юзер получает 15 ачивок за месяц => `132 * 10^6 * 10 / 86400 / 31 = 739.2 rps`. Данную таблицу **будем шардировать по `user_id`**

- **Таблица `UserItem`.** Размер записей в этой таблице равен 4.01 ТБ. Возьмем, что юзер покупает 1 предмет в месяц, тогда получаем `132 * 10^6 / 86400 / 31 = 73.9 rps`. Т.к. rps на запись в данную таблицу не такой большой, то можно **не шардировать данную таблицу**

- **Таблица `Screenshot`.** Не смотря на то, что rps для записи в эту таблицу не такой большой (257 rps в пике), размер данной таблицу очень большой (~43 ТБ), поэтому **будем шардировать по `user_id`**

**Резервирование:**
- MonogDB: будем использовать replica set: 1 primary + 2 secondary;
- Tarantool: для формирования бэкапов будем делать снапшоты каждые 30 минут, также будет использоваться WAL. Будем использовать Master-Slave репликацию;
- Redis: будем использовать 1 master + 1 replica;
- Elasticsearch: будем использовать snapshot-ы в качестве бэкапа.

### Клиентские библиотеки / интеграции
Т.к. планируем, что бэкэнд будет на Go, то для работы с MongoDB будем использовать драйвер `mongo-go-driver`. Сервисы, которые взаимодействуют с tarantool можно написать на lua, получится что у нас tarantool будет и application server и БД. Для работы с Redis будем использовать библиотеку `go-redis`. Для того, чтобы обеспечить консистентность данных между базами будем использовать `kafka`.


## 7. Алгоритмы

### Коллаборативная фильтрация
Поскольку основной функцией Steam является продажа игр, то имеет смысл внедрить хорошую рекомендательную систему. Так можно использовать **коллаборативную фильтрацию**, которая будет советовать игры, основываясь на предпочтениях пользователя.

Различают два вида коллаборативной фильтрации:
- **User-based**: сначала ищем пользователя, похожего на нашего таргетного пользователя (можно определять схожость по совпадению игр), затем берем игры, которые не установлены у таргетного пользователя, но есть у похожего
- **Item-based**: в данном методе строится матрица похожести для игр: т.е. берется статистика по пользователям, на выходе получаем, что тот кто играет в игру A также играет в игру B

Из плюсом данного метода можно отметить: не требует анализа содержимого игры, учитывает "коллективную мудрость", хорошо масштабируется. Основные минусы: требует большого объема данных, холодный старт (т.е. не работает для новых игр и пользователей)

#### Подробнее про алгоритм (Item-based)
Идея Item-based коллаборативной фильтрации заключается в следующем:
1. Строится матрица взаимодействий, строки -- пользователи, столбцы -- игры. В простом случае значение может быть бинарным (владеет ли игрок данной игрой или нет), однако более показательной метрикой будет -- количество часов в игре
2. Далее надо определить сходства между каждой парой игр. Для этого, для каждой пары игр (`game_i`, `game_j`) рассматриваем вектор столбцов (`v_i`, `v_j`). Сходство опередляется соотношением:
$$S(game_i, game_j) = \dfrac{(v_i \cdot v_j)}{||v_i||\cdot||v_j||}$$
3. Берем игры, в которые пользователь не играл, для каждой такой игры рассчитываем score (суммируем по всем `game_j`, в которые пользователь не играл):
$$Score(game_i, user_i) = \sum{S(game_i, game_j) \cdot Mat(game_i, user_i)}$$
4. Сортируем игры по значениям `Score`

### Кэширование и дедупликация для VPK (Valve Package System)
При обновлениях игры или скачивании новых патчей часто оказывается, что многие ресурсы не изменились между версиями. Алгоритм кэширования и дедупликации решает задачу: если на клиенте уже имеется файл с нужным содержимым (определяемым по хэш-сумме, например, SHA256), его не нужно скачивать снова. Все необходимые файлы храняться в VPK-контейнере.

**Описание алгоритма:**
1. **Формирование манифеста ресурсов:** на сервере генерируется манифест -- список всех файлов в VPK с их метаданными (путь файла, размер, хэш, ...), который передается клиенту, чтобы он знал, какие файлы требуются для текущей версии игры
2. **Кэширование:** на клиенте реализуется система кэширования, которая состоит из: локального кэша и проверки манифеста. Локальный кэш хранит ранее загруженные файлы (или манифесты). Проверки манифеста заключается в том, что при получении манифеста, клиент проходит по списку файлов и для каждого проверяет: если файл с таким хэшом уже есть в кэше, то повторно скачивать не надо; если файла нету (или его версия устарела, т.е. хэш не совпадает), то файл помечается для скачивания. Также можно использовать LRU кэш, для очистки редко используемых файлов в кэше.
3. **Дедупликация:** поскольку ключом к файлу служит его хэш, одинаковые файлы, встречающиеся в разных VPK, будут иметь один и тот же хэш. Таким образом, если один и тот же ресурс требуется в нескольких пакетах или обновлениях, он сохраняется в кэше только один раз. Также сервер может группировать запросы на скачивание по хэшу. Если несколько клиентов запрашивают один и тот же файл, CDN или прокси-сервер сможет отдать один и тот же кэшированный файл, что снижает нагрузку на исходный сервер.


## 8. Технологии
| Технология      | Область применения                            | Мотивация                                                                 |
|------------------|-----------------------------------------------|---------------------------------------------------------------------------|
| **Go**           | Backend                                       | Высокая скорость, хорошо поддерживаемый язык, хорошо работает с многоядерными процессорами |
| **Python (ML)**  | Анализ данных, машинное обучение              | Широкий набор библиотек для ML, простой язык                             |
| **Lua**          | Скриптовая логика, интеграция с Tarantool     | Для работы с tarantool, много библиотек (к примеру есть очереди на lua)  |
| **TypeScript**   | Фронтенд, веб-клиент                          | Для spa, есть типизация                                                  |
| **C++**          | Десктоп-клиенты, производительные модули      | Максимальная производительность, контроль над памятью                     |
| **Swift**        | Мобильное приложение для iOS                 | Язык для разработки по ios                                                |
| **Kotlin**       | Мобильное приложение для Android             | Современный язык, официальный стандарт для Android, улучшенный синтаксис  |
| **MongoDB**      | Основная БД                                  | Позволяет хранить данные в неструктурированном виде, что повышает скорость запросов |
| **Tarantool**    | БД для операций с оплатой                    | Быстрая работа с данными в памяти, позволяет писать доп. логику с помощью lua  |
| **Redis**        | Хранение сессий                              | Быстрый доступ к данным, отлично подходи для сессий                       |
| **Elasticsearch**| Поиск                                        | Мощный поиск для игр/пользователей, поддержка сложных запросов            |
| **Docker**       | Контейнеризация приложений                   | Упрощение сборки, тестирования и доставки, единая среда выполнения        |
| **Kubernetes**   | Оркестрация контейнеров                      | Масштабируемость, отказоустойчивость, автоматическое восстановление       |
| **Nginx**        | Балансировка нагрузки                        | Балансировка трафика, ssl-терминации, активно поддерживается              |
| **Kafka**        | Event streaming, обмен сообщениями            | Высоконагруженная очередь событий, надёжная доставка между микросервисами |
| **Prometheus**   | Сбор метрик и мониторинг                     | Хранение и анализ метрик, простота интеграции с Go, k8s, Redis и др.      |
| **Grafana**      | Визуализация мониторинга                     | Гибкие дашборды, наблюдаемость за системой, связка с Prometheus           |
| **Gitlab Runners**| CI/CD                                       | Удобная настройка пайплайнов, интеграция с gitlab                         |


## 10. Схема сервиса
Ниже приведена схема сервиса, также есть [файл drawio](steam-scheme.drawio).
![steam-scheme](imgs/steam-scheme.svg)


## Список источников
[^1]: [https://partner.steamgames.com/](https://partner.steamgames.com/)
[^2]: [https://store.steampowered.com/charts/](https://store.steampowered.com/charts/)
[^3]: [https://steam.quers.net/map](https://steam.quers.net/map)
[^4]: [https://worldpopulationreview.com/country-rankings/steam-users-by-country](https://worldpopulationreview.com/country-rankings/steam-users-by-country)
[^5]: [https://help.steampowered.com/en/faqs/view/7EFD-3CAE-64D3-1C31](https://help.steampowered.com/en/faqs/view/7EFD-3CAE-64D3-1C31)
[^6]: [https://www.demandsage.com/steam-statistics](https://www.demandsage.com/steam-statistics)
[^7]: [https://hypestat.com/info/api.steampowered.com](https://hypestat.com/info/api.steampowered.com)
[^8]: [https://ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc2018/presentations/Steam_in_2017.pdf](https://ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc2018/presentations/Steam_in_2017.pdf)
[^9]: [https://dtf.ru/gameindustry/1629749-korrelyaciya-otzyvov-i-prodazh-v-steam](https://dtf.ru/gameindustry/1629749-korrelyaciya-otzyvov-i-prodazh-v-steam)
[^10]: [https://www.trade.tf/volume](https://www.trade.tf/volume)
[^11]: [https://steamdb.info/instantsearch/](https://steamdb.info/instantsearch/)
[^12]: [https://steamdb.info/blog/scanning-all-steam-ids/](https://steamdb.info/blog/scanning-all-steam-ids/)
[^13]: [https://store.steampowered.com/stats/content/](https://store.steampowered.com/stats/content/)
[^14]: [https://partner.steamgames.com/doc/api/isteamfriends](https://partner.steamgames.com/doc/api/isteamfriends)
[^15]: [https://www.researchgate.net/figure/Average-Number-of-Hours-Spent-Gaming-in-a-Typical-Day-for-Female-and-Male-Gamers_fig2_283793880](https://www.researchgate.net/figure/Average-Number-of-Hours-Spent-Gaming-in-a-Typical-Day-for-Female-and-Male-Gamers_fig2_283793880)
[^16]: [https://www.playground.ru/misc/news/sudya_po_vsemu_igroki_ps5_tratyat_gorazdo_bolshe_vremeni_na_odinochnye_igry_chem_na_mnogopolzovatelskie-1674125?utm_source=chatgpt.com](https://www.playground.ru/misc/news/sudya_po_vsemu_igroki_ps5_tratyat_gorazdo_bolshe_vremeni_na_odinochnye_igry_chem_na_mnogopolzovatelskie-1674125?utm_source=chatgpt.com)
[^17]: [https://www.gamedeveloper.com/game-platforms/there-are-44-000-game-developers-on-steam-who-are-they-](https://www.gamedeveloper.com/game-platforms/there-are-44-000-game-developers-on-steam-who-are-they-)
