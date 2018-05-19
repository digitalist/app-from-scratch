# Domain

Для начала разберемся с доменной логикой. Это наши сущности, агрегаты, объекты-значения и службы.

Сделаем одну сущность Person в отдельном проекте.

Есть несколько способов моделировать состояние сущности в clojure

1. Использовать мапы:

   ```clojure
   {:id   1
    :type :person
    :name "Alise"}
   ```

   Его минус в том, что реализовать полиморфизм для мап можно только с помощью мультиметодов.
   Также нам явно нужно указывать тип.
2. Использовать записи:

   ```clojrue
   (defrecord Person [id name])
   ```

   При этом мы можем использовать как протоколы, так и мультиметоды.
   Каждая сущность имеет свой тип(класс). При этом записи поддерживают интерфейс мап.
   И их объявление является документацией того, какие поля они имеют.
3. Модель [datomic](https://docs.datomic.com/cloud/whatis/data-model.html) или
   [datascript](https://github.com/tonsky/datascript).
   Не рассматриваем.

## Моделирование состояния сущности

Для начала напишем тест на конструктор.
Конструктор - это обычная функция, возвращающая экземпляр `Person`.
Назовем наш конструктор `build`:

```clojure
(ns app.person
  (:require
   [clojure.test :as t]))

(declare build)
(declare person?)

(t/deftest build-test
  (let [params {:name "Alice"}
        person (build params)]
    (t/is (person? person))))
```

Добавим реализацию:

```clojure
(defrecord Person [id name])

(defn build [{:keys [name]}]
  (map->Person {:name name}))

(defn person? [x] (instance? Person x))
```

Отмечу, что наш конструктор не устанавливает идентификатор.
И наши сущности получаются неполноценными.

Напишем спецификацию на наш конструктор, чтобы проверять
корректность возващаемого значения.

```clojure
(ns app.person
  (:require
   [clojure.test :as t]
   [clojure.spec.alpha :as s]
   [orchestra.spec.test :as st]))

(s/def ::id pos-int?)
(s/def ::name string?)
(s/def ::person (s/keys :req-un [::id ::name]))

(defrecord Person [id name])

(s/fdef build
        :args (s/cat :params (s/keys :req-un [::name]))
        :ret ::person)

(defn build [{:keys [name]}]
  (map->Person {:name name}))

(defn person? [x] (instance? Person x))

;; подменяем функции на вариант, проверяющий спецификацию
(st/instrument)

(t/deftest build-test
  (let [params {:name "Alice"}
        person (build params)]
    (t/is (person? person))))
```

Ожидаемо наш тест не прошел, т.к. `build` возвращает структуру,
не соответствующую спецификации `::person`.

Мы пока не знаем, как мы будем сохранять наши сущности, но уже
сейчас нам нужно генерировать идентификаторы.
Отложем принятие решение о конкретной реализации генератора и
объявим абстракцию генератора:

```clojure
(defprotocol IdGenerator
  (-generate-id [this]))

(declare ^:dynamic *id-generator*)

(s/fdef generate-id
        :ret ::id)

(defn generate-id []
  (-generate-id *id-generator*))
```

Т.е. наш генератор должен реализовывать протокол `IdGenerator`
и его экземляр должен храниться в динамической переменной
`*id-generator`.

Теперь мы можем использовать наш генератор в конструкторе:

```clojure
(defn build [{:keys [name]}]
  (map->Person {:id (generate-id)
                :name name}))
```

Для тестов напишем фейковую реализацию, храняющую данные в памяти.
Продробнее про фейки, моки и стабы можно посмотреть
[тут](https://cleancoders.com/episode/clean-code-episode-23-p1/show).

```clojure
(deftype FakeIdGenerator [counter]
  IdGenerator

  (-generate-id [_]
    (swap! counter inc)))

(defn build-fake-id-generator []
  (FakeIdGenerator. (atom 0)))
```

Теперь перед каждым тестом нужно создавать экземпляр герератора и
устанавливать его в динамическую переменную. Для этого воспользуемся
[фикстурами](https://clojuredocs.org/clojure.test/use-fixtures):

```clojure
(t/use-fixtures :each (fn [test]
                        (binding [*id-generator* (build-fake-id-generator)]
                          (test))))
```

Вуаля, наш тест проходит.

[Весь пример полностью](/4-core/1-sources).

Очевидно, что мешать весь этот функционал в одином файле - плохая идея.
Не пугайтесь, я дам пример структуры.

## Идентичность

Мы смоделировали состояние сущности,
но не саму сущность. Нам еще нужна идентичность.
Если забыли, то прочитайте [урок про управление состоянием](/1-clojure/3-state-management.md).

Для моделирования сущностей идеально подходят Refs.
Подробнее в [продолжении урока про состояние](/1-clojure/3.1-other.md).

Функциональный подход










Дать ссылки на исходники и тесты domain в проекте.

Задание - реализовать любую сущность в проекте.