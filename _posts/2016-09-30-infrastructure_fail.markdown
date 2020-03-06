---
layout: post
title:  "Infrastructures fail"
date:   2016-09-30
categories: infrastructure
---

I visited Stutgart for a workshop over the last days. For the trip back to
Berlin, I chose the train over the plane. It takes longer, but you can book it
spontaneously, seats can be reserved and the route is covered with Wifi. A
relaxed trip where I could get some work done. Or so I thought.

<!--
At the start of my day, I tried to reserve a train connection in the Android "DB
Navigator" App. Nope. Not going to happen. The App would not start and not even
give me a reason for this rejection. But hey probably it is my own fault for
choosing the OpenSource CyanogenMod Version or for disabling some of the access
rights that all of these application seem to need.

<p><center><img src="{{site.baseurl}}/assets/blog/pictures/db-navigator.png" style="width:400px;height:230px" alt="DB Navigator giving up"></center></p>

```
D/AndroidRuntime(27311): Shutting down VM
E/AndroidRuntime(27311): FATAL EXCEPTION: main
E/AndroidRuntime(27311): Process: de.hafas.android.db, PID: 27311
E/AndroidRuntime(27311): java.lang.NullPointerException: Attempt to get length of null array
```
-->

Let me discuss some of the failures I experienced along the way. If you are a
(semi-)regular traveller you just might be able to relate. 
As my ticket App was mysteriously failing, I went directly to the nearest urban
railway line (actually "S-Bahn") to book a ticket at the nearest ticket machine.
With 5 minutes to spare, what could go wrong? 

1. After describing my preferences for a reservation in a multi-step process,
   the machine told me that in fact a reservation for the connection was not at
all possible. (30 seconds)

2. It would not take my Cash-Back Deutsche Bahn Card. Not from any direction.
   (But apparently that is the norm at ticket machines.) (1 minute)

3. It would also not take any other cash card. (3.5 minutes)

And then the urban railway arrived and I was pressed between missing it and
likely my train or taking a ride without a ticket...

At the main station I met another obstacle finding the right signs that
ultimatively led me to the track of my train. (An indoor navigation system would
have been pleasant.)

The remainder of the trip was rather uneventful. The train was overrun with
passengers squeezing past each other to find a seat, as they had to be booked
more than an hour in advance, the expensive Wifi option (5 Euro / day) has about
80-90% packet loss and the seat I finally found was broken so that it would
shift forward and backwards whenever I changed my position.

For me this day shows a commonly occurring failure of our infrastructure. The
point is not that my morning was made stressful by it (which in fact it was) or
that my day was less productive than it could have been. The biggest problem is
what it tells about this country. Can we not expect better from a large
industrial nation such as Germany?

And how many of these problems could be solved by the right software? Indoor
navigation showing empty seats and the path to the right track, flexible
reservations, and even a simple process optimization for the existing ticket
machines. Maybe even sensors on the seats to view their 'health'. The Wifi
quality might be one that is the most difficult to solve with software, if I get
to it I will use a Raspberry Pi to track the signal on my next train trip.
