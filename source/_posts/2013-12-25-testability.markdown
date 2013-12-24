---
layout: post
title: "Testability"
date: 2013-12-25 02:48:19 +1300
comments: true
categories: 
---

The more time in software development, the more I come to understand the importance of testing.  Unit tests, functional tests, integration tests...the important thing is to *have some tests*.

When I was younger and stupider (all of, oh, 2-3 years ago) the benefit of tests seemed more nebulous.  And stuff like [DI](http://en.wikipedia.org/wiki/Dependency_injection)?  That's some crazy Java crap.  Ain't nobody got time for that stuff!

## I was kind of dumb

So now I've got this fairly massive app (well, massive for a team of 2) we've been working on for a couple of years.  And I'm starting to really wish I'd been a lot more into testing when we started.  That little todo webapp you threw together to learn Ember or whatever probably doesn't need a lot of tests.  But we've got something like 9k lines of Coffeescript these days, plus endless Jade templates, a bunch of Stylus stylesheets, and a ton of random scripts, Ansible playbooks, config files, whatever.  And a complicated architecture.  We need tests!

...and we don't really have them.

## Meh, just add them!

If only it were that easy!  ...well, it kind of is.  We do have a server (a lightweight Express app mostly just exposing a somewhat RESTful API) and it is pretty easy to add tests to it.  Take a scooping helping of Mocha, a dash of Supertest, and a liberal coating of Nock, shake well, and you get some very nice, very fast, very useful functional tests.


``` coffeescript A simple test
app = express()
config = require './config'
require('./routes')(app, config)
require './nocks'

describe 'The  server', ->
  before (done) -> app.listen config.PORT, config.BIND, (err) -> done err

  it 'should reject logged out requests', (done) ->
    request(app)
    .get('/api/test')
    .expect(401, done)
```

Basically, I require my server code, spin up a testing instance, then make a serious of requests to the API endpoints, with all the calls to other servers mocked out with nock.  And it's amazing.

But our client is written with KnockoutJS, and it's not very well organised.

##The options

### Unit tests

This makes sense.  The server can be unit tested trivially, and Knockout is based on the MVVM pattern.  Just require the app, and test some view models!

**Problem:**  Over the months and now years, a few "quick hacks" have tangled the viewmodel code with the view code.  We have (just a few!) cases where viewmodel functions are referencing the window, or document globals directly, or using jQuery selectors.  This won't work in a headless unit test.

### Um, unit tests with JSDOM?

Doesn't implement enough.  The code relies on being in a web browser, and refactoring it now, especially without tests (funny how that works) will be a nightmare.

### Okay, ZombieJS!

Zombie is pretty awesome; it gives you a headless browser you can play with.  But like JSDOM, it isn't enough.  The code expects a Webkit browser.

### Webkit?  PhantomJS!

An obvious choice.  We can actually spin up a PhantomJS instance, and hit a dev server, and fetch an actual copy of the app.

But these aren't unit tests any more; we're not into the realm of acceptance or integration tests.  Which is fine, but...

**Problem 1:**  They're really slow.  It's a big, chunky app; it's meant to be cached and only loaded once.  Plus it does a lot of slow processing on initial load, so...  These tests are slow.

**Problem 2:**  The PhantomJS API is a huge pain to use.  Something simple like loading the app, then doing some tests to make sure the login form properly validates usernames and passwords, then logging in, waiting for the initial data to load, and then checking to make sure it loaded properly is, again, slow, but also a huge pain to write.

### Okay, CasperJS?

Not much better.  It's just a painful API.

### Right, Capybara?

Well, it's ruby-centric, and we're not a ruby shop.  But it works, and it has a nice API, and I even got a couple of Lettuce+Capybara tests written.

But it's still very slow to run, and very slow to write.  These aren't a replacement for the unit tests we so desperately need; at most I could write a couple as smoke tests.  (Basically "log into the app, connect to the test DB, and make sure a record displays; if that works it can't be too broken so let's ship it into production!")

But we still need unit tests.  And this path doesn't lead to unit tests.

## Now what?

That's a good question.  I've got over 8k lines of poorly organised Knockout code with no unit tests, and no ability to write unit tests unless I refactor it, which will be a nightmare without...unit tests.

At this point, we wince, accept the technical debt, and push forward, and hope we get the resources for a proper rewrite one day.

## And the lesson is?

***Testability matters***.  Which is why you should be writing tests from day one; not because you need them then, but because one day you'll need them, and having tests guarantees that you can write tests.  I look at the search code now, and I wince, because it's so tightly coupled to, well, everything else, that it's largely untestable now.  But it didn't have to be that way.

Also:  The Knockout docs say almost nothing about testing, and google for Knockout + testing turns up very little.  Culture matters, and I think that one of the big advantages Angular may have over Knockout is nothing technical; it's just the focus the Angular community puts on testing.