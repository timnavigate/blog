---
layout: post
title:  "Тесты в Clojure (второй фрагмент)"
permalink: /clj-book-tests-2/
tags: clojure book programming tests
lang: ru
---

*Продолжение [первой части](/clj-book-tests-1)*

{% include toc.html id="clj-book-tests-2" %}

<!-- more -->

### Транзакция с откатом

Другой способ избавиться от изменений в базе — обернуть все действия с ней в
особую транзакцию. Такая транзакция завершается оператором не `COMMIT`, а
`ROLLBACK`, что значит откатить все команды. С точки зрения базы это выглядит
так:

~~~sql
BEGIN;
INSERT INTO users ...
INSERT INTO profiles ...
UPDATE users SET name=...
ROLLBACK;
~~~

При выходе из транзакции мы не увидим последствий `INSERT`, `UPDATE` и других
команд, выполненных в ее рамках.

В пакет JDBC входит функция `db-set-rollback-only!`. Она принимает
транзакционное соединение и выставляет ему флаг rollback. Если флаг установлен,
JDBC завершает блок транзакции оператором ROLLBACK.

Вы уже знакомы с макросом `with-db-transaction`: внутри его тела действует
транзакционное соединение, которые получают из JDBC-спеки. Напишем свой макрос
with-db-transaction, который делает то же самое, но дополнительно устанавливает
флаг отката:

~~~clojure
(defmacro with-db-transaction
  [[t-conn & bindings] & body]
  `(jdbc/with-db-transaction [~t-conn ~@bindings]
     (jdbc/db-set-rollback-only! ~t-conn)
     ~@body))
~~~

Пример его работы:

~~~clojure
(with-db-rollback [tx *db*]
  (println "Inserting the data...")
  (jdbc/insert! tx :users {:name "Ivan"})
  (let [...]
    (do-something)))
~~~

Разработчик следит за тем, чтобы все действия — вставка данных, логика теста —
протекали через `tx`, а не `*db*`. Иначе изменения в рамках обычного соединения
останутся в базе. Прямо сейчас загрузчик load-data данных ссылается на
глобальную переменную `*db*`. Чтобы сообщить ему транзакционное соединение,
придется либо передать параметр, либо связать `*db*` с tx формой binding.

Пример с параметром: все действия с базой принимают tx, который мы установили на
вершине теста.

~~~clojure
(deftest test-user-with-rollback
  (with-db-rollback [tx *db*]
    (load-data tx)
    (let [user (get-user-by-name tx "Ivan")]
      (is (= "Ivan" (:name user))))))
~~~

Вариант со связанной переменной. В этом случае мы считаем, что все функции
ссылаются на глобальную *db*. Внутри макроса она станет транзакционным
соединением с откатом.

~~~clojure
(deftest test-user-with-rollback
  (with-db-rollback [tx *db*]
    (binding [*db* tx]
      (load-data)
      (let [user (get-user-by-name "Ivan")]
        (is (= "Ivan" (:name user)))))))
~~~

Реализация зависит от того, как в проекте устроена работа с базой. Решение с
откатом хорошо подходит для mount и похожей архитектуры, когда база это
глобальная переменная. В случае с системой компонентов могут возникнуть
трудности с передачей tx от теста к логике и наоборот.

В свободное время подумайте, как оформить фикстуру с макросом
with-db-rollback. Будет ли она работать с системой компонентов? Что необходимо в
этом случае?

## Тестирование веб-приложений

До сих пор мы тестировали отдельные функции, в основном связанные с
расчетами. Такие тесты, как говорят математики, необходимы, но недостаточны. Они
защищают проект от спонтанных изменений, но не обещают, что система
устойчива. Поднимемся на уровень выше и рассмотрим, как тестировать приложение
целиком.

В главе про веб-разработку мы пришли к важному выводу. Веб-приложение на каждом
уровне остается функций одного аргумента. Где-то внизу это обработчик конкретной
страницы, условный `(view-profile-page [request])`. Комбинация маршрутов и
middleware не меняет это соглашение. Конечный объект app, много раз обернутый в
middleware, по-прежнему принимает словарь запроса и возвращает словарь ответа.

Это определяет, как писать тесты для приложения. Типичный тест составляет запрос
и вызывает приложение как функцию. В ответе ищут HTTP-статус и проверяют на
успех (200, 201) или неудачу (404, 403). Если это JSON-ответ, его тело считывают
в коллекцию и сравнивают с образцом.

Вспомним приложение из первой главы. Отдельные страницы мы соединили в маршруты
с помощью Compojure. Получилось "голое" приложение. Так мы назвали его потому,
что оно многого не умеет. Например, читать параметры запроса, работать с JSON,
куками, сессиями и так далее. Приложение узнает все это из middleware, в которые
мы его оборачиваем.

~~~clojure
(defroutes app-naked
  (GET "/"      request (page-index request))
  (GET "/hello" request (page-hello request))
  page-404)

(def app
  (-> app-naked
      wrap-session
      wrap-keyword-params
      wrap-params
      wrap-json-body
      wrap-json-response))
~~~

Напишем несколько тестов для этого приложения. Пусть это будет главная страница
и любая другая, которой нет в дереве маршрутов. Для экономии места проверим
только статус ответа.

~~~clojure
(deftest test-app-index
  (let [request {:request-method :get :uri "/"}
        response (app request)
        {:keys [status body]} response]
    (is (= 200 status))))

(deftest test-app-page-not-found
  (let [request {:request-method :get :uri "/missing"}
        response (app request)
        {:keys [status body]} response]
    (is (= 404 status))))
~~~

Как видно из примеров, писать тесты для веб-приложения нетрудно. Однако, мы не
бросим читателя на этом месте. Перечислим несколько приемов, которые облегчат
работу с тестами.

**Приложение целиком.** Избегайте ситуации, когда тест вызывает не приложение, а
один из обработчиков. Плохой пример: (page-index request) вместо app. На текущем
уровне вызов конкретной функции ничего не дает. Даже если страница работает по
отдельности, нет гарантии, что запрос пройдет сквозь маршруты и все
middleware. В боевых проектах middleware несут весомую логику. Это права
доступа, разбор токенов и JWT, данные из прошлых запросов в сессии. Убрав все
это из теста, вы тем самым обманываете себя. Объект app, который вы тестируете,
должен быть максимально "заряжен", т.е. близок к настоящему веб-серверу.

**Библиотека запросов.** Выше мы объявили запрос в виде словаря. Это работает
только для простых случаев, когда нет ни параметров строки, ни тела. Если не
позаботиться об инструментах, вы дойдете до уровня

~~~text
/users/?page=2&order=name&name_contains=smith&search_type=relevance
~~~

, что совершенно нечитаемо и тяжело в поддержке. В словаре запроса легко
перепутать ключи :request-method и :method, что автор и сделал при написании
книги.

Чтобы избежать подобных ошибок, воспользуйтесь `ring-mock` — библиотекой для
запросов к Ring-приложениям. Ее функции покрывают основные сценарии в
тестах. Так, функция request принимает метод и путь. Если добавить словарь
параметров, то для GET они станут строкой запроса, а для POST — его
телом. URL-кодировку библиотека берет на себя. Другая функция json-body пишет в
запрос тело из коллекции. Рассмотрим несколько примеров.

Простая страница, запрос `GET /help`:

~~~clojure
(mock/request :get "/help")
~~~

Ввод данных в форму поиска, `GET /movies?search=batman&page=1`:

~~~clojure
(mock/request :get "/movies" {:search "batman" :page 1})
~~~

Новый пользователь из формы, `POST /users`. В этом запросе тело (ключ :body)
станет классом `ByteArrayInputStream`. Заголовок Content-Type примет значение
`"application/x-www-form-urlencoded"`.

~~~clojure
(mock/request :post "/users" {:name "Ivan" :email "test@test.com"})
~~~

Случай для JSON-API. Адрес /users ожидает не поля формы, а JSON-тело.Такой
запрос составляют из двух функций:

~~~clojure
(-> (mock/request :post "/users")
    (mock/json-body {:name "Ivan" :email "test@test.com"}))
~~~

`Проверка тела.` В тестах выше мы проверяли только статус ответа. На практике
одного статуса недостаточно: число 200 еще не говорит, что в ответе именно то,
что нужно. Проверка тела зависит от типа содержимого. Если это текст или HTML,
иногда хватит и регулярного выражения. Например, по фразе "Login" мы определим,
что на этой странице пользователь не авторизован.

Гораздо интересней вариант с JSON-API. В этом случае нужно восстановить данные
из ответа и сравнить с образцом. Для простоты вызовем приложение
sites-handler. Это заглушка, которой мы пользовались для тестировании
карт. Проверка ниже гарантирует, что при изменении ответа мы получим
ошибку. По-другому говорят, что ответ зафиксирован:

~~~clojure
(let [request (mock/request :get "/search/v1/" {:lat 11 :lon 22})
      response (sites-handler request)
      body (-> response :body (cheshire.core/parse-string true))]
  (is (= {...} body)))
~~~

Недостаток в том, что мы сравниваем данные как есть. Прием не сработает, если в
ответе даты, уникальные идентификаторы (UUID) или машинные номера. Перед
сравнением их исключают с помощью dissoc и map. Представим, что поиск кафе
вернул результат в таком виде:

~~~clojure
{:sites [{:name "Site1" :date-updated "2019-11-12" :id 42}
         {:name "Site2" :date-updated "2019-11-10" :id 99}]}
~~~

Код ниже вернет только те данные, которые не меняются от запроса к запросу. Их и
сравнивают с образцом.

~~~clojure
(update body :sites
        (fn [sites]
          (map (fn [site]
                 (dissoc site :id :date-updated))
               sites)))
;; {:sites ({:name "Site1"} {:name "Site2"})}
~~~

Иногда проверяют не конкретные значения, а структуру ответа. Это удобно на
больших данных, например, когда в каждом элементе двадцать и более полей. В
таком случае ответ проверяют спекой или JSON-схемой.

~~~clojure
(let [;; obtain the response
      body (-> response :body (cheshire.core/parse-string true))]
  (is (s/valid? :api.search/handler body)))
~~~

Затраты на спеку или схему окупаются в будущем. Ими проверяют входные параметры,
генерируют данные для тестов, описывают REST API (Swagger, RAML).

## Тестирование систем

Коротко о том, как пишут тесты в проектах с системами, о которых мы говорили в
отдельной главе. Напомним, система это набор компонентов со связями между
ними. Покрыть каждый компонент тестами нетрудно; проблемы возникают на стыке при
взаимодействии. В проекте обязательно должен быть тест, где система работает как
единое целое.

Очевидно, чтобы тест прошел, кто-то должен запустить систему и остановить ее
после. На эту роль подходит фикстура. Будем полагать, что переменная системы и
функции ее запуска и останова находятся в модуле system.clj. Напишем фикстуру
`fix-system`:

~~~clojure
(defn fix-system [t]
  (system/start!)
  (t)
  (system/stop!))
~~~

На время ее работы в переменной `system/system` будет рабочая система. Другие
фикстуры, например, для работы с базой, могут обратиться к ее компонентам
напрямую. Важно только, чтобы в вызове use-fixtures они шли в правильном порядке
(левее — раньше), иначе вы получите Null-ошибки и другие неприятности.

~~~clojure
(defn fix-db-data [t]
  (let [{:keys [db]} system/system]
    (prepare-test-data db)
    (t)
    (clear-test-data db)))

(use-fixtures :once fix-system fix-db-data)
~~~

Читатель заметит, что это противоречит правилу, о котором мы говорили в главе
про системы. Что к системе нельзя обращаться напрямую и копаться в ее
компонентах. Что ж, признаем, для тестов в этом плане действуют
послабления. Обращаться к системе нельзя в промышленном коде. Но тесты это не
промышленный код, поэтому на небольшие нарушения в них порой закрывают
глаза. Главное, чтобы программист понимал, что он намерен выиграть. В нашем
случае выигрыш в том, что мы не пробрасываем компонент базы в фикстуру. Это было
бы честнее, но заняло больше строк кода.

Фикстура `fix-system` неслучайно стоит под ключом `:once`. Запуск и остановка
системы занимают много времени. В наших интересах прогнать как можно больше
тестов, пока система работает. Если же делать это поштучно, процесс затянется
надолго. Именно с этой проблемой вы столкнетесь при запуске тестов из CIDER. При
попытке выполнить один тест вы будете ждать, пока сработают все фикстуры, в том
числе fix-system.

Кажется, что пять-десять секунд это немного. Но представьте, что работаете над
задачей и запускаете тест раз за разом — подобные паузы раздражают и сбивают с
ритма. Поделимся с читателем техникой, которая решит эту проблему.

Потребуется два шага. Первый — научить систему, чтобы она знала о своем
состоянии. Например, включена ли она сейчас или выключена. Проще всего это
сделать через поле с метаданными. Вынесем имя поля в отдельную
переменную. Перепишем start! так, чтобы в метаданных системы появился флаг со
значением true. Аналогично работает stop!, только флаг принимает ложь. Функция
started? возвращает флаг значение из текущей системы.

~~~clojure
(def state-field ::started?)

(defn start! []
  (let [sys (-> system
                component/start-system
                (with-meta {state-field true}))]
    (alter-var-root #'system (constantly sys))))

(defn started? []
  (some-> system meta (get state-field)))
~~~

Второй шаг — перед тем, как включить систему в фикстуре, определить, была ли она
запущена вручную. Если нет, фикстура работает как обычно: запуск, тест,
остановка. Если да, это значит, что системой управляют в ручном режиме. В таком
случае фикстура только выполняет тест, что гораздо быстрее, чем полный цикл.

~~~clojure
(defn fixture-system [t]
  (let [started-manually? (system/started?)]
    (when-not started-manually?
      (start!))
    (t)
    (when-not started-manually?
      (stop!))))
~~~

Выполните в REPL (system/start!). Теперь можно вызвать тест сколько угодно раз —
не придется ждать, пока запустится система.

## Интеграционные тесты

Не протяжении главы мы постепенно усложняли тесты. С каждым шагом они все меньше
зависят от технических деталей и делают упор на бизнес-логику. По-другому этот
принцип называют пирамидой тестов. В ее основании лежат юнит-тесты — множество
отдельных проверок. Поднимаясь к вершине, мы абстрагируемся от технического
слоя: в какой-то момент тестируем не отдельные функции, а часть приложения.

Каждый уровень в пирамиде требует специальных знаний. Полагаем, читатель готов к
тому, чтобы подняться на последний этаж — освоить интеграционное
тестирование. Это особая фаза тестов, когда окружение максимально похоже на
промышленное. По-другому их еще называют UI- или Selenium-тестами в честь
одноименного фреймворка. Для большей реалистичности мы не шлем запросы
программно, а имитируем действия человека. Например, управляем браузером: вводим
данные в форму, нажимаем кнопку и проверяем, что появился нужный текст.

Интеграционные тесты выполняются медленно, потому что включают полный цикл
приложения. Это загрузка страницы, выполнение скриптов, ожидание запроса, во
время которого сервер ходит в базу или очередь задач. Если возникнет ошибка, ее
трудно расследовать из-за длины цепи. Представьте, что вы нажали на кнопку, но
текст не появился. Возможны десятки причин, почему этого не произошло.

В этом разделе мы рассмотрим, как писать UI-тесты на Clojure. С подготовительной
частью вы уже знакомы: запускаем систему и наполняем базу тестовыми данными. А
вот тест ведет себя по-новому. Он захватывает контроль над браузером и командует
им. Например, открывает страницу `http://127.0.0.1:8080/` и щелкает по
ссылкам. В любой момент мы получим адрес страницы, ее заголовок и HTML-код. В
тест добавляют формы `(is (= ...))`, чтобы проверить, на какой странице мы
оказались или видит ли пользователь этот виджет.

Чтобы управлять браузером, понадобится драйвер и библиотека к нему. Под
драйвером понимают утилиту командной строки. Когда драйвер запущен, он принимает
запросы от библиотеки по протоколу HTTP. Одновременно драйвер запускает браузер
в особом режиме, и между ними образуется связь. Фактически, драйвер это
посредник между двумя акторами. Его работа сводится к переводу HTTP-команд в
бинарный протокол браузера и наоборот.

Каждый браузер работает со своим драйвером. Для Chrome он называется
`chromedriver`, для FireFox — `geckodriver`. Одноименные утилиты ставятся из
пакетных менеджеров `apt`, `yum` или `brew` в зависимости от операционной
системы. Пользователи Windows скачают бинарные файлы с сайта проекта. Драйвер к
Safari называется `safaridriver`. С версии 13 он идет в комплекте с Mac OS.

Для работы с драйвером подойдет библиотека Etaoin. Добавьте ее в зависимости
профиля :dev (библиотека нужна только для тестов):

~~~clojure
:dev {:dependencies [[etaoin "0.3.6"]]}
~~~

Убедитесь, что драйвер находится в одной из папок, перечисленных в PATH,
например, `/usr/local/bin`. Другими словами, его можно вызвать в терминале,
просто выполнив `chromedriver` или `geckodriver`. Путь до драйвера можно задать
в опциях библиотеки, но пока что мы сами положим его в нужное место.

Теперь напишем первый тест. Представим, что локальный сервер работает на
порту 8080. Тест открывает форму входа, заполняет поля и нажимает кнопку
"Login". Страница обновляется, наверху видно приветствие. Пользователь видит
элементы, которые прежде были скрыты (ссылки "My profile", "Logout").

~~~clojure
(ns project.integration-tests
  (:require [etaoin.api :as e]))

(deftest test-ui-login-ok
  (e/with-chrome {} driver
    (e/go driver "http://127.0.0.1:8080/login")
    (e/wait-visible driver {:fn/has-text "Login"})
    (e/fill driver {:tag :input :name :email} "test@test.com")
    (e/fill driver {:tag :input :name :password} "J3QQ4-H7H2V")
    (e/click driver {:tag :button :fn/text "Login"})
    (e/wait-visible driver {:fn/has-text "Welcome"})
    (is (e/visible? driver {:tag :a :fn/text "My profile"}))
    (is (e/visible? driver {:tag :button :fn/text "Logout"}))))
~~~

Разберем отдельные выражения из этого теста. Форма `with-chrome` это макрос,
который запускает Хром на время исполнения тела. Макрос необходим, чтобы
гарантированно закрыть драйвер при выходе или в случае ошибки. Без него пришлось
бы писать код с ручным `try/catch`, что порождает вложенность и в целом
неудобно:

~~~clojure
(let [driver (e/chrome)]
  (try
    (e/go driver "http://...")
    (e/click driver {:tag :button})
    (finally
      (e/quit driver))))
~~~

Функция `wait-visible` ждет до тех пор, пока элемент не появится на экране. К
ней прибегают довольно часто, чтобы дождаться, пока браузер отрисует
верстку. Если не разделить две инструкции ожиданием, между ними будет разница в
несколько миллисекунд. Второе действие либо не успеет выполниться, либо будет
отброшено.

Ожидание в UI-тестах это нормально. Основное время уходит на то, чтобы дождаться
загрузки и выполнить действия с задержкой, как это свойственно
человеку. `Wait-visible` это лишь одна из семейства wait-функций. В их число
входит `wait-has-text` (дождаться текст на экране), `wait-has-class` (ждать,
пока у элемента не появится класс) и многие другие.

Драйвер ищет элементы на странице с помощью селекторов. Это строки с особыми
выражениями; различают CSS- и XPath-селекторы. Сейчас мы не будем разбирать их
синтаксис: это долго и заслуживает отдельной главы. Для простоты рассмотрим
альтернативный прием. На элемент можно сослаться, задав его свойства
словарем. Ключи tag и id означают имя тега и его идентификатор. Другие ключи
становятся атрибутами тега. В примере выше селектор `{:tag :input :name :email}`
становится выражением `input[@name='email']` на языке XPath.

Подумаем, как улучшить наш тест? Для начала рассмотрим порт 8080. Мы уже знаем,
что подобные значения приходят из конфигурации. Исправьте тест так, чтобы и
сервер, и драйвер работали с одним и тем же портом.

Вспомним, как работает `with-chrome`: он создает новый драйвер, выполняет тело и
выключает его. Если каждый тест обернут в `with-chrome`, мы теряем время,
многократно повторяя эти шаги. Сделаем так, чтобы драйвер работал на протяжении
всего прогона тестов. Объявим динамическую переменную и фикстуру, которая
связывает драйвер на время тестов. Зарегистрируем ее с ключом `:once`.

~~~clojure
(defonce ^:dynamic *driver* nil)

(defn fix-chrome [t]
  (e/with-chrome {} driver
    (binding [*driver* driver]
      (t))))
~~~

Мы тестируем приложение в Хроме, самом популярном браузере. Но вот приходит
задача — убедиться, что мы также поддерживаем Firefox. Технически это значит,
что все тесты, которые мы написали для Хрома, должны сработать еще раз в другом
браузере. Скопировать их и заменить with-chrome на with-firefox — сомнительное
решение. Представьте, что через месяц нас попросят добавить Safari. Должен быть
способ, который не влечет за собой разрастание кода.

Поможет мульти-фикстура, с которой мы знакомились в середине главы. Она
пробегает по списку типов браузеров; в примере ниже это `:firefox` и
`:chrome`. Макрос `with-driver` это общий случай `with-chrome` и
аналогов. Отличие в том, что `with-driver` ожидает первым аргументом тип
браузера. На каждом шаге фикстура связывает драйвер и выполняет тест.

~~~clojure
(defn fix-multi-driver [t]
  (doseq [driver-type [:firefox :chrome]]
    (e/with-driver driver-type {} driver
      (binding [*driver* driver]
        (testing (format "Browser %s" (name driver-type))
          (t))))))
~~~

Теперь тесты по очереди сработают в каждом из браузеров. Для ясности мы
оборачиваем тест в сообщение о том, в рамках какого браузера его
вызывают. Поддержка нового браузера сводится к тому, чтобы добавить в список
ключ `:safari`, `:edge` и другие.

Еще один способ улучшить тесты — вынести одинаковые действия в фикстуру или
функцию. Например, каждый тест начинается с авторизации и заканчивается выходом
из системы. Чтобы не копировать эти действия каждый раз, напишем фикстуру
`fix-login-logout`. В отличии от предыдущих фикстур, ее регистрируют с ключом
`:each`.

~~~clojure
(defn fix-login-logout [t]
  (doto *driver*
    (e/go "http://127.0.0.1:8080/login")
    (e/fill {:tag :input :name :email} "test@test.com")
    (e/click {:tag :button :fn/text "Login"}))
  (t)
  (doto *driver*
    (e/click {:tag :button :fn/text "Logout"})
    (e/wait-has-text "Login")))
~~~

Попутно мы внедрили еще одну хорошую практику. Когда несколько функций принимают
одинаковый первый аргумент, их объединяют в макрос doto. Он подставит `*driver*`
на второе место в каждый список тела. С `doto` код становится немного короче и
чище.

## Другие решения

В последнем разделе мы перечислим другие библиотеки, полезные для тестов. Мы не
будем рассматривать их досконально: ограничимся кратким описанием и примером
кода. Все ответы ищите в документации к проектам.

### Продвинутые моки

На минуту вернемся к мокам — подмене функции через `with-redefs`. Этот макрос
слишком многословен, чтобы работать с ним напрямую. Появились библиотеки,
которые описывают мокинг короче и выразительнее. Одна из них называется
`mockery`. Библиотека предлагает макрос `with-mock` следующего вида:

~~~clojure
(with-mock mock
  {:target :project.path/get-geo-point
   :return {:lat 14.23 :lng 52.52}}
  (get-geo-point "cafe" "200m"))
~~~

В примере выше мы "замокали" `get-geo-point`, которая, судя по названию,
обращается к стороннему сервису карт. Внутри макроса объект `mock` это атом,
внутри которого словарь. Он наполняется данными по мере того, как мы вызываем
цель. Например, сколько раз ее вызвали и с какими аргументами. В выражении ниже
мы добавили проверки на то, что функцию вызвали один раз с аргументами "cafe" и
"200m".

~~~clojure
(let [{:keys [called? call-count call-args]} @mock]
  (is called?)
  (is (= 1 call-count))
  (is (= '("cafe" "200m") call-args)))
~~~

Библиотека `spy` устроена похожим образом. На функцию навешивается "шпион",
который копит данные о вызове.

~~~clojure
(defn adder [x y] (+ x y))
(def spy-adder (spy/spy adder))

(testing "calling the function"
  (is (= 3 (spy-adder 1 2))))

(testing "calls to the spy can be accessed via spy/calls"
  (is (= [[1 2]] (spy/calls spy-adder))))
~~~

### Альтернативный синтаксис

Проект `midje` предлагает другой способ писать тесты. В этой библиотеке мы имеем
дело с фактами. Факт это набор проверок, сгруппированных по смыслу. В примере
ниже факты о функции `split`:

~~~clojure
(facts "about `split`"
 (str/split "a/b/c" #"/") => ["a" "b" "c"]
 (str/split "" #"irrelvant") => [""])
~~~

Стрелка между выражениями это особый оператор, который называется extended
equality, продвинутое равенство. С ее помощью сравнение величин записывается
короче. Например, форма `1 => even?` приходит к виду `(even? 1)`. В `midje`
встречаются и другие, более сложные стрелки для проверки коллекций и макросов.

### Вывод XUnit

Плагин `test2junit` делает так, что отчет о тестах пишется в XML-файл формата
XUnit. Системы непрерывной интеграции, например, CircleCI или TeamCity понимают,
как отобразить его графически. Такой отчет легче просматривать, чем вывод
консоли. Проблемные места выделены красным, стектрейсы спрятаны под выпадающие
элементы. Плагину нужно задать путь к папке, куда писать файл.

~~~clojure
:plugins [[test2junit "1.1.2"]]
:test2junit-output-dir "target/test2junit"
~~~

### Генерация данных

Возможно, вы столкнетесь с тем, что для тестов нужен большой объем данных,
например, сто или двести тысяч записей. При этом данные должны быть разнообразны
— нас не устроит один и тот же набор, скопированный тысячу раз. Поможет
библиотека `test.check`. Ее модуль gen генерирует случайные данные по заданным
правилам. Особенно полезна генерация записей. В примере ниже мы получаем кортеж
строки, числа и булева типа. Затем применяем его к конструктору `->User`.

~~~clojure
(defrecord User [user-name user-id active?])

(def user-gen
  (gen/fmap (partial apply ->User)
            (gen/tuple (gen/not-empty gen/string-alphanumeric)
                       gen/nat
                       gen/boolean)))

(last (gen/sample user-gen))
;; => #user.User{:user-name "dfgJKSHF3"
;;               :user-id 5
;;               :active? false}
~~~

Библиотека `clojure.spec`, которой мы посвятили главу, идет еще дальше. С
помощью `test.check` она генерирует данные по спеке. Так проявляется еще одно
свойство спек: кроме проверки, они подходят для тестовых данных.

~~~clojure
(s/def :user/id int?)
(s/def :user/name string?)
(s/def :user/active? boolean?)
(s/def ::user (s/keys :req-un [:user/id :user/name :user/active?]))

(gen/generate (s/gen ::user))
{:id 88546920, :name "Z4MO7GH80k3mRD", :active? true}
~~~

Возможности `spec.gen` обширны. С ее помощью порождают связанные данные,
например, пользователей, которые ссылаются на профили и наоборот. Вместо
случайных величин можно опираться на допустимые значения (списки имен,
городов). Спеки бывают быть любой вложенности, что открывает поле для
экспериментов.

## Порядок аргументов

Необычный вопрос: как писать правильно, `(is (= 200 status))` или `(is (= status
200))`? На первый взгляд это абсурд. Разве может порядок аргументов влиять на
равенство? Значения либо равны, либо нет. Однако, макрос `is` устроен сложнее,
чем мы думаем. Он разбирает выражение `(= 200 status)` и выделяет ожидаемую и
фактическую части. По-английски они называются expected и actual.

Ожидаемое это значение, на которое рассчитывает тест. Как правило, это готовое
число или коллекция, которую посчитали заранее. Фактическое значение — то, к
которому мы пришли самостоятельно, например, вызвав функцию. Так, число 68 это
ожидаемое, а `(int (->fahr 20))` — действительное. Статус 200 это ожидаемое, а
`(:status response)` — действительное.

Такое разделение необходимо для отчетов. Когда значения не равны, нам бы
хотелось увидеть, где мы ошиблись. Предположим, что в отчете написано: `failed
(= 200 403)`. Не совсем ясно, как это трактовать. Мы ожидали успешный ответ, но
не хватило прав доступа? Или это брешь в безопасности — ожидали, что прав на эту
страницу нет, но пользовать все-таки ее увидел? Если же написано expected 200,
got 403, то все ясно — это первый случай (не хватило прав).

Теперь запомните правило: ожидаемое стоит на первом месте, а действительное на
втором. Поэтому пишите `(is (= 200 status))` вместо `(is (= status 200))`. Автор
согласен, что это непривычно и противоречит здравому смыслу. Как правило,
фактическое это число, а действительное — длинное выражение, поэтому хочется
записать их как слева. Увы, придется побороть себя и писать как справа:

~~~clojure
;; wrong                     ;; correct
(= (int (->fahr 20)) 68)     (= 68 (int (->fahr 20)))
~~~

Это не ошибка дизайна; правило уходит корнями в прошлое. Первый тестовый
фреймворк `JUnit` утвердил именно такой порядок в методах сравнения. Хорошо это
или плохо, судить уже поздно — принцип "expected слева" стал промышленным
стандартом. Аналогичное правило работает в языках Python, Ruby и
других. Отдельные фреймворки предлагают модули, чтобы "перестать говорить как
Йодо", то есть поменять семантику аргументов. Технически это возможно и в
Clojure, но сейчас мы не будем это рассматривать.

Особенность expected и actual видна при запуске тестов в CIDER. Один и тот же
тест проверяет статус ответа на 200. Пока все идет хорошо, нет разницы, в каком
порядке мы записали аргументы. Но в случе ошибки вариант слева вносит
путаницу. Согласно ему, нормальным считается статус 404, а не 200. Вариант
справа выводит статусы правильно.

~~~text
;; wrong                ;; correct
(is (= status 200))     (is (= 200 status))

Fail in test-...        Fail in test-...
expected: 404           expected: 200
  actual: 200             actual: 404
    diff: - 404             diff: - 200
          + 200                   + 404
~~~

Тем не менее, не стоит соблюдать это правило слишком рьяно. Иногда равенство с
перепутанными аргументами смотрится лучше, и потому код легче
поддерживать. Следить за порядком или нет остается на усмотрение команды. Автор
признаётся, что на этапе черновика перепутал аргументы везде, где это возможно.