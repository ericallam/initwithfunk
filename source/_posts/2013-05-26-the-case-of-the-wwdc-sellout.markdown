---
layout: post
title: "The Case of the WWDC Sellout"
date: 2013-05-26 17:32
comments: true
categories: [WWDC, Random Development]
published: true
---

<img src="http://i.imgur.com/A0WoI4T.jpg" />

It starts in the throat. Like a burly man with a size 13 boot stepping down on your neck, the feeling that something horrible has gone wrong. 

There is a bug in production.

Any programmer that has a modicum of experience shipping software knows this feeling, because programmers are the leading cause of bugs on Earth[^1].  It's a terrible feeling and almost always leads to the worst day in a programmer's life up until that point[^2].

But very few of us have experienced a day quite like the day one or more programmers had when WWDC "Sold Out" a couple weeks ago. The sale should have been a complete success: unprecedented demand, a rabid group of developers with their credit cards in hand.  After only [71 seconds][1] went by, the WWDC ticket site featured a prominent "Sold Out" banner and the dreams of thousands of developers were instantly crushed, yours truly included.

But then, just a few hours later, something strange happened: developers started getting mysterious phone calls from Apple letting them know they had tickets for them. I was one of the those developers. I was there as the clock rendered[^3] 10am PST and I was able to put a WWDC ticket in my Shopping Cart[^4], but when I submitted the payment form I was greated with a 500 error and when I went back to the cart it was empty, and the tickets were sold out.  The Apple Developer support person who called me only said that they knew that I had tried to purchase a ticket for WWDC and they still had it available for me to purchase, and that I'd be getting an email in the next 12 hours allowing me to grab it. 

There have been theories as to what happened, and Marco Arment [just posted][1] his theory this weekend:

{% blockquote %}
My best guess: some part of the infrastructure handling the purchases mistook 4,500 connections, transactions, or sessions for 4,500 sales. And when the front-end servers collapsed under the load of everyone hitting them at once — a first this year, since the availability time was preannounced — we all started refreshing, those connections started stacking up, and something on the back-end triggered the “Sold Out” state early because it was mistakenly counting all of those failed sessions
{% endblockquote %}

I think he's close but a little bit off. My guess is that there were multiple problems with their ticket reservation system. Reservation systems are very easy to screw up[^5], especially in distributed systems.  

Here is how I think it went down:

All the tickets were reserved (by placing a ticket into the cart you'd effectively "reserve" the ticket), and a majority of those reservations were fulfilled by the developer checking out and paying for the ticket.  The rest of the reservations were wrongly fulfilled when there was a bug in the checkout process that caused an exception somewhere in the distributed stack.  The checkout process failed in a way that isolated itself from the reservation system, so the final state was a fulfilled reservation, an empty cart, and an unpurchased ticket. What likely remained as evidence to the correct state of the system were log files scattered over dozens of servers and in a variety of formats.  That's why it took so long to begin calling (according to anecdotal twitter reports and my own call, I'm guessing it took 5+ hours), and why Apple couldn't make an immediate announcement as to what happened: they didn't know themselves. 

I don't envy the programmers who probably spent a couple sleepless nights figuring the mess out, and I am grateful that they've been reaching out to developers to try and get them the tickets they deserve for WWDC.  As a first-time attendant, I'm looking forward to meeting an Apple engineer or two, and maybe buying them a beer or a cranberry juice, giving them a pat on the back, and letting them know that we've been there, we knew how they were feeling, that something terrible happened but it's not the end of the world.

There was a bug in production.[^6]

[^1]: Unproven hypothesis: I've created more bugs than 99% of the population
[^2]: Programmers are very dramatic
[^3]: Clocks don't "strike" anymore, do they?
[^4]: We're going to need to rethink that metaphor soon. The Amazon Generation won't know what the hell we're talking about
[^5]: I know this because I've written one and I screwed it up multiple times
[^6]: One can hope for WWDC 2013, amirite?

[1]: http://www.marco.org/2013/05/24/71-seconds 
[2]: http://en.wikipedia.org/wiki/CAP_theorem