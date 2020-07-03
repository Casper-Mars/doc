# 基础模板项目

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
[t_dictionary_entity]:./imgs/t_dictionary.png

