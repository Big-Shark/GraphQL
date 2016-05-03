# GraphQL
[![Build Status](https://travis-ci.org/Youshido/GraphQL.svg?branch=master)](http://travis-ci.org/Youshido/GraphQL)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/Youshido/GraphQL/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/Youshido/GraphQL/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/Youshido/GraphQL/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/Youshido/GraphQL/?branch=master)
[![SensioLabsInsight](https://insight.sensiolabs.com/projects/8b8ab2a2-32fb-4298-a986-b75ca523c7c9/mini.png)](https://insight.sensiolabs.com/projects/8b8ab2a2-32fb-4298-a986-b75ca523c7c9)

This is a clean PHP realization of the GraphQL protocol based on the working draft of the specification located on https://github.com/facebook/graphql.

GraphQL is a modern replacement of the REST API approach. It advanced in many ways and has fundamental improvements over the old not-so-good REST:

 - self-checks embedded on the ground level of your backend architecture  
 - reusable API for different client versions and devices – no need in "/v1" and "/v2" anymore
 - a complete new level of distinguishing the backend and the frontend logic
 - easily generated documentation and incredibly easy way to explore API for other developers
 - once your architecture is complete – simple changes on the client does not require you to change API

It could be hard to believe, but give it a try and you'll be rewarded with much better architecture and so much easier to support code.

*Current package is and will be trying to be kept up with the latest revision of the GraphQL specification which is now of April 2016.*



## Table of Contents

* [Getting Started](#getting-started)
* [Installation](#installation)
* [Example – Creating Blog Schema](#example---creating-blog-schema)
  * [Inline approach](#inline-approach)
  * [Object Oriented approach](#object-oriented)
* [Define your Schema](#define-your-schema)
* [Query Documents](#query-documents)
* [Type System](#type-system)
  * [Scalar Types](#scalar-types)
  * [Objects](#objects)
  * [Interfaces](#interfaces)
  * [Unions](#unions)
  * [Enums](#enums)
  * [Input Objects](#input-objects)
  * [Lists](#list)
  * [Non-Null](#non-null)
* [Mutation structure](#mutation-structure)
* [Schema validation](#schema-validation)

## Getting Started

You should be better off starting with some examples, and "Star Wars" become a somewhat "Hello world" example for the GraphQL frameworks.
We have that too and if you looking for just that – go directly by this link – [Star Wars example](https://github.com/Youshido/GraphQL/Tests/StarWars).
On the other hand based on the feedback we prepared a step-by-step for those who want to get up to speed fast.

### Installation

You should simply install this package using the composer. If you're not familiar with it you should check out the [manual](https://getcomposer.org/doc/00-intro.md).
Add the following package to your `composer.json`:

 ```
 {
     "require": {
         "youshido/graphql": "*"
     }
 }
 ```
After you have created the `composer.json` simply run `composer update`.
Alternatively you can do the following sequence:
```sh
mkdir graphql-test && cd graphql-test
composer init
composer require youshido/graphql
```

After the successful message, Let's check if everything is good by setting up a simple schema that will return current time.
*(you can find this example in the examples directory – [01_sandbox](https://github.com/Youshido/GraphQL/examples/01_sandbox))*

```php
<?php
namespace Sandbox;

use Youshido\GraphQL\Processor;
use Youshido\GraphQL\Schema;
use Youshido\GraphQL\Type\Object\ObjectType;
use Youshido\GraphQL\Type\Scalar\StringType;

require_once 'vendor/autoload.php';

$processor = new Processor();
$processor->setSchema(new Schema([
    'query' => new ObjectType([
        'name' => 'RootQueryType',
        'fields' => [
            'currentTime' => [
                'type' => new StringType(),
                'resolve' => function() {
                    return date('Y-m-d H:ia');
                }
            ]
        ]
    ])
]));

$res = $processor->processRequest('{ currentTime }')->getResponseData();
print_r($res);
```

If everything was set up correctly – you should see response with your current time:
 ```js
 {
    data: { currentTime: "2016-05-01 19:27pm" }
 }
 ```

If not – check that you have the latest composer version and that you've created your test file in the same directory you have `vendor` folder in.
You can always use a script from `examples` folder. Simply run `php vendor/youshido/GraphQL/examples/01_sandbox/index.php`.

## Example – Creating Blog Schema

For our learning example we'll architect a GraphQL Schema for the Blog.

We'll keep it simple so our Blog will have Users who write Posts and leave Comments.
Here's an example of the query that returns title and summary of the latest Post:
 ```
 latestPost {
     title,
     summary
 }
 ```
As you can see, GraphQL query is a simple text query structured very much similar to the json or yaml format.

Supposedly our server should reply with a response structured like following:
 ```js
 {
    data: {
        latestPost: {
            title: "This is a post title",
            summary: "This is a post summary"
        }
    }
 }
 ```

Let's go ahead and create a backend that can handle that.

### Creating Post schema

We believe you'll be using our package along with your favorite framework (we have a Symfony version [here](http://github.com/Youshido/GraphqlBundle)).
But for the purpose of the current example we'll keep it as plain php code.
*(you can check out the complete example by the following link https://github.com/Youshido/GraphQL/examples/02_Blog )*

We'll take a quick look on different approaches you can use to define your schema.
Even though inline approach might seem to be easier and faster we strongly recommend to use an object based because it will give you more flexibility and freedom as your project grows.

#### Inline approach

So we'll start by creating the Post type. For now we'll have only two fields – title and summary:

**inline-schema.php**
```php
<?php
use Youshido\GraphQL\Type\Object\ObjectType;
use Youshido\GraphQL\Type\Scalar\StringType;

// creating a root query structure
$rootQueryType = new ObjectType([
    // name for the root query type doesn't matter, by a convention it's RootQueryType               
    'name'   => 'RootQueryType',
    'fields' => [
        'latestPost' => new ObjectType([ // our Post type will be extended from the generic ObjectType
            'name'    => 'Post', // name of our type – "Post"
            'fields'  => [                  
                'title'   => new StringType(),  // defining the "title" field, type - String
                'summary' => new StringType(),  // defining the "summary" field, type is also String
            ],
            'resolve' => function () {          // this is a resolve function
                return [                        // for now it will return a static array with data
                    "title"   => "New approach in API has been revealed",
                    "summary" => "This post will describe a new approach to create and maintain APIs",
                ];
            }
        ])
    ]
]);
```

Let's create an endpoint to work with our schema so we can actually test everything we do. it will eventually be able to handle requests from the client.

**router.php**
```php
<?php

namespace BlogTest;

use Youshido\GraphQL\Processor;
use Youshido\GraphQL\Schema;
use Youshido\GraphQL\Type\Object\ObjectType;
use Youshido\GraphQL\Validator\ResolveValidator\ResolveValidator;

require_once __DIR__ . '/vendor/autoload.php';
$rootQueryType = new ObjectType([
    'name' => 'RootQueryType',
]);

require_once __DIR__ . '/inline-schema.php';       // including our schema

$processor = new Processor();
$processor->setSchema(new Schema([
    'query' => $rootQueryType
]));
$payload = '{ latestPost { title, summary } }';
$response = $processor->processRequest($payload, [])->getResponseData();

print_r($response);
```

To check if everything is working well, simply execute it – `php router.php`
You should see a result similar to the one described in the previous section:
 ```js
 {
    data: {
        latestPost: {
            title: "New approach in API has been revealed",
            summary: "This post will describe a new approach to create and maintain APIs"
        }
    }
 }
 ```

As you can see our request was set to retrieve two fields, title and summary. You can try to play with the code by removing one field from the request or by changing the resolve function.

#### Object oriented approach

From now on we'll be focusing on the Object oriented approach, but you can find full examples of both in the examples folder – (https://github.com/Youshido/GraphQL/examples/02_Blog).

Let's create a folder for our Schema:
```sh
mkdir Schema
```

Using your editor create a file `Schema/PostType.php` and put the following content there:
```php
<?php
namespace Examples\Blog\Schema;

use Youshido\GraphQL\Type\Config\TypeConfigInterface;
use Youshido\GraphQL\Type\NonNullType;
use Youshido\GraphQL\Type\Object\AbstractObjectType;
use Youshido\GraphQL\Type\Scalar\IdType;
use Youshido\GraphQL\Type\Scalar\StringType;

class PostType extends AbstractObjectType   // extending abstract Object type
{

    public function build(TypeConfigInterface $config)  // implementing an abstract function where you build your type
    {
        $config->addField('title', new StringType())        // adding title field of type String
               ->addField('summary', new StringType());     // adding summary field of type String
    }

    public function resolve($value = null, $args = [])  // implementing resolve function
    {
        return [
            "title"   => "New approach in API has been revealed",
            "summary" => "This post will describe a new approach to create and maintain APIs",
        ];
    }    

    public function getName()
    {
        return "Post";  // important to use the real name here, it will be used later
    }

}
```

In order to make it work we need to update our `router.php` as well:
```php
<?php

namespace BlogTest;

use Youshido\GraphQL\Processor;
use Youshido\GraphQL\Schema;
use Youshido\GraphQL\Type\Object\ObjectType;
use Youshido\GraphQL\Validator\ResolveValidator\ResolveValidator;

require_once __DIR__ . '/vendor/autoload.php';
require_once __DIR__ . '/Schema/PostType.php';       // including PostType definition

$rootQueryType = new ObjectType([
    'name' => 'RootQueryType',
]);
// adding a field to our query schema
$rootQueryType->getConfig()->addField('latestPost', new PostType());    

$processor = new Processor();
$processor->setSchema(new Schema([
    'query' => $rootQueryType
]));
$payload = '{ latestPost { title, summary } }';
$response = $processor->processRequest($payload, [])->getResponseData();

print_r($response);
```

Once again, let's make sure everything is working properly by running `php router.php`. You should see the same response you saw for the inline approach.

## Define your GraphQL Schema

We would recommend to stick to object oriented approach for the several reasons that matter the most for the GraphQL specifically (also valid as general statements):
 - it makes your Types reusable
 - abilities to refactor your schema using IDEs
 - autocomplete to help you avoid typos
 - much easier to navigate through your Schema

> **User valid Names**
> We highly recommend to get familiar with GraphQL [specification](https://facebook.github.io/graphql/#sec-Language.Query-Document), but important thing for now is
> to remember that valid identifier in GraphQL should follow the pattern `/[_A-Za-z][_0-9A-Za-z]*/`.
> That means any identifier can has only latin letter, underscore, or digit but can't start with digit.
> *Names are case sensitive*

We'll continue on our Blog Schema throughout our exploration of all the details of GraphQL.

## Query Documents

Query Document in terms of GraphQL describes a complete request received by GraphQL service.
It contains list of *Operations* and *Fragments*. Both are fully supported by our PHP library.
There are two types of *Operations* in GraphQL:
- *Query* – a read only request that is not supposed to do any changes on the server
- *Mutation* – a request that changes data on the server followed by a data fetch

You've already seen `latestPost` and `currentTime` queries in our examples above, so let's define a simple Mutation to provide an API to like the Post.
Before we jump into writing the code let's think about it.
Any Operation, in our case a mutation, need to have a response type. Let's just see how a possible query from the client can look like:
```
{
  likePost(id: 23) {
    likeCount
  }
}
```

This mutation expects to receive a total amount of post's like as a result. Such a mutation can be describe in PHP:

```php



```

## Types System

*Type* is a atom of definition in GraphQL Schema. Every field, object, or argument has a type. That obviously means GraphQL is a strongly typed language.
Types are defined specific for the application, in our case we'll have types like `Post`, `User`, `Comment` and so on.
GraphQL has variety of build in types that are used to build your custom types.

### Scalar Types

List of currently supported types:
- Int
- Float
- String
- Boolean
- Id (serialized as String per [spec](https://facebook.github.io/graphql/#sec-ID))

We also implemented some extended types that we're considering to be scalar:
- Timestamp
- Date
- DateTime
- DateTimeTz
