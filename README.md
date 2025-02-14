# Проектирование высоконагруженного сервиса Steam

## Содержание

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
- Система профилей: можно добавлять друзей, чтобы играть с ними по сети

**Ключевые продуктовые решения**:
- **Клиент Steam**: написан на C++, использует Chromium Embedded Framework в качестве движка[^5];
- **Steamworks API**: используется для интеграции функций Steam в игры, реализован на основе HTTP и gRPC[^1];
- **Steam Guard (двухфакторная аутентификация)**: использует мобильное приложение Steam для генерации кодов[^6];

## Список источников
[^1]: [https://partner.steamgames.com/](https://partner.steamgames.com/)
[^2]: [https://store.steampowered.com/charts/](https://store.steampowered.com/charts/)
[^3]: [https://steam.quers.net/map](https://steam.quers.net/map)
[^4]: [https://worldpopulationreview.com/country-rankings/steam-users-by-country](https://worldpopulationreview.com/country-rankings/steam-users-by-country)
[^5]: [https://developer.valvesoftware.com/wiki/Chromium_Embedded_Framework](https://developer.valvesoftware.com/wiki/Chromium_Embedded_Framework)
[^6]: [https://help.steampowered.com/en/faqs/view/7EFD-3CAE-64D3-1C31](https://help.steampowered.com/en/faqs/view/7EFD-3CAE-64D3-1C31)