# API Терминала для партнеров

## Описание API Терминала
* [Структура проекта](#0011)
* [Работа с HTTP API](#0012)
* [Работа с WebView](#0013)
* [Работа с навигацией (Navigation API)](#0017)
* [Работа с БД Интеграции (Storage API)](#0014)
* [Работа с БД Терминала (Inventory API)](#0015)
* [Логирование данных (Logging API)](#0016)

* [Список стандартных API:](#001)
    * [1. Получить итоговый чек при переходе к оплате](#001)
    * [2. Разделить чек по позициям на несколько чеков (группы печати)](#002)
    * [3. Получить все подключенные пинпады](#003)
    * [4. Оплатить мультичек на пинпаде](#004)
    * [5. Записать в итоговый чек экстра данные интеграции](#005)

<a name="0011"></a>
### Структура проекта
Общая структура проекта:
```
client.yaml - файл с описанием разрешений для приложения, указаниями файлов, адреса view, для отображения и т.д.
```
Папка client:
```
daemons - папка, в которой находятся файлы js, которые будут выполнятся в фоне (в примере отслеживаются события кассы)
uiPlugins - папка, в которой находятся файлы js, которые выполняются перед отображением webView (может не отображаться)
view - папка с html файлами, стилями, скриптами и т.д., которые будут отображены в webView
``` 
Файл архива клиента распаковывается в папку assets андроид приложения. 
В корне должен находиться файл с описанием структуры проекта - client.yaml. 
Пример содержания:
```
version: 2 //Код версии приложения
versionName: "1.0.1" //Версия приложения
packageName: Test //Имя пакета
appName: "testApp" //Отображаемое пользователю имя приложения
appUUID: "2e6dc4b8-fdac-48c1-8a1a-ade402863947" //UUID приложения
iconColor: "#0f70b7" //Цвет иконки, если она размещена на рабочем столе
capabilities:
  - inventory
  - storage
  - http
  - event-bus
  - receipts
daemons:
  - name: check //имя
    events: //События, на которые он подписан
      - evo.receipt.opened
      - evo.receipt.productAdded
      - evo.receipt.productRemoved
      - evo.receipt.closed
      - evo.receipt.clear
      - app.suggestion.used
    behavior: check-daemon.js
plugins:
  - name: discount //имя
    moments: //События, на которые он подписан
    - evo.payments.process
    - evo.payments.beforePrintReceipt
    point: before
    behavior: before-receipt-fixed.js
views:
  - name: discount-loader
    header: "Подождите"
    source: client/views/discount-loader/view.html
    scripts: # список скриптов которые должны быть подключены
      - no-script
    styles: # список стилей которые должны быть подключены
      - "*.css" # может подключить все файлы
  - name: launcher //Launcher - обязательное view, если приложение на главном экране
    header: "Подождите" //Заголовок, при отображении в webView
    source: client/views/discount-loader/view.html
    scripts:
      - no-script
    styles:
      - "*.css"
```

<a name="0012"></a>
### Работа с HTTP API
Для использования возможности работы с сервером посредством http запросов, в файле скрипта используется java объект, контекст которого передается в Java Script. 
Работа в js с ним осуществляется, как с обычным js объектом. 
Перед использованием http api, необходимо явно указать необходимость использования данного функционала в скрипте: в начале js скрипта необходимо получить этот объект с помощью:
```
var http = require('http')
```
Далее в коде, после инициализации, можно вызывать метод этого объекта для отправки запроса.
Пример:
```
function generateSuggestions(items) {
  var response = http.send({       <---- Вызов метода
    method : "POST",               <---- Тип запроса
    path : "recommendations",      <---- Путь на сервере
    body : items                   <---- Тело запроса
  })
  var jsonObject = JSON.parse(response)  <---- В ответе получаем строку, которую 
                                               приводим к JSON объекту 
                                               (в следующей версии будет передаваться сразу объект)
}
``` 
Данный функционал возможно использовать не только в сервисах-демонах, в WebView данный код тоже будет работать.

<a name="0013"></a>
### Работа с WebView
Отображение webView происходит при вызове у интерфейса **Navigation** метода 
```
pushView(...)
```
куда передается адрес html страницы для открытия, где поддерживается использование css, javascript.
Работа с Java интерфейсами внутри webView несколько отличается от работы с ними в процессах-демонах:
Доступные интерфейсы в webView:
```
Navigation
jsData
Receipt
RPC
Storage
Logger
```
Для работы с ними, нет необходимости получать их через синтаксис required, они уже интегрированы во вью. 
Работать с ними можно, как с локальными переменными, без их объявления.

<a name="0017"></a>
### Работа с навигацией (Navigation API)
Navigation api используется для работы с WebView. Для использования функционала, необходимо явно указать о намерении использования в скрипте, работа происходит через java объект, контекст которого передается в Java Script, далее работа с ним ведется, как с обычным js объектом. Для инициализации объекта, в начале скрипта указываем:
Для работы с интерфейсом в **сервисе-демоне**:
```
var navigation = require('navigation');
```
Для работы с интерфейсом в **WebView**: ничего указывать не нужно, по умолчанию, интерфейс навигации уже передан во вью, для использования обращаемся к нему, как к уже созданному объекту с именем navigation, объявлять его **не нужно**.
Доступные функции в интерфейсе:
```
* pushNext
* pushView
```
*pushNext()* - используется для перехода к следующему экрану:
При открытом WebView - закрывает его и возвращает пользователя в EvoPos, при этом в кассу передается стек операций, который представляет из себя набор действий: добавление товара в чек, удаление товара из чека, применение скидки к чеку или к отдельно выбранному товару.
*pushView(String viewLocation, String data)* - где viewLocation - адрес html страницы для открытия в WebView, data - строка для данных, которые должны быть переданы в WebView, обычно используется json формат. 
Пример работы:
```
navigation.pushView("client/views/suggestion-list/view.html", {
  suggestions: suggestedProducts,
  receipt: receipt
});
```
Для получения данных в открывшемся webView используется интерфейс jsData, имеющий метод  getData(), возвращающий данные в строковом представлении, переданные через метод pushView(). Пример использования: 
```
var passedData  = JSON.parse(jsData.getData());
```

<a name="0014"></a>
### Работа с БД Интеграции (Storage API)

Storage – система хранения данных в формате ключ – значение

Для работы с API необходимо в манифесте приложения указать:
```
capabilities:
 - storage
```

Объект для работы с API вызывается функцией:
```
var storage = require('storage')
```

Далее используются две функции:

* Сохранение данных:
```
storage.set(key, value)
```
Функция возвращает boolean: true в случае успешного сохранения, false если произошла ошибка.
key и value – строковые переменные

* Получение данных:
```
storage.get(key)
```
Функция возвращает строковую переменную ранее записанную в хранилище, либо null, если значение не было найдено.
```
key – строковая переменная
```

<a name="0015"></a>
### Работа с БД Терминала (Inventory API)

Inventory api используется для доступа к базе данных устройства, конкретнее, к таблице, содержащей информацию о товарах. 
Для использования функционала, необходимо явно указать о намерении использования в скрипте.
Работа происходит через java объект, контекст которого передается в Java Script, далее работа с ним ведется, как с обычным js объектом. 
Для инициализации объекта, в начале скрипта указываем:
```
var inventory = require('inventory');
```
После этого, мы имеем доступ к методам этого объекта, на данный момент, реализован метод, для получения информации по конкретному товару в базе данных устройства. 
Для ее получения, необходимо передать в функцию уникальный uuid товара. 
Например:
```
function getProduct(productUID){
        return inventory.getProduct(productUID);
    }
```
Результатом работы функции будет JSON объект в строковом представлении, вида:
```
{ 
  "ID":"136",
  "UUID":"1196da34-e4a8-4915-8e92-bd7792875d76",
  "CODE":"4",
  "CODE_UPPER_CASE":"4",
  "NAME":"вино апсны",
  "NAME_UPPER_CASE":"ВИНО АПСНЫ",
  "IS_GROUP":"0",
  "IS_FAVORITE":"0",
  "MEASURE_ID":"1",
  "PRICE_OUT":"20000",
  "COST_PRICE":"0",
  "QUANTITY":"-1000",
  "TAX_NUMBER":"VAT_18",
  "ABBREVIATION":"ВНП",
  "TILE_COLOR":"-26624",
  "TYPE":"NORMAL",
  "ALCOHOL_BY_VOLUME":"0",
  "ALCOHOL_PRODUCT_KIND_CODE":"0",
  "TARE_VOLUME":"0",
  "SELL_FORBIDDEN":"0",
  "DESCRIPTION":"",
  "ARTICLE_NUMBER":"",
  "ARTICLE_NUMBER_UPPER_CASE":""
}
```
Если, товар с указанным uuid не будет найден в базе, то результатом будет строка с пустым JSON объектом : 
```
{ 
}
```
Данный функционал возможно использовать не только в сервисах-демонах, в WebView данный код тоже будет работать.

<a name="0016"></a>
### Логирование данных (Logging API)

Функционал для логгирования
Объект, через который осуществляется логгирование получается функцией require:
```
var logger = require('logger')
```
Далее у объекта вызывается функция:
```
logger.log(value)
```
После выполнения функции в logcat устройства будет выведена строка value

-----------------------------------------------------
<a name="001"></a>
### 1. Получить итоговый чек при переходе к оплате

Для получения всего чека используем:
```json
 receipt.getReceipt()
 он возвращает строку в JSON формате
 {
    "receiptData": {
        "totalSum": "218.50",
        "discountPercents": "0.000000",
        "totalSumWithoutDiscount": "218.50",
        "positionDiscountSum": "0.00",
        "positionsCount": "4",
        "extraData": "{}"
    },
    "receiptPositions": [{
        "uuid": "070efd1b-4f53-401a-b5c1-cb31b5e9072d",
        "type": "NORMAL",
        "code": "1",
        "measure": "шт",
        "price": "56",
        "priceWithDiscount": "56",
        "quantity": "1"
    }, {
        "uuid": "9d2fdacd-969c-41f2-b360-a5ebbddcbe9e",
        "type": "NORMAL",
        "code": "131",
        "measure": "шт",
        "price": "30.5",
        "priceWithDiscount": "30.5",
        "quantity": "5"
    }, {
        "uuid": "7c916c19-756d-4b3d-806e-63c090080a3d",
        "type": "NORMAL",
        "code": "129",
        "measure": "шт",
        "price": "9",
        "priceWithDiscount": "9",
        "quantity": "1"
    }, {
        "uuid": "9dd36300-647e-4863-81b2-eb9591bc599b",
        "type": "NORMAL",
        "code": "127",
        "measure": "шт",
        "price": "1",
        "priceWithDiscount": "1",
        "quantity": "1"
    }]
}
```

<a name="002"></a>
### 2. Разделить чек по позициям на несколько чеков (группы печати)

Группы печати, где для задания группы печати на позицию, используется метод:
```
edited 
receipt.addPositionPrintGroup(JSON String)
 в качестве параметра
 var addPositionPrintGroup = {
    "uuid": "e82a113b-0d76-424f-9ad5-c6595ca57770", -- uuid товара
    "code": "4", -- код товара
    "printGroupId" : "4dc2-3fcd", -- id группы печати
    "printGroupIsFiscal" : true, -- признак фискальности
    "printGroupOrgName" : "Andrew", -- имя группы
    "printGroupOrgInn": "88005553535", -- ИНН
    "printGroupPaymentSum" : "123.00" --Сумма товаров в группе
};
edited
```

<a name="003"></a>
### 3. Получить все подключенные пинпады
`В процессе разработки Терминала`

<a name="004"></a>
### 4. Оплатить мультичек на пинпаде
```
receipt.addPaymentOperation(JSON String)
```

Принимает на вход принимает JSON, который описывает структуру оплаты для разбиения общего чека на платежи:
```
var paymentOperation = {
  "uuid": "1ef45g5", //юид оплаты
  "deviceUUID": "smth", //юид устройства для оплаты
  "paymentSum" : "123.00", //сумма платежа
  "isCashless" : true, //признак картой/нал
  "printGroups" : [ //список групп печати в оплате
  {
      "printGroupId" : "4dc2-3fcd",
      "printGroupIsFiscal" : true,
      "printGroupOrgName" : "Andrew",
      "printGroupPaymentSum" : "123.00",
      "printGroupOrgInn": "88005553535"
   }
  ]
};
```

<a name="005"></a>
### 5. Записать в итоговый чек экстра данные интеграции
Все необходимые данные интеграция может добавить в нужном ей формате как дополнительную информацию по документу чека продажи:
```
receipt.addExtraReceiptData(String extraData)
```
