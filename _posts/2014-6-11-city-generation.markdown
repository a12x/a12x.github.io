---
layout: post
title:  "City Generation"
date:   2014-6-11
permalink: "/citygen"
---

I wanted to render a procedurally generated city. Here's how I did it.

##Generating the roads

To generate the roads for my city, I created a vector field. This allows the roads to flow naturally. The red dots affect the direction of the vectors close to them. Their circular outlines show their areas of influence.

<img src="{{ site.url }}/assets/1.png" height=450 style="display: block; margin-left: auto; margin-right: auto"/>

Now that the vector field has been set up, I trace paths through the vector field to get the roads. I also trace through the perpendicular vectors to get the crossroads. The red/green dots are the start/end points of each road.

<img src="{{ site.url }}/assets/2.png" height=450 style="display: block; margin-left: auto; margin-right: auto"/>

##Generating the buildings

Next, I find the intersections between each of the roads.

<img src="{{ site.url }}/assets/3.png" height=450 style="display: block; margin-left: auto; margin-right: auto"/>

I then traverse the graph of intersections to get the city blocks and color them.

<img src="{{ site.url }}/assets/4.png" height=450 style="display: block; margin-left: auto; margin-right: auto"/>

After that, I shrink the city blocks and slice them up to get the buildings.

<img src="{{ site.url }}/assets/5.png" height=450 style="display: block; margin-left: auto; margin-right: auto"/>

##Results

Here is how it looks without the debug symbols.

<img src="{{ site.url }}/assets/6.png" height=450 style="display: block; margin-left: auto; margin-right: auto"/>
<img src="{{ site.url }}/assets/7.png" height=450 style="display: block; margin-left: auto; margin-right: auto"/>
<img src="{{ site.url }}/assets/8.png" height=450 style="display: block; margin-left: auto; margin-right: auto"/>

And finally, here it is rendered in 3d.

<img src="{{ site.url }}/assets/9.png" height=450 style="display: block; margin-left: auto; margin-right: auto"/>
