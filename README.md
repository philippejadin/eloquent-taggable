# Eloquent-Taggable

Easily add the ability to tag your Eloquent models in Laravel 5.

[![Build Status](https://travis-ci.org/cviebrock/eloquent-taggable.svg?branch=master&format=flat)](https://travis-ci.org/cviebrock/eloquent-taggable)
[![Total Downloads](https://poser.pugx.org/cviebrock/eloquent-taggable/downloads?format=flat)](https://packagist.org/packages/cviebrock/eloquent-taggable)
[![Latest Stable Version](https://poser.pugx.org/cviebrock/eloquent-taggable/v/stable?format=flat)](https://packagist.org/packages/cviebrock/eloquent-taggable)
[![Latest Unstable Version](https://poser.pugx.org/cviebrock/eloquent-taggable/v/unstable?format=flat)](https://packagist.org/packages/cviebrock/eloquent-taggable)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/cviebrock/eloquent-taggable/badges/quality-score.png?format=flat)](https://scrutinizer-ci.com/g/cviebrock/eloquent-taggable)
[![SensioLabsInsight](https://insight.sensiolabs.com/projects/9e1bb86e-2659-4123-9b6f-89370ef1483d/mini.png)](https://insight.sensiolabs.com/projects/9e1bb86e-2659-4123-9b6f-89370ef1483d)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)


* [Installation](#installation)
* [Updating your Eloquent Models](#updating-your-eloquent-models)
* [Usage](#usage)
* [Query Scopes](#query-scopes)
* [The TagService Class](#the-tagservice-class)
* [Configuration](#configuration)
* [Bugs, Suggestions and Contributions](#bugs-suggestions-and-contributions)
* [Copyright and License](#copyright-and-license)


---

## Installation

> **NOTE**: Depending on your version of Laravel, you should install a different
> version of the package:
> 
> | Laravel Version | Taggable Version |
> |:---------------:|:----------------:|
> |       4.*       |        1.0       |
> | 5.1*, 5.2*, 5.3 |        2.0       |
> |       5.4       |      2.1, 3.0    |
> |       5.5       |        3.0       |

1. Install the `cviebrock/eloquent-taggable` package via composer:

    ```shell
    $ composer require cviebrock/eloquent-taggable
    ```
    
2. Unless you are using Laravel 5.5 and it's package auto-discovery, you will
need to add the service provider to `config/app.php`:

    ```php
    # Add the service provider to the `providers` array
    'providers' => [
        ...
        \Cviebrock\EloquentTaggable\ServiceProvider::class,
    ]
    ```

3. Publish the configuration file and migrations

    ```shell
    php artisan vendor:publish --provider="Cviebrock\EloquentTaggable\ServiceProvider"
    ```

4. Finally, use artisan to run the migration to create the required tables.

    ```sh
    composer dump-autoload
    php artisan migrate
    ```


## Updating your Eloquent Models

Your models should use the Taggable trait:

```php
use Cviebrock\EloquentTaggable\Taggable;

class MyModel extends Eloquent
{
    use Taggable;
}
```

That's it ... your model is now "taggable"!


## Usage

Tag your models with the `tag()` method:

```php
// Pass in a delimited string:
$model->tag('Apple,Banana,Cherry');

// Or an array:
$model->tag(['Apple', 'Banana', 'Cherry']);
```

The `tag()` method is additive, so you can tag the model again and those tags will be added to the previous ones:

```php
$model->tag('Apple,Banana,Cherry');

$model->tag('Durian');
// $model now has four tags
```

You can remove tags individually with `untag()` or entirely with `detag()`:

```php
$model->tag('Apple,Banana,Cherry');

$model->untag('Banana');
// $model is now just tagged with "Apple" and "Cherry"

$model->detag();
// $model has no tags
```

You can also completely retag a model (a short form for detagging then tagging):

```php
$model->tag('Apple,Banana,Cherry');

$model->retag('Etrog,Fig,Grape');

// $model is now just tagged with "Etrog", "Fig", and "Grape"
```

You can get the array of all tags (technically, an Eloquent Collection):

```php
foreach($model->tags as $tag)
{
    echo $tag->name;
}
```

You can also get the list of tags as a flattened array, or a delimited list:

```php
$model->tag('Apple,Banana,Cherry');

var_dump($model->tagList);

// string 'Apple,Banana,Cherry' (length=19)

var_dump($model->tagArray);

// array (size=3)
//  1 => string 'Apple' (length=5)
//  2 => string 'Banana' (length=6)
//  3 => string 'Cherry' (length=6)
```

Tag names are normalized (see below) so that duplicate tags aren't accidentally created:

```php
$model->tag('Apple');
$model->tag('apple');
$model->tag('APPLE');

var_dump($model->tagList);

// string 'Apple' (length=5)
```


## Query Scopes

For reference, imagine the following models have been tagged:

| Model Id | Tags                  |
|:--------:|-----------------------|
|     1    | - no tags -           |
|     2    | apple                 |
|     3    | apple, banana         |
|     4    | apple, banana, cherry |
|     5    | cherry                |
|     6    | apple, durian         |
|     7    | banana, durian        |
|     8    | apple, banana, durian |


You can easily find models with tags through some query scopes:

```php
// Find models that are tagged with all of the given tags
// i.e. everything tagged "Apple AND Banana".
// (returns models with Ids: 3, 4, 8)

Model::withAllTags('Apple,Banana')->get();

// Find models with any one of the given tags
// i.e. everything tagged "Apple OR Banana".
// (returns Ids: 2, 3, 4, 6, 7, 8)

Model::withAnyTags('Apple,Banana')->get();

// Find models that have any tags
// (returns Ids: 2, 3, 4, 5, 6, 7, 8)

Model::isTagged()->get();
```

And the inverse:

```php
// Find models that are not tagged with all of the given tags,
// i.e. everything not tagged "Apple AND Banana".
// (returns models with Ids: 2, 5, 6, 7)

Model::withoutAllTags('Apple,Banana')->get();

// To also include untagged models, pass another parameter:
// (returns models with Ids: 1, 2, 5, 6, 7)

Model::withoutAllTags('Apple,Banana', true)->get();

// Find models without any one of the given tags
// i.e. everything not tagged "Apple OR Banana".
// (returns Ids: 5)

Model::withoutAnyTags('Apple,Banana')->get();

// To also include untagged models, pass another parameter:
// (returns models with Ids: 1, 5)

Model::withoutAnyTags('Apple,Banana', true)->get();

// Find models that have no tags
// (returns Ids: 1)

Model::isNotTagged()->get();
```

Some edge-case examples:

```php
// Passing an empty tag list to a scope either throws an 
// exception or returns nothing, depending on the
// "throwEmptyExceptions" configuration option

Model::withAllTags('');
Model::withAnyTags('');

// Returns nothing, because the "Fig" tag doesn't exist
// so no model has that tag

Model::withAllTags('Apple,Fig');
```

Finally, you can easily find all the tags used across all instances of a model:

```php
// Returns an array of tag names used by all Model instances
// e.g.: ['apple','banana','cherry','durian']

Model::allTags();

// Same as above, but as a delimited list
// e.g. 'apple,banana,cherry,durian'

Model::allTagsList();

// Returns a collection of all the Tag models used by any Model instances

Model::allTagModels();
```


## The TagService Class

You can also use `TagService` class directly, however all the functionality is
exposed via the various methods provided by the trait, so you probably don't need to.

As always, take a look at the code for full documentation of the service class.


## Configuration

Configuration is handled through the settings in `/app/config/taggable.php`.  The default values are:

```php
return [
    'delimiters'           => ',;',
    'glue'                 => ',',
    'normalizer'           => 'mb_strtolower',
    'connection'           => null,
    'throwEmptyExceptions' => false,
];
```

### delimiters

These are the single-character strings that can delimit the list of tags passed to the `tag()` method.
By default, it's just the comma, but you can change it to another character, or use multiple characters.

For example, if __delimiters__ is set to ";,/", the this will work as expected:

```php
$model->tag('Apple/Banana;Cherry,Durian');

// $model will have four tags
```

### glue

When building a string for the `tagList` attribute, this is the "glue" that is used to join tags.
With the default values, in the above case:

```php
var_dump($model->tagList);

// string 'Apple,Banana,Cherry,Durian' (length=26)
```

### normalizer

Each tag is "normalized" before being stored in the database.  This is so that variations in the 
spelling or capitalization of tags don't generate duplicate tags.  For example, we don't want three 
different tags in the following case:

```php
$model->tag('Apple');
$model->tag('APPLE');
$model->tag('apple');
```

Normalization happens by passing each tag name through a normalizer function.  By default, this is 
PHP's `mb_strtolower()` function, but you can change this to any function or callable that takes a 
single string value and returns a string value.  Some ideas:

```php

    // default normalization
    'normalizer' => 'mb_strtolower',

    // same result, but using a closure
    'normalizer' => function($string) {
        return mb_strtolower($string);
    },

    // using a class method
    'normalizer' => ['Illuminate\Support\Str', 'slug'],
```

You can access the normalized values of the tags through `$model->tagListNormalized` and 
`$model->tagArrayNormalized`, which work identically to `$model->tagList` and `$model->tagArray` 
(described above) except that they return the normalized values instead.

And you can, of course, access the normalized name directly from a tag:

```php
echo $tag->normalized;
```

### connection

You can set this to specify that the Tag model should use a different database connection.
Otherwise, it will use the default connection (i.e. from `config('database.default')`).

### throwEmptyExceptions

Passing empty strings or arrays to any of the scope methods is an interesting situation.
Logically, you can't get a list of models that have all or any of a list of tags ... if the list is empty!

By default, the `throwEmptyExceptions` is set to false.  Passing an empty value to a query scope 
will "short-circuit" the query and return no models.  This makes your application code cleaner 
so you don't need to check for empty values before calling the scope.

However, if `throwEmptyExceptions` is set to true, then passing an empty value to the scope will 
throw a `Cviebrock\EloquentTaggable\Exceptions\NoTagsSpecifiedException` exception in these cases.
You can then catch the exception in your application code and handle it however you like.


## Bugs, Suggestions and Contributions

Thanks to [everyone](https://github.com/cviebrock/eloquent-taggable/graphs/contributors)
who has contributed to this project, with a big shout-out to 
[Michael Riediger](https://stackoverflow.com/users/502502/riedsio) for help optimizing the SQL.

Please use [Github](https://github.com/cviebrock/eloquent-taggable) for reporting bugs, 
and making comments or suggestions.
 
See [CONTRIBUTING.md](CONTRIBUTING.md) for how to contribute changes.


## Copyright and License

[eloquent-taggable](https://github.com/cviebrock/eloquent-taggable)
was written by [Colin Viebrock](http://viebrock.ca) and is released under the 
[MIT License](LICENSE.md).

Copyright (c) 2013 Colin Viebrock
