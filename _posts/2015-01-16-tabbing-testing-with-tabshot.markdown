---
layout: post
title: Tabbing Testing with TabShot
author: Josh Priestley
tags: Coding Testing iPlayer
---

[TabShot](https://github.com/joshrp/tabshot) is a tool I created to make videos of websites (sounds exciting I know). More importantly, videos of what it looks like to tab through a website. Here's the iPlayer homepage:

<video controls="" src="http://ldn.mrjrp.com/tabshot/homepage.mp4" style="width:50%;"></video>

## Why
Supporting tabbing on a website is hard; so hard that most websites don't bother "supporting" it. Letting the browser make its best guess at how your site should work is how most sites will deal with keyboard navigation. However, after a mix of image links with no descriptors, hidden links and a DOM order that a drunk would call out of alignment mean that getting around the web without a mouse is painful. People with visual impairments or motor impairment do not really have a choice of point and click so rely on DOM structures and site intentions being easily interpreted by the browser.

### Some background

Accessibility considerations on the web front end is something reasonably new to me. They've only been on my radar for about a year so I'm still getting to grips with them. The BBC itself has a big focus on accessibility, not just because it's "The Right Thing To Do" but because it has an obligation to; people who rely on screen readers for instance, pay a license fee for BBC services and they have every right to access them. 

This means we have some amazing people making sure those things happen, [Henny Swan](https://twitter.com/iheni) and [Ian Pouncy](https://twitter.com/IanPouncey) (there are, and have been, many others) have been the main contacts for iPlayer during my term. They are essentially internal consultants on making things accessibile. They help us assess whether designs are even nearly accessible, they also shout at us when we've managed to get all the way to Live with something that isn't. 

## Testing

Even with an awesome team focusing their energy on it, making a site accessible is hard, keeping it accessible is a challenge and a half. The slightest change to your markup and styling could have large consequences on your accessibility. Making sure you don't break tabbing with every change is hard; and by "break" I include making sure it's sensible and not infuriating.

A quick example: A modal dialog that has been written perfectly (never happens), is changed slightly; the "close" button is changed from a `<button>` to an `<a>` tag in the process. This doesn't seem like anything but a semantic problem, until you realise that `<a>` tags without a `href` aren't focusable. Your tab users are now stuck in an loop, unable to close the box without a page refresh. Mouse users can happily click "close" or click anywhere else in the page to dismiss it. This is a problem. This kind of constant change is still an unsolved problem for us right now.

It's hard enough to test tabbing manually, especially when you consider all the edge cases of your pages too. Somebody has to sit and diligently tab through the entire page in multiple states, making sure that all the links are focusable and it's in a sensible order. This can become quite subjective, which is poison to manual testing.

Automating these tabbing tests is nigh on impossible, they're partially subjective, non-defined behaviours that with even the subtlest of changes in behaviour, can be rendered utterly useless. Not something I want to try and automate. 

## The Answer

So this is the part where I'd proclaim the answer... Well this is awkward. 

We're still figuring this out constantly, thinking we've nailed it and stopped breaking things, then something silly comes along. This was why we needed TabShot, so we can cut down massively on the amount of time it takes to check pages. Coupled with [Fixturator]({{ site.base_url }}/2015/01/07/mocking-apis-with-fixturator.html) we can grab videos for our major pages in a variety of states. 

----------

## How

TabShot is built as a single entry point with a few parameters and backed by ready made Docker images. It has a few jobs to execute:

 1. Hit a URL
 2. Screenshot
 3. Hit Tab
 4. GoTo 1 until limit
 5. Resize and compress screenshots
 6. Encode video
 7. Cleanup

Apologies for the `GoTo` functional code is hard in markdown.

Steps 1- 4: So I decided on Selenium Grid after finding some issues in the usually awesome PhantomJS, [focus states don't appear in screenshots](https://github.com/ariya/phantomjs/issues/12796) (which is kind of a big issue for TabShot). Selenium Grid is somewhat meatier and not quite as easy to get going; they've done some [awesome work to get selenium in docker though](https://github.com/SeleniumHQ/docker-selenium) which made most of it easy. 

Step 5: This is way into ImageMagick territory, docker proves useful again with [this image](https://github.com/jujhars13/docker-imagemagick). 

Step 6: Easily taken care of by ffmpeg, unsuprisingly also [in a docker image](https://github.com/jrottenberg/ffmpeg).

Using Docker to run things like Selenium Grid and ffmpeg meant you don't have to install the bajllion dependencies <sup>[citation needed]</sup>, realise there's no package for the version that supports a thing, then end up wasting a day compiling ffmpeg, going insane and throwing yourself off a building. As long as you can get Node and Docker, you're set, and I'm considering moving the Node runner part into a docker container too to remove that dependency using something like [joyent/docker-node](https://github.com/joyent/docker-node).

It's customisable to any site, if there are particular things to take care. The BBC's cookie banner for instance can be closed before you start; or enable and disable things as you tab to show interactions too. The actual code which runs over the page is a simple JS Selenium script in the working directory so it's easily available for customising.

Happy Tabbing.

