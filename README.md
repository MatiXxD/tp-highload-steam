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
  - [Схема DNS балансировки](#схема-dns-балансировки)
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
Домен `steam.com` будет использоваться обслуживания клиентов (фронтэнд). Для обработки api запросов будут использоваться следующие домены:
- `auth.steam.com`: регистрация/авторизация;
- `store.steam.com`: магазин игр;
- `content.steam.com`: раздача игр;
- `community.steam.com`: работа с профилями, торговая площадка;
- `api.steam.com`: используется для работы с Steamworks API;

### Расположение ДЦ
Steam отрыто предоставляет статистику скачиваний в зависимости от региона [^13]:
![bandwidth-used](imgs/bandwidth-used.png)

**Пример расположения серверов:**
- *Северная Америка:* Сиэтл, Калифорния, Техас, Торонто, Северная Вирджиния, Монреаль, Чикаго;
- *Европа:* Франкфурт, Лондон, Амстердам, Париж, Варшава, Мадрид, Стокгольм, Милан;
- *Азия:* Токио, Сеул, Сингапур, Гонконг, Тайбэй, Шанхай, Мумбаи, Бангкок;
- *Россия:* Москва, Санкт-Петербург, Новосибирск, Екатеринбург;
- *Южная Америка:* Сан-Паулу, Буэнос-Айрес, Сантьяго, Богота;
- *Океания:* Сидней, Окленд;
- *Ближний Восток:* Эр-Рияд, Дубай;
- *Африка:* Йоханнесбург, Кейптаун;
- *Центральная Америка:* Мехико, Панама-Сити;

![dc-map](imgs/dc-map.png)

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

| Действие                      | Северная Америка | Европа | Азия | Россия | Южная Америка | Океания | Ближний Восток | Африка | Центральная Америка |
|-------------------------------|------------------|--------|------|--------|---------------|---------|----------------|--------|---------------------|
| Регистрация/авторизация       | 20.47            | 12.15  | 7.03 | 1.86   | 6.17          | 1.14    | 2.13           | 0.22   | 0.31                |
| Просмотр игр                  | 1774.13          | 1053.21| 610.01| 161.45 | 535.77        | 98.39  | 184.70         | 19.23  | 26.83               |
| Покупка игр                   | 4.75             | 2.82   | 1.63 | 0.43   | 1.43          | 0.26    | 0.49           | 0.05   | 0.07                |
| Комментарий к игре            | 0.13             | 0.08   | 0.04 | 0.01   | 0.04          | 0.01    | 0.01           | 0.001  | 0.002               |
| Скачивание игр                | 19.07            | 11.32  | 6.56 | 1.73   | 5.76          | 1.06    | 1.98           | 0.21   | 0.29                |
| Обновление игр                | 995.58           | 591.14 | 342.46| 90.56  | 300.68        | 55.22   | 103.56         | 10.77  | 15.06               |
| Совершить трейд               | 229.55           | 136.28 | 78.95 | 20.88  | 69.31         | 12.73   | 23.86          | 2.48   | 3.47                |
| Просмотр профилей             | 1583.94          | 940.37 | 544.76| 144.07 | 478.33        | 87.85   | 164.72         | 17.13  | 23.96               |
| Просмотр игровых предметов    | 3167.88          | 1880.74| 1089.52| 288.14 | 956.66        | 175.70  | 329.44         | 34.26  | 47.92               |
| Обновление информации профиля | 20.47            | 12.15  | 7.03  | 1.86   | 6.17          | 1.14    | 2.13           | 0.22   | 0.31                |
| Загрузка скриншотов/видео     | 102.13           | 60.63  | 35.12 | 9.29   | 30.84         | 5.66    | 10.62          | 1.10   | 1.54                |
| Обновление статуса            | 3801.71          | 2256.88| 1307.17| 345.96 | 1148.08        | 210.83  | 395.79         | 41.21  | 55.50               |
| Мультиплеер                   | 556.00           | 330.07 | 191.17| 50.60 | 167.91        | 30.83  | 57.88         | 6.03  | 8.41               |

### Схема DNS балансировки
Т.к. Steam-ом пользуются во всем мире имеет смысл применять Geo-based DNS, также можно рассмотреть вариант с Latency-based DNS, если хотим добиться минимального значения RTT.


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