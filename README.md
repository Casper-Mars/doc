# 基础模板项目

## 实体模型

![avatar][database_model]

[database_model]:./imgs/database.png


## 字典模块

### 模块说明

现阶段的设计是为了权限模块的配置，没有兼容业务需求。字典分了两个业务概念：

* 字典的索引
* 字典索引的明细

### 使用

出完接口后(不一定要实现)，需要手动对接口进行分组，每一组建立一个索引，每一组的接口就是索引的明细。新建菜单时，就根据索引查询明细，获取该菜单所有的操作。

### 目的

* 分离菜单的定义和创建。由相关人员对特定的菜单功能进行定义设计，由用户决定实际需要哪些功能。
* 适配数据权限的设计。对数据做权限控制的前提是对操作进行拦截，需要精确推断出用户所执行的操作是否有权限。因此，权限的设计需要精确到操作，而不仅仅是菜单。

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


