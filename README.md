# 基础模板项目

## 实体模型

> 具体的说明参阅数据库设计文档

![avatar][database_model]

[database_model]:./imgs/database.png


## 字典模块

### 模块说明

字典分了两个业务概念：

* 字典的索引
* 字典索引的明细

### 使用

字典的索引在数据库手动填写添加，添加后的索引可以在管理页面上添加其明细。

### 实体设计

业务上区分了两种概念：索引和索引的明细。在数据层面上，只有一种概念：索引。
索引是一种树状结构的数据，有父子的继承性质。
实体数据表：

![avatar][t_dictionary_entity]

[t_dictionary_entity]: ./imgs/t_dictonary.png

## 数据权限控制模块

### 模块说明

数据的权限关联到角色上，每个角色有一定的数据访问范围：本人，本组织，指定用户。

### 实体设计

实体数据表：

![avatar][t_role_databind_entity]

[t_role_databind_entity]:./imgs/t_role_databind.png

### 核心逻辑

* 第一阶段-创建角色

> 在创建角色时，需要定义好角色的数据访问范围。默认是本人的范围，并且必须有本人的范围。选择其余两种范围也会默认创建本人的范围。

* 第二阶段-拦截控制

> 在创建角色的时候，会绑定一些权限，其中包括菜单浏览权限和功能操作权限。在用户发起操作请求时，对请求进行拦截。结合用户的角色集合，推断操作权限所属的角色。
> 得到角色后，就可以查询到该角色的数据访问范围，并传递给业务操作。

## 用户管理模块

### 模块说明

* 管理用户的信息，提供手动添加或者批量导出导入的操作。
* 维护用户的数据访问控制信息。在添加用户的时候，会根据角色的数据访问信息和部门信息产生用户的数据访问控制信息。

### 核心逻辑

* 第一阶段-创建用户

* 第二阶段-关联用户和组织

* 第三阶段-关联用户和角色

* 第四阶段-产生数据访问控制信息

> 遍历用户的角色列表，对每个角色逐一处理。
>判断角色的数据访问范围类型，不同的类型采取不同处理策略。
>> 本人类型的:使用用户的id作为数据源id，添加一条本人类型的数据绑定记录。
>> 本组织类型的:遍历用户的组织，每一个组织都使用组织id作为数据源id添加本组织类型的数据绑定记录。
>> 指定人员类型的:和组织类似，不同的是遍历指定的人员而不是组织。

## 模板管理模块

### 模块说明

提供一个管理系统全局导入的Excel模板的功能。用户可以选择需要导入的业务数据表并自定义需要导入的字段。

### 实体模型

![avatar][import_excel_entity]

[import_excel_entity]:./imgs/ImportExcelEntity.png

### 持久化方式

此模块相对比较特殊，不能存进数据库，只能存储到本地文件。文件名作为主键id。数据的列表检索相当于是文件系统目录的展开，数据的详细信息检索相当于是文件的读取。
不支持事务管理。


### 核心逻辑

* 添加模板

> 把接收到的信息创建实体类对象，然后json格式化该对象并写入到文件中。文件名称使用id生成器产生，文件存储在指定的路径。

* 查询列表

> 先检查缓存的id列表和存储目录的文件名列表是否相同。相同则直接查询缓存的。不相同时，需要把缓存缺少的加入到缓存中，再查询缓存的信息。
> 查询以外的操作需要保证一个不变性：目录的文件名列表永远包含缓存的id列表，即id列表的id(文件名)永远存在于目录的文件名列表。
> 此模块的列表查询不再分页，在模块启动的时候，把所有的文件读取并封装成对象后，文件名作为key存在本地缓存中。查询列表就返回缓存的信息。

* 修改模板

> 修改模板需要删除缓存的信息和文件的信息。如果需要负载均衡，则需要中间件同步缓存数据。高并发下要使用锁对编辑的文件进行加锁。

* 下载模板

> 下载的模板是动态生成的。在下载操作的时候，再根据模板的定义信息生产excel文件。

* 数据导入

> 使用者按照导出的模块填写好数据后，上传文件后，判断是使用哪一个模板的，读取该模板的信息，并把数据导入到对应的数据库。

### 导入导出技术

#### 导入技术

##### 性能需求

至少能支持十万级的excel数据导入。

##### 分析

常规的数据量少的导入方式：把文件加载到内存后，进行excel格式解析，并把解析出来的数据添加到数据库或者其他数据源。
常规的方式需要把整个文件加载到内存中，目的为了对整个excel文件进行解析。在excel文件较大小的时候，需要解析的数据量较少，解析速度较快，主要是用年轻代堆内存。
随着数据量的增加，解析的速度逐渐降低，可能会导致许多本来朝生夕死的对象进入老年代堆，占用了老年代的空间直到FGC的到来。这种方式的浪费内存并有可能会导致内存溢出。

经过调研后，决定使用分步读取并导入的方式，并限定只操作07版的excel文件。分步读取文件内容到内存，可以减轻内存的压力并更好的发挥io的性能。

##### 主体思想

* 分步读取excel。
* 解析读取到的excel数据
* 缓存批量插入数据库

##### 技术细节

为什么只能使用07版的excel文件格式？因为这种格式是xml格式，之前的旧版是独有的二进制格式。对于xml格式的文件，可以实现按节点处理。

* 使用sax读取excel文件。
> sax是事件驱动的xml解析框架，当读到xml的一个节点时会触发相应的事件。07版的excel文件本质上是一个压缩包，打包了相应的数据文件，这些文件都是xml格式的。因此，利用sax事件驱动的特性，就可以分步读取excel文件，而不需要一次性读取到内存中。
* 使用disruptor高速队列。
> Disruptor框架是一个本地的内存式环形队列，速度快且占用的内存少。使用这个框架是为了把读取解析和导入的逻辑解耦。现阶段还未做到利用多核cpu的优势进行多线程读取excel文件，但是起码可以做到多线程添加到数据库中。

##### 导入引擎相关类图

![avatar][template_import_model]

[template_import_model]:./imgs/template_import_model.png

##### 导入流程

前端先上传文件，得到文件的地址后，用地址作为参数开启websocket链接。简要的流程图如下：

![avatar][import_process]

[import_process]:./imgs/importExcelProcess.png





