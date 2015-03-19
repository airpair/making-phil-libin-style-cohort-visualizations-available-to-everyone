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

The presentation was amazing on two levels. First, Evernote has very impressive retention story. Second, the visualizations were compelling; they told a story, visually without needing a lot of numbers. By just looking at the shapes, you could see what was going on.

So then I couldn’t get the idea out of my head — what if every product owner could see their user activity this way, visually, through Phil’s lens? I definitely wanted to see my products this way.

Googling around, I couldn’t find a product that provided anything like this, at an affordable price point. So we did what all self-starting developer/entrepreneurs do — we rolled up our sleeves, and built it ourself :)

The way systems like this work is that you have to reliably get events from the app that is being analyzed into a central database that is accessible to the analytics engine.

## Planning out the system

We figured we'd build something that looks like this:

<img src="http://blog.usercycle.com/content/images/2015/03/Screenshot-2015-03-18-20-41-38.png">

Not only did we need to build the data initially, but we want to keep the charts up-to-date throughout the day, refreshing the data hourly.

Four main components to the system:

1. **Event Collection** — getting user events from your app, and into a database that we can access.
2. **Analysis Engine** — run queries on the raw data and create data that is ready to be visualized. 
3. **Visualization** — we want to see the computed data in all it's glory, right?
4. **Dashboard** — if we want to make this system accessible to everyone, we need a place where people can signup and configure their project.

Here's a mockup of the sort of visualization we were going for:

<img src="http://blog.usercycle.com/content/images/2015/03/Screenshot-2015-03-01-12-35-39.png" />

Next, I'll share some details on how we built these components.

## 1. Event collection 

If you're not familiar with what I mean when I say "user events" it's simply an object:

```
  name: 'Viewed Dashboard'
  userId: 123
  platform: 'iOS'
```

For enterprise SaaS, there could be a few more properties:

```
  name: 'Viewed Dashboard'
  userId: 123
  accountId: 456
  locationId: 789
  platform: 'iOS'
```

Popular products could be sending millions of events per month (or day!). Building a system to both reliably and efficently collect this data and make it reliably and efficiently queryable is **a lot of work** that I wanted to avoid.

### Choosing Keen IO

I had recently heard of [Keen IO](https://keen.io), and after looking at it again, it seemed like a perfect fit. They are a venture-backed startup that do two things, but do them well: 1) collect event data 2) provide a great API to query the data. The icing on the cake for me was their [Funnel Analysis API](https://keen.io/docs/data-analysis/funnels/) — I knew that building on top of Keen IO would not only save us a lot of time, but also allow us to focus our energy on the analytics engine and visualization layers.

We knew that leveraging their system would cut our work down tremendously, because we wouldn't have to deal with building a reliable web service to collect user events. They also have a nice [command line tool](https://github.com/keen/keen-cli) that can be used for data import, which was on our list of needs. Perfect :)

By choosing Keen IO, we checked off the entire event collection bucket, and gave us a great API on which to build our analysis engine.

One component down, three to go.

## 2. Analysis Engine

To build a chart like the one above requires a funnel query **for every data point**. And depending on how much underlying data that's being querying, and the complexity of the query, these queries can take a "non-trivial" amount of time.

We had to build an application to manage the execution of these queries in accordance with Keen IO’s API limits. Also, to keep things simple, we used the [Monq](https://github.com/scttnlsn/monq) node library, which keeps track of the job queue right in MongoDB.

We’re big [Meteor](https://www.meteor.com/) fans, so we decided to use it (hosted on [Modulus.io](https://modulus.io/)), with our [MongoDB](http://www.mongodb.org/) hosted on [Compose.io](https://www.compose.io/).

### Batching queries

Keen IO has a query batching feature, allowing us to bundle multiple queries up into a batch to improve efficiency. To take advantage of this, we built a process to bundle individual queries into batches of 20.

### Large revenue cohorts workaround

One of our analysis shows how much revenue each cohort generates, for each time interval.

Implementing this in Keen requires two steps:

1. Extract the userIds who had a revenue event in the time period, for the cohort that's being currently analyzed.
2. Pass these userIds into a [sum query](https://keen.io/docs/api/reference/#sum-resource).

We discovered that for products with lots of paying users, the query will fail if there are too many userIds passed to the query.

So to compute *just one data point in one chart*, we need to run `n` queries by batching userIds into groups of 50, then summing up the data at the end. JUST FOR ONE DATA POINT on a chart. Crazy.

### Filters

Keen IO has a powerful filters feature, allowing you to filter just about every query based on event properties. We didn't support filters at first, to keep things simple, but eventually we decided it was time. Didn’t seem too hard conceptually, but let's see how many files adding the feature touched:

<img src="http://blog.usercycle.com/content/images/2015/03/Screenshot-2015-03-18-11-07-37.png">

In fact, the feature touched 37 files, in 195 places. It turned out to be not so easy, and did increase complexity throughout our codebase.

This is a great example of how an innocent-looking feature can be quite complicated underneath the surface.

### Limiting data points

To limit the # of data points per chart, we only show a maximum of 30 cohorts, and lump data older than 30 cohorts old into a "catch-all" bucket.

<img src="http://blog.usercycle.com/content/images/2015/03/Screenshot-2015-03-18-20-48-12.png">

Our max # of data points became: `30 x 29 / 2 + n` where n represents the number of periods for the catch all cohort. 

In the chart above, this would be around 500 data points & queries. If we allowed 100 cohorts, it would require about 10x more processing time (5000 queries).

But this innocent little feature created quite a bit of extra work for us. As each new cohort comes into existence, we have to merge the oldest cohort into this catch-all bucket, and prune the old data.

## 3. Visualization

For the MVP, we decided to use [Google's Area Chart Visualization](https://developers.google.com/chart/interactive/docs/gallery/areachart) because it has a lot of customization features that quickly got us to something we liked.

This was a situation where we could get to 80% of our vision with 10% of the work — and we took the opportunity to cut a corner, because the analysis engine really sucked up our time.

## 4. Dashboard

This too was fairly straightforward, but you should never underestimate the amount of time it takes to think through and build a good user experience for any app.

We didn't have a designer for the project, so we decided to follow Google's [Material Design spec](http://www.google.com/design/spec/material-design) via [Polymer](https://www.polymer-project.org/).

With each new feature we add to the app, we find ourselves refactoring the user experience to some degree.

## Where do we go from here?

We've been building the product for about three months. USERcycle is launched, and users are signing up and setting themselves up independently, so we're thrilled about that. We’ve been evangelizing the heck out of Keen IO. (We love the Keen IO team, so this really isn’t much of a burden.)

We’re working now to replace the quick-and-dirty, minimal Google Visualizations we whipped up with something a little nicer, using D3.

We’re also tuning our data processing engine to scale up to be capable of running millions of queries on Keen IO each day.

We feel lucky to have partners like Modulus.io, Keen IO, Compose.io, and MongoDB as strong partners to scale with us.

## The moral of the story...

> "Complexity is unavoidable when you're creating real value." -Ry Walker

THANK GOODNESS we decided to limit the complexity of our system and build on top of Keen IO, Google Visualization, and Meteor. We found all the complexity we could ever want building the analytics engine, generating cohort data and keeping it fresh.

---

**Shameless self promotion:** If you have a product with a signup button, and would like to see your product through Phil Libin's lens, I invite you to [sign up for USERcycle](https://usercycle.com).

