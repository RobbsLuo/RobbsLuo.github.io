---
title: Introduction to Rails Caching
date: 2016-11-18 16:32:38
tags:
 - Ruby
 - Cache
 - Rails
categories: Rails
description: A brief introduction to the caching mechanisms commonly used in Rails.
lang: en
---

## Introduction

Reasonable use of caching can greatly improve a website's performance, making it an indispensable part of performance optimization.

## Caching

*    Model-layer caching

     Through manual configuration, you can store selected query results in the corresponding cache system.

     ```ruby
     Rails.cache.fetch('all_products', expires_in: 1.days) do 
       Product.all.to_a
     end
     ```

     **Make sure to verify that the returned result is the final result set.**

- #### Controller-layer caching

  - ##### Action caching

    Caches the Action response, implemented with the help of Fragment Cache and callbacks.

    Various verification mechanisms can be added via `before_action`.

    ```
    before_action :authentication, only: :show
    cache_action :show, expires_in: 1.hour
    ```

    **This method was removed in Rails 4, but it can be re-enabled through a Gem.**

  - ##### Page caching

    Caches the page as a static page; it cannot perform authentication or similar checks.

    ```
    class ProductsController < ActionController
      caches_page :index
    end
    ```

    **This method was removed in Rails 4, but it can be re-enabled through a Gem.**

- #### View-layer caching

  - ##### Fragment Cache

    As a page becomes more complex, full-page caching is no longer feasible; the page must instead be split into different fragments.

    ```
    - cache('xxx' , expires_in: 1.days ) do 
      %ul
        = @product.name
    ```

    **Fragment caching can be nested to form a special pattern known as Russian Doll Caching.**

- #### SQL caching

    This is a built-in feature of the Rails framework; it caches the result set of each query.

    ```
    class ProductsController < ActionController
      def index
        # First Query
        @products = Product.all
      
        # Second Query (Cache)
        @products = Product.all
      end
    ```

    When the second query runs, it reads the result set cached in memory by the first query directly.

    **Tips: The cache is valid for the lifetime of the action.**

## Configuration

```bash
 # Enable caching
 config.action_controller.perform_caching = true
 # Cache store backend
 config.cache_store = :memory_store  # memory_store mem_cache_store file_store
```

