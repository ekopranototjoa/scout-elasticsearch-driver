# Scout Elasticsearch Driver

[![Packagist](https://img.shields.io/packagist/v/babenkoivan/scout-elasticsearch-driver.svg)](https://packagist.org/packages/babenkoivan/scout-elasticsearch-driver)
[![Packagist](https://img.shields.io/packagist/dt/babenkoivan/scout-elasticsearch-driver.svg)](https://packagist.org/packages/babenkoivan/scout-elasticsearch-driver)
[![Gitter](https://img.shields.io/gitter/room/nwjs/nw.js.svg)](https://gitter.im/scout-elasticsearch-driver/Lobby)
[![Donate](https://img.shields.io/badge/donate-PayPal-blue.svg)](https://www.paypal.me/ivanbabenko)

:coffee: If you like my package, it'd be nice of you [to buy me a cup of coffee](https://www.paypal.me/ivanbabenko).
 
:octocat: The project has a [chat room on Gitter](https://gitter.im/scout-elasticsearch-driver/Lobby)!

---

This package offers advanced functionality for searching and filtering data in Elasticsearch.
Check out its [features](#features)!

## Contents

* [Tutorial](#tutorial)
* [Features](#features)
* [Requirements](#requirements)
* [Installation](#installation)
* [Configuration](#configuration)
* [Index configurator](#index-configurator)
* [Searchable model](#searchable-model)
* [Usage](#usage)
* [Console commands](#console-commands)
* [Search rules](#search-rules)
* [Available filters](#available-filters)
* [Zero downtime migration](#zero-downtime-migration)
* [Debug](#debug)

## Tutorial

For your convenience I wrote step-by-step tutorial - [How to make Laravel and Elasticsearch become friends](https://medium.com/@babenko.i.a/how-to-make-laravel-and-elasticsearch-become-friends-55ed7690331c). 
There are information about Elasticsearch installation and the package usage examples, don't miss it!    

## Features

* An easy way to [configure](#index-configurator) and [create](#console-commands) an Elasticsearch index.
* A fully configurable mapping for each [model](#searchable-model).
* A possibility to add a new field to an existing mapping [automatically](#configuration) or using [the artisan command](#console-commands).
* Lots of different ways to implement your search algorithm: using [search rules](#search-rules) or a [raw search](#usage).
* [Various filter types](#available-filters) to make a search query more specific.
* [Zero downtime migration](#zero-downtime-migration) from an old index to a new index.
* Bulk indexing, see [the configuration section](#configuration).

## Requirements

The package has been tested in the following configuration: 

* PHP version &gt;= 7.0
* Laravel Framework version &gt;= 5.5
* Elasticsearch version &gt;= 5.5

## Installation

Use composer to install the package:

```
composer require babenkoivan/scout-elasticsearch-driver
```

## Configuration

To configure the package you need to publish settings first:

```
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
php artisan vendor:publish --provider="ScoutElastic\ScoutElasticServiceProvider"
```

Then, set the driver setting to `elastic` in the `config/scout.php` file and configure the driver itself in the `config/scout_elastic.php` file.
The available options are:

Option | Description
--- | ---
client | A setting hash to build Elasticsearch client. More information you can find [here](https://www.elastic.co/guide/en/elasticsearch/client/php-api/current/_configuration.html#_building_the_client_from_a_configuration_hash). By default the host is set to `localhost:9200`.
update_mapping | The option that specifies whether to update a mapping automatically or not. By default it is set to `true`.
indexer | Set to `single` for the single document indexing and to `bulk` for the bulk document indexing. By default is set to `single`.

Note, that if you use the bulk document indexing you'll probably want to change the chunk size, you can do that in the `config/scout.php` file.

## Index configurator

An index configurator class is used to set up settings for an Elasticsearch index.
To create a new index configurator use the following artisan command:

```
php artisan make:index-configurator MyIndexConfigurator
```

It'll create the file `MyIndexConfigurator.php` in the `app` folder of your project. 
You can specify index name, settings and default mapping like in the following example:

```php
<?php

namespace App;

use ScoutElastic\IndexConfigurator;

class MyIndexConfigurator extends IndexConfigurator
{
    // It's not obligatory to determine name. By default it'll be a snaked class name without `IndexConfigurator` part.
    protected $name = 'my_index';  
    
    // You can specify any settings you want, for example, analyzers. 
    protected $settings = [
        'analysis' => [
            'analyzer' => [
                'es_std' => [
                    'type' => 'standard',
                    'stopwords' => '_spanish_'
                ]
            ]    
        ]
    ];

    // Common mapping for all types.
    protected $defaultMapping = [
        '_all' => [
            'enabled' => true
        ],
        'dynamic_templates' => [
            [
                'es' => [
                    'match' => '*_es',
                    'match_mapping_type' => 'string',
                    'mapping' => [
                        'type' => 'string',
                        'analyzer' => 'es_std'
                    ]
                ]    
            ]
        ]
    ];
}
```

More about index settings and default mapping you can find in the [index management section](https://www.elastic.co/guide/en/elasticsearch/guide/current/index-management.html) of Elasticsearch documentation.

To create an index just run the artisan command:
 
```
php artisan elastic:create-index App\\MyIndexConfigurator
```

## Searchable model

To create a model with the ability to perform search requests in an Elasticsearch index use the command:

```
php artisan make:searchable-model MyModel --index-configurator=MyIndexConfigurator
``` 

After executing the command you'll find the file `MyModel.php` in you `app` folder:

```php
<?php

namespace App;

use ScoutElastic\Searchable;
use Illuminate\Database\Eloquent\Model;

class MyModel extends Model
{
    use Searchable;

    protected $indexConfigurator = MyIndexConfigurator::class;

    protected $searchRules = [
        //
    ];

    // Here you can specify a mapping for a model fields.
    protected $mapping = [
        'properties' => [
            'text' => [
                'type' => 'string',
                'fields' => [
                    'raw' => [
                        'type' => 'string',
                        'index' => 'not_analyzed',
                    ]
                ]
            ],
        ]
    ];
}
```

Each searchable model represents an Elasticsearch type.
By default a type name is the same as a table name, but you can set any type name you want through the `searchableAs` method.
You can also specify fields which will be indexed by the driver through the `toSearchableArray` method.
More information about these options you will find in [the scout official documentation](https://laravel.com/docs/5.5/scout#configuration).

The last important option you can set in the `MyModel` class is the `$searchRules` property. 
It allows you to set different search algorithms for a model. 
We'll take a closer look at it in [the search rules section](#search-rules).

After setting up a mapping in your model you can update an Elasticsearch type mapping:

```
php artisan elastic:update-mapping App\\MyModel
```

## Usage

Once you've created an index configurator, an Elasticsearch index itself and a searchable model, you are ready to go.
Now you can [index](https://laravel.com/docs/5.5/scout#indexing) and [search](https://laravel.com/docs/5.5/scout#searching) data according to the documentation.

Basic search usage example:

```php
App\MyModel::search('phone') 
    ->where('color', 'red')
    ->orderBy('price', 'asc')
    ->take(10)
    ->get();
```

If you need to load relations you can use the `with` method:

```php
App\MyModel::search('phone') 
    ->with('makers')
    ->get();
```

In addition to standard functionality the package offers you the possibility to filter data in Elasticsearch without specifying a query string:
  
```php
App\MyModel::search('*')
    ->where('id', 1)
    ->get();
```

Also you can override model [search rules](#search-rules):

```php
App\MyModel::search('Brazil')
    ->rule(App\MySearchRule::class)
    ->get();
```

And use [variety](#available-filters) of `where` conditions: 

```php
App\MyModel::search('*')
    ->whereRegexp('name.raw', 'A.+')
    ->where('age', '>=', 30)
    ->whereExists('unemployed')
    ->get();
```

At last, if you want to send a custom request, you can use the `searchRaw` method:

```php
App\MyModel::searchRaw([
    'query' => [
        'bool' => [
            'must' => [
                'match' => [
                    '_all' => 'Brazil'
                ]
            ]
        ]
    ]
]);
```

This query will return raw response.

## Console commands

Available artisan commands are listed below:

Command | Arguments | Description
--- | --- | ---
make:index-configurator | `name` - The name of the class | Creates a new Elasticsearch index configurator.
make:searchable-model | `name` - The name of the class | Creates a new searchable model.
make:search-rule | `name` - The name of the class | Creates a new search rule.
elastic:create-index | `index-configurator` - The index configurator class | Creates an Elasticsearch index.
elastic:update-index | `index-configurator` - The index configurator class | Updates settings and mappings of an Elasticsearch index.
elastic:drop-index | `index-configurator` - The index configurator class | Drops an Elasticsearch index.
elastic:update-mapping | `model` - The model class | Updates a model mapping.
elastic:migrate | `model` - The model class, `target-index` - The index name to migrate | Migrates model to another index.

For detailed description and all available options run `php artisan help [command]` in the command line.

## Search rules

A search rule is a class that describes how a search query will be executed. 
To create a search rule use the command:

```
php artisan make:search-rule MySearchRule
```

In the file `app/MySearchRule.php` you will find a class definition:

```php
<?php

namespace App;

use ScoutElastic\SearchRule;

class MySearch extends SearchRule
{
    // This method returns an array that represents a content of bool query.
    public function buildQueryPayload()
    {
        return [
            'must' => [
                'match' => [
                    'name' => $this->builder->query
                ]
            ]
        ];
    }
}
```

You can read more about bool queries [here](https://www.elastic.co/guide/en/elasticsearch/reference/5.2/query-dsl-bool-query.html).

The default search rule returns the following payload:

```php
return [
   'must' => [
       'match' => [
           '_all' => $this->builder->query
       ]
   ]
];
```

This means that by default when you call `search` method on a model it tries to find the query string in any field.

To determine default search rules for a model just add a property:

```php
<?php

namespace App;

use ScoutElastic\Searchable;
use Illuminate\Database\Eloquent\Model;

class MyModel extends Model
{
    use Searchable;
    
    // You can set several rules for one model. In this case, the first not empty result will be returned.
    protected $searchRules = [
        MySearchRule::class
    ];
}
```

You can also set a search rule in a query builder:

```php
// You can set either a SearchRule class
App\MyModel::search('Brazil')
    ->rule(App\MySearchRule::class)
    ->get();
    
// or a callable
App\MyModel::search('Brazil')
    ->rule(function($builder) {
        return [
            'must' => [
                'match' => [
                    'Country' => $builder->query
                ]
            ]
        ];
    })
    ->get();
```

## Available filters

You can use different types of filters:

Method | Example | Description
--- | --- | ---
where($field, $value) | where('id', 1) | Checks equality to a simple value.
where($field, $operator, $value) | where('id', '>=', 1) | Filters records according to a given rule. Available operators are: =, <, >, <=, >=, <>.    
whereIn($field, $value) | where('id', [1, 2, 3]) | Checks if a value is in a set of values. 
whereNotIn($field, $value) | whereNotIn('id', [1, 2, 3]) | Checks if a value isn't in a set of values. 
whereBetween($field, $value) | whereBetween('price', [100, 200]) | Checks if a value is in a range.
whereNotBetween($field, $value) | whereNotBetween('price', [100, 200]) | Checks if a value isn't in a range.
whereExists($field) | whereExists('unemployed') | Checks if a value is defined.
whereNotExists($field) | whereNotExists('unemployed') | Checks if a value isn't defined.  
whereRegexp($field, $value, $flags = 'ALL') | whereRegexp('name.raw', 'A.+') | Filters records according to a given regular expression. [Here](https://www.elastic.co/guide/en/elasticsearch/reference/5.2/query-dsl-regexp-query.html#regexp-syntax) you can find more about syntax.

In most cases it's better to use raw fields to filter records, i.e. not analyzed fields.

## Zero downtime migration

As you might know, you can't change the type of already created field in Elasticsearch. 
The only choice in such case is to create a new index with necessary mapping and import your models into the new index.      
A migration can take quite a long time, so to avoid downtime during the process the driver reads from the old index and writes to the new one.
As soon as migration is over it starts reading from the new index and removes the old index.
This is how the artisan `elastic:migrate` command works.  

Before you run the command, make sure that your index configurator uses the `ScoutElastic\Migratable` trait.
If it's not, add the trait and run the artisan `elastic:update-index` command using your index configurator class name as an argument:

```
php artisan elastic:update-index App\\MyIndexConfigurator
```

When you are ready, make changes in the model mapping and run the `elastic:migrate` command using the model class as the first argument and desired index name as the second argument:

```
php artisan elastic:migrate App\\MyModel my_index_v2
``` 

Note, that if you need just to add new fields in your mapping, use the `elastic:update-mapping` command.

## Debug

There are two methods that can help you to analyze results of a search query:

* [explain](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-explain.html)
 
    ```php
    App\MyModel::search('Brazil')
        ->explain();
    ```
    
* [profile](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-profile.html)

    ```php
    App\MyModel::search('Brazil')
        ->profile();
    ```
    
Both methods return raw data from ES.

Besides, you can get a query payload that will be sent to ES, by calling the `buildPayload` method.

```php
App\MyModel::search('Brazil')
    ->buildPayload();
```

Note, that this method returns a collection of payloads, because of possibility of using multiple search rules in one query. 
