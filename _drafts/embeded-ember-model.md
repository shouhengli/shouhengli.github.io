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
      "id": 1,
      "book_count": 23,
      "keywords": ["money", "ember", "developer"]
    }
  }
}
```
In this post, I'll explain how to map the structure with embeded model, and then optimize network traffic using lazy loading and making use of Ember Data's [identity map][identity-map] (store cache).

### 1. One model for all
Intuitively, a model which exactly maps the DB structure seems to be the way to go.

```javascript
// model/author.js
import DS from "ember-data";

export default DS.Model.extend({
  books:DS.attr(),
  first_name: DS.attr('string'),
  last_name: DS.attr('string'),
  meta: DS.attr()
});
```
This would work only if you have a very few number of routes. However, if you have a complex routing structure like I do, this would not work. Having a massive model like this means:

1. You can only do CRUD on model.

   While you can probably use `author.books.0` as the "model" for a handlebar template, you can't do CRUD on it since it is not really an Ember model. Sure you can do your customized ajax to update parts of the model, but it would be silly: why use Ember Data if you are not taking the advantage of it

2. All the routes/pages can only have the same model.

   This is simply not going to work. For example, if you have a nested route `author/1/books/1`, and you use the *category* field in the handlebar like `model.books.0.category`. This is super nasty and rots easily, not to mention the overhead of tracking the index *0* in all sub-routes.

3. No field definition to reply on.

   One of the reasons for using Ember Data is that we define the field and types so that we can trust them to exist when use them later. Having a generic `DS.attr()` doesn't provide any information on what to expect from that field, which may worry a lot of people.

### 2. Use model fragments

The Ember community is great enough to have a Ember CLI addon created for this kind of situation: [Ember Data Model Fragments][model-fragments]. As it says in the description:

> This package provides support for sub-models that can be treated much like belongsTo and hasMany relationships are, but whose persistence is managed completely through the parent object.

[Ember Data Model Fragments][model-fragments] is a mature library with a comprehensive documentation with exmaples. It basically allows you to use a fragment of a model as another model. For example, with its help, the *author* can be defined as:

```javascript
// model/author.js
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
// model/book.js
import DS from "ember-data";
import Fragment from 'model-fragments/fragment';

export default Fragment.extend({
  title: DS.attr('string'),
  category: DS.attr('string')
});
```

```javascript
// model/meta.js
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

So now we have 3 models/fragments, and the fields and types are all defined. Then I can pass the fragments as models to my child routes, for exmaple, *author* for `/authors`, *book* for `/authors/0/books/1` and *meta* for `/author/0/meta`. Looks great. Now I can code my Ember app as normal, I can even add default value, nested fragments, and extend fragments in the polymorphsm way. 

But did I miss anything? 
Yes, I missed this sentence in the library's description:

> ...but whose persistence is managed completely through the parent object. 

Oops. It means that although I can pass the fragments around and use them like normal models, I can't persist them through Ember Data, i.e. `fragment.save()` or `fragment.destroyRecord()` would not work.

Luckily, not being able to persist fragments is definitely not the end of the world, it can actually be workarounded easily. For instance, you can always use `this.controllerFor('author').get('model').save()`, or the neater solution: bubble the action up to the parent route:

```javascript
// controller/books.js
import Ember from "ember";

export default Ember.Controller.extend({
  actions:{
    addNewBook(){
      let books = this.get('model');
      books.createFragment({
        title: 'new book',
        category: 'thrill'
      });
      this.send('saveAuthor');
    }
  }
})
```
```javascript
// route/author.js
import Ember from "ember";

export default Ember.Route.extend({
  actions: {
    saveAuthor() {
      this.modelFor('author').save();
    }
  }
})
```

The solution is good enough to get things working in the Ember way, but there's one last thing that annoys me: **Network Effciency**.

1. The grand model is usually large. We will send large data chunk through network while most of the data are the same.
2. Frequent trafficing. We are not using Ember store cache (aka [identity map][identity-map]), more HTTP requests will be made than required.

Network inefficiency is indeed a perforamnce issue, and it will only get worse if the site is SSL encrypted.

So what's the reason that stops [Ember Data Model Fragments][model-fragments] from persisting fragments? The answer is identifiers.

[Ember Data Model Fragments][model-fragments] is made for Non-SQL document which in most cases do not have an ID for its field! That's right, generally in the example, *books* don't get a generated ID, only *author* gets one because it is a document in the DB while `books` are not.

### 3. Use EmbededRecordsMixin

The solution I finally go with is nothing fancy, instead, I just used the [EmbededRecordsMixin][embed-record-mixin] which is included in the standard Ember Data library. 

```javascript
// model/author.js
import DS from "ember-data";

export default DS.Model.extend({
  books:DS.hasMany('book', { async: false }),
  first_name: DS.attr('string'),
  last_name: DS.attr('string'),
  meta: DS.belongTo('book', { async: false })
});
```
```javascript
// model/book.js
import DS from "ember-data";

export default DS.Model.extend({
  title:DS.attr('string'),
  category:DS.attr('string')
});
```
```javascript
// model/meta.js
import DS from "ember-data";

export default DS.Model.extend({
  book_count:DS.attr('string'),
  keywords:DS.attr()
});
```
```javascript
// serializer/author.js
import DS from "ember-data";

export default DS.RESTSerializer.extend(DS.EmbeddedRecordsMixin, {
  attrs: {
    books: {embedded: 'always'},
    meta: {embedded: 'always'}
  }
});
```
This is a simple and elegant solution, it allows us to pass the nested models around, take advantage of caching as well as perform persistence CRUD on each models.

This solution is recommended by Ember API and documented in [detail][embed-record-mixin], but why did I use it at the first place?

Well, because there is a prerequiste: **embedded models need to have unique identifiers**. Ember Data needs an ID to work with adapter, and the ID has to be unique so that the caching identity map doesn't get confused.

You may have noticed, in my exmample payload at the very begining, the embeded models (*book* and *meta*) have an *id* field. In fact the IDs are generated by the backend for its own reference as a shotcut to lookup records quickly with projection queries. 

Easy enough that we can reuse the backend IDs for our own convinience except for one project: The IDs are only unique within the scope of its parent, e.g. *book* *1* is unique within author *0* but author *1* can have a totally different *book* *1*. This was causing problems in caching since identity map is not scope aware and once *book* *1* is cached it will be served for all authors regardless of who the author is. 

It would be ideal that if Ember Data provides a way to customize identity map to be scope-aware, but since it is not support, we have to workaround it. One workaround is to unload the cache whenever a page's scope is changed. However, this is pretty bad as it leaves the cache useless. The workaround I finally chose is to enhance the backend's id generator to generate generally unique IDs. I need to fiddle a bit with the backend code to get this working, but as long as the IDs doesn't run out, this is a safe solution.

### Future work
There are still rooms to improve for the solution. One of them is that with `{embedded: 'always'}`, the payload for *author* needs to include *books* and *meta*, which is unnessary and a potential performance pitfall. A better way is to hide the embedded structure from Embed and emulate the behaviour of a relational database in the REST layer. 

```json
// Asuming using DS.RESTAdapter
{
  "author": {
    "id": 1,
    "books": [1, 2],
    "first_name": "John",
    "last_name": "Sting",
    "meta": 1
  }
}
```
You may find this is nothing like what our MongoDB document structure is, but if you think about, does our models have to reflect our DB structure? Certainly not. Actually one of the advantages of having a multi-layer architecture is the flexibility of the ways to use DB records. In some places, exposing DB schema is even considered as a security breach.

Doing this will certainly require some work to transform data format. Luckily, both MongoEngine and pyMongo have a good support for MongoDB projection query. By leveraging the power of projection, the transformation shouldn't be a hard work after all.

### :wq
This is my very first post on Ember.js. I like Ember and I believe by following and understading the Ember conventions, people are educated to be a better programmer. I hope you find this post helpful, if you do, please feel free to share with acknowledgement. If you have any qustions or you find my argument absurd, please leave a comment and I will be happy to discuss.

[model-fragments]:	https://github.com/lytics/ember-data-model-fragments
[embed-record-mixin]:	http://emberjs.com/api/data/classes/DS.EmbeddedRecordsMixin.html 
[identity-map]:		https://haagen-software.no/blog/post/2014_11_07_understanding_ember_data
