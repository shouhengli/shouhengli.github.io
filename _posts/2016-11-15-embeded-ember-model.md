---
title:  "Using MongoDB with Ember Data"
date:   2016-11-15 22:00:00 +11
categories: [Web App]
tags: [Ember.js, MongoDB, RESTful]
---
I have been working with Ember.js + MongoDB for over two years now, it is a great tech combination and I enjoyed it most of the time. But sometimes I also got annoyed by problems that I just can't find a seamless solution online. Get Ember Data working with the nesting structure of MongoDB documents is definitely one of them. Therefore, I decide to write it down so that I don't forget and also help others who encounters the same problem.

### The Problem

My DB document has a nested structure, for example:

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

Serialize the document and serve it is pretty staight forward with the help of [Django Rest Framework MongoEngine][django_rest_framework_mongoengine]. However, mapping it with Ember Data Model is a different story. I have thought and tried a few approach and there seems no perfect solution. Although I finally settled on one, it is still arguable.

### One model for all
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

1. You can only persist one model.

   While I can use `author.books.0` as the "model" to render data in a handlebar template, I can only do CRUD on the model, not the individual fragments. I thought about do my own customized ajax to update fragments and force the model to reload, but it is silly.

2. All the routes/pages can only have the same model.

   This is simply not going to work. For example, if you have a nested route `author/{id}/books/{id}`, and you can access the *category* field in the handlebar in the way of `model.books.{index}.category`, the code will look so bad that it will rot quickly, not to mention the non-trival work to track and keep `{index}` in sync.

3. No field definition to reply on.

   One of the benefits for using Ember Data is that we define the field and its type so that we are confident that they are there. Having a generic `DS.attr()` doesn't provide any information on what to expect. 

### Use model fragments

The Ember community has provided an Addon for using ID-less document fragments as models [Ember Data Model Fragments][model-fragments]. As it says in the description:

> This package provides support for sub-models that can be treated much like belongsTo and hasMany relationships are, but whose persistence is managed completely through the parent object.

[Ember Data Model Fragments][model-fragments] is a mature library with a comprehensive documentation with exmaples. It basically allows you to use a fragment of a model as another model. By using it, the *author* can be defined as:

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

So now we have 3 models, and we can define the fields and types as normal Ember Model. Then I can pass the fragments as my child routes' models: *author* for `/authors`, *book* for `/authors/0/books/1` and *meta* for `/author/0/meta`. Looks great. Now I can code my Ember app as normal, I can even add default value, nested fragments, and extend fragments in the polymorphsm way. 

But did I miss anything? 
Yes, in the library's description:

> ...but whose persistence is managed completely through the parent object. 

It means that although I can fragments like normal models, I can't persist them through Ember Data, in another word, `fragment.save()` or `fragment.destroyRecord()` would not work.

Luckily, not being able to persist fragments is definitely not the end of the world, it can be workarounded. For instance, you can always use `this.controllerFor('author').get('model').save()`, or the way I prefer, bubble the action up to the parent route:

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

1. The parent model is usually large, it is not efficient to send it via network for every change, while what I really need to send is the fragment that is changed.
2. Without an ID'd model, the Ember store cache (aka [identity map][identity-map]) is useless, more HTTP requests will be send than required.

Network inefficiency is indeed a perforamnce issue, and it will only get worse if the site is SSL encrypted.

[Ember Data Model Fragments][model-fragments] is made for Non-SQL document fragments which in most cases do not have an ID. Without identifier, it is not possible to persist the fragments individually. 

### Use EmbededRecordsMixin

This is the solution I now use. It is actually documented in Ember Data API: [EmbededRecordsMixin][embed-record-mixin]. And here are my models:

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
This approach allows me to pass the nested models around, take advantage of caching as well as perform CRUD operations on each models individually without worrying about going out of sync.

It is recommended by Ember API and well documented at [here][embed-record-mixin], but why didn't I use it at the first place?

Well, because there is a prerequiste: **embedded models need to have unique identifiers**. Ember Data needs IDs to work with adapter and its identity map.

You may have noticed, in my exmample payload at the very begining, the fragments (*book* and *meta*) do have *id* field. In fact the IDs are generated by the backend for its own reference and I can easily reuse the backend IDs for my own convinience, except for one problem: the IDs are only unique within the scope of its parent, in another word, *book*'s ID *1* is only unique within the scope of author *0*, and author *1* can have a totally different *book* with the same ID *1*. Unfortunately, identity map is not scope aware and once *book* *1* is cached it will be cached regardless of what the parent scope *author* is. 

It would be ideal that if Ember Data provides a way to customize identity map to be scope-aware, but for now I just have to workaround it. One workaround is to unload the cache explictly when loading a model. However, this is pretty bad as it leaves the cache useless. The workaround I finally go with is to alter the backend's ID generator to generate universally unique IDs. It is not ideal that I have to change the backend's ID generations logic to get around it, but it is the compromise I have to make.

### There is always a better way
There are still rooms to improve. One thing I am planning to do is to get rid of *EmbededRecordsMixin* and use the async style lazy-loading instead. With the current approach the payload for *author* needs to include *books* and *meta* because we explicitly say they are always embedded. However, this is unnessary and just like the second solution, it is sending unnessary data through network. A better way is to hide the embedded structure from Embed and emulate the behaviour of a relational database in the REST layer. Something like:

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
You may find this is nothing like what our MongoDB document structure is, but if you think about it, does payload really have to reflect DB structure? Certainly not, in fact, some information sensitive places even consider it as a secruity pitfall. Also, one of the advantages of having a multi-layer architecture is the flexibility to transform DB records.

Doing this will certainly require some extra work to transform data format on the server side. Luckily, both MongoEngine and pyMongo have a good support for MongoDB projection query. By leveraging the power, it shouldn't be a task that is too hard to do.

### :wq
This is my very first post on Ember.js. I like Ember and I believe the Ember conventions educate me to be a better programmer. I hope you find this post helpful, if you do, please feel free to share with acknowledgement. If you have any qustions or suggestions, please leave a comment and I will be happy to discuss.

[model-fragments]:	https://github.com/lytics/ember-data-model-fragments
[embed-record-mixin]:	http://emberjs.com/api/data/classes/DS.EmbeddedRecordsMixin.html 
[identity-map]:		https://haagen-software.no/blog/post/2014_11_07_understanding_ember_data
[django_rest_framework_mongoengine]:	https://github.com/umutbozkurt/django-rest-framework-mongoengine
