---
title: odoo模块加载机制
tags:
  - odoo
  - python
abbrlink: 51a2a1bd
date: 2019-11-08 20:36:34
---

**Odoo的启动通过openerp-server脚本完成，它是系统的入口。**

**然后加载配置文件openerp-server.conf 或者 openerp_serverrc；**

openerp-server.conf的主要内容：

这个文件缺省是没有的，Odoo系统会有一个默认值，但是一般情况我们都需配置这个文件。

启动http服务器，监听端口。

**模块加载：**

模块加载外层就是封装一个Registry(Mapping)对象:实际是一个字典，它包含对应的db，model等映射关系，一个DB对应一个Registry。后续的操作都会围绕这个Registry进行，将相关的数据赋值给相应的属性项。

## 初始化数据库（初次运行)

**1)加载base模块下的base.sql文件并执行。**
此时数据库表为：

```sql
CREATE TABLE ir_actions (
  id serial,
  primary key(id)
);
CREATE TABLE ir_act_window (primary key(id)) INHERITS (ir_actions);
CREATE TABLE ir_act_report_xml (primary key(id)) INHERITS (ir_actions);
CREATE TABLE ir_act_url (primary key(id)) INHERITS (ir_actions);
CREATE TABLE ir_act_server (primary key(id)) INHERITS (ir_actions);
CREATE TABLE ir_act_client (primary key(id)) INHERITS (ir_actions);


CREATE TABLE ir_model (
  id serial,
  model varchar NOT NULL,
  name varchar,
  state varchar,
  info text,
  primary key(id)
);

CREATE TABLE ir_model_fields (
  id serial,
  model varchar NOT NULL,
  model_id integer references ir_model on delete cascade,
  name varchar NOT NULL,
  relation varchar,
  select_level varchar,
  field_description varchar,
  ttype varchar,
  state varchar default 'base',
  relation_field varchar,
  translate boolean default False,
  serialization_field_id integer references ir_model_fields on delete cascade, 
  primary key(id)
);

CREATE TABLE res_lang (
    id serial,
    name VARCHAR(64) NOT NULL UNIQUE,
    code VARCHAR(16) NOT NULL UNIQUE,
    primary key(id)
);

CREATE TABLE res_users (
    id serial NOT NULL,
    active boolean default True,
    login varchar(64) NOT NULL UNIQUE,
    password varchar(64) default null,
    -- No FK references below, will be added later by ORM
    -- (when the destination rows exist)
    company_id integer, -- references res_company,
    partner_id integer, -- references res_partner,
    primary key(id)
);

create table wkf (
    id serial,
    name varchar(64),
    osv varchar(64),
    on_create bool default false,
    primary key(id)
);

CREATE TABLE ir_module_category (
    id serial NOT NULL,
    create_uid integer, -- references res_users on delete set null,
    create_date timestamp without time zone,
    write_date timestamp without time zone,
    write_uid integer, -- references res_users on delete set null,
    parent_id integer REFERENCES ir_module_category ON DELETE SET NULL,
    name character varying(128) NOT NULL,
    primary key(id)
);

CREATE TABLE ir_module_module (
    id serial NOT NULL,
    create_uid integer, -- references res_users on delete set null,
    create_date timestamp without time zone,
    write_date timestamp without time zone,
    write_uid integer, -- references res_users on delete set null,
    website character varying(256),
    summary character varying(256),
    name character varying(128) NOT NULL,
    author character varying(128),
    icon varchar,
    state character varying(16),
    latest_version character varying(64),
    shortdesc character varying(256),
    category_id integer REFERENCES ir_module_category ON DELETE SET NULL,
    description text,
    application boolean default False,
    demo boolean default False,
    web boolean DEFAULT FALSE,
    license character varying(32),
    sequence integer DEFAULT 100,
    auto_install boolean default False,
    primary key(id)
);
ALTER TABLE ir_module_module add constraint name_uniq unique (name);

CREATE TABLE ir_module_module_dependency (
    id serial NOT NULL,
    create_uid integer, -- references res_users on delete set null,
    create_date timestamp without time zone,
    write_date timestamp without time zone,
    write_uid integer, -- references res_users on delete set null,
    name character varying(128),
    module_id integer REFERENCES ir_module_module ON DELETE cascade,
    primary key(id)
);

CREATE TABLE ir_model_data (
    id serial NOT NULL,
    create_uid integer,
    create_date timestamp without time zone,
    write_date timestamp without time zone,
    write_uid integer,
    noupdate boolean,
    name varchar NOT NULL,
    date_init timestamp without time zone,
    date_update timestamp without time zone,
    module varchar NOT NULL,
    model varchar NOT NULL,
    res_id integer,
    primary key(id)
);

-- Records foreign keys and constraints installed by a module (so they can be
-- removed when the module is uninstalled):
--   - for a foreign key: type is 'f',
--   - for a constraint: type is 'u' (this is the convention PostgreSQL uses).
CREATE TABLE ir_model_constraint (
    id serial NOT NULL,
    date_init timestamp without time zone,
    date_update timestamp without time zone,
    module integer NOT NULL references ir_module_module on delete restrict,
    model integer NOT NULL references ir_model on delete restrict,
    type character varying(1) NOT NULL,
    name varchar NOT NULL,
    primary key(id)
);

-- Records relation tables (i.e. implementing many2many) installed by a module
-- (so they can be removed when the module is uninstalled).
CREATE TABLE ir_model_relation (
    id serial NOT NULL,
    date_init timestamp without time zone,
    date_update timestamp without time zone,
    module integer NOT NULL references ir_module_module on delete restrict,
    model integer NOT NULL references ir_model on delete restrict,
    name varchar NOT NULL,
    primary key(id)
);  

CREATE TABLE res_currency (
    id serial,
    name varchar NOT NULL,
    primary key(id)
);

CREATE TABLE res_company (
    id serial,
    name varchar NOT NULL,
    partner_id integer,
    currency_id integer,
    primary key(id)
);

CREATE TABLE res_partner (
    id serial,
    name varchar,
    company_id integer,
    primary key(id)
);

```

这20张表是odoo系统级的，它是模块加载及系统运行的基础。后续模块生成的表及相关数据都可以在这20张中找到蛛丝马迹。
## 数据库表初始化后，就可以加载模块数据（addons）到数据库了，这个也是odoo作为平台灵活的原因，所有的数据都在数据库。
找到addons-path下所有的模块,然后一个一个的加载到数据库中。
Info就是load模块的__openerp__.py文件，它是一个dict。

根据__openerp__.py中定义的category创建分类信息：
将模块信息写入ir_module_module表：
将module信息写入ir_model_data表：
一个module要写两次ir_model_data表，
写module的dependency表：

根据依赖关系进行判断，递归更新那些需要auto_install的模块状态为“to install”。


到目前为止，模块的加载都是在数据库级别，只是将“模块文件”信息存入数据库表，但是还没有真正加载到程序中。
Odoo运行时查找object是通过Registry.get()获取的，而不是通过python自己的机制来找到相应的object，所以odoo在加载模块时会把模块下包含的model全部注册到models.py的module_to_models字典中。

**下面的步骤就是加载模块到内存：

## 加载base模块

创建一个包含model层级的节点图，第二行代码将从数据库更新数据到graph中。然后调用load_module_graph方法加载模块，最终执行加载的方法：

这个方法是odoo加载model的核心，通过 __import__方法加载模块，这个是python的机制，当import到某个继承了BaseModel类的class时，它的实例化将有别于python自身的实例化操作，
后者说它根本不会通过python自身的__new__方法创建实例，所有的实例创建都是通过 _build_model 方法及元类创建，并注册到module_to_models中。通过这种方式实例化model就可以解决我们在xml中配置model时指定的继承，字段，约束等各种属性。

## 标记需要加载或者更新的模块（db）

## 加载被标记的模块（加载过程与加载base模块一致）

##  完成及清理安装

## 清理菜单

## 删除卸载的模块

## 核实model的view

## 运行post-install测试