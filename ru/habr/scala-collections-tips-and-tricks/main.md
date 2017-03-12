Секреты и трюки Scala коллекций
===============================

<img src="https://pavelfatin.com/images/scala-collections.jpg"
     style="float:left">

Представляю вашему вниманию перевод статьи
[Павла Фатина](https://pavelfatin.com/about)
[Scala Collections Tips and Tricks](https://pavelfatin.com/scala-collections-tips-and-tricks/).
Павел работает в [JetBrains](https://www.jetbrains.com/) и занимается разработкой
[Scala плагина](https://confluence.jetbrains.com/display/SCA/Scala+Plugin+for+IntelliJ+IDEA)
для IntelliJ IDEA.


Данная статья представляет собой набор оптимизаций упрощений для типичного
[интерфейса коллекций Scala](https://www.scala-lang.org/docu/files/collections-api/collections.html) usages.

Некоторые советы основаны на тонкостях реализации библиотеки коллекций, однако
большинство рецептов -- являются разумными преобразованиями, которые на практике
часто упускаются из виду.

Этот список вдохновлен моими попытками разработать практичные
[инспекции для Scala коллекций](https://youtrack.jetbrains.com/oauth?state=%2Fissues%2FSCL%3Fq%3Dby%253A%2BPavel.Fatin%2Bcollection%2Border%2Bby%253A%2Bcreated)
для [Scala плагина IntelliJ](https://confluence.jetbrains.com/display/SCA/Scala+Plugin+for+IntelliJ+IDEA).
Сейчас мы в процессе осуществления данных инспекций,
так что если вы используете этот плагин в IDEA
вы автоматически получаете пользу от статического анализа кода

Тем не менее, эти рецепты ценны сами по себе и могут помочь вам углубить ваше
понимание стандартной библиотеки коллекций Scala и сделать ваш код быстрее и
чище


**Содержание:**

  1. Легенда
  2. Композиция
  3. Побочные эффекты
  4. Последовательности (Sequences)
    4.1. Создание
    4.2. Длина
    4.3. Равенство
    4.4. Индексация
    4.5. Существование
    4.6. Фильтрация
    4.7. Сортировка
    4.8. Свертка
    4.9. Сопоставление
    4.10. Переписываем
  5. Множества (Sets)
  6. Options
    6.1. Значение
    6.2. Null
    6.3. Обработка
    6.4. Переписываем
  7. Maps
  8. Дополнение

Все примеры кода доступны в [репозитории на GitHub](https://github.com/pavelfatin/scala-collections-tips-and-tricks).


## 1. Легенда
Чтобы сделать примеры кода более понятными, я придерживался следующих условных
обозначений:

  * `seq` — экземпляр основанной на `Seq` коллекции, вроде `Seq(1, 2, 3)`
  * `set` — экземпляр `Set`, например `Set(1, 2, 3)`
  * `array` — массив, такой как `Array(1, 2, 3)`
  * `option` — экземпляр `Option`, например, `Some(1)`
  * `map` — экземпляр `Map`, подобный `Map(1 -> "foo", 2 -> "bar")`
  * `???` — произвольное выражение
  * `p` — предикат функции типа `T => Boolean`, например `_ > 2`
  * `n` — целочисленное значение
  * `i` — целочисленный индекс
  * `f`, `g` — простые функции, `A => B`
  * `x`, `y` — некоторые произвольные значения
  * `z` — начальное, или значение по-умолчанию
  * `P` — паттерн

## 2. Композиция
Имейте в виду что несмотря на то что все рецепты изолированны и самостоятельны
мы можем соединять (compose), итеративно превращая в более прогрессивные
выражения, например:

    seq.filter(_ == x).headOption != None

    // seq.filter(p).headOption -> seq.find(p)

    seq.find(_ == x) != None

    // option != None -> option.isDefined

    seq.find(_ == x).isDefined

    // seq.find(p).isDefined -> seq.exists(p)

    seq.exists(_ == x)

    // seq.exists(_ == x) -> seq.contains(x)

    seq.contains(x)

Таким образом, мы можем полагаться на "заменяющую модель применения рецептов"
(по аналогии с [SICP](https://mitpress.mit.edu/sicp/full-text/sicp/book/node10.html))
для упрощения комплексных выражений.

## 3. Побочные эффекты
"Побочный эффект" является необходимой концепцией, которую следует рассмотреть
перед тем, как перечислять основные преобразования.

По сути, побочный эффект это некоторые действие, помимо возврата значения,
наблюдаемое вне тела функции, такое как:

  * операция ввода-вывода,
  * модификация переменной (доступной вне тела области видимости),
  * изменение состояния объекта (наблюдаемое вне области видимости),
  * возбуждение исключения (которое не обрабатывается внутри области видимости).

О функциях или выражение содержащих любое из вышеперечисленных действий,
говорят, что они содержат побочные эффекты, в противном случае их называют
"чистыми".

Почему побочные эффекты так важны?
Потому что при наличии побочных эффектов, порядок исполнения начинает иметь
значение. Например, ниже представлены два "чистых" выражения, связанных с
(соответствующими значениями):

    val x = 1 + 2
    val y = 2 + 3

Поскольку они не содержат побочных эффектов (т.е. эффектов, наблюдаемых вне
выражения), мы можем вычислить эти выражения в различном порядке — сначала
`x`, а затем `y`, или сначала `y`, а затем `x` — это не повлияет на корректность
полученных результатов (мы можем даже закешировать результирующие значения, если
того захотим). Теперь, давайте представим следующую модификацию:

    val x = { print("foo"); 1 + 2 }
    val y = { print("bar"); 2 + 3 }

Теперь у нас совсем другая история — мы не можем изменить порядок выполнения,
и увидеть "barfoo" вместо "foobar" в нашем терминале (и это не то, что мы
ожидали).

Таким образом, присутствие побочных эффектов **уменьшает количество возможных
преобразований** (включая упрощения и оптимизации), которые мы можем применить
к коду.

Подобное объяснение может быть применено и к выражениям связанным с коллекциями.
Давайте представим, что у нас есть некий `builder` вне области видимости,
(с методом `append`, имеющим побочный эффект).

    seq.filter{ x => builder.append(x); x > 3 }.headOption

В принципе конструкция, `seq.filter(p).headOption` сократима до вызова
`seq.find(p)`, однако наличие побочных эффектов не позволяет нам это сделать:

    seq find { x => builder.append(x); x > 3 }

Не смотря на то, что эти выражения эквивалентны с точки зрения результирующего
значения, они не эквивалентны из-за побочных эффектов. Предыдущий пример добавит
все элементы, в то время как последний пропустит элементы после первого
соответствия предикату. Таким образом, подобное упрощение не сможет иметь место.

Что можно сделать для того, чтобы сделать автоматические упрощения возможными?
Ответ является **золотым правилом** которое должно быть применимо ко всем
побочным эффектам в вашем коде (даже включая код, не имеющий коллекций совсем):

  * избегать побочных эффектов, пока это возможно
  * иначе, изолировать побочные эффекты от чистого кода.

Поэтому, мы необходимо либо избавиться от `builder`а (вместе с его API,
содержащим побочные эффекты), или отделить вызов `builder`а от чистого выражения
Предположим, что этот  `builder` является неким второстепенным объектом, от
которого мы можем избавиться, чтобы изолировать вызов:

    seq.foreach(builder.append)
    seq.filter(_ > 3).headOption

Сейчас мы можем безопасно выполнить преобразование:

    seq.foreach(builder.append)
    seq.find(x > 3)

Чисто и красиво! Изоляция побочных эффектов делает автоматические преобразования
возможными. Дополнительной пользой будет и то, что в виду наличия чистого
отделения, результирующий код легче понять человеку.

Наименее очевидной, однако наиболее заметной выгодой от изоляции побочных
эффектов будет повышение надежности вашего кода, относительно других возможных
оптимизаций.

Применительно к примеру, первоначальное выражение может порождать различные
побочные эффекты зависящие от текущей реализации `Seq`. Для `Vector`, например,
оно добавит все элементы, для `Stream` оно пропустит все элементы после первого
удачного сопоставления с предикатом (потому что стримы являются "ленивыми" —
элементы вычисляются только тогда, когда это необходимо). Изоляция побочных
эффектов позволяет нам избежать этого неопределенного поведения.


## 4. Последовательности (Sequences)
While tips in the current section are intended for classes
that are descendants of `Seq`, some transformations are applicable
to other collection (and non-collection) classes,
like `Set`, `Option`, `Map` and even `Iterator`
(because all of them provide similar interfaces with monadic methods).

### 4.1 Создание
#### Создавайте пустые коллекции явно

    // До
    Seq[T]()

    // После
    Seq.empty[T]

Некоторые неизменяемые (immutable) классы коллекций имеют синглтон с реализацией
метода `empty`. Однако, далеко не все из этих фабричных методов проверяют длину
созданных коллекций. Таким образом, делая пустоту очевидной на этапе компиляции,
вы можете сохранить либо место в куче (путем переиспользования экземпляра), либо
такты процессора (потраченные бы на проверки размерности во время выполнения).

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.

### 4.2 Длины
#### Используйте `length` вместо `size` для массивов

    // До
    array.size

    // После
    array.length

Несмотря на то что, `size` и `length` являются практически синонимами, в
Scala 2.11 вызовы `Array.size` по прежнему выполняются через неявное
преобразование (implicit conversion), таким образом создаются промежуточные
объекты-обертки для каждого вызова метода. Если вы конечно не включите
[эскейп анализ](https://en.wikipedia.org/wiki/Escape_analysis) для JVM, подобные
временные объекты будут обузой для сборщика мусора и деградируют
производительность кода (особенно внутри циклов).

#### Не отрицайте проверки на пустоту

    // До
    !seq.isEmpty
    !seq.nonEmpty

    // После
    seq.nonEmpty
    seq.isEmpty

Простые свойства добавляют меньше визуального шума, нежели составные выражения.

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


#### Не вычисляйте длину при проверке на пустоту.

    // До
    seq.length > 0
    seq.length != 0
    seq.length == 0

    // После
    seq.nonEmpty
    seq.nonEmpty
    seq.isEmpty

С одной стороны, простое свойство воспринимается гораздо легче, нежели
составное выражение. С другой стороны, для коллекций-наследников `LinearSeq`
(таких как `List`) может потребоваться `O(n)` времени на вычисление длины
списка (вместо `O(1)` для `IndexedSeq`), таким образом мы можем ускорить наш
код избегая вычисления длины когда нам вобщем-то, это значение и не нужно.

Имейте также в виду, что вызов `.length` для бесконечных стримов может никогда
не закончиться, поэтому всегда проверяйте стрим на пустоту явно.

Также применимо к: `Set`, `Map`.


### Don’t compute full length for length matching

    // До
    seq.length > n
    seq.length < n
    seq.length == n
    seq.length != n

    // После
    seq.lengthCompare(n) > 0
    seq.lengthCompare(n) < 0
    seq.lengthCompare(n) == 0
    seq.lengthCompare(n) != 0

Поскольку вычисление размера коллекции может быть достаточно "дорогим" вычислением
для некоторых классов коллекций, мы можем сократить время сравнения с `O(length)`
до `O(length min n)` для наследников `LinearSeq` (которые могут быть спрятаны
под `Seq`-подобными значениями).

Кроме того, подобный подход незаменим ели вы имеете дело с бесконечными
стримами.


### 4.3 Равенство
#### Не полагайтесь на `==` для сравнения содержания массивов

    // До
    array1 == array2

    // После
    array1.sameElements(array2)

Проверка на равенство всегда будет выдавать `false` для различных экземпляров
массивов.

Также применимо к: `Iterator`.


#### Не проверяйте на равенство коллекции различных категорий

    // До
    seq == set

    // После
    seq.toSet == set

Проверки на равенство могут быть использованы для сравнения коллекций и
различных категорий (например `List` и `Set`).

Прошу вас дважды подумать о смысле данной проверки (касательно примера выше —
как рассматривать дубликаты в последовательности).


#### Не используйте `sameElements` для сравнения обычных коллекций

    // До
    seq1.sameElements(seq2)

    // После
    seq1 == seq2

Проверка равенства это способ, которым следует сравнивать коллекции одной и той
же категории. В теории, это может улучшить производительность из-за наличия
возможных низлежащих проверок экземпляра (`eq`, обычно намного быстрее).


#### Не используйте corresponds явно

    // До
    seq1.corresponds(seq2)(_ == _)

    // После
    seq1 == seq2

У нас уже есть встроенный метод, который делает тоже самое. Оба выражения берут
во внимание порядок элементов. И мы таки сможем выиграть в производительности.

## 4.4 Индексация
#### Не получайте первый элемент по индексу

    // До
    seq(0)

    // После
    seq.head

Обновленный подход может быть слегка быстрее для некоторых типов коллекций
(для примера, ознакомьтесь с кодом `List.apply`). Кроме того, доступ к свойству
намного проще (как синтаксически, так и семантически), чем вызов метода с
аргументом.


#### Не получайте последний элемент по индексу

    // До
    seq(seq.length - 1)

    // После
    seq.last

Последнее выражение является более очевидным и при этом позволяет избежать
избыточного вычисления длины коллекции (а для линейных последовательностей это
может занять не мало времени). Более того, некоторые классы коллекций могут
отдавать последний элемент более эффективно, в сравнении с доступом по индексу.


#### Не проверяйте нахождение индекса в границах коллекции явно

    // До
    if (i < seq.length) Some(seq(i)) else None

    // После
    seq.lift(i)

Семантически, второе выражение эквивалентно, однако более выразительно


#### Не эмулируйте headOption

    // До
    if (seq.nonEmpty) Some(seq.head) else None
    seq.lift(0)

    // После
    seq.headOption

Оптимизированное выражение более лаконично.


#### Не эмулируйте `lastOption`

    // До
    if (seq.nonEmpty) Some(seq.last) else None
    seq.lift(seq.length - 1)

    // После
    seq.lastOption

Оптимизированное выражение короче (и потенциально быстрее).


#### Будьте осторожны с типами аргументов для `indexOf` и `lastIndexOf`

    // До
    Seq(1, 2, 3).indexOf("1") // скомпилируется
    Seq(1, 2, 3).lastIndexOf("2") // скомпилируется

    // После
    Seq(1, 2, 3).indexOf(1)
    Seq(1, 2, 3).lastIndexOf(2)

В виду особенностей работы [вариантности](http://stackoverflow.com/questions/2078246/why-does-seq-contains-accept-type-any-rather-than-the-type-parameter-a/2078619#2078619),
методы `indexOf` и `lastIndexOf` принимают аргументы типа `Any`.

На практике, это может приводить к труднонаходимым багам, которые невозможно
обнаружить на этапе компиляции. Вот где вспомогательные инспекции вашей IDE
придуться к месту.


#### Не создавайте диапазон индексов последовательности вручную

    // До
    Range(0, seq.length)

    // После
    seq.indices

Существует встроенный метод, который возвращает диапазон из всех индексов
последовательности.


#### Не связывайте коллекции с индексами вручную

    // До
    seq.zip(seq.indices)

    // После
    seq.zipWithIndex

Начнем с того, что последнее выражение короче. Помимо этого, мы можем ожидать
некий прирост производительности, так как мы избегаем скрытого вычисления
размера коллекции (что, в случае линейных последовтаельностей может обойтись
недешево).
Дополнительное преимущество последнего выражения в том -- что оно хорошо
работает с возможно бесконечными коллекциями (например `Stream`).


### 4.5 Существование
#### Не используйте предикат сравения, для проверки наличия элемента

    // До
    seq.exists(_ == x)

    // После
    seq.contains(x)

Второе выражение семантически эквивалентно, однако более выразительно.

Когда эти выражения используются применительно к `Set`, производительность может
отличаться значительно отличаться, из-за того что поиск у множеств стремится к
O(1) (из-за внутреннего индексирования, не использующегося при вызове `exists`).

Также применимо к: `Set`, `Option`, `Iterator`.

#### Будьте осторожны с типом аргумента `contains`

    // До
    Seq(1, 2, 3).contains("1") // компилируется

    // После
    Seq(1, 2, 3).contains(1)


Так же как методы `indexOf` и `lastIndexOf`, `contains` принимает аргументы
типа `Any`, что может привести к труднонаходимым багам, которые не находятся на
этапе компиляции. Будьте осторожны с аргументами этих методов.

#### Не используйте предикат неравенства для проверки отсутствия элемента

    // До
    seq.forall(_ != x)

    // После
    !seq.contains(x)

И снова последнее выражение чище и, вероятно быстрее (особенно для множеств).

Также применимо к: `Set`, `Option`, `Iterator`.


#### Не считайте вхождения для проверки существования

    // До
    seq.count(p) > 0
    seq.count(p) != 0
    seq.count(p) == 0

    // После
    seq.exists(p)
    seq.exists(p)
    !seq.exists(p)

Очевидно, когда нам нужно знать, содержится ли удовлетворяющий предикату элемент
в коллекции, подсчет количества удовлетворяющих элементов избыточен.

Упрощенное выражение выглядит проще и работает быстрее.

Предикат `p` должен быть чистой функцией.

Также применимо к: `Set`, `Map`, `Iterator`.

TODO:
#### Don’t resort to filtering to check existence

    // Before
    seq.filter(p).nonEmpty
    seq.filter(p).isEmpty

    // After
    seq.exists(p)
    !seq.exists(p)

The call to `filter` creates an intermediate collection which takes heap
space and loads GC. Besides, the former expressions find all occurrences,
when only the first one is needed
(which might slowdown code, depending on likely collection contents).

The potential performance gain is less significant for lazy collections
(like `Stream` and, especially, Iterator).

The predicate `p` must be pure.

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


### 4.6 Filtering
#### Don’t negate filter predicate

    // Before
    seq.filter(!p)

    // After
    seq.filterNot(p)

The latter expression is syntactically simpler
(while semantically they are equal).

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


#### Don’t resort to filtering to count elements

    // Before
    seq.filter(p).length

    // After
    seq.count(p)

The call to `filter` creates an intermediate collection
(which is not really needed) that takes heap space and loads GC.

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


#### Don’t use filtering to find first occurrence

    // Before
    seq.filter(p).headOption

    // After
    seq.find(p)

Unless `seq` is a lazy collection (like `Stream`),
filtering will find all occurrences (and create a temporary collection),
while only the first one is needed.

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


### 4.7 Sorting

#### Don’t sort by a property manually

    // До
    seq.sortWith(_.property <  _.property)

    // После
    seq.sortBy(_.property)

We have a special method for that, which is more clear and concise.


#### Don’t sort by identity manually

    // До
    seq.sortBy(identity)
    seq.sortWith(_ < _)

    // После
    seq.sorted

There’s a special method for that, more clear and concise.


#### Perform reverse sorting in one step

    // До
    seq.sorted.reverse
    seq.sortBy(_.property).reverse
    seq.sortWith(f(_, _)).reverse

    // После
    seq.sorted(Ordering[T].reverse)
    seq.sortBy(_.property)(Ordering[T].reverse)
    seq.sortWith(!f(_, _))

That’s how we can avoid a temporary collection allocation and
exclude the additional transformation step
(to save both heap space and CPU cycles).


### 4.8 Reduction
#### Don’t calculate sum manually

    // Before
    seq.reduce(_ + _)
    seq.fold(z)(_ + _)

    // After
    seq.sum
    seq.sum + z

The advantage is clarity and conciseness.

Other possible methods are:
`reduceLeft`, `reduceRight`, `foldLeft`, `foldRight`.

The second transformation might be replaced with the first when `z` is `0`.

Также применимо к: `Set`, `Iterator`.


#### Don’t calculate product manually

    // Before
    seq.reduce(_ * _)
    seq.fold(z)(_ * _)

    // After
    seq.product
    seq.product * z

Rationale is the same as in the previous tip.

The second transformation might be replaced with the first when z is 1.

Также применимо к: `Set`, `Iterator`.


#### Don’t search for the smallest element manually

    // Before
    seq.reduce(_ min _)
    seq.fold(z)(_ min _)

    // After
    seq.min
    z min seq.min

Rationale is the same as in the previous tips.

Также применимо к: `Set`, `Iterator`.


#### Don’t search for the largest element manually

    // До
    seq.reduce(_ max _)
    seq.fold(z)(_ max _)

    // После
    seq.max
    z max seq.max

Rationale is the same as in the previous tips.

Также применимо к: `Set`, `Iterator`.


#### Don’t emulate `forall`

    // Before
    seq.foldLeft(true)((x, y) => x && p(y))
    !seq.map(p).contains(false)

    // After
    seq.forall(p)

The goal of the simplification is clarity and conciseness.

The predicate `p` must be pure.

Также применимо к: `Set`, `Option` (for the second line), `Iterator`.


#### Don’t emulate `exists`

    // Before
    seq.foldLeft(false)((x, y) => x || p(y))
    seq.map(p).contains(true)

    // After
    seq.exists(p)

Besides clarity and conciseness,
the latter expression might be faster
(because it stops element processing when a single occurrence is found)
and it can work with infinite sequences.

The predicate `p` must be pure.

Также применимо к: `Set`, `Option` (for the second line), `Iterator`.


#### Don’t emulate `map`

    // До
    seq.foldLeft(Seq.empty)((acc, x) => acc :+ f(x))
    seq.foldRight(Seq.empty)((x, acc) => f(x) +: acc)

    // После
    seq.map(f)

That’s a “classical” implementation of map via folding
in functional programming.
While it’s surely didactic, there’s no need to resort to such a
formulation when we have the concise built-in method
(which is also faster, because it uses a simple `while` loop under the hood).

Также применимо к: `Set`, `Option`, `Iterator`.


#### Don’t emulate `filter`

    // До
    seq.foldLeft(Seq.empty)((acc, x) => if (p(x)) acc :+ x else acc)
    seq.foldRight(Seq.empty)((x, acc) => if (p(x)) x +: acc else acc)

    // После
    seq.filter(p)

Rationale is the same as in the previous tip.

Также применимо к: `Set`, `Option`, `Iterator`.


#### Don’t emulate `reverse`

    // До
    seq.foldLeft(Seq.empty)((acc, x) => x +: acc)
    seq.foldRight(Seq.empty)((x, acc) => acc :+ x)

    // После
    seq.reverse

Again, the built-in method is cleaner and faster.

Также применимо к: `Set`, `Option`, `Iterator`.


### 4.9 Сопоставление
Here are several dedicated tips that are related to Scala’s
[pattern matching](http://docs.scala-lang.org/tutorials/tour/pattern-matching.html)
and [partial functions](https://www.scala-lang.org/api/current/index.html#scala.PartialFunction).


#### Use partial function instead of function with pattern matching

    // Before
    seq.map {
      _ match {
        case P => ??? // x N
      }
    }

    // After
    seq.map {
      case P => ??? // x N
    }

The updated expression produces the same result, yet looks simpler.

It’s possible to apply this transformation to any functions,
not only to `map` arguments.
This tip is not actually collection-related at all. However, because
[higher-order functions](https://en.wikipedia.org/wiki/Higher-order_function)
are so ubiquitous in Scala collection API, the tip is especially handy.


#### Convert `flatMap` with partial function to `collect`

    // Before
    seq.flatMap {
      case P => Seq(???) // x N
      case _ => Seq.empty
    }

    // After
    seq.collect {
      case P => ??? // x N
    }

The updated expression produces the same result, but looks much simpler.

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


#### Convert `match` to `collect` when the result is a collection

    // Before
    v match {
      case P => Seq(???) // x N
      case _ => Seq.empty
    }

    // After
    Seq(v) collect {
      case P => ??? // x N
    }

When all the case statements produce collections,
it’s possible to simplify the expression by converting the `match`
statement to a `collect` call.
In that way we can write collection creation only once and we can omit
the explicit default `case` branch.

Personally, I often use this trick with `Option` rather than sequences per se.

Также применимо к: `Set`, `Option`, `Iterator`.


#### Don’t emulate collectFirst

    // Before
    seq.collect{case P => ???}.headOption

    // After
    seq.collectFirst{case P => ???}

We have a special method for such a use case,
which also works faster on non-lazy collections.

The partial function must be pure.

Также применимо к: `Set`, `Map`, `Iterator`.


### 4.10 Rewriting
Merge consecutive `filter` calls

    // Before
    seq.filter(p1).filter(p2)

    // After
    seq.filter(x => p1(x) && p2(x))

That’s how we can avoid creation of an intermediate collection
(after the first filter call), and thus relieve GC’s burden.

We may also apply a generalized approach that relies on views (see below),
so the result will be `seq.view.filter(p1).filter(p2).force`.

The predicates `p1` and `p2` must be pure.

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


#### Merge consecutive `map` calls

    // Before
    seq.map(f).map(g)

    // After
    seq.map(f.andThen(g))

Again, we’re creating the resulting collection directly,
without an intermediate one.

We may also apply a generalized approach that relies on views (see below),
so the result will be `seq.view.map(f).map(g).force`.

The functions `f` and `g` must be pure.

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


#### Do sorting after filtering

    // До
    seq.sorted.filter(p)

    // После
    seq.filter(p).sorted

Sorting is computationally expensive procedure.
There’s no need to sort elements that will be potentially
filtered out in the next step.

The same applies to the other possible sorting methods,
like `sortWith` and `sortBy`.

The predicate `p` must be pure.


#### Don’t reverse collection explicitly before calling `map`

    // До
    seq.reverse.map(f)

    // После
    seq.reverseMap(f)

The first expression creates an intermediate (reversed) collection before
transforming elements, which is quite reasonable sometimes (e.g. for `List`).
Other times it’s possible to perform the required transformation directly,
without creating the intermediate collection, which is more efficient.


#### Don’t reverse collection explicitly to acquire reverse iterator

    // До
    seq.reverse.iterator

    // После
    seq.reverseIterator

Again, the latter expression is simpler and might be more efficient.


#### Don’t convert collection to `Set` to find distinct elements

    // До
    seq.toSet.toSeq

    // После
    seq.distinct

There’s no need to create a temporary set (at least explicitly) to
find distinct elements.


#### Don’t emulate slice

    // Before
    seq.drop(x).take(y)

    // After
    seq.slice(x, x + y)

For linear sequences all we gain is clear intent and conciseness.
For indexed sequences, however,
we may expect a potential performance improvements.

Также применимо к: `Set`, `Map`, `Iterator`.


#### Не эмулируйте `splitAt`

    // До
    val seq1 = seq.take(n)
    val seq2 = seq.drop(n)

    // После
    val (seq1, seq2) = seq.splitAt(n)

Оптимизированное выражения исполняется быстрее для линейных последовательностей.
(как для `List`, так и для `Stream`), в виду того что оно вычисляет результаты
за один проход. Также применимо к: `Set`, `Map`.


#### Не эмулируйте `span`

    // До
    val seq1 = seq.takeWhile(p)
    val seq2 = seq.dropWhile(p)

    // После
    val (seq1, seq2) = seq.span(p)

Вот как мы можем пройти последовательность и проверить предикат всего
один раз, а не два.

Предикат `p` не должен иметь побочных эффектов.
Также применимо к: `Set`, `Map`, `Iterator`.


#### Не эмулируйте `partition`

    // До
    val seq1 = seq.filter(p)
    val seq2 = seq.filterNot(p)

    // После
    val (seq1, seq2) = seq.partition(p)

Опять-таки, преимуществом -- однопроходное вычисление

Предикат `p` не должен иметь побочных эффектов.
Также применимо к: `Set`, `Map`, `Iterator`.


#### Не эмулируйте `takeRight`

    // До
    seq.reverse.take(n).reverse

    // После
    seq.takeRight(n)

Последнее выражение является более выразительным и потенциально более
эффективным (Как для индексированных, так и для линейных последовательностей).


#### Не эмулируйте `flatten`

    // До (seq: Seq[Seq[T]])
    seq.reduce(_ ++ _)
    seq.fold(Seq.empty)(_ ++ _)
    seq.flatMap(identity)

    // После
    seq.flatten

Нет необходимости вручную уплощать коллекции, когда у
нас уже есть специализированный метод для этого.

Также применимо к: `Set`, `Option`, `Iterator`.


#### Не эмулируйте `flatMap`

    // До (f: A => Seq[B])
    seq.map(f).flatten

    // После
    seq.flatMap(f)

Again, there’s no need to emulate built-in method manually. Besides
improved clarity, we can skip the creation of an intermediate collection.

Также применимо к: `Set`, `Option`, `Iterator`.


#### Don’t use `map` when result is ignored

    // До
    seq.map(???) // результат игнорируется

    // После
    seq.foreach(???)

When side effect is all that is needed,
there’s no justification for calling `map`.
Such a call is misleading (and less efficient).

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


#### Don’t use `unzip` to extract a single component

    // Before (seq: Seq[(A, B]])
    seq.unzip._1

    // After
    seq.map(_._1)

There’s no need to create additional collection(s)
when only a single component is required.

Other possible method: `unzip3`.

Также применимо к: `Set`, `Option`, `Map`, `Iterator`.


#### Don’t create temporary collections

This recipe is subdivided into three parts
(depending on transformation final result).

1) Transformation reduces collection to a single value.

    // Before
    seq.map(f).flatMap(g).filter(p).reduce(???)

    // After
    seq.view.map(f).flatMap(g).filter(p).reduce(???)

In place of `reduce` might be any method that reduces collection to
a single value, for example:
`reduceLeft`, `reduceRight`, `fold`, `foldLeft`, `foldRight`, `sum`,
`product`, `min`, `max`, `head`, `headOption`, `last`, `lastOption`,
`indexOf`, `lastIndexOf`, `find`, `contains`, `exists`, `count`,
`length`, `mkString`, etc.

The exact order of transformations is not relevant — what matters
is that we’re creating one or more temporary, intermediate collections,
that are not needed by themselves, yet they will take heap space and burden GC.
That happens because, by default, all collection transformers
(`map`, `flatMap`, `filter`, `++,` etc.) are “strict” (except for `Stream`)
and construct a new collection with all its elements as a result of
transformation.

That’s where views come to the rescue — we may think of view as some
kind of `Iterator`, that allows re-iteration:

  * Views are “lazy” — elements constructed only when they are needed.
  * Views don’t hold created elements in memory (which even `Stream` does).

To go from a collection to its view, we use the `view` method.

2) Transformation produces a collection of the same class.

It’s possible to use views when the final result of transformation is still a
collection — the `force` method will build a collection of the original class
(while creation of all the intermediate collections is still avoided):

    // Before
    seq.map(f).flatMap(g).filter(p)

    // After
    seq.view.map(f).flatMap(g).filter(p).force

If the only intermediate transformation is filtering, you may consider using
`withFilter` method as an alternative:

    seq.withFilter(p).map(f)

The method is originally [intended](https://www.scala-lang.org/old/node/3698.html#comment-14546)
to be used in for comprehensions. It works much like a view — it creates a
temporary object that restricts the domain of subsequent collection
transformations (so, it reorders possible side effects).
However, there’s no need to explicitly convert collection to / from temporary
representation (by calling `view` and `force`).

While view-based approach is more universal, in this particular case you may
prefer `withFilter` method because of the conciseness.

3) Transformation creates a collection of different class.

    // Before
    seq.map(f).flatMap(g).filter(p).toList

    // After
    seq.view.map(f).flatMap(g).filter(p).toList

This time we use a suitable converter method
(instead of the generic `force` call),
so the result will be a collection of different class.

There also exists an alternative way of handling “transformation + conversion”
case that relies on `breakOut`:

    seq.map(f)(collection.breakOut): List[T]

Such an expression is functionally equivalent to using a view,
however this approach:

  * needs expected type to be explicit
  (which often requires an additional type annotation),
  * is limited to a single transformation
  (like `map`, `flatMap`, `filter`, `fold`, etc.),
  * looks rather tricky
  (because implicit builders are usually [omitted](https://www.scala-lang.org/api/current/index.html#scala.collection.Seq)
  from the Scala Collections API docs).

Thus, it’s usually better to substitute `breakOut` for a view, which is more
clear, more concise and more flexible.

Views are especially efficient when collection size is relatively large.

All the functions (like `f` and `g`) and predicates (`p`) must be pure
(because view might delay, skip or reorder computations).

Также применимо к: `Set`, `Map`.


#### Use assignment operators to reassign a sequence

    // Before
    seq = seq :+ x
    seq = x +: seq
    seq1 = seq1 ++ seq2
    seq1 = seq2 ++ seq1

    // After
    seq :+= x
    seq +:= x
    seq1 ++= seq2
    seq1 ++:= seq2

Scala offers a syntactic sugar known as “assignment operators” —
it automatically converts `x <op>= y` statements into `x = x <op> y`
where `<op>` is some character operator (like `+,` `-,` etc.).
Note, that if `<op>` ends with `:` it is considered right-associative
(i. e. invoked on the right expression, instead of the left).

There’s also a special syntax for lists and streams:

    // Before
    list = x :: list
    list1 = list2 ::: list

    stream = x #:: list
    stream1 = stream2 #::: stream

    // After
    list ::= x
    list1 :::= list2

    stream #::= x
    stream1 #:::= stream2

The optimized statements are more concise.

Также применимо к: `Set`, `Map`, `Iterator` (with operator specifics in mind).


#### Don’t convert collection type manually

    // Before
    seq.foldLeft(Set.empty)(_ + _)
    seq.foldRight(List.empty)(_ :: _)

    // After
    seq.toSet
    seq.toList

We have corresponding build-it methods for doing that, which are
cleaner and faster.

If you need to transform or filter values in the course of the conversion,
consider using views or similar techniques, as described above.

Также применимо к: `Set`, `Option`, `Iterator`.


#### Be careful with `toSeq` on non-strict collections

    // Before (seq: TraversableOnce[T])
    seq.toSeq

    // After
    seq.toStream
    seq.toVector

Because `Seq(...)` creates a strict collection (namely, [Vector](https://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Vector)),
we may be tempted to use `toSeq` to convert a non-strict entity
(like `Stream`, `Iterator`, or “view”) to a strict collection. However,
`TraversableOnce.toSeq` returns a `Stream` under the hood, which is a lazy
collection, and that might result in hard-to-find bugs or performance problems.
Even if a stream is exactly what you expect,
such an expression might confuse other people who read your code.

Here’s a typical example of the pitfall:

    val source = Source.fromFile("lines.txt")
    val lines = source.getLines.toSeq
    source.close()
    lines.foreach(println)

Such a code will throw an `IOException` complaining that
the stream is already closed.

To clarify the intent, it’s better to write `toStream` explicitly, or,
if we need a strict collection, after all, to use `toVector` instead of `toSeq`.

#### Don’t convert to `String` manually

    // Before (seq: Seq[String])
    seq.reduce(_ + _)
    seq.reduce(_ + separator + _)
    seq.fold(prefix)(_ + _)
    seq.map(_.toString).reduce(_ + _) // seq: Seq[T]
    seq.foldLeft(new StringBuilder())(_ append _)

    // After
    seq.mkString
    seq.mkString(prefix, separator, "")

The latter approach is cleaner and potentially faster,
because it uses a single `StringBuilder` under the hood.

Other possible methods are:
`reduceLeft`, `reduceRight`, `foldLeft`, `foldRight`.

Также применимо к: `Set`, `Option`, `Iterator`.


## 5. Множесва (Sets)
Most of the tips for sequences are applicable to sets as well.
Additionally, there are several set-specific tips available.

#### Don’t use `sameElements` to compare unordered collections

    // Before
    set1.sameElements(set2)

    // After
    set1 == set2

This rule was introduced earlier (for sequences), yet for sets the
rationale is even more sound.

The `sameElements` method might return indeterministic results for
unordered collections, because this method respects order of elements,
while we cannot rely on the order in a set.

Classes that explicitly guarantee predictable iteration order
(like `LinkedHashSet`) are exceptions from this rule.

Также применимо к: `Map`.


#### Don’t compute set intersection manually

    // Before
    set1.filter(set2.contains)
    set1.filter(set2)

    // After
    set1.intersect(set2) // or set1 & set2

The latter expression is more clear and concise
(while performance is the same).

This transformation can be performed on sequences,
however we should keep in mind, that in such a case,
duplicate elements will be handled differently.


#### Don’t compute set difference manually

    // Before
    set1.filterNot(set2.contains)
    set1.filterNot(set2)

    // After
    set1.diff(set2) // or set1 &~ set2

Again, the updated expression is more clear and concise
(while performance is the same).

The transformation is potentially applicable to sequences,
though we should consider duplicate elements.


## 6. Options
Технически, `Option` не является частью коллекций Scala, однако он предоставляет
похожий интерфейс (с монадическими методами и тд) и ведет также как специальный
тип коллекций который может иметь, а может не иметь значение.

Многие из приведенных советов для последовательностей также применимы и к
option. Кроме того, здесь представлены советы характерные для `Option` API.

### 6.1 Значение
#### Не сравнивайте значения option и `None`

    // До
    option == None
    option != None

    // После
    option.isEmpty
    option.isDefined

В то время как сравнение является вполне законным, у нас есть более простой
способ, позволяет проверить объявлен ли option.

Другое преимущество данного упрощения в том, что если вы решите изменить тип от
`Option[T]` к `T`, scalac скомпилирует предшествующее выражение (выдав только
одно предупреждение), в то время как компиляция последнего, справедливо
закончится ошибкой


#### Не сравнивайте значения option с `Some`

    // До
    option == Some(v)
    option != Some(v)

    // После
    option.contains(v)
    !option.contains(v)

Данный совет является дополнением к предыдущему.


#### Не прибегайте к проверке типа, для проверки существования

    // До
    option.isInstanceOf[Some[_]]

    // После
    option.isDefined

В подобном трюкачестве нет нужды.


#### Не прибегайте к сопоставлению с образцом, для проверки существования

    // До
    option match {
        case Some(_) => true
        case None => false
    }

    option match {
        case Some(_) => false
        case None => true
    }

    // После
    option.isDefined
    option.isEmpty

Опять же, хотя первое выражение и является корректным, оправдания подобной
экстравагантности нет. Помимо того, упрощенное выражение будет работать быстрее.

Также применимо к: `Seq`, `Set`.


#### Не отрицайте значения свойств, связанных с существованием

    // До
    !option.isEmpty
    !option.isDefined
    !option.nonEmpty

    // После
    seq.isDefined
    seq.isEmpty
    seq.isEmpty

Обоснование такое же как и для последовательностей — простое свойство добавляет
меньше визуального шума, чем составное выражение.

Отметьте, что мы имеем синонимы: `isDefined` (специфичный для option) и
`nonEmpty` (специфичный для последовательностей). Возможно, было бы разумно
отдать предпочтение первому, для явного отделения option и последовательностей.


### 6.2 Null
#### Не выполняйте явное сравнение значений с `null` чтобы создать `Option`

    // До
    if (v != null) Some(v) else None

    // После
    Option(v)

Для этого у нас есть специальный, более выразительный синтаксис.


#### Не предоставляйте `null` как явную альтернативу

    // До
    option.getOrElse(null)

    // После
    option.orNull

TODO:
We can rely on the predefine method in such a case,
so the expression will become shorter.


### 6.3 Обработка

TODO
It’s possible to single out a groups of tips that are related to how `Option`
value is processed.

TODO
As `Option`‘s API [documentation](https://www.scala-lang.org/api/current/index.html#scala.Option)
says that “the most idiomatic way to use an
`Option` instance is to treat it as a collection or monad and use
map, flatMap, filter, or foreach”, the basic principle here is to
avoid “check & get” chains, implemented either via
`if` statement or via pattern matching.

TODO
The goal is conciseness and robustness, the “monadic” code is:

  * more concise and intelligible,
  * safeguarded against `NoSuchElementException` nor `MatchError`
  exceptions at runtime.

TODO
This rationale is common for all the following cases.


#### Не эмулируйте `getOrElse`

    // До
    if (option.isDefined) option.get else z

    option match {
      case Some(it) => it
      case None => z
    }

    // После
    option.getOrElse(z)


#### Не эмулируйте `orElse`

    // До
    if (option1.isDefined) option1 else option2

    option1 match {
      case Some(it) => Some(it)
      case None => option2
    }

    // После
    option1.orElse(option2)


#### Не эмулируйте `exists`

    // До
    option.isDefined && p(option.get)

    if (option.isDefined) p(option.get) else false

    option match {
      case Some(it) => p(it)
      case None => false
    }

    // После
    option.exists(p)


#### Не эмулируйте `forall`

    // До
    option.isEmpty || (option.isDefined && p(option.get))

    if (option.isDefined) p(option.get) else true

    option match {
      case Some(it) => p(it)
      case None => true
    }

    // После
    option.forall(p)


#### Не эмулируйте `contains`

    // До
    option.isDefined && option.get == x

    if (option.isDefined) option.get == x else false

    option match {
      case Some(it) => it == x
      case None => false
    }

    // После
    option.contains(x)


#### Не эмулируйте `foreach`

    // До
    if (option.isDefined) f(option.get)

    option match {
      case Some(it) => f(it)
      case None =>
    }

    // После
    option.foreach(f)


#### Не эмулируйте `filter`

    // До
    if (option.isDefined && p(option.get)) option else None

    option match {
      case Some(it) && p(it) => Some(it)
      case _ => None
    }

    // После
    option.filter(p)


#### Не эмулируйте `map`

    // До
    if (option.isDefined) Some(f(option.get)) else None

    option match {
      case Some(it) => Some(f(it))
      case None => None
    }

    // После
    option.map(f)


#### Не эмулируйте `flatMap`

    // До (f: A => Option[B])
    if (option.isDefined) f(option.get) else None

    option match {
      case Some(it) => f(it)
      case None => None
    }

    // После
    option.flatMap(f)


### 6.4 Переписываем
#### Приведите цепочку из `map` и `getOrElse` в `fold`*

    // До
    option.map(f).getOrElse(z)

    // После
    option.fold(z)(f)

Данные выражения семантически эквиваленты (в обоих случаях `z` будет вычислен
лениво, по требованию), однако последнее выражение короче.
Трансформация может требовать дополнительного указания типа (из-за того что
вывод типов в Scala так работает), и, в подобных случаях, предыдущее выражение
предпочтительнее.

Имейте в виду, что данное упрощение весьма
[противоречиво](https://www.reddit.com/r/scala/comments/2z411u/scala_collections_tips_and_tricks/cqiip08/),
в виду того что последнее выражение выглядит менее ясно, особенно если вы к
нему не привыкли.


#### Не имитируйте `exists`

    // До
    option.map(p).getOrElse(false)

    // После
    option.exists(p)

Мы представили довольно похожее правило для последовательностей (которое
также применимо к optionам). Данная трансформация является специальным
случаем вызова `getOrElse`.


#### Не эмулируйте `flatten`

    // До (option: Option[Option[T]])
    option.map(_.get)
    option.getOrElse(None)

    // После
    option.flatten

Последнее выражение смотрится чище.


#### Не конвертируйте option в sequence вручную

    // До
    option.map(Seq(_)).getOrElse(Seq.empty)
    option.getOrElse(Seq.empty) // option: Option[Seq[T]]

    // После
    option.toSeq

Для этого есть специальный метод, который делает это кратко и эффективно.


## 7. Maps
Так же как и с другими классами коллекций, многие советы для последовательностей
так же применимы к таблицам, поэтому перечислим только специфичные для таблиц.


#### Не выполняйте поиск значений вручную

    // До
    map.find(_._1 == k).map(_._2)

    // После
    map.get(k)

В принципе, первый фрагмент кода будет работать, однако производительность
будет недостаточной в виду того что `Map` не является простой коллекцией
пар (ключ, значение) — она может выполнять поиск в наиболее
эффективным способом.

Более того, последнее выражение проще и легче для понимания.


#### Не используйте `get`, когда необходимо сырое значение

    // Before
    map.get(k).get

    // After
    map(k)

Нет необходимости плодить промежуточный `Option`, когда необходимо сырое (raw)
значение.


#### Не используйте `lift` вместо `get`

    // Before
    map.lift(k)

    // After
    map.get(k)

Нет необходимости рассматривать значение таблицы как частичную функцию для
получения опционального результата (что полезно для последовательностей),
потому что у нас есть встроенный метод с той же функциональностью.
Хотя `lift` отлично работает, он выполняет дополнительное преобразование
(от `Map` до` PartialFunction`) и может показаться запутанным.


#### Не вызывайте `get` и `getOrElse` раздельно

    // До
    map.get(k).getOrElse(z)

    // После
    map.getOrElse(k, z)

Единственный вызов метода проще, как с синтаксически, так и с точки зрения
производительности. В обоих случаях `z` вычисляется лениво, по требованию.


#### Не извлекайте ключи вручную

    // До
    map.map(_._1)
    map.map(_._1).toSet
    map.map(_._1).toIterator

    // После
    map.keys
    map.keySet
    map.keysIterator

Оптимизированные выражения являются более понятными (и потенциально более
быстрыми).


#### Не извлекайте значения вручную

    // До
    map.map(_._2)
    map.map(_._2).toIterator

    // После
    map.values
    map.valuesIterator

Упрощенные выражения понятней (и потенциально быстрее).


#### Будьте осторожны с `filterKeys`

    // До
    map.filterKeys(p)

    // После
    map.filter(p(_._1))

TODO:
The `filterKeys` methods wraps the original map without copying any elements.
There’s nothing wrong with that per se, however such a behaviour is hardly
expected from `filterKeys` method.
Because it unexpectedly behaves like a view, code performance might be
substantially degraded in some cases (e. g. for `filterKeys(p).groupBy(???)`).

TODO:
Another possible pitfall is unexpected “laziness” (collection transformers
are expected to be strict by default) – the predicated is not evaluated at
all withing the method call itself,
so possible side effects might be reordered.

TODO:
The `filterKeys` method should probably be deprecated, because it’s now
impossible to [make it strict](https://issues.scala-lang.org/browse/SI-4776)
without breaking backward compatibility. A more suitable name for the current
implementation is `withKeyFilter` (by analogy with the `withFilter` method).

TODO:
All in all, it seems reasonable to follow the
[Rule of Least Surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)
and to filter keys manually.

TODO:
Nevertheless, as the view-like functionality of `filterKeys` is potentially
useful (when only a few entries will be accessed while the map is relatively
large), we may still consider using the method. To keep other people who read
(or modify) our code from confusion, it’s better clarify our intent by defining
an appropriate method synonym:

    type MapView[A, +B] = Map[A, B]

    implicit class MapExt[A, +B](val map: Map[A, B]) extends AnyVal {
      def withKeyFilter(p: A => Boolean): MapView[A, B] =
        map.filterKeys(p)
    }

Мы используем псевдоним типа `MapView` для того чтобы обозначить что
результирующая таблица является view-подобной.

Другим вариантом будет объявление простого вспомогательного метода:

    def get(k: T) = if (p(k)) map.get(k) else None


#### Будьте осторожны с `mapValues`

    // До
    map.mapValues(f)

    // После
    map.map(f(_._2))

Основная причина такая же как и в предыдущем случае.
Подобным способом мы можем объявить недвусмысленный синоним:

    type MapView[A, +B] = Map[A, B]

    implicit class MapExt[A, +B](val map: Map[A, B]) extends AnyVal {
      def withValueMapper[C](f: B => C): MapView[A, C] =
        map.mapValues(f)
    }

Более простым способом будет объявить вспомогательный метод вроде:

    def get(k: T) = map.get(k).map(f)


#### Не фильтруйте ключи вручную

    // До
    map.filterKeys(!seq.contains(_))

    // После
    map -- seq

Мы можем полагаться на упрощенный синтаксический сахар, чтобы отфильтровать
ключи.


#### Используйте операторы переприсваивания таблиц

    // До
    map = map + x -> y
    map1 = map1 ++ map2
    map = map - x
    map = map -- seq

    // После
    map += x -> y
    map1 ++= map2
    map -= x
    map --= seq

Также как и с последовательностями, мы можем полагаться на синтаксический
сахар, для упрощения подобных операторов.


## 8. Дополнение
В дополнение к приведенным рецептам, я рекомендую вам посмотреть на официальную
[документацию библиотеки коллекций Scala](http://docs.scala-lang.org/overviews/collections/introduction.html),
которую, на удивление легко читать.

Смотрите также:

  * [Scala for Project Euler](https://pavelfatin.com/scala-for-project-euler/) —
    Выразительные функциональные решения проблем проекта Эйлер на Scala.
  * [Ninety-nine](https://pavelfatin.com/ninety-nine/) — Девяносто девять
    проблем на Scala, Java, Clojure и Haskell (с множеством решений).

Данные головоломки неоценимо помогли мне сформировать и углубить мое понимание
коллекций Scala.

Несмотря на обширность данного списка, он наверняка далек от завершения.
Более того, в виду сильно варьирующейся применимости данных рецептов,
некоторая тонкая настройка определенно необходима.
Ваши предложения приветствуются.

