# Piwik/Cache

This is a PHP caching library based on [Doctrine cache](https://github.com/doctrine/cache) that supports different backends. 
At [Piwik](http://piwik.org) we developed this library with the focus on speed as we make heavy use of caching and 
sometimes fetch hundreds of entries from the cache.

[![Build Status](https://travis-ci.org/piwik/component-cache.svg?branch=master)](https://travis-ci.org/piwik/component-cache)

## Installation

With Composer:

```json
{
    "require": {
        "piwik/cache": "*"
    }
}
```

## Supported backends
* Array (holds cache entries only during one request but is very fast)
* Null (useful for development, won't cache anything)
* File (stores the cache entry on the file system)
* Redis (stores the cache entry on a Redis server, requires [phpredis](https://github.com/nicolasff/phpredis))
* Chained (allows to chain multiple backends to make sure it will read from a fast cache if possible)

Doctrine cache provides support for many more backends and adding one of those is easy. For example:
* APC
* Couchbase
* Memcache
* MongoDB
* Riak
* WinCache
* Xcache
* ZendData

Please send a pull request in case you have added one. 

## Different caches

This library comes with three different types of caches. The naming is not optimal right now.

### Persistent

This can be considered as the default cache to use. The persistent cache works with any backend so you can decide
whether you want to persist cache entries between requests or not. It does not support the caching of any objects. 
Only boolean, numbers, strings and arrays are supported. Whenever you request an entry from the cache it will fetch the 
entry from the defined backend again which can cause many reads depending on your application.

### Transient

This class is used to cache any data during one request. It won't be persisted.

Compared to the persistent cache it does not support setting any life time. All cache entries will be only cached during 
one request in a simple array meaning it is very fast to save and fetch cache entries. It can also hold all sorts of 
values, even objects. For fast performance it won't validate any cache ids.

### Multi

This cache stores all its cache entries under one "cache" entry in a configurable backend.

This comes handy for things that you need very often, nearly in every request. Instead of having to read eg.
a hundred cache entries from files it only loads one cache entry which contains the hundred keys. Should be used only 
for things that you need very often and only for cache entries that are not too large to keep loading and parsing the 
single cache entry fast. This cache is even more useful in case you are using a slow backend such as a file or a database.
 Instead of having a hundred stat calls there will be only one. All cache entries it contains have the same life time. 
 For fast performance it won't validate any cache ids. It is not possible to cache any objects using this cache.

## Usage

### Creating a file backend

```php
$options = array('directory' => '/path/to/cache');
$factory = new \Piwik\Cache\Backend\Factory();
$backend = $factory->buildBackend('file', $options);
```

### Creating a Redis backend

```php
$options = array(
    'host'     => '127.0.0.1',
    'port'     => '6379',
    'timeout'  => 0.0,
    'database' => 15, // optional
    'password' => 'secret', // optional
);
$factory = new \Piwik\Cache\Backend\Factory();
$backend = $factory->buildBackend('redis', $options);
```

### Creating a chained backend

```php
$options = array(
    'backends' => array('array', 'file'),
    'file'     => array('directory' => '/path/to/cache')
);
$factory = new \Piwik\Cache\Backend\Factory();
$backend = $factory->buildBackend('redis', $options);
```

Whenever you set a cache entry it will save it in the array and in the file cache. Whenever you are trying to read a cache
entry it will first try to get it from the fast array cache. In case it is not available there it will try to fetch
the cache entry from the file system. If the cache entry exists on the file system it will cache the entry automatically
using the array cache so the next read within this request will be fast and won't cause a stat call again. If you delete
 a cache entry it will be removed from all configured backends. You can chain any backends. It is recommended to list 
 faster backends first.

### Creating a persistent cache

```php
$factory = new \Piwik\Cache\Backend\Factory();
$backend = $factory->buildBackend('file', array('directory' => '/path/to/cache'));

$cache = new \Piwik\Cache\Persistent($backend);
$cache->get('myid');
$cache->has('myid');
$cache->delete('myid');
$cache->set('myid', 'myvalue', $lifeTimeInSeconds = 300);
$cache->flushAll();
```

### Creating a transient cache

```php
$cache = new \Piwik\Cache\Transient();
$cache->get('myid');
$cache->has('myid');
$cache->delete('myid');
$cache->set('myid', new \stdClass(), $lifeTimeInSeconds = 300);
$cache->flushAll();
```

### Creating a multi cache

```php
$cache = new \Piwik\Cache\Multi();
$cache->populate($backend, $cacheId = 'multicache', $lifeTimeInSeconds = 300);
$cache->get('myid');
$cache->has('myid');
$cache->delete('myid');
$cache->set('myid', new \stdClass(), $lifeTimeInSeconds = 300);
$cache->persistCacheIfNeeded();
$cache->flushAll();
```

It will cache all set cache entries under the cache entry `multicache`.

## License

The Cache component is released under the [LGPL v3.0](http://choosealicense.com/licenses/lgpl-3.0/).
