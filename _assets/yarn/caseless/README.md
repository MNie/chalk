## Caseless -- wrap an object to set and get property with caseless semantics but also preserve caseing.

This library is incredibly useful when working with HTTP headers. It allows you to get/set/check for headers in a caseless manner while also preserving the caseing of headers the first time they are set.

## Usage

{% highlight javascript %}
var headers = {}
  , c = caseless(headers)
  ;
c.set('a-Header', 'asdf')
c.get('a-header') === 'asdf'
```

## has(key)

Has takes a name and if it finds a matching header will return that header name with the preserved caseing it was set with.

{% highlight javascript %}
c.has('a-header') === 'a-Header'
```

## set(key, value[, clobber=true])

Set is fairly straight forward except that if the header exists and clobber is disabled it will add `','+value` to the existing header.

{% highlight javascript %}
c.set('a-Header', 'fdas')
c.set('a-HEADER', 'more', false)
c.get('a-header') === 'fdsa,more'
```

## swap(key)

Swaps the casing of a header with the new one that is passed in.

{% highlight javascript %}
var headers = {}
  , c = caseless(headers)
  ;
c.set('a-Header', 'fdas')
c.swap('a-HEADER')
c.has('a-header') === 'a-HEADER'
headers === {'a-HEADER': 'fdas'}
```
