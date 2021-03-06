# Lockable event usage

В предыдущем посте я показал реализацию `lockable event` - обертку над событием которая позволяет нам лочить или анлочить исходное событие с возможностью подписки на не заблокированные события, а если попроще то - фильтровать события.

Теперь непосредственно к примеру использования в жизни.

---

## Задача

Есть кнопка нажатия по которой вызывают отправку запросов по `api`, проблема в том что если нажать на нее несколько раз то запросов отправиться соответствующее количество. Нужно блокировать отправку запросов пока предыдущий не выполниться, после выполнения разрешить отправку следующего запроса.

Из условий мы имеем событие `sendRequest` - нажатие на кнопку и эффект `sendRequestFx` - запрос который будет отправляться по событию `sendRequest`.

---

## Effect

`effect` - это контейнер для сайд эффектов. Так же в эффекторе принято для эффектов делать постфикс `Fx`.

Эффект предоставляет набор полезных событий:

- `done` - эффект завершился успешно
- `fail` - с ошибкой
- `finally` - срабатывает в конце, полная аналогия с `finally` из `js`

...и булево значение `panding` - которое значит что эффект в процессе выполнения. Остальные свойства и события разберем позже.

---

## Начальные юниты

`Unit` - тип данных, используемый для описания бизнес-логики приложений. Есть четыре типа юнитов: `store`, `event`, `effect` и `domain`.

Итак по нашей логике мы создали три юнита:

```js
const sendRequest = createEvent();

const sendRequestFx = createEffect(async (ms) => {
  return new Promise((resolve) => setTimeout(resolve, ms));
});

const $completedRequests = createStore(0).on(
  sendRequestFx.done,
  (state) => state + 1
);
```

- `sendRequest` - событие которое будет навешено на кнопку
- `sendRequestFx` - эффект который делает запрос к `api`, для примера делаем таймаут с задержкой которую будем передавать при вызове эффекта `sendRequestFx(1000)`
- `$completedRequests` - стор для хранения количества завершенных эффектов, при реагировании на событие `done` с эффекта увеличиваем наш стейт на `1`

В компоненте навешиваем событие на клик и с помощью хука `useStore` из `effector-react` подключаем наш компонент к стору `$completedRequests`

```js
function App() {
  const completedRequests = useStore($completedRequests);

  return (
    <div>
      {completedRequests}
      <button onClick={() => sendRequest(1000)}>Start</button>
    </div>
  );
}
```

---

Если этот код запустить сейчас, то при нажатии на кнопку ничего не произойдет, так как наше событие улетает в никуда. Нам нужно реагировать на событие `sendRequest` чтобы вызывать эффект `sendRequestFx`.

Обратите внимание что мы вызываем событие с аргументом `1000`, это наша задержка которая будет передаваться в наш эффект.

---

## Forward

`forward` - метод создания связи между юнитами декларативным способом. Отправляет обновления с (`from`) одного юнита в (`to`) другой.

В нашем случае нужно с юнита `sendRequest` перенаправить данные в `sendRequestFx`.

```js
forward({
  from: sendRequest,
  to: sendRequestFx,
});
```

Вот теперь пример уже рабочий, разберем по шагам что тут происходит:

1. Пользователь нажимает на кнопку
2. Вызывается событие `sendRequest` с `payload` - `1000`
3. `forward` реагирует на событие `sendRequest` и вызывает `sendRequestFx` передавая первым параметром `1000`
4. `sendRequestFx` ожидает окончания таймаута, после его выполнения триггерит внутреннее событие `done`
5. Стор `$completedRequests` реагирует на событие `sendRequestFx.done` вызывая редюсер для вычисления нового значения счетчика
6. `useStore` реагирует на обновление стора вызывая рендер компонента

Текущую реализацию можно потыкать в [репле](https://share.effector.dev/DDDO0NHB), что бы посмотреть на кнопку перейдите в правом верхнем углу на таб `DOM`

---

## Lockable sendRequest event

Осталось реализовать блокирование события пока эффект находиться в процессе выполнения.

Обернем пока событие `sendRequest` в обертку `lockable`, ознакомится с которой можно в [первой части](./lockable-event.md)

```js
const [lock, unlock, filteredSendRequest] = lockable(sendRequest);
```

Теперь распишем нашу логику по шагам:

1. Слушаем фильтруемые события `sendRequest`
2. Когда срабатывает триггер вызываем `lock` для включения фильтрации `sendRequest`
3. Запускаем эффект `sendRequestFx`
4. Слушаем событие окончания эффекта `sendRequestFx.finally`
5. По завершению эффекта вызываем событие `unlock` для отключения фильтрации `sendRequest`

Описываем инструкции эффектором

```js
forward({
  from: filteredSendRequest,
  to: [lock, sendRequestFx],
});

forward({
  from: sendRequestFx.finally,
  to: unlock,
});
```

Да в ключах `from` и `to` может быть массив юнитов, в первом случает это значит что при триггере `filteredSendRequest` вызови `lock` и `sendRequestFx` по порядку.

---

## TakeFirst

Этот набор логики может понадобиться где-то еще в приложении, да и понять описание будет легче если вынести это в отдельную функцию, например `takeFirst`

```js
function takeFirst(event, effect) {
  const [lock, unlock, filteredEvent] = lockable(event);

  forward({
    from: filteredEvent,
    to: [lock, effect],
  });

  forward({
    from: effect.finally,
    to: unlock,
  });
}
```

Теперь для ее использования достаточно передать `event` и `effect`, а уже функция построит все нужные связи между юнитами.

```js
takeFirst(sendRequest, sendRequestFx);
```

[Репл](https://share.effector.dev/FsQCBdVP) c полным исходным кодом
