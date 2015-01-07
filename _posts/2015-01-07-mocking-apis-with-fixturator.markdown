---
layout: post
title: Mocking APIs with Fixturator
author: Josh Priestley
tags: Coding Testing iPlayer
---

At iPlayer we use several back end APIs to build the pages you see on the front end. We test with unit, integration and acceptance tests; Fixturator is one way we saved ourselves a bunch of time and pain at all of those levels.

Fixturator was written specifically for iPlayer and is of very little use to anyone else. It's our way of implementing Live Fixtures, and the reasons why should be applicable to any app consuming an API. As an application Fixturator is a steaming pile of untested hackery, however, the principles behind it  have helped us enormously.

## We mock services because

They suck. No matter which service and who looks after it, including you, it will suck. Murphy's law dictates that the 0.0001% error rate will always happen while your tests are running or while you're trying to impress someone with your excellent new feature. Relying on the API being available, correct and fast to do your development and testing will make a fool of you. Every time.

## Mock them. 

Having local copies of the data you need to render your page, process your stats etc. means you can iterate a whole lot faster and makes TDD and other methods a doddle. There's no scheduled downtime or flaky staging environments when you've got a local copy.

This is a little bit of a misnomer though. Mocking an entire API is hard, almost impossible to do well and usually counter productive. If you've managed to mock every part of the API and all it's eventualities, well done, you've rebuilt the API, statically. This is what we had in our code base for a long time, a massive amount of JSON files in various states of relevance. The things you actually want to cater for are: changeable data and edge cases. 

## Mocking 101

Before you can `expect` anything of your code you need to be certain of the data, which is what make changeable data difficult. For instance, checking that we display a list of "Most Popular" items is hard because there's no constant (although Eastenders is close). By mocking that feed we had a hard list of most popular items, automated tests can run riot checking those values.

Edge cases are also fairly easy to automate once mocked, take the feed, save it 20 times for your different data structures and variations; create tests around them and rejoice, you have full coverage. 

Then hope nothing ever changes.

## Ooops

Something Changed. The first thing that you notice is user issues. All of your tests are passing in the glorious walled garden of fixtures while your users are looking at [Burning Clowns]. Turns out your API removed a field that you thought you weren't using, you were wrong. You probably should have checked it in some kind of integration test, "But wait" you say (probably) "I thought my automated tests were integration tests! They use the API data." 

Code rot applies just as much to these fixtures as to any other piece of your code. More so in some cases, as these are more like comments, nobody is reading them until they're wrong. 

## Enter: Fixturator
Cue heroic music, explosions and swooning on lookers. Maybe a little over the top.

The idea is to not hardcode things that you don't care about. That field that disappeared earlier, until it vanished and broke things we didn't care about it. Our tests continued to pass because we'd hard coded it in the fixture. If you could use live data the whole time you'd never hit these issues, but then we're back to the top of this rant, we like mocking. 

Changing the live data on the way through is the key. 

Grab the live data, modify it in **and only in** the ways you care about, then save it and use it. Next time you want to see that edge case, grab the live data again and modify it. All the things you truly don't care about can change underneath you, but you'll break early if your assumptions are wrong. Fields can come and go, whole data structures can change, thus proving you don't care, or that you cope with it. 

A small example:
We want to check the most popular list is displayed properly, by checking the title, image URL and link HREF:

 - Grab the live most popular feed
 - Change the title of the first episode to "Cake"
 - Change the image URL to "Cake image"
 - Change the link URL to "http://Cake_test_url"
 - Save

We can then simply check that "Cake" is on the page where the title should be, and leave gracefully. If something weird changes then the appropriate code will either blow up or work, as happens in an integration test. We're still testing the other parts of that feed as if it were an integration test.

Getting live data each time you run the tests is a little cumbersome though, what if we cache it for an hour? We can code and test for an hour and only hit the live system once, then we refresh the cached feeds and go round again. In our CI environments we don't cache the feeds at all, which means we get live data within our tests for every application build and Pull Request test run. We do have an override, to force using the cache, so we can still build if the API is having Bad Day. As long as the live API is available and correct *sometimes* then we don't have a problem. If it's not, well, we have a problem.

## In anger?

We've been using this solution in our code base for 9 months now (March 2014) as the primary way of mocking our data service. We first used it in our web front end to replace our existing static fixtures, then began using it to weed out more edge cases. We then realised it could have a lot of use in our data library too, whose main job is fetching and representing JSON from the API, which makes a lot of the library difficult to unit test. For instance, constructing a PHP class for each kind of JSON object; to unit test this means mocking out the fetching and providing massive amounts of JSON through your tests. Adding in Fixturator meant that we could store and run a massive amount of permutations to check that logic.

It's reaped all the benefits of mocking, but it's also helped a massive amount with being able to answer questions about our automated and manual test coverage. Previously, either certain special edge cases were just tested once and left, or their cases were hidden in a mountain of JSON files which may or may not work anymore. Now it's trivial for anyone in the team to make the site behave with any edge case they like.

Everyone in the team contributes to the fixtures, whether they have JavaScript skills or not. We keep them short and easy, using a simple and defined API, so creating a new one is mostly a copy, paste and modify job. They are reviewed as they enter the code base, as rigorously as the rest of our code, which is to say, sometimes stupid things creep in. We've had a few occasions where silly things have crept in to our fixtures and caused the build to fail but, because they're small easy scripts, the cleanup was a few seconds and we were on our way again with a lesson learned.


----------

## Implementation

The initial idea was for some kind of proxy service, it would make the call for the live feed, then modify it in predefined ways. This meant you wouldn't care that it was happening, we'd just have every feed available in every permutation. Awesome. That does mean however that you've made an API, and we established that all APIs suck and will make a fool of you. We also didn't need all the extra functionality provided by a proxy fixturing method, like cache checks and authentication.

A little hacking on a coach proved that it really isn't hard to make JavaScript read JSON (who knew?). The solution was to create small scripts using a well defined library to request the live data, modify it and save it to a file. JavaScript is easy in small doses, 50% of the team already knows it, the other 50% wouldn't take much coaching for such simple scripts. 

The harder bit then was creating a sensible API with which to manipulate the data. The obvious way was to define a block of JSON to "merge" with the live data, which would have worked but is very hard to read and maintain. We decided on a good set of base helper method such as:
 
 - `fixture.addEpisode()` 
 - `fixture.getElement()` 
 - `fixture.removeAllItems()`

`fixture` here is the base feed in JS "class" form so each feed type could do the appropriate task when it was asked to `addEpisode`, given where and how episodes are structured in that feed. The same with `removeAllItems`, each feed has "items" in different places and of different types.

A quick example of a fixture:

{% highlight javascript %}
module.exports = function (creator, fixtureName) {
    return creator.createFixture('categories/films/highlights').then(function (fixture) {

        fixture.getEpisode().set({
            title: 'Hai from the fixture'
        });

        fixture.save(fixtureName)
    })
}; 
{% endhighlight %}

Another important part was that the API shouldn't block any kind manipulation. Feeds such as "Schedule" didn't have any fixturator support when it was released by our API, but it didn't stop anyone. They could simply do `fixture.data....` to get access to the feed's raw object and make any direct modifications they wanted. It didn't look as pretty, but they could continue working when circumstances changed. Then we could implement those new features upstream back into Fixturator to make them pretty. 

Keeping all these fixture scripts small and reproducible meant they could be versioned along with the rest of the code on their own. Their output should be very strictly reproducible on any machine at any time, so anyone can still run the tests. This left zero hard coded JSON in the code base for that API. 

## All in All

APIs are bad. Mocking is good. Live Mocking is the bomb. I can't really overstate how much pain this saved us, and how much more visibility we have of our apps edge cases. There are lots of "API Mocking" solutions out there you could strap onto your app to do faking, or, write a small crap JS library to do it for you. Either way, scripted changes to live data are the way to go.

[Burning Clowns]: {{ site.base_url }}/images/burning_clown.jpg

