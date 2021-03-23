# Lockable event

`event` - в рамках Effector это какое-то событие в системе, например нажатие кнопки пользователем

`lockable` это своя самописная обертка которая принимает переданное ей событие и возвращает два события lock и unlock, а так же событие target на которое можно подписаться для получения не проигнорированных событий

---

## Пример использования:

```js
const trigger = createEvent();

const [lock, unlock, target] = lockable(trigger);

target.watch(console.log);

trigger("A"); // => "A"

lock();
trigger("B"); // nothing happens
unlock();

trigger("C"); // => "C"
```

---

## Разберем по строчно

```js
// создаем событие trigger, которое может быть вызвано нами или юзером
const trigger = createEvent();

// передаем наше событие в обертку которая возвращает
// нам два события lock и unlock, а так же target
const [lock, unlock, target] = lockable(trigger);

// подписываемся на уведомления от target
// и вызываем наше событие попутно дергая lock или unlock

target.watch(console.log);
trigger("A"); // => "A"

lock();
trigger("B"); // nothing happens
unlock();

trigger("C"); // => "С"
```

---

## Реализация lockable

```js
function lockable(event) {
  const lock = createEvent();
  const unlock = createEvent();

  const $unlocked = createStore(true)
    .on(lock, () => false)
    .on(unlock, () => true);

  const target = guard(event, {
    filter: $unlocked,
  });

  return [lock, unlock, target];
}
```

---

## Разбор

1. Создаем два события для локов
2. Создаем стор который отвечает за состояние лока - `$unlocked` (примечание: в effector для сторов принято приписывать префикс `$` чтобы стразу было понятно что мы работаем со стором)
3. Через on мы навешиваем наши события и во втором параметре функции редюсере мы обрабатываем то как наши события изменяют стор, в нашем случае мы просто меняем флаг состояния лока
4. получения именно тех событий которые прошли фильтр

В следующем посте я хочу показать как мы можешь работать с этим на практическом примере в реакте и то как мы можешь комбинировать наше полученное апи lockable с остальным апи effector

Весь код можно потыкать в [репле](https://share.effector.dev/hZ40ElwJ), если что-то не совсем понятно то велком в коментарии
