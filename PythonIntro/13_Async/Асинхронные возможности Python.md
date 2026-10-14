В данной главе будет рассмотрена одна из наиболее важных возможностей Python. **Асинхронность** - это последовательное выполнение произвольных кусков кода.

# Быстрый поиск

 + [Общее понимание асинхронности](#общее-понимание-асинхронности)
 + [Модель асинхронной работы кода](#модель-асинхронной-работы-кода)
   + [Асинхронность как произвольное исполнение частей кода между yield-ами](#асинхронность-как-произвольное-исполнение-частей-кода-между-yield-ами)
   + [Ещё модели](#ещё-модели)
   + [Синтаксис Async](#синтаксис-async)
 + [Asyncio](#asyncio)
   + [Введение в высокоуровневое API](#введение-в-высокоуровневое-api)

---

# Общее понимание асинхронности

_Прямой ассинхронности_ (непосредственно, параллелизма) в Python нет. Существуют модули, позволяющие воспользоваться _псевдопараллелизмом ОС_, например, [subprocess](https://docs.python.org/3/library/subprocess). Там предусмотрены средства синхронизации, это тоже не совсем параллелизм, но в работу ОС мы здесь погружаться не будем.

Термину асинхронности соответствует целая парадигма [событийно-ориентированного программирования](https://ru.wikipedia.org/wiki/%D0%A1%D0%BE%D0%B1%D1%8B%D1%82%D0%B8%D0%B9%D0%BD%D0%BE-%D0%BE%D1%80%D0%B8%D0%B5%D0%BD%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%BD%D0%BE%D0%B5%20%D0%BF%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5), в которой программа — это обработчик независимо порождаемых событий. Как правило, понятие «события» привязано к внешним источникам, а программа представляет собой более или менее обычный [цикл обработки](https://ru.wikipedia.org/wiki/%D0%A6%D0%B8%D0%BA%D0%BB_%D1%81%D0%BE%D0%B1%D1%8B%D1%82%D0%B8%D0%B9), то есть цикл и вызов методов-обработчиков. Примерами модулей, поддерживающих такую парадигму, являются [tkinter](https://docs.python.org/3/library/tkinter) или игровые движки [PyGame](https://www.pygame.org/) и [The Python Arcade Library](https://api.arcade.academy/).

Нас же будет в большей степени интересовать _сопрограммная асинхронность_. Описывается несколько сопрограмм с естественными синхронными участками, асинхронность заключается в алгоритме переключения между выполнения таких участков.


# Модель асинхронной работы кода

Сопрограммная асинхронность явно описывает работу python-генераторов. Поэтому для понимания работы асинхронных модулей и синтаксиса `async` вспомним, как работают [генераторы](../07_Iterators/Итераторы.html#генераторы).

При описании генератора используется ключевое слово `yield` для «замораживания» генератора в текущем состоянии и передачи параметра выше по стеку фреймов. После повторного обращения к генератору происходит возврат в точку выхода (в `yield`) и продолжение работы генератора с этого места. Для вызова генераторов внутри других генераторов с пробросом значений наверх используется `yield from`:

```python
>>> def subr():
...     yield "One"
...     yield "Two"
...
... def task():
...     for i in range(3):
...         yield from subr()
...         yield f"{i} Pass"
...
... for res in task():
...     print(res)
...
One
Two
0 Pass
One
Two
1 Pass
One
Two
2 Pass
>>>
```

На время выполнения yield from код генератора task() логически не исполняется, так что можно считать, что на это время его замещает subr(), как раз демонстрируя асинхронность работы кода.

Для передачи данных из генератора во внешний фрейм:
 + От `yield from` параметры приезжают сами;
 + По завершении генератора через `return` можно добавить параметр в поле `.value` исключения `StopIteration`.

```python
>>> def subr(n):
...     yield f"One: {n}"
...     yield f"Two: {n}"
...     return f"Done: {n}"
...
... def task():
...     for i in range(3):
...         result = yield from subr(i)
...         yield result
...     return "*END*"
...
... core = task()
... try:
...     while (res := next(core)):
...         print(res)
... except StopIteration as E:
...     print(E.value)
...
One: 0
Two: 0
Done: 0
One: 1
Two: 1
Done: 1
One: 2
Two: 2
Done: 2
*END*
>>>
```

Для передачи в генератор используется `.send()` (помним, что первый вызов всегда `.send(None)` или `.next()`: это _запуск_ генератора с указанными при создании параметрами):

```python
>>> def task(initial):
...     value = initial
...     while True:
...         value = yield f"<{value * 2}>"
...
... core = task(100500)
... print(f"Start: {next(core)}")
... for i in range(5):
...     print(core.send(i + 1))
...
Start: <201000>
<2>
<4>
<6>
<8>
<10>
>>>
```

При передаче параметров из внешнего фрейма они попадают в тот итератор, который совершал yield. И всё равно не всегда бывает просто с ходу понять, куда какие параметры передаются:

```python
>>> def subr():
...     x = yield "Wait for x"
...     y = yield f"Wait for y ({x=})"
...     return x, y
...
... def task():
...     while True:
...         value = yield from subr()
...         _ = yield value
...
... core = task()
... print(next(core))
... for i in range(8):
...     print(core.send(i))
...
Wait for x
Wait for y (x=0)
(0, 1)
Wait for x
Wait for y (x=3)
(3, 4)
Wait for x
Wait for y (x=6)
(6, 7)
>>>
```

## Асинхронность как произвольное исполнение частей кода между yield-ами

Вспомнив логику работы генераторов, опишем её в терминах асинхронности:
 + Непрерывно выполняемые блоки кода генераторов (между входом в него, yield-ами и выходом) — _синхронные фрагменты_;
 + Внешний фрейм, который _по какому-то правилу_ обращается к генераторам (т.е. произвольным синхронным фрагментам) — _образующий цикл_.

Алгоритм, согласно которому образующий цикл обращается к фрагментам, может быть _абсолютно любым_. Даже простое последовательное обращение к генераторам формально уже считается асинхронным. Но обращение может быть и более сложным:

```python
>>> def subr(n):
...     x = yield f"({n}) Wait for x"
...     y = yield f"({n}) Wait for y ({x=})"
...     return x, y
...
... def task(n):
...     yield f"Start {n}"
...     while True:
...         value = yield from subr(n)
...         _ = yield f"[{n}]: {value}"
...
... cores = task(0), task(1)
... print(next(cores[0]), next(cores[1]), sep="\n")
... for i in range(20):
...     print(cores[not i % 3].send(i))
...
Start 0
Start 1
(1) Wait for x
(0) Wait for x
(0) Wait for y (x=2)
(1) Wait for y (x=3)
[0]: (2, 4)
(0) Wait for x
[1]: (3, 6)
(0) Wait for y (x=7)
[0]: (7, 8)
(1) Wait for x
(0) Wait for x
(0) Wait for y (x=11)
(1) Wait for y (x=12)
[0]: (11, 13)
(0) Wait for x
[1]: (12, 15)
(0) Wait for y (x=16)
[0]: (16, 17)
(1) Wait for x
(0) Wait for x
>>>
```

Здесь из образующего цикла поступает поток целых чисел, `subr()` их попарно умножает, а две задачи складывают эти произведения. Очередное число попадает в `subr()` выбранной задачи, выбор задач делает образующий цикл. Синхронные фрагменты из `task[0]` выполняются в два раза чаще синхронных фрагментов из `task[1]`.

Рассмотрим более сложный пример:  три конечных задачи с разным количеством синхронных фрагментов.

```python
>>> def subr():
...     x = yield
...     y = yield
...     return [x,  y]
...
... def task(num):
...     res = []
...     for i in range(num):
...         res += yield from subr()
...     return res
...
... def loop(*tasks):
...     queue, result = list(tasks), []
...     print("Start:", *queue, sep="\n\t")
...     for task in tasks:
...         next(task)
...     step = 0
...     while queue:
...         task = queue.pop(0)
...         try:
...             task.send(step)
...         except StopIteration as ret:
...             result.append((hex(id(task)), ret.value))
...         else:
...             queue.append(task)
...         step += 1
...     return result
...
... print("Done:", *loop(task(7), task(2), task(5)), sep="\n\t")
...
Start:
        <generator object task at 0x7f1368fd1a80>
        <generator object task at 0x7f1368fd1d20>
        <generator object task at 0x7f1368fd1c40>
Done:
        ('0x7f1368fd1d20', [1, 4, 7, 10])
        ('0x7f1368fd1c40', [2, 5, 8, 11, 13, 15, 17, 19, 21, 23])
        ('0x7f1368fd1a80', [0, 3, 6, 9, 12, 14, 16, 18, 20, 22, 24, 25, 26, 27])
>>>
```

Образующий цикл здесь вынесен в отдельную функцию. В нём генерируется последовательность целых чисел и отдаётся поштучно на обработку очередному заданию. Если задание закончилось, запоминается его результат, иначе оно ставится в конец очереди. Для реализации этой логики используется явная обработка StopIteration. Значения, возвращаемые yield, при этом не используются вообще: yield служит только для разметки синхронных фрагментов.

Если ещё усложнить логику образующего цикла, мы сможем управлять его поведением с помощью возвращаемых yield-значений:

```python
>>> from random import randint
... from string import ascii_uppercase
... from collections import deque
...
... def subr():
...     return (yield int) * (yield str)
...
... def task(num):
...     res = ""
...     for i in range(num):
...         res += yield from subr()
...     return res
...
... def loop(*tasks):
...     queue, result = deque((task, None) for task in tasks), []
...     print("Start:", *queue, sep="\n\t")
...     idx = -1
...     while queue:
...         task, request = queue.popleft()
...         if request is int:
...             data = randint(1, 4)
...         elif request is str:
...             data = ascii_uppercase[idx := idx + 1]
...         else:
...             data = request
...         try:
...             request = task.send(data)
...         except StopIteration as ret:
...             result.append((task, ret.value))
...             task.close()
...         else:
...             queue.append((task, request))
...     return result
...
... print("Done:", *loop(task(10), task(3), task(5)), sep="\n\t")
...
...
Start:
        (<generator object task at 0x7f1368fd1c40>, None)
        (<generator object task at 0x7f1368fd1e00>, None)
        (<generator object task at 0x7f1368fd1d20>, None)
Done:
        (<generator object task at 0x7f1368fd1e00>, 'BEEHHHH')
        (<generator object task at 0x7f1368fd1d20>, 'CCCFFFIKMMMM')
        (<generator object task at 0x7f1368fd1c40>, 'ADDDGGJLLLNNNNOOOOPPQQQR')
>>>
```

Здесь subr() возвращает тип параметра, который она хотела бы получить в следующем yield. Этот тип хранится в очереди вместе с заданием, чей subr() запросил данный параметр. Образующий цикл генерирует параметр сообразно типу.

 + Бонусом здесь очередь заданий образующего цикла это _именно очередь_, а не список.

## Ещё модели

Общая логика такой асинхронной модели понятна: образующий цикл согласно некоторым правилам обращается к генераторам, те выполняют свои синхронные фрагменты, возвращают в образующий цикл либо результат работы, либо какое-то служебное данное, согласно которому производится следующее к ним обращение.

Кроме такой существует ещё несколько моделей асинхронной работы:
 1. _Цикл событий_: образующий цикл получает откуда-то «события», определяет, кто их должен обрабатывать и вызывает функции-обработчики с параметром _обработчик(событие)_ (возможно, не функции, а генераторы _обработчик.send(событие)_).
 2. _Цикл обратных вызовов_ как частный случай цикла событий: каждый обработчик «регистрируется» — по заранее определённому протоколу указывает, в каких случаях его надо вызывать (это и есть событие) —, а образующий цикл при наступлении события вызывает все обработчики, которые на нём зарегистрировались (в виде функций или в виде генераторов)
 3. _Цикл с фьючами_ (future, promise). future — это генератор, в котором есть поле «готовность / результат»; изначально фьюча _не готова_.

Фьюча состоит из двух синхронных сегментов:
 + Настройка и yield себя в образующий цикл
 + return готового результата пользователю

Алгоритм работы фьючи описывается буквально в [6 строк](https://github.com/python/cpython/blob/2149a979aa254f581b7bbe15d9376ff7d1a3a57f/Lib/asyncio/futures.py#L292):
 + Неготовая фьюча заводится в данном образующем цикле
 + Образующий цикл вызывает `next(сопрограмма)`
 + Сопрограмма делает `yield from фьюча`
 + Фьюча, в свою очередь, тут же выпадает в образующий цикл, потому что она ещё не готова (первый сегмент)
   + Если фьюча уже готова, вызовы _yield from_ фьюча сразу возвращают результат (второй сегмент)
 + Образующий цикл продолжает работу, проверяя, что фьюча не готова
 + В какой-то момент некто выставляет фьюче готовность / результат
 + На этом основании образующий цикл возвращает управление фьюче `next(фьюча)` (во второй сегмент)
   + Если фьюча всё ещё не готова — это ошибка алгоритма в образующем цикле, так делать нельзя
 + Фьюча возвращает значение пользователю

## Синтаксис Async

Для полноценного различия генераторов и асинхронного кода (и далее для использования в модулях, связанных с асинхронностью) используется отдельный синтаксис для описания _сопрограмм_ (корутин). Сопрограмма это, своего рода, генератор на один шаг. Описание сопрограммы выглядит как описание обычной функции с ключевым словом `async`. Ключевое слово `yield` в общем случае не используется, его роль берёт на себя `return` (потому генератор и на _один_ шаг). Для вызова других сопрограмм вместо `yield from` используется ключевое слово `await`.

```python
>>> async def hello(name):
...     print(f'Hello, {name}!')
...     return 42
...
>>> coro = hello("You")
>>> coro
<coroutine object hello at 0x7f13690336b0>
>>> coro.send(None)
Hello, You!
Traceback (most recent call last):
  File "<python-input-9>", line 1, in <module>
    coro.send(None)
    ~~~~~~~~~^^^^^^
StopIteration: 42
>>>
```

`yield` может использоваться в сопрограммах. Получаются _асинхронные генераторы_, которые можно проходить с помощью `async for`.

Рассмотрим последний пример асинхронной работы, переписав его под async:

```python
from random import randint
from string import ascii_uppercase
from types import coroutine
from collections import deque

@coroutine
def subr():
    return (yield int) * (yield str)

async def task(num):
    res = ""
    for i in range(num):
        res += await subr()
    return res

def loop(*tasks):
    queue, result = deque((task, None) for task in tasks), []
    print("Start:", *queue, sep="\n\t")
    idx = -1
    while queue:
        task, request = queue.popleft()
        if request is int:
            data = randint(1, 4)
        elif request is str:
            data = ascii_uppercase[idx := idx + 1]
        else:
            data = request
        try:
            request = task.send(data)
        except StopIteration as ret:
            result.append((task, ret.value))
            task.close()
        else:
            queue.append((task, request))
    return result

print("Done:", *loop(task(10), task(3), task(5)), sep="\n\t")
```

Здесь дополнительно используется специальный декоратор `@coroutine`: низкоуровневая сопрограмма, которая может делать и `return` значение, и `yield`, то есть напрямую обращаться к образующему циклу.

# Asyncio

Самое сложное в описании асинхронного кода — логика образующего цикла. При этом эта самая логика совершенно не важна. Достаточно понимать, как пользоваться образующим циклом. Именно такой логики и придерживались авторы модуля [Asyncio](https://docs.python.org/3/library/asyncio):
 + Запрограммируем образующий цикл заранее, насуём туда инструментов
 + Упростим протокол управления до одного понятия — Future
 + Обмажем протокол верхним уровнем (задания, события, очереди и т. п.)
   + До такой степени, что ни одна из наших сопрограмм не делает yield (если это не _асинхронный генератор_)
 + (asyncio specific) обмажем огромным количеством применений IRL

В Asyncio используются следующие понятия:
 + Mainloop — образующий цикл, скрытый от разработчика.
 + Task — сопрограмма, управляемая образующим циклом.

```python
>>> import asyncio
... from time import strftime
...
... async def late(delay, msg):
...     await asyncio.sleep(delay)
...     print(msg)
...
... async def main():
...     print(f"> {strftime('%X')}")
...     await late(1, "One")
...     print(f"> {strftime('%X')} + 1")
...     await late(2, "Two")
...     print(f"> {strftime('%X')} + 2")
...
...     task3 = asyncio.create_task(late(3, "Three"))
...     task4 = asyncio.create_task(late(4, "Four"))
...     await task3
...     print(f"> {strftime('%X')} + 3")
...     await task4
...     print(f"> {strftime('%X')} + <<1>>")
...
... asyncio.run(main())
...
> 12:12:06
One
> 12:12:07 + 1
Two
> 12:12:09 + 2
Three
> 12:12:12 + 3
Four
> 12:12:13 + <<1>>
>>>
```

 + `asyncio.run(main())` — запуск «приложения» main() в образующем цикле asyncio()
 + «приложение» asyncio — корутина, который заполняет очередь mainloop-а и немножко командует им
 + `asyncio.sleep(тайм-аут)` — это команда mainloop-у «верни мне управление после тайм-аута»
   + Чуть ли не единственная команда asyncio-шному mainloop-у на поверхности

Если просто написать _await_, корутина «просто запустится», и асинхронность её работы будет незаметна (даже если она и выходила в mainloop, выполнения выглядит синхронным). Так в примере первая корутина спит секунду, а вторая — _после этого_ ещё две.

Если же написать `create_task(корутина)`, корутина регистрируется в mainloop-е, а возвращается нечто вроде фьючи — _задание_. `await(задание)` запускает его. И вот здесь асинхронность заметна: при регистрации задания таймер сразу начинает идти. И поскольку две корутины планируются одновременно, и первая из них спит три секунды, а вторая — четыре, вторая отрабатывает _через секунду_ после первой.

Asyncio предлагает множество вариантов создания заданий и их запуска. Первое — `gather`: атомарная операция создания и запуска над несколькими корутинами.

```python
>>> import asyncio
...
... async def late(delay, msg):
...     await asyncio.sleep(delay)
...     print(msg)
...     return delay
...
... async def main():
...     res = await asyncio.gather(
...             late(3, "A"),
...             late(1, "B"),
...             late(2, "C"),
...     )
...     print(res)
...
... asyncio.run(main())
...
B
C
A
[3, 1, 2]
>>>
```

В первом примере регистрация заданий была не различима для человека, но всё же последовательна, в данном случае задание и запуск были именно атомарно одновременны. Кроме этого `gather` позволяет получить результат в порядке описания сопрограмм, а не в порядке из завершения.

Для одновременного запуска заданий можно использовать [группы заданий](https://docs.python.org/3/library/asyncio-task.html) и обращение к ним в виде контекстного менеджера:

```python
>>> import asyncio
...
... async def late(delay, msg):
...     await asyncio.sleep(delay)
...     print(msg)
...     return delay
...
... async def main():
...     async with asyncio.TaskGroup() as tg:
...         tg.create_task(late(3, "A"))
...         tg.create_task(late(1, "B"))
...         tg.create_task(late(2, "C"))
...     print("Done")
...
... asyncio.run(main())
...
B
C
A
Done
>>>
```

## Введение в высокоуровневое API

Здесь мы рассмотрим несколько направлений высокоуровнего использования Asyncio.

При асинхронной работе нескольких корутин несомненно появляется вопрос о [синхронизации](https://docs.python.org/3/library/asyncio-sync.html) из работы. Asyncio предлагает большой спектр синхронизирующих сопрограмм. Рассмотрим [события](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Event). Эта корутина представляет собой объект, который _можно ожидать_ (тогда управление из ожидающей сопрограммы переходит в образующий цикл) и _можно задать_ (тогда событие считается произошедшим, и все, кто его ожидал смогут быть вызваны образующим циклом и продолжать своё выполнение).

```python
>>> async def waiter(name, event):
...     print(f'{name} waits for {event}…')
...     await event.wait()
...     print(f'…{name} got it!')
...
... async def eventer(wait, event):
...     print(f"Emitting {event} in {wait} seconds")
...     await asyncio.sleep(wait)
...     print(f"Emitting {event}…")
...     event.set()
...
... async def main():
...     event = asyncio.Event()
...     await asyncio.gather(
...         waiter("One", event),
...         waiter("Two", event),
...         eventer(1, event))
...
... asyncio.run(main())
...
One waits for <asyncio.locks.Event object at 0x7f13685a41a0 [unset]>…
Two waits for <asyncio.locks.Event object at 0x7f13685a41a0 [unset, waiters:1]>…
Emitting <asyncio.locks.Event object at 0x7f13685a41a0 [unset, waiters:2]> in 1 seconds
Emitting <asyncio.locks.Event object at 0x7f13685a41a0 [unset, waiters:2]>…
…One got it!
…Two got it!
>>>
```

Похожим свойством обладают [барьеры](https://docs.python.org/3/library/asyncio-sync.html#asyncio.Barrier). Их отличие заключается в определении готовности: она не задаётся кем-то, а определяется автоматически, когда нужное количество сопрограмм «упирается» в барьер — ожидает его.

Для асинхронной передачи данных между сопрограммами (тоже обеспечивая этим синхронизацию работы) используются [очереди](https://docs.python.org/3/library/asyncio-queue.html). В случае добавления объекта в очередь и взятия объекта из очереди выхода в образующий цикл не происходит. Он срабатывает при обращении в пустую очередь (что логично: нужно же передать управление кому-то, кто в эту очередь что-то положит, а то все зависнут).

```python
>>> async def ham(queue, size):
...     for i in range(size):
...         await asyncio.sleep(1)
...         res = await queue.get()
...         print(f"\tGot {res}")
...
... async def spam(wait, queue):
...     for i in range(6):
...         await asyncio.sleep(wait)
...         val = f"{wait}:{i}"
...         await queue.put(val)
...         print(f"Put {val}")
...
... async def main():
...     queue = asyncio.Queue()
...     await asyncio.gather(
...         ham(queue, 12),
...         spam(0.4, queue),
...         spam(1.6, queue))
...
... asyncio.run(main())
...
Put 0.4:0
Put 0.4:1
        Got 0.4:0
Put 0.4:2
Put 1.6:0
Put 0.4:3
        Got 0.4:1
Put 0.4:4
Put 0.4:5
        Got 0.4:2
Put 1.6:1
        Got 1.6:0
Put 1.6:2
        Got 0.4:3
        Got 0.4:4
Put 1.6:3
        Got 0.4:5
Put 1.6:4
        Got 1.6:1
        Got 1.6:2
Put 1.6:5
        Got 1.6:3
        Got 1.6:4
        Got 1.6:5
>>>
```

С помощью asyncio можно обеспечить работу с множеством потоков ввода-вывода. И если добавить туда сеть, получается классический обработчик сетевых соединений. Asyncio поддерживает создание собственного TCP-сервера, который асинхронно обрабатывает получаемые запросы с _произвольным_ количеством соединений.

```python
>>> import asyncio
...
... async def echo(reader, writer):
...     while data := await reader.readline():
...         writer.write(data.swapcase())
...     writer.close()
...     await writer.wait_closed()
...
... async def main():
...     server = await asyncio.start_server(echo, '0.0.0.0', 1337)
...     async with server:
...         await server.serve_forever()
...
... asyncio.run(main())
...


```

```console
[papillon_rouge@BaseALT-Papillon ~]$ netcat localhost 1337
Hello
hELLO
Python
pYTHON
^C
[papillon_rouge@BaseALT-Papillon ~]$
```