---
title:  "Using embedded model with Ember Data"
date:   2016-10-30 22:00:00
categories: [ember]
tags: [ember]
---

### Background

#### DB structure
In my Ember project, I"m using MongoDB+MongoEngine to serve data. As a common usage of Non-SQL, my document has a nested structure, for example:

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
        "title": "book 2"
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
All the IDs are a managed by our backend code and is unique within the parent scope, e.g. a book's ID `1` is unique within the author, but other authors may have a different book with the same ID `1`. The IDs are generated not just for the convinience of frontend, but also for the usage of backend computation engine.

#### Ember model
And here is how my Ember models look like:

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

```

Check out the [Jekyll docs][jekyll] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll’s dedicated Help repository][jekyll-help].

[jekyll]:      http://jekyllrb.com
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-help]: https://github.com/jekyll/jekyll-help
