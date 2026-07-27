+++
title = "Understanding Historical Trends of LEGO and Modularity Through Data Analytics"
date = "2025-08-20T00:00:00+13:00"
draft = false
+++

Many moons ago when I was starting out on my tech journey, I came across two things that perked my interest. These were PowerBI by Microsoft which is a tool for visualising data, and a book called Narconomics by Tom Wainwright.

![](/images/narconomics.png)

This book seemed incredibly interesting, with my main draw being in how they came up with the data to showcase the trafficking of an illegal substance. Noticing how powerful visual data analysis is, I wanted to set out on picking a non-traditional topic (hopefully not drugs), which I could explore at a meticulous level and craft insights from spanning from decades.

Enter LEGO. A play system admired by the children, creative people, and the obsessed that love to collect and categorise.

LEGO is a toy company founded in 1932 that has gone through many changes as a modular creativity toy consisting of certain bricks and miscellaneous parts to build whatever you want. You could make a car with five wheels, a technical working crane, and maybe even a magic steam robot duck. LEGO is a powerhouse of a brand, but also a catalyst for creativity. Lego used to forego “spoonfeeding” of detailed creations to children and adults, and instead, promoted the idea of breaking out of instruction sets to instead encourage building something new.

Sadly, I believe this is what LEGO used to stand for. My hypothesis statement is that**LEGO is less modular and instead are designed more for kitsets/model creation**. I will be testing this hypothesis through collecting data across data platforms and centralised datasets and then move on to modelling trends around sets and pricing for Lego with help from PowerBI to provide justification to my hypothesis.

### Phase 1/4 – Getting the Data for Historical Lego Set Information
We initially have to know that we have the data to justify the time investment for this project, we begin by checking Kaggle.com, a community platform for analysts and data appreciators alike. We begin this by checking for Lego and find [https://www.kaggle.com/datasets/rtatman/lego-database](https://www.kaggle.com/datasets/rtatman/lego-database) which has 453 upvotes and a whopping 39.3k downloads. This is perfect and has a number of things such as:

- Parts > Sets > Thems to map which sets have which parts and set individual counts and what not.
- When did certain set composition trends instigate and what this would tell us about Lego as a medium?
- How have the size of sets changed over time?
- What colors are associated with witch themes? Could you predict which theme a set is from just by the bricks it contains?
- What sets have the most-used pieces in them? What sets have the rarest pieces in them?
- Have the colors of LEGOs included in sets changed over time?
(The last four were specified by the author)

![](/images/kaggle.png)

This seemed great until the eagle eyed of you would notice that this was current as of July 2017. This meant that our data would be missing the 8 most recent years of business which would be nothing short of insufficient data. As suggested by the author, we need to go to the Rebrickable API.

Funnily enough, I think Rebrickable themselves noticed people doing large API pulls and this affecting their systems, so by any means we have at [https://rebrickable.com/downloads/](https://rebrickable.com/downloads/) the ability to download the whole LEGO catalog database to download with this being updated via daily pulls, saving time to work with the API and provision a key. The schema for the set is as such:

![](/images/rebrickable_schema.png)

Funnily enough all of this information that Rebrickable has accounts to a total of 128mb. Despite this we actually have a plethora of data. To run some quick stats:

- A total of 25,477 sets
- Publishing of sets between the years of 1949 to 2025 with a total of 76 years of data
- A total of 15,924 minifigures, with a personal favourite of mine being [LEGO Minifigure - Mr. Shrimps](https://cdn.rebrickable.com/media/sets/fig-013835.jpg)
There are some slight inconsistencies, such as the date of four separate sets posted as being released in the future in 2026, and duplicate entries, however this is a >0.01% issue and across the board, the Rebrickable’s service reigns supreme for our selected usecase.

### Phase 2/4 – Installing PowerBI and Doing the Initial Setup
Now that we have the data, we need to visualise and make sense of it. There are of course other means of analysis we can perform from structured data queries, a living documentation through a Python Jupyter notebook, or even a few excel formulas.

We’ll be using PowerBI to do this. You can download a local copy of PowerBI from [Microsoft Power BI Desktop Download (v2.146.1026.0, published 8/19/2025)](https://www.microsoft.com/en-us/download/details.aspx?id=58494) and after that, once you log in with a Microsoft account.

So now you'll have something like this:

![](/images/powerbi_start.png)

What you’ll do from here is you’ll click “Get data from other sources” and specify CSV, selecting all 12 tables we have

![](/images/import_csv.png)

So, we have the data all loaded in, we now need to save the project and move on to constructing the model view, or how each table associates with one another and shares values (i.e., Primary/Foreign Keys). Attached below is a screenshot which articulates this:

![](/images/schema_edit.png)

The model view should be a carbon copy ported over to PowerBI with the schema of what Rebrickable outlines earlier. I assume either you’ll be able to figure out this linking in Model View, or there are many tutorials around. Either way you should have something like this:

![](/images/schema.png)

It should be noted that the schema autosuggestion feature can be useful for this, however I’d be careful to fully rely on this due to the part identifiers and general being incorrectly set to certain cardinalities by PowerBI. We also need to attempt to showcase this data, we’ll be doing two test graphs being:

- Trending stacked graph of number of unique sets released by year
![](/images/test_graph1.png)

- Top 10 table of sets by largest count of pieces
![](/images/test_graph2.png)

With both the data to manipulate, and the mechanism to achieve this through visualisation, we now move onto constructing a dashboard of statistics to understand the nature of LEGO and how it is evolving.

### Phase 3/4 – Finalising Selected Metrics and Identifying Additional Data Sources
In our last step before we move onto our final phase of opinion and relation to the initial thesis statement, we’ll be looking into the metrics we’ll be collecting to answer the question.

Firstly, considering that what the average number of pieces in a set each year compared to number of sets released this year to see the , we’ll start with mapping this by year and showing these as a bar graph.

![](/images/graph1.png)

For understanding unique distinctive parts by set for year on year to track if this goes up or down. We will be adding a line graph with this information requiring a combination of data tables. To assist with this we start with the DAX query, which will take a calculation of distinct part numbers by the inventory parts set and use the relationship of set numbers in sets to set numbers in the LEGO inventory, using the query below:`CALCULATE ( DISTINCTCOUNT ( inventory_parts [part_num] ), USERELATIONSHIP ( sets [set_num], inventories [set_num] ) )`

And are treated with a line chart encapsulating the total number of unique parts by year on year:

![](/images/graph2.png)

We also need to know the number of distinct sets and the breakdown by year, as well as just the number overall. As this is a single value, we just add a PowerBI Card visual along with a value of count of [‘sets’]name to create:

![](/images/graph3.png)

To which we’ll use a similar creation workflow to create one for overall distinct parts:

![](/images/graph4.png)

We created a donut chart visualising the number the themes LEGO have released and counts of unique sets by this theme. This should be noted this doesn’t always mean how popular a set is. Although a set that sells well will get more subsequent sets, you could also state that a one hit wonder set that goes viral might also do amazing, a good example being Harry Potter which despite being popular only has 195 sets).

![](/images/graph5.png)

A final metric that I collected partially through the analysis stage was the average number of pieces per set which we could either leave across every LEGO set ever made, or filter by theme/year.

![](/images/graph6.png)

Our final dashboard when performing minor visual adjustments and themes to enhance the vibe and readability of our PowerBI interactive dashboard, we are given the following which we publish:

![](/images/dashboard.png)

You can actually drill into any part of the data by clicking a theme/section/year that you are interested in, this can be as hyper specific as how many distinct LEGO parts are made from plastic materials which the answer is 93% (this is by distinctive parts, and not counting all parts in existence, which would be more like 99.9%), other parts include metal (LEGO Technic), stickers, and cloth (i.e., a cape on a minifigure):

![](/images/dashboard2.png)

As part of my analysis before drilling into my final opinion, I produced the following insights about LEGO to help either disprove or prove my hypothesis statement, these are:

- For every LEGO set that is released, an average of 2.26 unique parts is introduced into the LEGO ecosystem. Comparative to the 164.82 average number of pieces per set, this showcases strong modularity and reuse of existing pieces.
- We see the trend of average number of parts per set growing consistently (doubling from 2015 to 2025) over the past 10 years with:

- 2015 having a 147.12 average part count per set
- 2016 having a 159.65 average part count per set
- 2017 having a 162.14 average part count per set
- 2018 having a 161.36 average part count per set
- 2019 having a 174.18 average part count per set
- 2020 having a 190.56 average part count per set
- 2021 having a 217.86 average part count per set
- 2022 having a 233.24 average part count per set
- 2023 having a 264.11 average part count per set
- 2024 having a 276.04 average part count per set
- 2025 having a 294.90 average part count per set

- When removing the filter on the donut chart for LEGO themes we notice that there is a large spread of themes with the most prominent theme (LEGO Star Wars) only taking 3.99% of total share of the LEGO ecosystem. Most themes were released from 1999 onwards (i.e., Brickheadz in 2016, Star Wars in 1999, Mario in 2020, Harry Potter in 2001, keychains in 2001, and other misc bags and merch in 2015)
![](/images/theme_explore.png)

Sadly, the distinct number of unique pieces metric is static due to not being mapped to release date or set (i.e. we don’t have the data to state when a unique part was introduced, only that it exists currently).

One final thing we are treated with is an amazing statistic being that in terms of set releases in the early 2000’s, the company was at times releasing more Bionicle than any other set that existed. This was a pop culture hit and was one of LEGO’s best sellers and always a huge contributor for number of distinct parts released in a year due to their unique build style being completely different to traditional to LEGO (while unlike Duplo, falling under the same LEGO set umbrella). The success and iconic hit Bionicle had accounts why after Star Wars and Technic, Bionicle is the 3rd most popular theme of all time. They also got a re-release for a 2016 range however not much came of it and despite adding even more unique parts, the sales didn’t match and was discontinued after two years. Bionicle also has so many unique parts in comparison to any other theme that it can be difficult to apply Bionicle to other LEGO sets, to the point that it isn’t really LEGO anymore (i.e., interchangeable bricks where any set can be combined with any other set). With half the number of average pieces per set to other LEGO sets on average, this was an interesting theme but removed from the “LEGO universe”.

![](/images/bionicle.png)

### Phase 4/4 – Final Opinion in Relation to Initial Thesis Statement
My final opinion of**Lego is less modular and instead are designed more for kitsets/model creation**is incorrect and I must throw in the towel with this one. Looking at the analysis section LEGO sets are getting more complicated with sets containing more pieces on average by each year. Performing analysis on sampled LEGO sets, the average number of distinct pieces also tends to be low enough to allow sets to be modular with one another. Sets are less generic these days with more sets from 1999 onwards being connected to brand collaborations. LEGO has pivoted more on collaborations than their own IP singularly, however this is not yet overshadowed their brand statistically, especially considering how spread their collaborations have been. Additionally, the number of distinct pieces haven’t blown out of proportion due to collaborations and limited edition sets as was expected, so LEGO has in fact done a good job at letting a customer combine, let’s say, a LEGO racers set from 2005 to a 2025 LEGO Formula 1 collaboration set.

I feel like just because the company changed, doesn’t mean it’s a bad thing. Having looked more closely at the company and understanding their business model more, I realised that LEGO has grown to be more to cater to everyone. The minimal-instructions creative LEGO still exists through Classic, Creator, and other themes that help to keep the older style of LEGO over that which has become overly complex and focused less on creativity and more of unique pieces and booklets which ultimately makes LEGO no more than an overly complicated model sets. LEGO hasn’t lost anything but has built other styles of LEGO around it. It would be nice however if LEGO went back to promoting this more than collaborations and things that can’t be as interchangeable.