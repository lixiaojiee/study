1、关于mysql中的`limit`和`offset`

- **limit**

  mysql中的`limit`语法可以限制所要查询记录的笔数，使用sql的`limit`语法时，通常会伴随着`order by`，如果没有指定`order by`，那么记录的排序就以mysql的默认值为准，这样有可能会导致查询到的记录不正确，因此建议在使用`limit`时，最好加上`order by`，以确保结果的正确性

  limit语法有两种情况：

  **a、limit n**

  ```mysql
  select * from table order by id desc limit 1
  ```

  语法说明：`limit`后面直接加一个整数，代表限制查询到的记录数，如`limit 5`，就代表需要查询5条记录，如果不够五条，则有几条返回几条

  **b、limit index, count**

  ```mysql
  select * from table order by id desc limit 5,2
  ```

  `limit`后面接`[index,count]`,例如上面sql中的`limit 5,2`，代表：

  **index:**从哪个`index`开始查询，`index`从0开始计算，上面语句中的5表示开始从第6条记录开始查询

  **count:**由index开始，总共需要查询的记录数，从上边的sql可以看出，最终可能查询出第6、7两条记录

- **offset**

- ```mysql
  select * from table order by id desc limit 2 offset 4
  ```

  有些关系型数据库不支持`[offset,count]`，必须要用`offset`来取代，我们可以把`offset`想成需要略过的笔数，以offset=2为例，便可想成在找到的记录笔数中略过前两笔，即从第五笔开始计算，返回限制的两条记录

