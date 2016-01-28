# RQG - Recursive Query Generator #
RQG - рекурсивный генератор запросов на языке SQL.
## Для чего? ##
Ускорить разработку, отказаться от привязки к конкретной БД в пользу подключаемых драйверов.
## Предыстория ##
Недавно взявшись за новый проект я заметил, что большую часть времени у меня занимает написание функций по работе с базой данных (добавление, удаление, редактирование, выборки с сортировками и без итд). К тому же это занятие нудное и скучное. Но как же это дело упростить? При этом не потеряв в производительности?
Нужно как то абстрагироваться от конкретной базы данных, от этих многострочных SQL запросов и циклов с выборками...
## Идея ##
Идея не нова и информацию по теме можно почерпнуть из статей по ORM (Object-relational mapping). Идея ORM по сути состоит в предоставлении некой объектной модели доступа к данным. Например, у нас есть список стран с их столицами, численностью населения. В терминах объектно-ориентированного программирования они будут представляться объектами класса Страна, содержащими некий список полей: столица, численность населения.
## Плюсы и минусы ##
ORM избавляет программиста от написания большого количества кода, часто однообразного и подверженного ошибкам, тем самым значительно повышая скорость разработки.
С точки зрения программиста такая система является лишь хранилищем данных и ему не нужно заботится о том, как эти данные представлены на более низком уровне.

Такой подход позволяет полностью абстрагироваться от источника данных и это, несомненно, один из основных плюсов. Это также означает то, что продукт, использующий ORM не привязан к какой - либо конкретной БД, а имеет несколько "драйверов", реализующих требуемые методы для конкретной БД. Это тоже безусловный плюс.

А что же с минусами? Ну а как же без них)))

Системы ORM обычно ограничивают возможности оптимизации современных БД из за унифицированных запросов, хотя обычно позволяют программисту при необходимости самому жёстко задать код SQL-запроса, который будет использоваться при тех или иных действиях (сохранение в базу данных, загрузка, поиск и т. д.). Программист не может гарантировать что код, сгенерированный ORM будет быстрым и эффективным, а большое количество порождаемых объектов увеличивает объем потребляемой памяти.

## Цели ##
Если существует множество готовых решений, почему я взялся изобретать велосипед?
Существующие реализации ORM слишком громоздки для использования их каждый день, на мой взгяд черезчур избыточны. Большая доля расмотренных мной ORM требует много лишних телодвижений (например для начала описать таблицы на XML). Я привык проектировать структуру БД вручную и мне не нужно для этого никаких автоматизированых систем. К тому же сложно повлиять на генерируемый код и улучшить оптимизацию, да и порой нужных "фишек" просто не хватает.
Разрабатывая RQG я ставил перед собой несколько задач:
  * максимально упростить работу с базой
  * сохранить возможность "ручного управления"
  * легкость встраивания
  * возможность писать "как хочу" (те использование собственных алиасов для комманд, регистронезависимость)
  * использование "драйвера" для запросов к БД
## С чего я начинал ##
Начал я с того, что попытался представить как оно будет выглядеть в конечном варианте, те как я это дело буду непосредственно использовать. На листочке нарисовались такие варианты:
```
$obj -> table('coumn1', 'column2') -> all;
$obj -> table -> sort('column1') -> limit(20, 10) -> get;
$obj -> table -> filter('id = 1') -> delete;
$obj -> table('column1') -> add('value');
$obj -> table('column1') -> filter('id = 1') -> update('value');
```
Вроде легко воспринимается, это то что надо )))
## Рекурсивный генератор ##
Для реализации таких цепочек объектов несомненно требуется рекурсивное создание объектов, но как это реализовать на php?
Выручили магические методы call и get - они вызываются в тех случаях, когда вызываемый метод объекта, или запрашиваемое свойство не существуют.
Таким образом я смог использовать цепочки объектов неограниченной длинны, объекты рекурсивно создаются и заполняют масссив параметров:
```

    /*
     *  \brief Конструктор
     */
    public function __construct($propertys = array()) {
        $this -> propertys = $propertys;
    }

    /*
     *  \brief Обработка запроса несуществующего свойства
     */
    public function __get($property) {
        $this -> propertys[strtolower($property)] = true;
        if (!in_array(strtolower($property), $this -> finals))
            return new self($this -> propertys);
        return $this -> __process($this -> propertys);
    }

    /*
     *  \brief Обработка вызова несуществующего метода
     */
    public function __call($method, $args) {
        $this -> propertys[strtolower($method)] = $args;
        if (!in_array(strtolower($method), $this -> finals))
            return new self($this -> propertys);
        return $this -> __process($this -> propertys);
    }

```
Возникает вопрос: а как же определить что цепочка закончилась?
Для этого используются слова - финализаторы, встреча с которыми означает что цепочка закончена и нужно совершить некоторое действие.
```
    // Финализаторы - свойства и методы, завершающие рекурсию
    protected $finals = array('all','byid','delete','count','add','update','insert','get','one');
```

Ну а дальше, как говорится, дело техники. Разбираем массив параметров, первым обязательно идет имя таблицы, потом фильтры, сортировщики итд, оканчивается все финализатором.
## Работа с базой ##
Как же RQG будет работать с базой, при этом не привязваясь к конкретной БД?
Работа с базой возлагается на объект dbAPI, ссылка на который должна быть передана пользователем в статичное свойство db. В коде это выглядит так:
```
...
    // Ссылка на объект, реализующий dbAPI
    static $db = null;
...
    // Выборка из базы
    $result = self::$db -> fetch($this -> sql);
...
    /*
     *  \brief Выполняет запрос (dbApi)
     */
    public static function query($sql) {
        if (!is_null(self::$db))
            return self::$db -> query($sql);
        return false;
    }

    /*
     *  \brief Выполняет запрос и формирует объект с результатом (dbApi)
     */
    public static function fetch($sql) {
        if (!is_null(self::$db))
            return self::$db -> fetch($sql);
        return false;
    }

    /*
     *  \brief Выполняет запрос на добавление (dbApi)
     */
    public static function insert($sql) {
        if (!is_null(self::$db))
            return self::$db -> insert($sql);
        return false;
    }
```
## Как использовать ##
В данном разделе я постараюсь показать на сколько RQG легок в использовании и какие запросы в каких случаях он формирует.
Начнем с того, что RQG позволяет работать в двух режимах. Рассмотрим оба варианта и покажем какой лучше использовать и в каких случаях.

Один объект на всех:
```
$rqg = new RQG();
$rqg -> users;
if ($order) $rqg -> sort($order);
if ($limit) $rqg -> limit($limit);
$users = $rqg -> all;
```
Мы создаем экземпляр объекта RQG и можем начинать с ним работу. При таком подходе каждый новый запрос будет использовать один и тот же экземпляр объекта. Мы можем собирать запрос по кусочкам до тех пор, пока не будет вызван финализатор. С точки зрения потребления памяти, наверное это лучший вариант, да и выглядит он короче чем
```
RQG::create() -> users -> all;
```
Это синглтон, экземпляр объекта, гарантирующий что он не будет использован дважды (мы не создаем на него ссылку и поэтому он существует только в нашей цепочке и после финализатора мы никогда его больше не сможем использовать). Вариант с одним экземпляром на всех - очень удобен для создания запросов, основываясь на каких то передаваемых данный, но приводит к совершенно непредсказуемым последствиям при использовании вложенных запросов, например добавление двух записей в разные таблицы, при которых вторая запись добавляется на основе уникального идентификатора первой:
```
$rqg = new RQG();
$rqg -> info('id_user', 'phone') -> add($rqg -> users('name') -> add('Вася'), '12345');
```
Такой код обречен быть нерабочим из-за того, что в объект $rqg собираются поля для двух запросов сразу и тк первым выполнится финализатор из вложенного запроса, а данные в объекте будут от первого - это приведет к ошибке.
### С чего начинать ###
Давайте разберемся как использовать rqg для выборки данных.
Хотя rqg и не обладает строгим синтаксисом, все же необходимо придерживаться некоторых правил:
  * Любой запрос начинается с таблицы, из которой получаем данные
```
    $rqg -> mytable
```
> В данном случае такая запись будет воспринята как
```
    SELECT * FROM mytable 
```
  * Если мы хотим определить список получаемых полей, то используем такую запись:
```
    $rqg -> mytable('column1', 'column2') 
```
> что соответствует
```
    SELECT column1, column2 FROM mytable 
```
  * Любой запрос должен оканчиваться финализатором
### Финализаторы ###
От последнего элемента в цепочке зависит то, какое действие мы ждем от rqg. Вот список финализаторов, и ответные действия rqg:
  * all - соответствует SELECT ... FROM ...
  * get - от all ничем не отличается и служит лишь для того, чтобы логически разделить выборки, при которых используются или не используются фильтры
  * byid - SELECT ... FROM ... WHERE id = 'id'
  * one - от get отличается тем, что в результате всегда возвращается самая первая запись из массива результата (с индексом 0)
  * count соответствует SELECT count(**) as count FROM ...
  *****delete****DELETE FROM ...
  *****add,insert****INSERT INTO ...
  *****update****UPDATE ...
### Фильтры ###
Для выборки данных нужны фильтры. Для использования фильтра в цепочку rqg достаточно добавить
```
-> filter() 
```
Есть 2 способа формирования фильтров:
  * В фильтр передаем массив ключ - значение, при формировании SQL условия будут объединены по AND
```
$rqg -> mytable -> filter(array('id' => 1)) -> get; 
```
  * В фильтр передаем блок WHERE sql запроса (в данном случае filter лучше заменить на where, для логического разделения)
```
$rqg -> mytable -> where('id = 1') -> get; 
```
### Постфильтр ###
Все аналогично фильтрам. Для использования надо добавить
```
-> having() 
```**

### Сортировка ###
Для сортировки используются следующие методы: sort, order, orderby, sortby - работают они все одинаково и являются лишь алиасами. В качестве параметра передаются имена полей, по которым производить сортировку в требуемом порядке.
```
$rqg -> news -> sort('date') -> all;
```
Результатом будет служить
```
SELECT * FROM news ORDER BY date
```
### Группировка ###
Для сортировки используются методы group, groupby - работают они одинаково и являются лишь алиасами. В качестве параметра передается поле, по которому производится группировка.
```
$rqg -> news -> sort('date') -> group('tag') -> all;
```
Результатом будет служить
```
SELECT * FROM news ORDER BY date GROUP BY tag
```
### Ограничение вывода ###
Для ограничения вывода используется limit() - параметры аналогичны тем, что передаются в sql LIMIT ...
```
$rqg -> mytable -> limit(10) -> get;
$rqg -> mytable -> limit(10, 5) -> get;
```
### Добавление данных ###
Существует два варианта добавления данных. Покажу на примере:
```
$rqg -> mytable('column1', 'column2') -> add('value1', 'value2');
$rqg -> mytable -> add(array('column1' => 'value1', 'column2' => 'value2'));
```
### Редактирование ###
По синтаксису аналогично добавлению, но обязательно использование фильтра.
```
$rqg -> mytable('column1', 'column2') -> filter(array('id' => '1')) -> update('value1', 'value2');
```
### Запросы с JOIN ###
Для связи таблиц можно использовать join, ljoin, leftjoin, rjoin, rightjoin - первые три являются алиасами для LEFT JOIN, последние два для RIGHT JOIN.

join всегда используется совместно с on. В join передается имя таблицы, к которой будем джойнится, а в on - два параметра: первый - поле первой таблицы, второй - имя поля второй таблицы, те это поля по котрым будет происходить join.
```
$rqg -> users -> join('info') -> on('id', 'id_user') -> get;
```
Результат выполнения:
```
SELECT * FROM users LEFT JOIN info ON users.id = info.id_user
```

## БД драйверы ##
### dbAPI ###
Для написания своего драйвера необходимым условием является наличие трех публичных функций, получающим запрос SQL в качестве параметра:
```
public function query($sql) {}
public function insert($sql) {}
public function fetch($sql) {}
```
### Mysql ###
```

// Пример класса, реализующего dbAPI для mysql
class mysqlAPI {

    public function __construct($h, $debug=false) {
        // $h - link на поключение
        $this -> handle = $h;
        $this -> debug = $debug;
    }

    public function fetch($sql) {
        if ($this -> debug) echo "<pre>SQL:", $sql, "</pre>";
        if ($h = mysql_query($sql, $this -> handle)) {
            $r = array();
            while ($tmp = mysql_fetch_object($h)) {
                if (count($tmp) > 0) {
                    $r[] = $tmp;
                }
            }
            return $r;
        }
        echo mysql_error($this -> handle);
        return false;
    }

    public function insert($sql) {
        if ($this -> debug) echo "<pre>SQL:", $sql, "</pre>";
        if ($h = mysql_query($sql, $this -> handle))
            return mysql_insert_id($this -> handle);
        echo mysql_error($this -> handle);
        return false;
    }

    public function query($sql) {
        if ($this -> debug) echo "<pre>SQL:", $sql, "</pre>";
        if ($h = mysql_query($sql, $this -> handle))
            return true;
        echo mysql_error($this -> handle);
        return false;
    }
}
```
### Sqlite ###
```

// Пример класса, реализующего dbAPI для sqlite
class sqliteAPI {

    public function __construct($file, $debug=false) {
        if (is_file($file)) {
            $this -> handle = sqlite_open($file);
            $this -> debug = $debug;
        }  else
            die("$file is not a valid filename");
    }

    public function fetch($sql) {
        if ($this -> debug) echo "<pre>SQL:", $sql, "</pre>";
        if ($h = sqlite_query($sql, $this -> handle)) {
            $r = array();
            while ($tmp = sqlite_fetch_object($h)) {
                if (count($tmp) > 0) {
                    $r[] = $tmp;
                }
            }
            return $r;
        }
        echo sqlite_last_error($this -> handle);
        return false;
    }

    public function insert($sql) {
        if ($this -> debug) echo "<pre>SQL:", $sql, "</pre>";
        if ($h = sqlite_query($sql, $this -> handle))
            return sqlite_last_insert_rowid($this -> handle);
        echo sqlite_last_error($this -> handle);
        return false;
    }

    public function query($sql) {
        if ($this -> debug) echo "<pre>SQL:", $sql, "</pre>";
        if ($h = sqlite_query($sql, $this -> handle))
            return true;
        echo sqlite_last_error($this -> handle);
        return false;
    }

}

```
## Пример использования ##
Небольшой примерчик по использованию rqg с драйвером sqlite:
```
include 'sqlite.lib.php';
include 'rqg.lib.php';
// Удаляем старый файлик базы
@unlink('temp.db');
// И создаем новый
touch('temp.db');
// Передадим ссылку на объект dbApi (второй параметр - включение отладки)
RQG::$db = new sqliteAPI('temp.db', true);
// Создаем табличку =) Просто sql запрос
RQG::query("CREATE TABLE test_temp (key VARCHAR(32), value VARCHAR(32))");
// Зададим префикс для таблиц, чтобы обращаться по короткому имени без префикса
RQG::$prefix = 'test_';
// Заполним табличку данными из массива $_SERVER
foreach($_SERVER as $key => $value)
    RQG::create() -> temp('key', 'value') -> add($key, $value);
echo "<pre>";
//Посмотрим сколько записей было добавлено
echo RQG::create() -> temp -> count;
// Посмотрим как выглядит первая запись
print_r(RQG::create() -> temp -> limit(1) -> one);
// Обновим значение в записи с ключом DOCUMENT_ROOT
RQG::create() -> temp('value') -> where("key = 'DOCUMENT_ROOT'") -> update('qweqweqwe');
// Посотрим поменялось ли значение
echo RQG::create() -> temp('value') -> where("key = 'DOCUMENT_ROOT'") -> one -> value;
// Удалим все данные
RQG::create() -> temp -> delete;
// Убедимся что все удалено =)
echo RQG::create() -> temp -> count;
```
Вывод скрипта с отладкой
```
SQL:CREATE TABLE test_temp (key VARCHAR(32), value VARCHAR(32))

SQL:INSERT INTO  test_temp (key,value) VALUES ('HTTP_HOST','deadlink.local')

SQL:INSERT INTO  test_temp (key,value) VALUES ('HTTP_USER_AGENT','Mozilla/5.0 (X11; Linux i686; rv:2.0b7) Gecko/20100101 Firefox/4.0b7')

SQL:INSERT INTO  test_temp (key,value) VALUES ('HTTP_ACCEPT','text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8')

SQL:INSERT INTO  test_temp (key,value) VALUES ('HTTP_ACCEPT_LANGUAGE','ru-ru,ru;q=0.8,en-us;q=0.5,en;q=0.3')

SQL:INSERT INTO  test_temp (key,value) VALUES ('HTTP_ACCEPT_ENCODING','gzip, deflate')

SQL:INSERT INTO  test_temp (key,value) VALUES ('HTTP_ACCEPT_CHARSET','windows-1251,utf-8;q=0.7,*;q=0.7')

SQL:INSERT INTO  test_temp (key,value) VALUES ('HTTP_KEEP_ALIVE','115')

SQL:INSERT INTO  test_temp (key,value) VALUES ('HTTP_CONNECTION','keep-alive')

SQL:INSERT INTO  test_temp (key,value) VALUES ('SERVER_SOFTWARE','Apache/2.2.16 (Ubuntu)')

SQL:INSERT INTO  test_temp (key,value) VALUES ('SERVER_NAME','deadlink.local')

SQL:INSERT INTO  test_temp (key,value) VALUES ('SERVER_ADDR','192.168.0.48')

SQL:INSERT INTO  test_temp (key,value) VALUES ('SERVER_PORT','80')

SQL:INSERT INTO  test_temp (key,value) VALUES ('REMOTE_ADDR','192.168.0.48')

SQL:INSERT INTO  test_temp (key,value) VALUES ('DOCUMENT_ROOT','/home/deadlink/www/deadlink.local')

SQL:INSERT INTO  test_temp (key,value) VALUES ('SERVER_ADMIN','webmaster@localhost')

SQL:INSERT INTO  test_temp (key,value) VALUES ('SCRIPT_FILENAME','/home/deadlink/www/deadlink.local/orm/test.php')

SQL:INSERT INTO  test_temp (key,value) VALUES ('REMOTE_PORT','51610')

SQL:INSERT INTO  test_temp (key,value) VALUES ('GATEWAY_INTERFACE','CGI/1.1')

SQL:INSERT INTO  test_temp (key,value) VALUES ('SERVER_PROTOCOL','HTTP/1.1')

SQL:INSERT INTO  test_temp (key,value) VALUES ('REQUEST_METHOD','GET')

SQL:INSERT INTO  test_temp (key,value) VALUES ('QUERY_STRING','')

SQL:INSERT INTO  test_temp (key,value) VALUES ('REQUEST_URI','/orm/test.php')

SQL:INSERT INTO  test_temp (key,value) VALUES ('SCRIPT_NAME','/orm/test.php')

SQL:INSERT INTO  test_temp (key,value) VALUES ('PHP_SELF','/orm/test.php')

SQL:INSERT INTO  test_temp (key,value) VALUES ('REQUEST_TIME','1292313179')

SQL:INSERT INTO  test_temp (key,value) VALUES ('argv','Array')

SQL:INSERT INTO  test_temp (key,value) VALUES ('argc','0')

SQL:SELECT  count(*) as count FROM test_temp

31

SQL:SELECT  * FROM test_temp LIMIT 1

stdClass Object
(
    [key] => HTTP_HOST
    [value] => deadlink.local
)

SQL:UPDATE test_temp SET value = 'qweqweqwe' WHERE key = 'DOCUMENT_ROOT'

SQL:SELECT  value FROM test_temp WHERE key = 'DOCUMENT_ROOT'

qweqweqwe

SQL:DELETE FROM test_temp

SQL:SELECT  count(*) as count FROM test_temp

0
```
# RQG Source #
```

/*
 *      rqg.lib.php
 *
 *      Copyright 2010 deadlink <dev.link@yandex.ru>
 *
 *      This program is free software; you can redistribute it and/or modify
 *      it under the terms of the GNU General Public License as published by
 *      the Free Software Foundation; either version 2 of the License, or
 *      (at your option) any later version.
 *
 *      This program is distributed in the hope that it will be useful,
 *      but WITHOUT ANY WARRANTY; without even the implied warranty of
 *      MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 *      GNU General Public License for more details.
 *
 *      You should have received a copy of the GNU General Public License
 *      along with this program; if not, write to the Free Software
 *      Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston,
 *      MA 02110-1301, USA.
 */

/**
 *  \brief Класс RQG (Recursive Query Generator)
 *  \date 01.12.2010
 *  \update 14.12.2010
 *  \version 1.5 alpha
 */
class RQG {
    // Массив переданных свойств
    private $propertys = array();
    // Сюда собираются параметры для запроса
    private $params = array();
    // Тут сам sql запрос
    private $sql = '';
    // Финализаторы - свойства и методы, завершающие рекурсию
    protected $finals = array('all','byid','delete','count','add','update','insert','get', 'one');
    // Префикс для таблиц
    static $prefix = '';
    // Ссылка на объект, реализующий dbAPI
    static $db = null;

    /*
     *  \brief Возвращает новый объект RQG
     */
    static function create() {
        return new self;
    }

    /*
     *  \brief Конструктор
     */
    public function __construct($propertys = array()) {
        $this -> propertys = $propertys;
    }

    /*
     *  \brief Обработка запроса несуществующего свойства
     */
    public function __get($property) {
        $this -> propertys[strtolower($property)] = true;
        if (!in_array(strtolower($property), $this -> finals))
            return new self($this -> propertys);
        return $this -> __process($this -> propertys);
    }

    /*
     *  \brief Обработка вызова несуществующего метода
     */
    public function __call($method, $args) {
        $this -> propertys[strtolower($method)] = $args;
        if (!in_array(strtolower($method), $this -> finals))
            return new self($this -> propertys);
        return $this -> __process($this -> propertys);
    }

    /*
     *  \brief Чистка переменных
     */
    private function free() {
        unset($this -> propertys);
        unset($this -> params);
    }

    /*
     *  \brief Обработка
     */
    private function __process($propertys) {
        // Берем первый переданный параметр - это имя таблицы
        $this -> params['table'] = self::$prefix . array_shift(array_keys($propertys));
        // Значение первого параметра - массив полей
        $columns = array_shift($propertys);
        if ($columns === true || count($columns) == 0)
            $this -> params['columns'] = '*';
        else $this -> params['columns'] = implode(',', $columns);
        // Финализатор - последний элемент
        $final = array_pop(array_keys($propertys));
        // Аргументы финализатора
        $args = array_pop($propertys);
        // Пройдемся по всем свойствам и заполним массив параметров
        foreach ($propertys as $name => $value)
            $this -> set($name, $value);
        // Построим sql запрос
        $this -> build($final, $args);
        // Почистим переменные
        $this -> free();
        // Вызовем нужную функцию из dbAPI
        if (!is_null(self::$db)) {
            if (in_array($final, array('add', 'insert')))
                $result = self::$db -> insert($this -> sql);
            elseif (in_array($final, array('delete', 'update')))
                 $result = self::$db -> query($this -> sql);
            elseif ($final == 'one') {
                $result = self::$db -> fetch($this -> sql);
                if ($result[0])
                    $result = $result[0];
            }
            elseif ($final == 'count') {
                $result = self::$db -> fetch($this -> sql);
                if ($result[0])
                    $result = $result[0] -> count;
            } else {
                $result = self::$db -> fetch($this -> sql);
            }
            return $result;
        }
    }

    /*
     *  \brief В зависимости от имени свойства добавляет его в массив параметров
     */
    private function set($name, $value) {
        // ORDER BY
        if (in_array($name, array('order','orderby', 'sort', 'sortby'))) {
            if ($value === true || count($value) == 0)
                $this -> params['order'] = $value;
            else $this -> params['order'] = implode(',', $value);
        }
        // GROUP BY
        elseif (in_array($name, array('group','groupby'))) {
            if ($value === true || count($value) == 0)
                $this -> params['group'] = $value;
            else $this -> params['group'] = implode(',', $value);
        }
        // LIMIT
        elseif ($name == 'limit') {
            if ($value === true || count($value) == 0)
                $this -> params['limit'] = $value;
            else $this -> params['limit'] = implode(',', $value);
        }
        // DISTINCT
        elseif (in_array($name, array('unique','distinct'))) {
            $this -> params['distinct'] = true;
        }
        // LEFT JOIN
        elseif (in_array($name, array('join','ljoin','leftjoin'))) {
            $this -> params['leftjoin'] = self::$prefix . $value[0];
        }
        // RIGHT JOIN
        elseif (in_array($name, array('rjoin','rightjoin'))) {
            $this -> params['rightjoin'] = self::$prefix . $value[0];
        }
        // ON (IF JOIN)
        elseif ($name == 'on') {
            $this -> params['on'] = $value;
        }
        // ORDER BY RAND()
        elseif ($name == 'random') {
            $this -> params['random'] = $value;
        }
        // WHERE
        elseif (in_array($name, array('filter','where'))) {
            $this -> params['where'] = $value;
        }
        // HAVING
        elseif (in_array($name, array('having'))) {
            $this -> params['having'] = $value;
        }
    }

    /*
     *  \brief Формирует sql на основе массива параметров запроса
     */
    private function build($type, $args) {
        $where = array();
        $having = array();
        $distinct = '';
        // DISTINCT
        if (isset($this -> params['distinct'])) $distinct = 'DISTINCT';
        // SELECT
        if (in_array($type, array('all','get','byid','one'))) {
            $this -> sql = "SELECT $distinct {$this -> params['columns']} FROM {$this -> params['table']}";
        }
        // INSERT
        elseif (in_array($type, array('add','insert'))) {
            $this -> sql = "INSERT INTO $distinct {$this -> params['table']} ";
            if ($this -> params['columns'] == '*') {
                $this -> sql .= "(".implode(',', array_keys($args[0])).") ";
                $this -> sql .= "VALUES (".self::arr2mstr($args[0]).") ";
            } else {
                $this -> sql .= "({$this -> params['columns']}) ";
                $this -> sql .= "VALUES (".self::arr2mstr($args).")";
            }
        }
        // UPDATE
        elseif ($type == 'update') {
            $set = array();
            $this -> sql = "UPDATE {$this -> params['table']} ";
            if ($this -> params['columns'] == '*') {
                $keys = array_keys($args[0]);
                $vals = array_values($args[0]);
            } else {
                $keys = explode(',', $this -> params['columns']);
                $vals = $args;
            }
            for ($i = 0; $i < count($keys); $i++) {
                $set[] = "{$keys[$i]} = '{$args[$i]}'";
            }
            $this -> sql .= "SET ".implode(',', $set);
        }
        // SELECT count()
        elseif ($type == 'count') {
            $this -> sql = "SELECT $distinct count(*) as count FROM {$this -> params['table']}";
        }
        // DELETE
        elseif ($type == 'delete') {
            $this -> sql = "DELETE FROM {$this -> params['table']}";
        }
        // LEFT JOIN
        if (isset($this -> params['leftjoin']) && isset($this -> params['on'])) {
            $this -> sql .= " LEFT JOIN {$this -> params['leftjoin']} ON ".
                "{$this -> params['table']}.{$this -> params['on'][0]} = ".
                "{$this -> params['leftjoin']}.{$this -> params['on'][1]}";
        }
        // RIGHT JOIN
        if (isset($this -> params['rightjoin']) && isset($this -> params['on'])) {
            $this -> sql .= " RIGHT JOIN {$this -> params['rightjoin']} ON ".
                "{$this -> params['table']}.{$this -> params['on'][0]} = ".
                "{$this -> params['rightjoin']}.{$this -> params['on'][1]}";
        }
        // GROUP BY
        if (isset($this -> params['group']))
            $this -> sql .= " GROUP BY {$this -> params['group']}";
        // HAVING
        if (isset($this -> params['having'])) {
            if (is_array($this -> params['having'][0])) {
                foreach ($this -> params['having'][0] as $col => $val)
                    if (is_string($val))
                        $having[] = "$col LIKE '$val'";
                    else $having[] = "$col = '$val'";
            } else $having[] = $this -> params['having'][0];
        }
        if (count($having) > 0)
            $this -> sql .= ' HAVING '.implode(' AND ', $having);
        // WHERE id
        if($type == 'byid') {
            $where[] = "{$this -> params['table']}.id = '{$args[0]}'";
        }
        // WHERE
        if (isset($this -> params['where'])) {
            if (is_array($this -> params['where'][0])) {
                foreach ($this -> params['where'][0] as $col => $val)
                    if (is_string($val))
                        $where[] = "$col LIKE '$val'";
                    else $where[] = "$col = '$val'";
            } else $where[] = $this -> params['where'][0];
        }
        if (count($where) > 0)
            $this -> sql .= ' WHERE '.implode(' AND ', $where);
        // ORDER BY
        if (isset($this -> params['order']))
            $this -> sql .= " ORDER BY {$this -> params['order']}";
        // ORDER BY RAND()
        if (isset($this -> params['random']))
            $this -> sql .= " ORDER BY RAND()";
        // LIMIT
        if (isset($this -> params['limit']))
            $this -> sql .= " LIMIT {$this -> params['limit']}";
    }

    /*
     *  \brief Преобразует массив в строку с разделителями и оберткой =)
     */
    private static function arr2mstr($arr, $delimiter = ",", $wrap = "'") {
        $res = array();
        foreach ($arr as $val)
            $res[] = "$wrap$val$wrap";
        return implode($delimiter, $res);
    }

    /*
     *  \brief Выполняет запрос (dbApi)
     */
    public static function query($sql) {
        if (!is_null(self::$db))
            return self::$db -> query($sql);
        return false;
    }

    /*
     *  \brief Выполняет запрос и формирует объект с результатом (dbApi)
     */
    public static function fetch($sql) {
        if (!is_null(self::$db))
            return self::$db -> fetch($sql);
        return false;
    }

    /*
     *  \brief Выполняет запрос на добавление (dbApi)
     */
    public static function insert($sql) {
        if (!is_null(self::$db))
            return self::$db -> insert($sql);
        return false;
    }

}

```