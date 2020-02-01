# User Long Poll API

### План документации:
1. [Подключение](https://github.com/danyadev/longpoll-doc#подключение)
2. [Возвращаемые ошибки](https://github.com/danyadev/longpoll-doc#возвращаемые-ошибки)
3. [Получение устаревшей истории событий](https://github.com/danyadev/longpoll-doc#получение-устаревшей-истории-событий)
4. [Структура событий](https://github.com/danyadev/longpoll-doc#структура-событий)
5. [Дополнительные данные](https://github.com/danyadev/longpoll-doc#дополнительные-данные)
 * Флаги сообщений
 * Вложения
 * Идентификаторы платформ

Документация написана для __11__ версии Long Poll.

## Подключение

Long Polling - это способ получения событий в реальном времени используя бесконечную цепочку запрос - ответ - запрос... При подаче запроса сервер возвращает ответ не сразу, а когда придет новое событие (либо истечет время ожидания).

Ссылку для запроса нужно генерировать следующим образом:

> https://**`server`**?act=a_check&key=**`key`**&ts=**`ts`**&wait=**`wait`**&mode=**`mode`**&version=**`version`**

* `server`, `key` и `ts` получаются методом [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer)
* `version` - Версия Long Poll
* `wait` - Время ожидания нового события в секундах
* `mode` - Дополнительные опции ответа:
  * `2` - Возвращать вложения
  * `8` - Возвращать нормальные данные некоторых событий
  * `32` - Возвращать `pts`
  * `64` - Возвращать данные о платформе в событии онлайна юзера
  * `128` - Возвращать `random_id`

В `Node.js` ссылку можно составить следующим образом:
```js
// server, key и ts нужно заранее получить. 
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

## Возвращаемые ошибки

1. Устарела история событий. Решается [получением и обработкой устаревшей истории событий](https://github.com/danyadev/longpoll-doc#получение-устаревшей-истории-событий) и использованием переданного `ts` далее.
2. Истекло время действия ключа. Решается использованием `key` из метода [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer).
3. Информация о пользователе утрачена. Решается [получением и обработкой устаревшей истории событий](https://github.com/danyadev/longpoll-doc#получение-устаревшей-истории-событий) и использованием `key` и `ts` из метода [`messages.getLongPollServer`](https://vk.com/dev/messages.getLongPollServer).
4. Передана неправильная версия. Вместе с ошибкой передаются поля `min_version` и `max_version`, так что проблем здесь не возникнет.

## Получение устаревшей истории событий

## Структура событий

## Дополнительные данные
