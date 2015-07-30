---
layout: post
title: Breaking an Open Source Monolith
comments: true
---

We (at [Wikia](http://www.wikia.com/)) recently started down the path of
breaking apart the [MediaWikia](https://www.mediawiki.org/wiki/MediaWiki)
Monolith that the platform is built upon. There are
[many](http://engineeringblog.yelp.com/2015/03/using-services-to-break-down-monoliths.html)
[examples](http://developers.soundcloud.com/blog/building-products-at-soundcloud-part-1-dealing-with-the-monolith)
of organizations breaking apart their monoliths, but this
one I think is interesting because it's happening in the open.

![Easter Island Monolith](/assets/monolith.png)  
<sub><sup>(["AhuTongariki" by Ian Sewell - IanAndWendy.com Photo gallery from
Easter Island. Licensed under CC BY 2.5 via Wikimedia
Commons](https://commons.wikimedia.org/wiki/File:AhuTongariki.JPG#/media/File:AhuTongariki.JPG)</sup></sub>

Wikia's current source code policy is open by default. That means almost all of
the company's source code is publicly visible. For the curious, it's all
[here](https://github.com/Wikia/). The MediaWiki portion is
[here](https://github.com/Wikia/app).

Earlier this year we decided to start building the infrastructure to enable more
service development within the organization and accelerate our movement towards
a mobile friendly experience with different types of contribution. One of the
first pieces of functionality that we wanted to separate from the monolith was
user preferences.

A lot went into building the infrastructure, but the part that I want to focus
on here are some of the details about how we are preparing to replace existing
functionality in the monolith with a service while preventing any
major disruptions to our users. 

One of the first steps was to build the service for user preferences that will
replace the existing functionality within MediaWikia. We decided to start
building our service infrastructure in Java using
[dropwizard](http://www.dropwizard.io/). The service
repository is [here](https://github.com/Wikia/pandora) and the user-preference
portion is
[here](https://github.com/Wikia/pandora/tree/master/service/user-preference).

The next step was to refactor MediaWikia to separate out the functionality that
we wanted to decouple from the monolith. Ideally we would have a well defined
seam that provided an interface that we could swap out. By seam I'm referring to
the place where two parts of the software meet and where something can be
injected. I first encountered this term in _[Working Effectively with Legacy
Code](http://www.amazon.com/Working-Effectively-Legacy-Michael-Feathers/dp/0131177052)_
by Michael Feathers. Unfortunately there was no seam and to make matters worse,
preferences were muddled together with user attributes and flags in the concept of
an option within the `User` object in MediaWiki.

If I'm not mistaken this is the kind of problem that Eric Evan's was trying to
avoid with the notion of a [Bounded
Context](http://martinfowler.com/bliki/BoundedContext.html). In the case of
MediaWiki an "option" had come to mean different things depending on the
context. In some places an option referred to a marketing preference, in others
it referred to the user's website in still others it referred to whether or not
their account had been disabled.

To solve this problem, we decided to separate the option concept into
preferences, attributes, and flags and replace the call sites to `getOption`
with `getPreference`, `getAttribute` and `getFlag` respectively. This effort
took 7 people 250+ commits and touched 228 files. You can see the details in
this [pull request](https://github.com/Wikia/app/pull/7648).

Once we hand the context clearly defined we could create the seam where the
service could be injected. To make this move cautiously, and allow us to back it
out if we got into to trouble, we created a [persistence
interface](https://github.com/Wikia/app/pull/7607) to preferences.

This interface created the seam but still used the database
underneath as the only concrete implementation. We then used a [feature
flag](https://github.com/Wikia/app/blob/1cdfad4aac1e7397a733aa02547e23caf2f9986f/includes/User.php#L2510-2529)
to toggle the behavior of the `getPreference`, `getAttribute`, and `getFlag`
functions to use the legacy `getOption` or the new persistence interface.

We have been slowly increasing the communities we are deprecating `getOption` on
to make sure our extensive replacements are working as expected. Several issues
have surfaced that caused us disable the feature flag, fix the issues then try
again.

Once we are confident we have worked out the issues with the context separation,
we will create a client to the user preference service and [inject
it](https://github.com/Wikia/app/tree/dev/lib/Wikia/src/DependencyInjection). We
will also use a feature flag to slowly swap out the database
implementation with the service implementation so we can monitor the behavior of
the new service as we increase the traffic.

Whew! All of that to replace a small piece of thirteen year old legacy system!
