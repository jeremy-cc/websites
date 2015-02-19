---
layout: post
title:  "Scaling Tech Support"
date:   2015-02-13 16:07:00
author: Jeremy Botha
categories: scalability support visualisation
---

### Preamble

One of the largest issues facing the Tech team at Currency Cloud is the growing volume of work related to customer and business queries.  When I started here in June 2012 our system was
small and relatively simple - it was possible to get a good understanding of most of the areas of the system in a short timeframe, and it was usually possible to work out what was going
on in any given workflow.

Since then we have shipped a number of large projects - including but not limited to

* Integration with third-party Compliance and KYC services
* A fully-featured account management system which receives feeds from our bank accounts
* An entirely new version of our API

This has greatly increased the surface area of our system, and while we have worked hard to organise our technical workflows so that code resides in the system responsible for the code's domain,
we still face a constantly-growing set of functionality that we need to support for either legislative or business reasons.

Recently, we have reached the point where we have to schedule significant amounts of developer time to provide support to our Client Services, Relationship Management and Payments teams, simply
to keep the system ticking over smoothly.  Ultimately it is us, Tech, who have to delve into the servers and databases with a torch and a club when things go wrong, or when one of our clients believes that things have gone wrong.

Up until now, this has involved logging into a server, and either firing up MySQL if it's a database query or digging through logfiles if it's not.  Either way, if I have to log into a server
in order to resolve a query then I can pretty much guarantee that I'm going to lose 15 minutes of my day, and I can multiply that estimate by a scaling factor of at least 2 if I was doing development work at the time,
because I will need to recover my mental save point in order that that I can resume my development work.

I understandably felt this was not a good use of my time, and I knew from discussions with several other developers here that they felt the same way.  We had projects that needed to be shipped, business-critical functionality to deliver,
and instead we were herding cats - Tech support was, quite literally, preventing us from shipping software - it was preventing us in some cases from shipping fixes to the very items that were causing the tech support in the first place!

### Prior journeys into the darkness

At the end of December 2014, the Currency Cloud Tech team held a hack day.  The developers were basically given free reign - we could run amok for the day, doing whatever we liked, working on whatever technology or idea we wanted to, from 8am till 4pm on company time.
 The only requirement was that anything we did or produced was in some way applicable to the company as a whole.

I decided to see if I could shrink the amount of time I needed to spend hunting workflow issues in the system.  I'd already done some work in this area in my own time, but I wanted to run the idea past the rest of the team and get feedback from them.

At a previous company, I had worked on a massive legacy enterprise Java system, and had spent several months just getting up to speed on the [Big Ball of Mud][bbom] that the system was.  I had found it so difficult to effectively convey the complexity of this system
to non-developers that I decided I needed to do it in pictures.  To this end, I wrote a Java program that used the Java compiler to parse the application source tree and output [DOT][dot] files.  I then fed these through [Graphviz][graphviz] in order to generate [SVG][svg] images which I linked
 into a dependency graph using embedded hyperlinks.  I excluded various core Java classes and common code, and was left with what was effectively a massive interlinked net of nodes that represented the objects in the system, and how many other objects in the
  system used them.

I found that this generated graph was an effective way to represent the complexity of the system - people  could see the (sometimes hundreds) of dependencies on classes in the system, and it made it much easier to motivate my belief that the constant increase in time needed
to deliver features was in no small part due to this complexity.

I was therefore curious to see whether a similar approach would be useful at CurrencyCloud.

### Connecting the dots

Anyone who's worked on a system that's been alive for more than a few months will be familiar with how the system will change.  A significant proportion of this change will turn either out to be unnecessary, or
be a solution for a problem that didn't actually exist in the way originally described.  We have numerous examples of this sort of creep in our system, and while most of it is in legacy code that we are phasing out,
some of it is inherent in the structure of our databases.  This is one of the factors that makes tech support such a headache at times.

There are a couple of common requests we deal with.

* Why is X happening when it shouldn't?
* Why is X not happening when it should?
* Who changed X?
* Who was the last person to change X?
* Who can we blame for X?

The first two could, of course, be answered with "Magic" or "Because of the alignment of the planets" but Customer Service and our Payments team do not always appreciate Tech's sense of humour.
Anyway, all of these classes of question will involve a trip to the production database, and a corresponding spike in our head of Infrastructure's blood pressure.

I realised that if we could just provide a single place that showed both the structure of data in our system, and the ability to find the audit log for each entity in that structure, we could trim a significant amount of time off the duration
of a tech support request.

Our database data is hierarchical - we have some core entities which lie at the heart of what we do - Accounts, Conversions, Payments and Settlements to name a few.  Everything else in our system either permutes these or logs permutations of these.  If we
can represent these structures in their hierarchy, and link to them the audit logs - we've solved a significant portion of our headache.

The trick was going to be to make it usable.  I thought of using some sort of graph, and did some brief investigations on the kind of visualisation libraries available in Javascript, since the application I would be using as the container for my
idea is a Rails 3.2 / Twitter Bootstrap / Knockout application, and I wanted something that would integrate with this.

I found [D3][d3js], and I felt it was pretty much perfect for my needs.

### To the batmobile!

I did some research into the sort of visualisations that were possible with D3, and decided on a [force-directed][d3-force] graph as the most obvious fit, since it would permit me to lay out the primary elements of an object hierarchy graph and
let the secondary elements distribute themselves around it.  There were various challenges I knew I would face, notably trying to find a layout algorithm that would be "good enough" for the vast bulk of the ways in which we'd use the tool.
I also needed to limit the amount of data we showed at any one time, since I wanted the tool to be easy to use.

I started with the simplest use-case, which crops up several times a day.

<pre>
Can we please find out the market rate, client buy and sell amounts and
commission for this conversion: 20141107-OMGBBQ
</pre>

Typically, this request would involve someone logging into the production database and running a database select against the conversion table.  Simple enough, right?

But I'm lazy.  I don't want to have to do this.  I'd far rather use my existing session in our back office application, and simply be able to do my work there.
Even better, I'd like to have a tool that our CS or RM guys could use themselves.  This would save me from a context switch, and would mean they wouldn't have to wait
for my feedback.

So I stared, mocking up a very simple search screen in our backoffice app.

This would perform a search on our system, using the conversion reference provided above, and would return a JSON structure that modelled the conversion and its linked items.  D3
requires a data structure in the format:

{% highlight js %}
    {
        "nodes":[
                N1,
                N2,
                N3
            ],
         "links":[
            L1,
            L2,
            L3
         ]
    }
{% endhighlight %}

Where *N<sub>1</sub>..N<sub>n</sub>* are nodes in the graph and *L<sub>1</sub>..L<sub>n</sub>* are links between the nodes.  This is a pretty easy structure to build, but requires some massaging in our code in order to generate the object tree.

I spent some time working on generating this structure, and identified some additional flags and categories I would need to apply in order to lay out the graph in a meaningful manner.  Firstly, I wanted the root node of the graph to always be in a
similar starting location on the screen.  This makes using the screen easier.  Secondly, I wanted to scale the primary entities and secondary entities in our system differently, to quickly indicate their relative importance in a visual manner.  Thirdly,
I wanted to group objects from similar parts of the system by colour, again to make it apparent that they were related.

The end result is nodes that look something like this:
{% highlight js %}
{
    name: "Trade",
    reference: "20141107-OMGBBQ",
    type: "trades",
    idx: 0,
    group: 0,
    root: true,
    child_count: 3,
    audited: true,
    matchable: true,
    identifier: "<system uuid>",
    primary: true,
    parent: null
}
{% endhighlight %}

D3 uses the index of a node to link it to other nodes via the links structure above, so I needed to maintain the index of the nodes in code so that I could make use of it.  I also needed to indicate the root node to my rendering code,
hence the root attribute.  I added child_count so that I could try to locate the root horizontally on the canvas according to how messy the graph was likely to be.  The audited and matchable flags were state indicators that I would use later.

I wanted to use a radial layout, since this is a compact way to lay out a small object tree of limited depth.  Each node that represented a primary object in the system would be placed at a radial angle from its parent that was determined from the
number of children that parent had.  Polar coordinates were a great choice here - calculate the angle of rotation you need for a node with *N* children, and place each child at *N * angle* degrees from the parent of the children.  Obviously, some work was needed
here to ensure a node didn't overlap its grandparent, so some random factors had to be introduced.   But the end result was a reasonably uncluttered layout for the less complex entities in our system.

<div style="display: block; border: 1px solid #cccccc; padding: 5px;">
    {% image scaling-tech-support/render.png alt="Zeus Render" [no-autosize] %}
</div>
<br>
This graph gives an immediate visual overview of this particular object, which happens to be a conversion.  It gives me the following information:

* It is a non-spot conversion, with a deposit lodged against it
* Only a single payment was made against it
* It formed part of a Settlement - a collection of trades for the same customer that are paid out either in single payments per conversion, or a single bulk payment comprising all payments of a particular currency.
* It was likely booked by a white label partner due to the presence of a broker record against the conversion.

A more complex example might be the following:

<div style="display: block; border: 1px solid #cccccc; padding: 5px;">
  {% image scaling-tech-support/more-complex.png alt="Zeus Render" [no-autosize] %}
</div>

This shows me the following:

* The conversion that I am looking at is a non-spot trade that has had its value drawn down on request by our client.
* Two payments have been issued against this conversion

For a change of pace, here's the object graph of a drawdown itself.

<div style="display: block; border: 1px solid #cccccc; padding: 5px;">
  {% image scaling-tech-support/drawdown.png alt="Zeus Render" [no-autosize] %}
</div>

This shows me the following:

* The Drawdown I am looking at is against a conversion that has likely had multiple other drawdowns performed against it.
* The Drawdown has not yet been closed off, and is still potentially active in the system.

These graphs are very pretty, but of course they don't actually tell me much of substance with regards to the individual entities.

Fortunately, our Back Office system already provides many hooks to get the data for these elements.  I was able to repurpose these in order to provide a crude but useful view into the state of the system, invoked by clicking on
one of the nodes in the graph:

<div style="display: block; border: 1px solid #cccccc; padding: 5px;">
  {% image scaling-tech-support/details.png alt="Zeus Render" [no-autosize] %}
</div>

Two additional sections of functionality are available here.  Firstly, the Funds Matches button, which loads data from our Financial backend, permitting us to determine whether we have received funds from clients and sent out funds for payments, and secondly,
the History button, which permits us to see the audit trail for certain key objects in the system, for example payments:

<div style="display: block; border: 1px solid #cccccc; padding: 5px;">
  {% image scaling-tech-support/history.png alt="Zeus Render" [no-autosize] %}
</div>

While this is still pretty rough, it gives us a quick visual diff over the history of an entity in the system, allowing us to see when changes were made, and answering questions 3, 4 and 5 above.

### Use in anger

This is very much a work in progress, but I have been using it as part of my day-to-day tech support work and I estimate it cuts 20% to 40% of the database lookups I need to do on average.  Some queries with regards to the internal state of the system
can be answered purely from this tool, whereas previously I would have had to inspect upwards of five tables across at least two databases at any one time.  As we work to extend it and make it more useful, I expect to reap additional time savings.

In this case, at least, the old adage that a picture is worth a thousand words remains correct.

[bbom]:             https://en.wikipedia.org/wiki/Big_ball_of_mud
[graphviz]:         https://en.wikipedia.org/wiki/Graphviz
[svg]:              https://en.wikipedia.org/wiki/Scalable_Vector_Graphics
[dot]:              https://en.wikipedia.org/wiki/DOT_%28graph_description_language%29
[d3js]:             http://d3js.org/
[d3-force]:         https://github.com/mbostock/d3/wiki/Force-Layout