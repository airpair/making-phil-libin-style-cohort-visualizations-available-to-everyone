One day I saw a link to [Phil Libin’s talk in 2010](https://vimeo.com/11932184#t=705s) where he described Evernote’s growth, visualized in signup cohorts.

<table style="max-width: 640px;">
<tr>
<td><a href="https://vimeo.com/11932184#t=705s"><img style="padding: 5px;" src="http://blog.usercycle.com/content/images/2014/Aug/1.png"></a></td>
<td><a href="https://vimeo.com/11932184#t=705s"><img style="padding: 5px;" src="http://blog.usercycle.com/content/images/2014/Aug/2.png"></a></td>
</tr>
<tr>
<td><a href="https://vimeo.com/11932184#t=705s"><img style="padding: 5px;" src="http://blog.usercycle.com/content/images/2014/Aug/3.png"></a></td>
<td><a href="https://vimeo.com/11932184#t=705s"><img style="padding: 5px;" src="http://blog.usercycle.com/content/images/2014/Aug/4.png"></a></td>
</tr>
</table>

The presentation was amazing on two levels. First of all, Evernote itself has amazing retention. But secondly, the visualizations were stunning — and told a visual story that didn't require any numbers to understand what was going on.

And I couldn’t get the idea out of my head — what if every product owner could see their user activity this way, visually, through Phil’s lens? It’d be so eye opening to see your product this way.

Googling around, I couldn’t find a product that provided this at a low price point. So we did what all self-starting developer/entrepreneurs do — we rolled up our sleeves, and built it ourself.

## Choosing a tech stack

First challenge was to decide where we'd store our raw event data. I had recently heard of [Keen IO](https://keen.io) and it seemed like a perfect fit, because they have a cool [Funnel Analysis API](https://keen.io/docs/data-analysis/funnels/) that would save us a lot of time. 

And we knew that leveraging their system would cut our work down tremendously, because we wouldn't have to deal with building a reliable web service to collect user events. And they had a nice [command line tool](https://github.com/keen/keen-cli) that can be used for data import. Perfect :)

## Building the cohort area chart

We decided to use [Google's Area Chart Visualization](https://developers.google.com/chart/interactive/docs/gallery/areachart) because it has a lot of customization features that quickly got us to something we liked.

Here's a mockup of what we were going for:

<img src="http://blog.usercycle.com/content/images/2015/03/Screenshot-2015-03-01-12-35-39.png" />

## Generating the data

To build a chart like the one above requires a funnel query **for every data point**. And depending on how much underlying data that's being querying, and the complexity of the query, these queries can take a "non-trivial" amount of time.

So this introduces some complexity, but nothing in life is free.

We had to build an app to manage the execution of these queries, and execute them in accordance with Keen IO’s API limits.

## Building the app

We’re big [Meteor](https://www.meteor.com/) fans, so we decided to use Meteor (hosted on [Modulus.io](https://modulus.io/)), and put our [MongoDB](http://www.mongodb.org/) on [Compose.io](https://www.compose.io/).

Also, to keep things simple, we used a node library called [Monq](https://github.com/scttnlsn/monq) which keeps track of the job queue right in MongoDB.

Also, we didn't have a designer for the project, so we decided to follow Google's [Material Design spec](http://www.google.com/design/spec/material-design) via [Polymer](https://www.polymer-project.org/).

So our process looked like this:

<img src="http://blog.usercycle.com/content/images/2015/03/Screenshot-2015-03-18-20-41-38.png">

*Note: Not only did we need to build the data initially, but we want to keep the charts up-to-date throughout the day, refreshing the data hourly.*

## Limiting data points

To limit the # of data points per chart, we only show a maximum of 30 cohorts, and lump data older than 30 cohorts old into a "catch-all" bucket.

<img src="http://blog.usercycle.com/content/images/2015/03/Screenshot-2015-03-18-20-48-12.png">

Our max # of data points became: `30 x 29 / 2 + n` where n represents the number of periods for the catch all cohort. 

In the chart above, this would be around 500 data points & queries. If we allowed 100 cohorts, it would require about 10x more processing time (5000 queries).

But this innocent little feature created quite a bit of extra work for us. As each new cohort comes into existence, we have to merge the oldest cohort into this catch-all bucket, and prune the old data.

## Batching queries

Keen IO has a query batching feature, allowing us to bundle multiple queries up into a batch to improve efficiency. To take advantage of this, we built a process to bundle individual queries into batches of 20.

## Large revenue cohorts workaround

**This part is pretty technical, but it is an example of the realities of building systems like this.**

In one visualization we built, we show how much revenue each cohort generates, for each time interval (revenue by cohort).

This requires us to pass a collection of userIds to a sum query. For small products with few paying users, this method works just fine. But we discovered that for big products, the query will fail if there are too many userIds passed to the query.

So to compute just one data point in one chart, it's possible that we may need to run several, dozens, or even hundreds of queries, batching userIds into groups of 50, then summing up the data at the end. JUST FOR ONE DATA POINT on a chart. Crazy.

## Adding filters

Keen IO has a powerful filters feature, allowing you to filter results based on event properties. Eventually we decided it was time to support that. Didn’t seem too hard at first, but let's see how many files adding the feature touched:

<img src="http://blog.usercycle.com/content/images/2015/03/Screenshot-2015-03-18-11-07-37.png">

In fact, the feature touched 37 files, in 195 places. So the feature turned out to be not so easy, and increased complexity throughout our codebase. This is a great example of how an innocent feature can be quite complicated underneath the surface.

## Adding multiple retention events

This feature wasn’t actually too hard, but did require us to use the ‘inverted’ option in the funnel query API, which slows down queries. So we need to add some logic to only use it when it's required.

## Where do we go from here?

We've been building the product for about three months. USERcycle is launched, and users are signing up and setting themselves up independently, so we're thrilled about that. We’ve been evangelizing the heck out of Keen IO. (We love the Keen IO team, so this really isn’t much of a burden.)

We’re working now to replace the quick-and-dirty, minimal Google Visualizations we whipped up with something a little nicer, using D3.

We’re also tuning our data processing engine to scale up to be capable of running millions of queries on Keen IO each day.

We feel lucky to have partners like Modulus.io, Keen IO, Compose.io, and MongoDB as strong partners to scale with us.

## The moral of the story...

> "Complexity is unavoidable when you're creating real value." -Ry Walker

THANK GOODNESS we decided to limit the complexity of our system and build on top of Keen IO. We found all the complexity we could ever want on the analytics side of things — just generating cohort data and keeping it fresh.

And of course I invite you to [sign up for USERcycle](https://usercycle.com), if you're intrigued.

