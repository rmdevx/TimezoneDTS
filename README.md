# Сервис TimezoneDTS
Сначала необходимо будет скачать все доп. пакеты, для этого наберите в консоли:
```bash
$ composer update --dev
```
Затем, для запуска сервиса необходимо запустить docker-контейнеры.
Если в вашей системе есть команда `make`, то можно просто набрать: 
```bash
$ make init
```
Все команды в **Makefile** уже прописаны. 
Либо, если команда `make` недоступна, можно набрать в консоли:
```bash
$ docker-compose up -d --build
```

После того, как соберутся образы, будет создана БД mysql и таблицы в ней, сервис будет доступен по адресу `http://localhost:8080/`.
Доступны следующие запросы:
+ POST `http://localhost:8080/time/` - получения локального времени в городе по переданному идентификатору города и метке времени по UTC+0
+ POST `http://localhost:8080/time/local/` - обратное преобразование из локального времени и идентификатора города в метку времени по UTC+0
+ GET `http://localhost:8080/time/refresh/` - запуск процесса обновления данных

Формат данных для POST-запросов:
```json
{
  "city_id": "Идентификатор_Города",
  "time":"2021-10-30 19:00:00"
}
```

Все настройки хранятся в файле `/src/config.php` - подключение к БД, настройки роутинга и TimeData-адаптера. 

## Подключение к БД
Используемая БД - MySql. Настройки подключения указаны в конфиге.
При сборе докер-контейнеров база инициализируется отсюда `/docker/mysql/scripts` - создаются 2 таблицы `city` и `city_tm`, в таблицу `city`
загружаются тестовые данные. 

## Настройки роутинга
Настройки роутинга в config.php выглядят так:
```php
    'routes' => [
        'url_prefix' => '',
        'controllers' => [
            'time' => 'Mirai\\Timezone\\Controller\\TimeController',
        ]
```
`url_prefix` - это возможный префикс к адресу(вида `http://localhost:8080/url_prefix/`), который будет отсекаться

`controllers` - здесь массив контроллеров, где **key** - это часть адреса, а **value** - класс контроллера.

На текущих настройках у нас по адресу `http://localhost:8080/time/` подключается класс контроллера `\Mirai\Timezone\Controller\TimeController`.

Каждый контроллер наследуется от абстрактного класса `Controller` и должен реализовывать метод `Controller::getRoutes()`. 
Этот метод должен возвращать массив вида:
```php
        return [
            'post' => [
                '/' => 'getTimeWithUTC',
                '/local' => 'getTimeWithLocal',
            ],
            'get' => [
                '/refresh' => 'refreshTimezone',
            ]
        ];
```
где для каждого метода HTTP-запроса(POST,GET,PUT и т.д.) задается массив, где **key** - часть адреса, а **value** - имя функции в текущем контроллере.

## Работа с Timezone
Данные по timezone и смещениям могут браться либо из `https://timezonedb.com/references/get-time-zone`, либо из csv-файла `/public/timezone.csv` - в нем просто продублирована информация, полученная от timezonedb.
Для timezonedb нужен api key, его необходимо указать в конфиге.

Можно выбрать, откуда получать информацию, указав в конфиге класс адаптера. Каждый адаптер должен реализовывать интерфейс `\Mirai\Timezone\Time\TimeDataInterface`.

Данные, которые получаем от timezonedb для каждого города:
+ **zone1** - время начала текущей временной зоны по UTC+0
+ **zone2** - время окончания текущей временной зоны и начала следующей временной зоны по UTC+0
+ **zone3** - время окончания следующей временной зоны по UTC+0
+ **gmtOffset1** - смещение от по UTC+0 для текущей временной зоны
+ **gmtOffset2** - смещение от по UTC+0 для следующей временной зоны

## Что еще можно будет реализовать в дальнейшем?
+ Избавится от дублирования кода в классе `\Mirai\Timezone\Controller\TimeController`
+ Вынести обновление данных на cron, обновлять по php-cli
+ Добавить для класса Config еще один уровень абстракции, чтобы можно было использовать разные форматы конфиг-файлов
+ Прикрутить логирование
+ Увеличить покрытие кода тестами
+ Перенести запуск composer update в docker-контейнер
