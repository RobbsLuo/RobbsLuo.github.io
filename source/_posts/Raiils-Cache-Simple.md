---
title: Rails 缓存简介
date: 2016-11-18 16:32:38
tags:
 - Ruby
 - Cache
 - Rails
categories: Rails
description: 简单的介绍 Rails 中常用的缓存机制
---

## 前言

合理的使用缓存可以很大程度上提高网站性能，是网站性能优化必不可少的一部分。

## 缓存

*    Model 层缓存

     通过手动设置可以将部分查询结果存储在对应的缓存系统中。

     ```ruby
     Rails.cache.fetch('all_products', expires_in: 1.days) do 
       Product.all.to_a
     end
     ```

     **需要确认执行结果是否为最终的结果集**

- #### Controller 层缓存

  - ##### Action 缓存

    缓存 Action Response，借助 Fragement Cache 和 Callback 实现。

    可通过 before_action 加入各种验证机制。

    ```
    before_action :authentication, only: :show
    cache_action :show, expires_in: 1.hour
    ```

    **该方法在 Rails4 已经移除了，可通过Gem包开启**

  - ##### Page 缓存 (页面缓存)

    将页面缓存成静态页面，无法进行权限认证等。

    ```
    class ProductsController < ActionController
      caches_page :index
    end
    ```

    **该方法在 Rails4 已经移除了，可通过Gem包开启**

- #### View 层缓存

  - ##### Fragement Cache (片段缓存)

    随着页面复杂的程度的提高，已经无法做整页缓存，只能将其切割成不同的片段。

    ```
    - cache('xxx' , expires_in: 1.days ) do 
      %ul
        = @product.name
    ```

    **片段缓存可以通过嵌套使用的方式形成特殊的俄罗斯套娃(Russian Doll Caching)**

- #### SQL 缓存

  这是 Rails 框架自带的一个特性，会缓存每一次查询的结果集。

  ```
  class ProductsController < ActionController
    def index
      # First Query
      @products = Product.all
      
      # Second Query (Cache)
      @products = Product.all
    end
  ```

  当第二次查询时，会直接从内存中读取第一次查询缓存入内存的结果集。

  **Tips: 缓存的有效时间是 action 的生命周期**

## 配置

```bash
 # 开启缓存
 config.action_controller.perform_caching = true
 # 缓存存储方式
 config.cache_store = :memory_store  # memory_store mem_cache_store file_store
```

