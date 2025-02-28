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
| Просмотр игр [^7]             | 3.0000                   |
| Покупка игр [^8]              | 0.0075                   |
| Комментарий к игре [^9]       | 0.0002                   |
| Скачивание игр                | 0.0301                   |
| Обновление игр                | 1.5714                   |
| Совершить трейд [^10]         | 0.3623                   |
| Просмотр профилей             | 2.5000                   |
| Просмотр игровых предметов    | 5.0000                   |
| Обновление информации профиля | 0.0323                   |
| Загрузка скриншотов/видео     | 0.1612                   |

- пользователь в среднем авторизируется в Steam 1 раз в месяц;
- согласно SteamSpy[^8] (статистика за 2017) на пользователя, в среднем, приходится ~11 игр, а средний возраст аккаунтов ~4 лет;
- согласно [^9] медианное значение множителя для игр ~40, т.е. на один отзыв приходится 40 проданных игр;
- если учесть, что у пользователя в среднем появляется 11 игр за год, а также, то что в steam игры обычно обновляются раз в неделю, получим, что в среднем пользователь обновляет 11 игр в неделю;
- согласно [^10] за последний месяц было в среднем 2.5M трейдов в день => количество трейдов = 2.5 / DAU;

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
$3DAU \times 16мб \div 86400 с \approx 38.3 гб/c = 306.4 гбит/с$
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
| Просмотр игр         | 306.4                      |  612.8                           | 3 310 080                          |
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
| Просмотр игр                  | 2395.83    | 4791.66         |
| Покупка игр                   | 5.99       | 11.98           |
| Комментарий к игре            | 0.16       | 0.32            |
| Скачивание игр                | 1254.94    | 2509.88         |
| Обновление игр                | 20.04      | 40.08           |
| Совершить трейд               | 289.34     | 578.68          |
| Просмотр профилей             | 1996.53    | 3993.06         |
| Просмотр игровых предметов    | 3993.06    | 7986.12         |
| Обновление информации профиля | 25.80      | 51.60           |
| Загрузка скриншотов/видео     | 128.74     | 257.47          |

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