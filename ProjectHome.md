## Recursive Query Generator (RQG) ##

php класс для автоматического составления SQL запросов на основе построеной "цепочки" параметров, выполнения запросов через подключаемый драйвер БД.

Пример использования:

```
   // Добавление
   RQG::create() -> temp('key', 'value') -> add($key, $value);
   // Кооличество записей
   RQG::create() -> temp -> count;
   // Выборка одного значения
   RQG::create() -> temp -> limit(1) -> one;
   // Выборка с WHERE условием
   RQG::create() -> temp('value') -> where("key = 'DOCUMENT_ROOT'") -> one -> value;
   // Обновление данных
   RQG::create() -> temp('value') -> where("key = 'value'") -> update('new_value');
   // Удаление
   RQG::create() -> temp -> delete;
```
Для ознакомления с проектом посмотрите Readme в вики.