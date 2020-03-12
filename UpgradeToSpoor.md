# Основные изменения

Для лучшей работы курса и сбора статистики было необходимо переписать текущее расширение `adapt-contrib-xapi`, добавив в него нужный функционал, а именно:
1. Сменить текущий тип запуска на `Spoor`
2. Добавить в поле `context` значения:
    - `os` – текущая операционная система
    - `browser` – текущий браузер
    - `screen-size` – текущее разрешение экрана 
3. Обновить видеопрофиль для компонента `media`
4. Генерировать стейтмент `experienced` при каждом запуске компонентов, а не только `completed` один раз
5. Генерировать карту курса в `tincan.xml` с подсчетом количества знаков)

Ниже описан процесс изменения исходного кода, который позволил добиться нужного функционала.

## Изменение типа запуска на Spoor
Изначально, расширение `adapt-contrib-xapi` посылает запросы в LRS через GET-запрос (вся информация о пользователе хранится и передается через URL). 

Нам такой тип передачи данных не подходит, поэтому нужно изменить его на `Spoor`. Изначальные настройки профиля нам подходят, нужно только поменять модель передачи данных.

Для этого нам сначала нужно изменить конфигурацию `xAPIWrapper`. 
1. Находим функцию `initializeWrapper`, которая отвечает за запуск враппера
2. Находим строчку, содержащую стандартный конфиг и меняем ее на: 
    ```javascript
    let conf = {
      actor,
      registration: this.queryString().registration,
      endpoint: this.getConfig("_endpoint"),
      auth: "Basic " + toBase64(this.getConfig("_user") + ":" + this.getConfig("_password"))
    }; 
    ```
3. Устанавливаем значение инициализации: `this.set({isInitialised: true})`

Для отправки состояний нужно также переписать функцию отправки и передачи данных. Сначала поменяем отправку на сервер. Для этого нужно найти функцию `sendStatementsSync`, и изменить отправляемые заголовки (`headers`)
```javascript
var headers = {
  'Content-Type': 'application/json',
  'Authorization': "Basic " + toBase64(this.config.get("_user") + ":" + this.config.get("_password")),
  'X-Experience-API-Version': ADL.XAPIWrapper.xapiVersion,
  'Access-Control-Allow-Origin': "*"
};
``` 
  
## Добавление полей контекста  
Все стейтменты посылаются функцией `onStatementReady`, поэтому нужно ее найти и отредактировать.
1. Задаем игрока: 
```javascript
statement.actor = {"mbox": "mailto:" + pipwerks.SCORM.data.get('cmi.core.student_id')};
```
2. Получаем расширения:
     ```javascript
    let extensions = {
      'https://sberbank.ru/os': Adapt.device.OS,
      'https://sberbank.ru/browser': Adapt.device.browser,
      'https://sberbank.ru/screen-size': outerWidth + "*" + outerHeight
    };
    ```
    В самом адапте уже хранится информация о браузере и ОС, поэтому придумывать что-то свое нет необходимости. Ну а для размера экрана хватит стандартных JavaScript-свойств окна – `window.outerWidth` и `window.outerHeight`
3. Нужно добавить эти расширения в стейтмент, но так как их может изначально не быть, нужно это проверить: 
    ```javascript
    if (!statement.context) // проверяем что контекст вообще есть
      statement.context = { // его нет, значит, нужно добавить поле
        extensions // и в него уже записать расширения
      };
    else if (!statement.context.extensions) // контекст есть, но вдруг нет расширений
      statement.context.extensions = extensions;
    else // все объекты есть, значит просто добавим новые расширения к уже имеющимся
      statement.context.extensions = Object.assign(statement.context.extensions, extensions);
    ``` 
4. Используя такую же логику, нужно добавить поле `result`, в котором будет храниться время выполнения стейта:
    ```javascript
    if (!statement.result)
      statement.result = {
        duration: this.convertMillisecondsToISO8601Duration(this.getAttemptDuration())
      };
    else if (!statement.result.duration)
      statement.result.duration = this.convertMillisecondsToISO8601Duration(this.getAttemptDuration());
    ```
   Функция `convertMillisecondsToISO8601Duration()` стандартная в `adapt-contrib-xapi`, она переводит время в секундах в нужный формат.
5. И остается только передать стейтмент в LRS:
    ```javascript
    let xapiState = new ADL.XAPIStatement(statement);
    let resp = ADL.XAPIWrapper.sendStatement(xapiState);
    ```