---
layout: post
title: "How to Remove a Property from a Mongoose.js Schema"
comments: true
canonical: "http://bytes.goodeggs.com/post/36553128854/how-to-remove-a-property-from-a-mongoosejs-schema"
categories: 
---

This should be simple, but Mongoose really clings to data in existing documents.  I'll walk through all the ways I wanted it to work that failed.  We'll remove an `organic` flag from a toy `Food` model so we can replace it with [Bittman's dream label](http://www.nytimes.com/2012/10/14/opinion/sunday/bittman-my-dream-food-label.html).  If you just came for the solution, I arrived at:

{% codeblock lang:js %}
Food.collection.update({}, 
  {$unset: {organic: true}}, 
  {multi: true, safe: true}
);
{% endcodeblock %}

<!-- more -->Our well-loved `Food` schema might look something like:

{% codeblock lang:js %}
var Food = db.model('Food', new mongoose.Schema({
  name: {type: String, required: true},
  organic: Boolean
}, {
  strict: true
}));
{% endcodeblock %}

and it might be populated with documents like organic frozen broccoli:

{% codeblock lang:js %}
var broccoli = new Food({
  name: 'frozen broccoli',
  organic: true
});
{% endcodeblock %}

Alright, time to get rid of that `organic` property.  Adding a property with Mongoose is as easy as declaring it in the schema.  Could removing be just as easy?

{% codeblock lang:diff %}
  var Food = db.model('Food', new mongoose.Schema({
    name: {type: String, required: true},
-   organic: Boolean
  }, {
    strict: true
  }));
{% endcodeblock %}

If we reload our broccoli doc, will mongoose strip out the undeclared properties?  We did tell Mongoose to be `strict` with our `Food`…

{% codeblock lang:js %}
Food.findById(broccoli, function(err, broccoli) {
  console.log(broccoli.get('organic'));
});

> true
{% endcodeblock %}

No.  Too slick.  I suppose it's comforting that mongoose isn't silently manipulating our docs.  Maybe we just need to re-save `broccoli`.  Surely mongoose will be `strict` now…

{% codeblock lang:js %}
broccoli.save(function() {
  Food.findById(broccoli, function(err, broccoli) {
    console.log(broccoli.get('organic'))
  })
});

> true
{% endcodeblock %}

Nope.  [Mr. Heckmann rationalizes this behavior](http://grokbase.com/t/gg/mongoose-orm/123ya4qp0a/mongoose-removing-an-existing-field-from-a-collection#20120330swrofqtizat6i3kalhvfrusz5a) as

> Mongoose "plays nice" with existing data in the db, not deleting it unless you tell it to.


I'll have to be more explicit with this broccoli, more meticulous with my cleanup.  I'll unset `organic` directly.

{% codeblock lang:js %}
broccoli.set('organic', undefined);
broccoli.save(function() {
  Food.findById(broccoli, function(err, broccoli) {
    console.log(broccoli.get('organic'));
  });
});

> true
{% endcodeblock %}

Wow.  Fine.  Now `strict` decides to help out.

Mongoose isn't cooperating.  Time to talk directly to Mongo.  Maybe Mongoose can at least offer me some [update sugar](http://mongoosejs.com/docs/api.html#model_Model-update):

{% codeblock lang:js %}
Food.update({}, 
  {$unset: {organic: true}}, 
  {multi: true, safe: true}, 
  function(err) {
    Food.findById(broccoli, function(err, broccoli) {
      console.log(broccoli.get('organic'));
    });
  }
);

> true
{% endcodeblock %}

![Y U NO UNSET!?](/images/yuno.jpg)

This must be `strict` still [keeping us safe](https://groups.google.com/d/topic/mongoose-orm/ypvL3Fximjc/discussion).

Okay.  Last chance Mongoose.  Just give me the collection.

{% codeblock lang:js %}
Food.collection.update({}, 
  {$unset: {organic: true}}, 
  {multi: true, safe: true}, 
  function(err) {
    Food.findById(broccoli, function(err, brocolli) {
      console.log(broccoli.get('organic'));
  }
);

> undefined
{% endcodeblock %}

Phew.  

Here's a [Mocha spec](https://gist.github.com/4008255) reproducing this frustrating sequence.  How should we make it be better?
