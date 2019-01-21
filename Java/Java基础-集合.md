#### Tava基础-集合

##### 1.思维导图

##### 2. 集合定义

##### 	2.1 List

##### 	2.2 Set 

##### 	2.3 Queue

##### 	2.4 Map

##### 思维导图

![](https://raw.githubusercontent.com/ljingya/LearningNotes/master/Image/Java%E9%9B%86%E5%90%88.jpg)

#####  集合定义

######   	2.1 Lsit

​	有序的，可重复的集合，集合中每个元素有都有对应的索引。

###### 	2.2  Set

​	Set集合不包含，相同的元素。

###### 	2.3 Queue

​	队列是一种先进先出的容器，队列头部是存储最久的元素。

​	

| 方法      | 返回值  | 说明                                                         |
| --------- | ------- | ------------------------------------------------------------ |
| add()     | boolean | 插入成功返回true,当空间不足时报IllegralStateException。      |
| element() | T       | 返回队列头部数据。但不移除，当队列为空时，报NoSuchElementException错误 |
| offer()   | boolean | 插入数据，返回true，当空间不足时无法插入不会报错。           |
| peek()    | T       | 返回头部数据，但不移除，当队列为null时，返回null             |
| poll()    | T       | 返回队列头部数据，并移除头部，队列为空时，返回null、         |
| remove()  | T       | 返回队列头部数据，并删除头部，队列为空时，报NoSuchElementException错误 |
|           |         |                                                              |

###### 	2.4 Map

Key不允许重复，Value允许重复。一组key可以看做Set，一组Value可以看做是List。

​		