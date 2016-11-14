---
title:  "Using embedded model with Ember Data"
date:   2016-10-30 22:00:00
categories: [ember]
tags: [ember]
---

I"m using MongoDB+MongoEngine to serve data in my Ember.js project (I'm using Ember Data too). My DB document has a nested structure, for example:

```json
{
  "author": {
    "id": 1,
    "books": [
      {
        "id": 1,
        "title": "book 1",
        "category": "sci-fi"
      },{
        "id": 2,
        "title": "book 2",
        "category": "thrill"
      }
    ],
    "first_name": "John",
    "last_name": "Sting",
    "meta": {
      "book_count": 23,
      "keywords": ["money", "ember", "developer"]
    }
  }
}
```
In this post, I'll explain how to map the structure with embeded model, and then optimize network traffic using lazy loading and making use of Ember Data's identity map (store cache).

### One model for all
Intuitively, a model which exactly maps the DB structure seems to be the way to go.

```javascript
// author.js
import DS from "ember-data";

export default DS.Model.extend({
  books:DS.attr(),
  first_name: DS.attr('string'),
  last_name: DS.attr('string'),
  meta: DS.attr()
});
```

This would work only if you have a very few number of routes. However, if you have a complex routing structure like I do, this would not work.  Having a massive model like this means:

1. You can only do CRUD on model.

   While you can probably use `author.books.0` as the "model" for a handlebar template, you can't do CRUD on it since it is not really an Ember model. Sure you can do your customized ajax to update parts of the model, but it would be silly: why use Ember Data if you are not taking the advantage of it

2. All the routes/pages can only have the same model.

   This is simply not going to work. For example, if you have a nested route `author/1/books/1`, and you use the `categorty` field in the handlebar like `model.books.0.category`. This is super nasty and rots easily, not to mention the overhead of tracking the index `0` in all sub-routes.

3. No field definition to reply on.

   One of the reasons for using Ember Data is that we define the field and types so that we can trust them to exist when use them later. Having a generic `DS.attr()` doesn't provide any information on what to expect from that field, which may worry a lot of people.

### Model Fragments

The Ember community is great enough to have a Ember CLI addon created for this kind of situation: [Ember Data Model Fragments][model-fragments]. As it says in the description:

> This package provides support for sub-models that can be treated much like belongsTo and hasMany relationships are, but whose persistence is managed completely through the parent object.

Ember Data Model Fragments is a mature library with a comprehensive documentation with exmaples. It basically allows you to use a fragment of a model as another model. For example, with its help, the `author` can be defined as:

```javascript
// author.js
import DS from "ember-data";
import {
  fragment,
  fragmentArray,
} from 'model-fragments/attributes';

export default DS.Model.extend({
  books: fragmentArray('book'),
  first_name: DS.attr('string'),
  last_name: DS.attr('string'),
  meta: fragment('meta')
});
```

```javascript
// book.js
import DS from "ember-data";
import Fragment from 'model-fragments/fragment';

export default Fragment.extend({
  title: DS.attr('string'),
  category: DS.attr('string')
});
```

```javascript
// meta.js
import DS from "ember-data";
import {
  array,
} from 'model-fragments/attributes';
import Fragment from 'model-fragments/fragment';

export default Fragment.extend({
  book_count: DS.attr('string'),
  keywords: array()
});
```

So now we have 3 models/fragments, and the fields and types are all defined. Then I can pass the fragments as models to my child routes, for exmaple, `author` for `/authors`, `book` for `/authors/0/books/1` and `meta` for `/author/0/meta`. Looks great. Now I can code my Ember app as normal, I can even add default value, nested fragments, and extend fragments in the polymorphsm way. 

But did I miss anything? 
Yes, I missed this sentence in the library's description:
> ...but whose persistence is managed completely through the parent object. 
Oops. It means that although I can pass the fragments around and use them like normal models, I can't persist them through Ember Data, i.e. `fragment.save()` or `fragment.destroyRecord()` would not work. Why is that? 

Ember Data Model Fragments is made for Non-SQL document fragments which in most cases do not have an ID! That's right, generally in the example, `books` don't get a generated ID, only `author` gets one because it is a document in the DB while `books` are not.

### Use embedded models


```javascript
// author.js
import DS from "ember-data";

export default DS.Model.extend({
  books:DS.hasMany('book', { async: false }),
  first_name: DS.attr('string'),
  last_name: DS.hasMany('datum', {async: false}),
  meta: DS.attr()
});
```

```javascript
// book.js
import DS from "ember-data";

export default DS.Model.extend({
  title:DS.attr('string'),
  category:DS.attr('string')
});
```

### Make use of store cache

All the IDs are a managed by our backend code and is unique within the parent scope, e.g. a book's ID `1` is unique within the author, but other authors may have a different book with the same ID `1`. The IDs are generated not just for the convinience of frontend, but also for the usage of backend computation engine.

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
[model-fragments]: https://github.com/lytics/ember-data-model-fragments
