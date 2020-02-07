# User Long Poll API

### План документации:
1. [Подключение](#подключение)
2. [Возвращаемые ошибки](#возвращаемые-ошибки)
3. [Получение истории прошлых событий](#получение-истории-прошлых-событий)
4. [Структура сообщения](#структура-сообщения)
5. [Описание событий](#описание-событий)
   - [Событие 2. Установка флагов сообщения](#событие-2-установка-флагов-сообщения)
   - [Событие 3. Сброс флагов сообщения](#событие-3-сброс-флагов-сообщения)
   - [Событие 4. Новое сообщение](#событие-4-новое-сообщение)
   - [Событие 5. Редактирование сообщения](#событие-5-редактирование-сообщения)
   - [Событие 6. Прочтение входящих сообщений](#событие-6-прочтение-входящих-сообщений)
   - [Событие 7. Прочтение исходящих сообщений](#событие-7-прочтение-исходящих-сообщений)
   - [Событие 8. Друг появился в сети](#событие-8-друг-появился-в-сети)
   - [Событие 9. Друг вышел из сети](#событие-9-друг-вышел-из-сети)
   - [Событие 10. Неизвестное](#событие-10-неизвестное)
   - [Событие 12. Неизвестное](#событие-12-неизвестное)
   - [Событие 13. Удаление всех сообщений в диалоге](#событие-13-удаление-всех-сообщений-в-диалоге)
   - [Событие 18. Добавление сниппета к сообщению](#событие-18-добавление-сниппета-к-сообщению)
   - [Событие 19. Сброс кеша сообщения](#событие-19-сброс-кеша-сообщения)
   - [Событие 51. Изменение данных чата (не следует использовать)](#событие-51-изменение-данных-чата-не-следует-использовать)
   - [Событие 52. Изменение данных чата](#событие-52-изменение-данных-чата)
   - [Событие 63. Статус набора сообщения](#событие-63-статус-набора-сообщения)
   - [Событие 64. Статус записи голосового сообщения](#событие-64-статус-записи-голосового-сообщения)
   - [Событие 80. Изменение количества непрочитанных диалогов](#событие-80-изменение-количества-непрочитанных-диалогов)
   - [Событие 81. Изменение состояния невидимки друга](#событие-81-изменение-состояния-невидимки-друга)
   - [Событие 114. Изменение настроек пуш-уведомлений в беседе](#событие-114-изменение-настроек-пуш-уведомлений-в-беседе)
   - [Событие 115. Звонок](#событие-115-звонок)
6. [Дополнительные данные](#дополнительные-данные)
   - [Флаги сообщений](#флаги-сообщений)
   - [Сервисные сообщения](#сервисные-сообщения)
   - [Клавиатура для ботов](#клавиатура-для-ботов)
   - [Вложения](#вложения)

Документация написана для __10__ версии Long Poll.

Актуальная версия Long Poll - __11__, но в этой версии удалили события `8` и `9` (онлайн и оффлайн друга).
По этой причине я и не рекомендую вам использовать данную версию.

## Подключение

__Long Polling__ - это способ получения событий в реальном времени используя бесконечную цепочку запрос - ответ - запрос... При подаче запроса сервер возвращает ответ не сразу, а когда придет новое событие или истечет время ожидания.

Ссылку для запроса нужно генерировать следующим образом:

> https://**`server`**?act=a_check&key=**`key`**&ts=**`ts`**&wait=**`wait`**&mode=**`mode`**&version=**`version`**

* `server`, `key` и `ts` получаются один раз методом [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer)
* `version` - Версия Long Poll
* `wait` - Время ожидания нового события в секундах, максимум 90
* `mode` - Дополнительные опции ответа:
  * `2` - Возвращать вложения
  * `8` - Возвращать нормальные данные некоторых событий
  * `32` - Возвращать `pts`
  * `64` - Возвращать данные о платформе в событии онлайна друга
  * `128` - Возвращать `random_id`

В `Node.js` ссылку можно составить следующим образом:
```js
// server, key и ts нужно получить заранее. 
const link = `https://${server}?` + require('querystring').stringify({
  act: 'a_check',
  key: key,
  ts: ts,
  wait: 10,
  mode: 2 | 8 | 32 | 64 | 128,
  version: 11
});
```

После выполнения запроса сервер вернет ответ следующего вида:
```js
{
  updates?: Array,
  ts?: Number,
  pts?: Number,
  failed?: Number,
  min_version?: Number,
  max_version?: Number
}
```

Затем нужно будет обработать пришедшие в `updates` события и повторить запрос, перед этим заменив `ts` на новый из ответа.

## Возвращаемые ошибки

Иногда вместо поля `updates` в ответе может прийти поле `failed`. В первой половине случаев эта ошибка очень легко решается и не связана с потерей событий (2 и 4), но во второй, чтобы восстановить все "потерянные" события, нужно будет немножко попотеть.

1. Устарела история событий. Решается [получением истории прошлых событий](#получение-истории-прошлых-событий) и использованием переданного `ts` далее.
2. Истекло время действия ключа. Решается использованием `key` из метода [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer).
3. Информация о пользователе утрачена. Решается [получением истории прошлых событий](#получение-истории-прошлых-событий) и использованием `key` и `ts` из метода [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer).
4. Передана неправильная версия. Вместе с ошибкой передаются поля `min_version` и `max_version`, так что проблем здесь не возникнет.

## Получение истории прошлых событий

Чтобы получить историю прошлых событий, нам необходим `pts`, получение которого включается полем `need_pts: 1` в [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer) и добавлением в `mode` флага `32` при [выполнении запроса](#подключение).

Для получении истории мы будем использовать метод [`messages.getLongPollHistory`](https://vk.com/dev/messages.getLongPollHistory) с указанием следующих параметров при запросе:
- `ts` - последний полученный `ts` из лонгпула
- `pts` - последний полученный `pts` из лонгпула
- `msgs_limit` - минимум `200`, рекомендую `500`
- `max_msg_id` - `id` последнего полученного сообщения из лонгпула
- `onlines` - `1` если возвращать события `8` и `9` (онлайн и оффлайн друга), `0` если нет
- `lp_version` - последняя версия лонгпула
- `fields` - поля [пользователей и групп](https://vk.com/dev/objects/user), которые придут вместе с историей.

Ответ выглядит следующим образом:
```js
{
  history: Array,
  messages: { count: Number, items: Array },
  conversations: Array,
  from_pts: Number,
  new_pts: Number,
  profiles?: Array,
  groups?: Array,
  more?: Number
}
```

Поле `history` идентично полю `updates` в Long Poll, за исключением сокращения некоторых событий:
- `3` - сброс флага сообщения
- `4` - новое сообщение
- `5` - редактирование сообщения
- `18` - добавление сниппета к сообщению

В результате сокращения события будут выглядеть так:
```
[event_id, msg_id, flags, peer_id]
```

Основную информацию о сообщениях и беседах нужно будет брать из полей `messages` и `conversations`.

Если в ответе придет поле `more`, то после обработки всех событий нужно будет повторить запрос, указав в поле `pts` пришедший `new_pts`, а в поле `max_msg_id` `id` последнего полученного сообщения отсюда.

## Структура сообщения

Самой сложной структурой события в LongPoll является структура сообщения, которая идентична сразу для нескольких событий. В ней есть еще 4 довольно сложных момента, которые помечены как `<Type>` и описаны в отдельных абзацах:

- [Flags](#флаги-сообщений) - Флаги сообщений
- [Keyboard](#клавиатура-для-ботов) - Клавиатура для ботов (в том числе инлайн)
- [Action](#сервисные-сообщения) - Сервисное сообщение
- [Attachments](#вложения) - Вложения

```js
[
  event_id: 3 || 4 || 5 || 18,
  msg_id: Number,
  flags<Flags>: Number,
  peer_id: Number,
  timestamp: Number,
  text: String,
  {
    title?: ' ... ', // Приходит только в лс
    emoji?: '1', // Наличие emoji (кто бы мог подумать...)
    from?: String, // id автора сообщения. Приходит только в беседах
    fwd_count?: Number, // На данный момент перестал приходить, но скоро вернется
    has_template?: '1', // Наличие карусели
    mentions?: Array, // Массив с id людей, которых пушнули или ответили на их сообщения
    marked_users?: Array, // Плохой аналог mentions, игнорируем
    keyboard<Keyboard>?: Object, // Клавиатура для ботов (в т.ч. инлайн)
    <Action> // Описание сервисного сообщения
  },
  {
    fwd?: '0_0', // При пересланных сообщениях и ответах на сообщения
    <Attachments> // Описание вложений
  },
  random_id: Number,
  conversation_msg_id: Number,
  edit_time: Number // 0 (не отредактированно) или timestamp (время редактирования)
]
```

Стоит отметить, что в поле `text` переносы строк обозначаются как `<br>`,
а символы `"`, `&`, `<` и `>` экранируются, чтобы избежать проблем с версткой
и путаницей с ненужными переносами строк.

## Описание событий

### Событие 2. Установка флагов сообщения

Это событие вызывается в 5 случаях, которые отличаются [флагами сообщения](#флаги-сообщений):
1. Пометка важным (`important`)
2. Пометка как спам (`spam`)
3. Удаление сообщения (`deleted`)
4. Удаление для всех (`deleted` и `deleted_all`)
5. Прослушка голосового сообщения (`audio_listened`)

Структура: `[2, msg_id, flags, peer_id]`

### Событие 3. Сброс флагов сообщения

Это событие вызывается в 4 случаях, которые отличаются [флагами сообщения](#флаги-сообщений):
1. Прочитано сообщение (`unread`). Рекомендую не использовать, потому что иногда просто перестает приходить.
2. Отмена пометки важным (`important`)
3. Отмена пометки сообщения как спам (`spam` и `cancel_spam`)
4. Восстановление удаленного сообщения (`deleted`)

В 1 и 2 случаях структура у события такая: `[2, msg_id, flags, peer_id]`.
В остальных (3 и 4) случаях возвращается [сообщение](#структура-сообщения).

### Событие 4. Новое сообщение

Возвращает сообщение, описание которого есть [здесь](#структура-сообщения).

### Событие 5. Редактирование сообщения

Также возвращает сообщение, которое описано [здесь](#структура-сообщения).

### Событие 6. Прочтение входящих сообщений

Структура: `[6, peer_id, msg_id, count]`

Вы прочитали в беседе `peer_id` сообщения до `msg_id` включительно.  
`count` - количество непрочитанных в беседе сообщений.

### Событие 7. Прочтение исходящих сообщений

Структура: `[7, peer_id, msg_id, count]`

Собеседник прочитал в беседе `peer_id` сообщения до `msg_id` включительно.  
`count` - количество непрочитанных в беседе сообщений __у собеседника__ или непрочитанных у вас.

### Событие 8. Друг появился в сети

Структура: `[8, -user_id, platform, timestamp, app_id]`
- `user_id` приходит отрицательным, поэтому стоит минус
- `platform` означает тип устройства, с которого онлайн друг:
   - `1` - `m.vk.com` или неизвестное мобильное приложение
   - `2` - iPhone
   - `3` - iPad
   - `4` - Android
   - `5` - windows Phone
   - `6` - Windows 8
   - `7` - `vk.com` или неизвестное десктопное приложение
- `timestamp` - время онлайна в секундах
- `app_id` - идентификатор приложения, с которого онлайн друг

### Событие 9. Друг вышел из сети

Структура: `[9, -user_id, isTimeout, timestamp, app_id]`
- `user_id` - `id` друга
- `isTimeout` - `1` - оффлайн возник из-за отсутствия активности, `0` - из-за выхода из `vk.com`
- `timestamp` - время наступления оффлайн в секундах
- `app_id` - идентификатор приложения, с которого был онлайн друг

### Событие 10. Неизвестное

`TODO`

### Событие 12. Неизвестное

`TODO`

### Событие 13. Удаление всех сообщений в диалоге

Структура: `[13, peer_id, last_msg_id]`
- `peer_id` - беседа, в которой были удалены все сообщения
- `last_msg_id` - `id` последнего сообщения в беседе (т.е. были удалены все сообщения до `last_msg_id` включительно)

### Событие 18. Добавление сниппета к сообщению

Структура идентична [данной](#структура-сообщения) структуре сообщения.

Событие добавляет к сообщению вложение типа `link`.

### Событие 19. Сброс кеша сообщения

Структура: `[19, msg_id]`

Событие означает, что если сообщение сохранено в вашем кеше, то его нужно переполучить.  
Это значит, что сообщение изменилось, но пользователь его напрямую (или вообще) не редактировал.

## Событие 51. Изменение данных чата (не следует использовать)

Структура: `[51, chat_id]`

Событие означает, что в беседе `chat_id` изменились какие-то данные.
Подробнее об этих данных можно узнать в [52 событии](#событие-52-изменение-данных-чата).

## Событие 52. Изменение данных чата

Структура: `[52, type, peer_id, extra]`
- `peer_id` - `id` беседы, в которой произошло событие
- `extra` - дополнительная информация, которая зависит от `type`
- `type` - тип события:
   - `1` - Изменилось название беседы
      - Основная информация приходит в [сервисном сообщении](#сервисные-сообщения)
      - `extra` = `0`
   - `2` - Обновилась аватарка беседы
      - Основная информация приходит в [сервисном сообщении](#сервисные-сообщения)
      - `extra` = `0`
   - `3` - Назначен новый администратор
      - `extra` = `id` нового администратора
   - `5` - Закрепление или открепление сообщения
      - `extra` = `conversation_msg_id` при закреплении и `0` при откреплении
   - `6` - Вступление в беседу (по ссылке, прямое добавление и возвращение в беседу)
      - `extra` = `id` добавленного юзера
   - `7` - Выход из беседы
      - `extra` = `id` вышедшего из беседы
   - `8` - Исключение из беседы
      - `extra` = `id` исключенного из беседы
   - `9` - Разжалован администратор
      - `extra` = `id` разжалованного из администраторов
   - `11` - Появление и скрытие клавиатуры
      - `extra` = `id` диалога, в котором произошло действие

### Событие 63. Статус набора сообщения

Структура: `[63, peer_id, [from_ids], from_ids_count, timestamp]`

### Событие 64. Статус записи голосового сообщения

Идентичен событию 63, только вызывается в случае записи голосового сообщения, а не при наборе текста.

### Событие 80. Изменение количества непрочитанных диалогов

Структура: `[80, count, count_with_notifications]`
- `count` - Количество непрочитанных сообщений
- `count_with_notifications` - Количество непрочитанных сообщений со включенными уведомлениями

### Событие 81. Изменение состояния невидимки друга

Структура: `[-user_id, state, timestamp]`
- `user_id` - `id` друга, который включил или выключил невидимку
- `state` - `1` - включена, `0` - выключена
- `timestamp` - время включения или выключения невидимки у друга

### Событие 114. Изменение настроек пуш-уведомлений в беседе

Структура: `[{ peer_id, sound, disabled_until }]`
- `peer_id` - `id` беседы, в которой включили или выключили уведомления
- `sound` - нестабильный параметр, не рекомендую его обрабатывать
- `disabled_until` может быть трех видов:
   - `0` - Включены
   - `-1` - Выключены
   - `Другое число` - Выключены, время их включения

### Событие 115. Звонок

`TODO`

## Дополнительные данные

### Флаги сообщений

Маска представляет собой сумму флагов, которые являются степенью двойки.
Обычно маску записывают таким образом:
```js
// Так принято: 
const mask = 1 | 2 | 8 | 64 | 1024 | 1 << 11 | 1 << 15;
// Но можно и так:
// Скобки необходимы, потому что у "+" приоритет выше чем у "<<"
const mask = 1 + 2 + 8 + 64 + 1024 + (1 << 11) + (1 << 15);
```

Так как при увеличении степени двойки числа тоже увеличиваются, и довольно быстро,
можно использовать `1 << n` или `2 ** n`, что означает `2` в степени `n`.

|    Название    |                 Описание                 |  Значение  |
| :------------: | :--------------------------------------: | :--------: |
| unread         | Непрочитанное сообщение                  | `1 << 0`   |
| outbox         | Исходящее сообщение                      | `1 << 1`   |
| important      | Важное сообщение                         | `1 << 3`   |
| chat           | Отправка сообщения в беседу через vk.com | `1 << 4`   |
| friends        | Исходящее; входящее от друга в лс        | `1 << 5`   |
| spam           | Пометка сообщения как спам               | `1 << 6`   |
| deleted        | Удаление сообщения локально              | `1 << 7`   |
| audio_listened | Прослушано голосовое сообщение           | `1 << 12`  |
| chat2          | Отправка в беседу через клиенты          | `1 << 13`  |
| cancel_spam    | Отмена пометки как спам                  | `1 << 15`  |
| hidden         | Приветственное сообщение от группы       | `1 << 16`  |
| deleted_all    | Удаление сообщения для всех              | `1 << 17`  |
| chat_in        | Входящее сообщение в беседе              | `1 << 19`  |
| noname_flag    | Неизвестный флаг                         | `1 << 20`  |
| reply_msg      | Ответ на сообщение                       | `1 << 21`  |

Пример определения флага в маске:
```js
const mask = 1 | 2 | 32;

8 & mask // 0 (false)
2 & mask // 2 (true)
// Однако, если указать в качестве флага не степень двойки,
// то там будет другое число, отличное от 0.
```

### Сервисные сообщения

Сервисное сообщение описывается одним или несколькими ключами в объекте. Тип сервисного сообщения определяет ключ `source_act`.

Возможные значения `source_act`:
- __`chat_photo_update`__ - Обновление фотографии беседы
   - Нет дополнительных ключей, но добавляет фотографию во вложениях
- __`chat_photo_remove`__ - Удаление фотографии беседы
   - Нет дополнительных ключей
- __`chat_create`__ - Создание беседы
   - `source_text` - Название беседы
- __`chat_title_update`__ - Обновление название беседы
   - `source_old_text` - Старое название беседы (но при полчении сообщения из API его не будет)
   - `source_text` - Новое название беседы
- __`chat_pin_message`__ - Закрепление сообщения
   - `source_mid` - `id` закрепившего сообщение
   - `source_message` - обрезанное закрепляемое сообщение
   - `source_chat_local_id` - `conversation_message_id` закрепляемого сообщения
- __`chat_unpin_message`__ - Открепление сообщения
   - `source_mid` - `id` открепившего сообщение
   - `source_chat_local_id` - `conversation_message_id` открепляемого сообщения
- __`chat_invite_user_by_link`__ - Вступление по ссылке
   - Нет дополнительных ключей
- __`chat_invite_user`__ - Вступление или возвращение в беседу
   - `source_mid` - `id` вступившего или вернувшегося
   - Если `source_mid` совпадает с `from`, то он вернулся сам, иначе - его пригласили
- __`chat_kick_user`__ - Исключение или выход из беседы
   - `source_mid` - `id` кикнутого или вышедшего
   - если `source_mid` совпадает с `from`, то он вышел сам, иначе - его кикнули

Пример описания сервисного сообщения:
```js
{
  source_act: 'chat_pin_message',
  source_mid: '88262293',
  source_message: 'Сообщение, которое будет в закрепе',
  source_chat_local_id: '5517'
}
```

### Клавиатура для ботов

Клавиатура для ботов представляет собой обьект с описанием ее типа и кнопок.
Основная структура представлена ниже, остальную информацию можно узнать в [документации](https://vk.com/dev/bots_docs_3?f=4.2.%20%D0%A1%D1%82%D1%80%D1%83%D0%BA%D1%82%D1%83%D1%80%D0%B0%20%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85).
```js
{
  one_time: true || false, // Скрывать ли при клике на кнопку (не работает для inline)
  inline: true || false, // Клавиатура для сообщения или для беседы
  buttons: Array // Массив с массивами кнопок
}
```

### Вложения

__Спойлер:__ Скоро формат ответа может измениться.

Список известных на данный момент вложений: `geo`, `doc`, `link`, `poll`, `wall`, `call`, `gift`, `story`, `photo`, `audio`, `video`, `event`, `market`, `artist`, `sticker`, `article`, `podcast`, `graffiti`, `wall_reply`, `audio_message`, `money_request`, `audio_playlist`.

Однако названия вложений, полученных через лонгпул, могут не совпадать с теми, что приходят через апи:
- `event`, приходящий в апи, в лонгпуле обозначается как `group`
- `audio_message` и `graffiti` из лонгпула обозначаются как `doc`, но при этом добавляется ключ `attach*_kind` в ответе вложения со значением `audiomsg` или `graffiti`
- `artist`, `audio_playlist` и `article`, которые приходят в лонгпуле, через апи отображаются как `link`

Вложение `geo` (прикрепленное местоположение) является еще одним исключением и приходит не как `attach*`, а просто как ключи `geo` и иногда `geo_provider`. Также при получении сообщения через [`messages.geyById`](https://vk.com/dev/messages.getById) ключ `geo` будет находиться не во вложениях, а в "корне" сообщения.

На данный момент вложения передаются достаточно неудобным для обработки способом. Вот пример вложений, состоящих из фотографии, документа и аудиозаписи:
```js
{
  attach1: '88262293_457290160',
  attach1_type: 'photo',
  attach2: '88262293_532324610',
  attach2_type: 'doc',
  attach3: '88262293_535133534',
  attach3_kind: 'audiomsg',
  attach3_type: 'doc'
}
```

Для того, чтобы вытащить их названия нужно писать дополнительный код. Однако вытаскивать идентификаторы вложений нет смысла, потому что они не представляют никакой ценности. Чтобы получить нормальную информацию о вложениях, нужно получить сообщение через API, используя метод [`messages.geyById`](https://vk.com/dev/messages.getById). Пример кода для получения массива с названиями вложений на `JavaScript`:
```js
function getAttachments(data) {
  const attachments = [];

  if(data.geo) attachments.push({ type: 'geo' });

  for(const key in data) {
    const match = key.match(/attach(\d+)$/);

    if(match) {
      const id = match[1];
      const kind = data[`attach${id}_kind`];
      let type = data[`attach${id}_type`];

      if(kind == 'audiomsg') type = 'audio_message';
      if(kind == 'graffiti') type = 'graffiti';
      if(type == 'group') type = 'event';

      attachments.push({ type });
    }
  }

  return attachments;
}
```

При работе с вложениями можно попробовать найти необходимый элемент в [документации](https://vk.com/dev/objects), однако у некоторых вложений документация не обновлена или вовсе отсутствует.
