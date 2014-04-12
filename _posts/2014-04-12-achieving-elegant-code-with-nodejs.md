---
title: Achieving elegant code with node.js
layout: post
---

# The problem

I've spent a whole week, trying to figure out how to achieve readable code with node.js. 
Consider writing a redis queue. Let's first start with a straightforward callback-based
approach.

## Vanilla Javascript
Here is our redis queue implementation:

{% highlight js %}
// redis_queue.js
module.exports = function(url) {
  var client = redis.createClient(url.port || 6379, url.hostname);

  this.enqueue = function(message, callback) {
    client.lpush(inQueue, JSON.stringify(message), callback);
  };

  this.dequeue = function(callback) {
    client.rpop(inQueue, function(err, data){
      if (err) throw err;
      var message = JSON.parse(data);
      callback(message);
    });
  };
};
{% endhighlight %}

Nothing really horrible, isn't it? Think again. Here comes the spec:

{% highlight js %}
describe('redis transport', function() {
  it('works like fifo queue', function(done) {
    transport.enqueue({a: 1}, function() {
      transport.enqueue({b: 1}, function() {
        transport.dequeue(function(msg) {
          msg.should.eql({a: 1});
          transport.dequeue(function(msg) {
            msg.should.eql({b: 1});
            done();
          });
        });
      });
    });
  });
});
{% endhighlight %}

Getting worse, right? Note that there's no exception handling logic here - it would make the code even more complex and long.

But what bothers me most, is that the test is completely unreadable. In a synchronous language,
you would write (I'll use ruby in this example):

{% highlight ruby %}
describe 'redis transport' do
  it 'works like fifo queue', do
    transport.enqueue {a: 1}
    transport.enqueue {b: 1}
    transport.dequeue.should.eql {a: 1}
    transport.dequeue.should.eql {b: 1}
  end 
end
{% endhighlight %}

It's easy to notice that the payload of the test in ruby is pretty equal to tests code. No endless "function" definitions, loads of curly braces, empty braces and semicolons.

So what can we do? Let's try...

## CoffeeScript
Let's write our transport in coffeescript:

{% highlight coffeescript %}
# redis.coffee
module.exports = class Redis
  constructor: (url) ->
    client = redis.createClient(url.port || 6379, url.hostname)

  enqueue: (message, callback) ->
    client.lpush(inQueue, JSON.stringify(message), callback)

  dequeue: (callback) ->
    client.rpop(inQueue, (err, data) ->
      throw err if err
      var message = JSON.parse(data);
      callback(message);
{% endhighlight %}

And the spec:
{% highlight coffeescript %}
# redis_spec.coffee
describe 'redis transport', ->
  it 'works like fifo queue', (done) ->
    transport.enqueue {a: 1}, ->
      transport.enqueue {b: 1}, ->
        transport.dequeue (msg) ->
          msg.should.eql {a: 1});
          transport.dequeue (msg) ->
            msg.should.eql {b: 1}
            done()
{% endhighlight %}

Looks much cleaner to me. But! Coffeescript has some problems:

- Additional compilation step, which means 
  - no monkey patching
  - compilation infrastructure should be in place
- Hell to debug
  - Coffeescript rarely considers providing you the line number of the error
  - Node.js executes Javascript and thus it's errors don't correspond 1:1 to your code

So I don't want to use CoffeeScript (though it's amazing for rails client-side development),
and the callbacks syntax makes me made. People say promises to the resque. Let's try.

## Promises

{% highlight js %}
// redis_queue.js
var Q = require('Q');

module.exports = function(url) {
  var client = redis.createClient(url.port || 6379, url.hostname);

  this.enqueue = function(message) {
    var deferred = Q.defer();

    client.lpush(inQueue, JSON.stringify(message), function(err){
      if (err) throw err;
      deferred.resolve();
    });
    return deferred.promise;
  };

  this.dequeue = function() {
    var deferred = Q.defer();

    client.rpop(inQueue, function(err, data){
      if (err) throw err;
      var message = JSON.parse(data);
      deferred.resolve(message);
    });
    return deferred.promise;
  };
});
{% endhighlight %}

So the server-side code becomes a bit more complex, but let's see if it will help us with the tests:

{% highlight js %}
// redis_spec.js
describe('redis transport', function() {
  it('works like fifo queue', function*() {
    transport.enqueue({a:1})
      .then(function() {
        transport.enqueue({b: 1})
      })
      .then(function() {
        transport.dequeue()
      })
      .then(function(msg) {
        msg.should.eql({a: 1})
        transport.dequeue()
      })
      .done(function(msg) {
        msg.should.eql({b: 1});
      });
  });
});
{% endhighlight %}

So what we've got here? Test's overhead doubled, the code is still unreadable, though 
exceptions can be handled easier, and at least we won't have infinite indentation.
But it doesn't impress me much. It looks like code horror, it reads like code horror, it's code horror indeed.

Seems like we're doomed. Seems like Javascript was born to be an unreadable-loads-of-code-horror language. But there's some light at the beginning of year 2015. Yep. This is when the
ES6 standard is estimated to be approved.

## ES6 Javascript generator functions & yield

But we already have it functioning in development builds of node.js. Just install the development branch of node and run it like this:

{% highlight sh %}
  node --harmony
{% endhighlight %}

So we can still run the server side logic on the release branch of node, but use generators in our tests.
Just add this to your package.json:

{% highlight json %}
{
  "...", "...",
  "scripts": {
    "test": "node --debug --harmony ./node_modules/.bin/grunt test",
    "start": "node --harmony ./node_modules/.bin/grunt serve"
  }
}
{% endhighlight %}

And now assuming we leave the promises code in redis_queue.js, we can use "suspend" package to write our tests like:

{% highlight js %}
// redis_spec.js
describe('redis transport', function() {
  it('works like fifo queue', function(done) {
    suspend.run(function*() {
      yield transport.enqueue({a:1});
      yield transport.enqueue({b:1});
      (yield transport.dequeue()).should.eql({a: 1});
      (yield transport.dequeue()).should.eql({b: 1});
    }, done);
  });
});
{% endhighlight %}

## Making it DRY

The test itself now reads much better. But what bothers me is the boilerplate code:

{% highlight js %}
function(done) {
  suspend.run(function*(){
    ...
  }, done);
}
{% endhighlight %}

Can we get rid of it? Yes we can.

Let's add this to a file called "test_helper.js":

{% highlight js %}
var suspend = require('suspend');

// Add suspend support to "it-blocks"
var originalIt = it;                  // remember the original it
it = function(title, test) {          // override the original it by a wrapper

  // If the test is a generator function - run it using suspend
  if (test.constructor.name === 'GeneratorFunction') {
    originalIt(title, function(done) {
      suspend.run(test, done);
    });
  }
  // Otherwise use the original implementation
  else {
    originalIt(title, test);
  }
}
{% endhighlight %}

And then our test finally can look like this: 
{% highlight js %}
require('../test_helper.js');

describe('redis transport', function() {
  it('works like fifo queue', function*() {
    yield transport.enqueue({a:1});
    yield transport.enqueue({b:1});
    (yield transport.dequeue()).should.eql({a: 1});
    (yield transport.dequeue()).should.eql({b: 1});
  });
});
{% endhighlight %}

Compare it to the callbacks/promises horror. Looks better, ha?
