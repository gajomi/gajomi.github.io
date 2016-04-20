---
layout: post
title: "Solving Pricenomics Treefort BnB challenge with pandas"
date: 2016-04-19
---

[Pricenomics](http://priceonomics.com/) has put out a [data analysis challenge](http://priceonomics.com/the-priceonomics-data-puzzle-treefortbnb/) as part of an effort to promote and recruit for their [freelance portion](http://priceonomics.com/priceonomics-freelance-gigs-for-data-journalists/) of the [The Priceonomics Data Studio](http://priceonomics.com/the-priceonomics-data-studio/). The challenge revolves around a fictional company called "Treefort BnB", which has released a dataset of their treehouse listings. It seems to me like it could be pretty cool to freelance for Pricenomics as I have enjoyed reading their blog posts in the past, so I decided to solve the puzzle.

## The problem and my motivation

The basic goal is to determine the median price of a booking for the the hundred most represented cities by number of listings.  The technical portion of the application for the freelance gig is restricted to the specific computation mentioned above, but it seemed to me like a somewhat more extended analysis and exposition could make for a nice blog post (an opportunity not to be wasted). So in what follows I'll show how I would solve the basis challenge using Python's [`pandas` library](http://pandas.pydata.org/). If you haven't seen `pandas` in action before this should give a good idea of how it can be useful for solving simple analysis problems. After solving the basic problem I'll show how I would go about poking around the dataset to learn more about TreefortBnB and show off some more `pandas` functionality.

## The basic solution

If you want to follow along using [Jupyter](http://jupyter.org/) you can find all the code [here](https://gist.github.com/gajomi/a0997b02af570c3e50bb8ca58790677f).Using IPython I downloaded the data and read it into a `pandas` (it is conventional to import the module as `pd`) dataframe with

```python
!curl https://s3.amazonaws.com/pix-media/Data+for+TreefortBnB+Puzzle.csv > Puzzle.csv
df = pd.read_csv("./Puzzle.csv")
```

We can inspect the first few rows by calling `head`

```python
df.head(5)
```

which in a Jupyter notebook will render nicely as an HTML table:

<div><table border="1" class="dataframe">  <thead>    <tr style="text-align: right;">      <th></th>      <th>Unique id</th>      <th>City</th>      <th>State</th>      <th>$ Price</th>      <th># of Reviews</th>    </tr>  </thead>  <tbody>    <tr>      <th>0</th>      <td>1</td>      <td>Portland</td>      <td>OR</td>      <td>75</td>      <td>5</td>    </tr>    <tr>      <th>1</th>      <td>2</td>      <td>San Diego</td>      <td>CA</td>      <td>95</td>      <td>3</td>    </tr>    <tr>      <th>2</th>      <td>3</td>      <td>New York</td>      <td>NY</td>      <td>149</td>      <td>37</td>    </tr>    <tr>      <th>3</th>      <td>4</td>      <td>Los Angeles</td>      <td>CA</td>      <td>199</td>      <td>45</td>    </tr>    <tr>      <th>4</th>      <td>5</td>      <td>Denver</td>      <td>CO</td>      <td>56</td>      <td>99</td>    </tr>  </tbody></table></div>

So we have columns corresponding to a unique id (it's a good idea to check that it actually is unique!), a state, a city a price and total number of reviews for each listing. Having glimpsed the basic layout of the data, one can plan out a sequence of operations to convert the full table into desired summary. Here is one way to do this:

```python
basicsummary = df.assign(City = df.City.str.title()) \
                 .groupby(["City","State"])["$ Price"] \
                 .describe().unstack().sort_values("count").tail(100)["50%"] \
                 .sort_values(ascending=False).to_frame() \
                 .rename(columns = {"50%":"median price ($)"})
basicsummary.head(5)
```
<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>median price ($)</th>
    </tr>
    <tr>
      <th>City</th>
      <th>State</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Indianapolis</th>
      <th>IN</th>
      <td>650</td>
    </tr>
    <tr>
      <th>Malibu</th>
      <th>CA</th>
      <td>304</td>
    </tr>
    <tr>
      <th>Park City</th>
      <th>UT</th>
      <td>299</td>
    </tr>
    <tr>
      <th>Truckee</th>
      <th>NV</th>
      <td>275</td>
    </tr>
    <tr>
      <th>Healdsburg</th>
      <th>CA</th>
      <td>275</td>
    </tr>
  </tbody>
</table>
</div>

That first chunk of Python deserves a bit of explanation, which I am happy to provide.

- The first line converts all the city names to title case. This is necessary because the original column contains rows with both title and lower cases (e.g both "New York" and "new york").
- The second line creates a grouping data structure based on the city and ctate and pulls out the price in each group (the other columns aren't needed in this exercise). The grouping is based on both cities and states to avoid grouping distinct cities that happen to have the same name (e.g. both Carmel, California and Carmel, Indiana are present in this dataset).
- The third line is perhaps the most obscure. In it I make use of `pandas` useful `describe` method, which returns several statistics about a column, including the total number of entries (as `count`) and the median (as `50%`). The `unstack` method is used to get these statistics into columns, from which point we can pull out the hundred most numerous cities and extract the median price.
- The fourth line sorts the resulting "`Series`" data structure by price and converts the result to a dataframe.
- The last line simply renames the "50%" to "median price ($)".

These five lines, counted as such by way of an ad-hoc continuation break up, are not necessarily the most Pythonic, but more or less reflect how I think about the steps in this exercise. I oftentimes find myself chaining method calls like this when working with `pandas` interactively in Jupyter or vanilla IPython, as it allows me to build up a single operation from many while getting the output by evaluating the statement in one go.

## An exploratory analysis and improved solution

The above exercise focused on on median prices. This is perhaps a reasonable metric to get a sense for the typical sort of bookings one might expect to make, since it is robust to possible extreme outliers which might be present in the data. One of the nice things about using `pandas` interactively, however, is that we don't have to guess, when we can see! As was already mentioned, the `describe` method returns a bunch of statistics which can be useful in developing a rough idea of the distribution of data in a column. We are interested in the distributions of prices in particular:

```python
df["$ Price"].describe().to_frame() #The call to `to_frame` is a bit superfluous (I did it here to get a nice HTML table).
```
<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>$ Price</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>42802.000000</td>
    </tr>
    <tr>
      <th>mean</th>
      <td>187.117074</td>
    </tr>
    <tr>
      <th>std</th>
      <td>263.002109</td>
    </tr>
    <tr>
      <th>min</th>
      <td>10.000000</td>
    </tr>
    <tr>
      <th>25%</th>
      <td>90.000000</td>
    </tr>
    <tr>
      <th>50%</th>
      <td>130.000000</td>
    </tr>
    <tr>
      <th>75%</th>
      <td>200.000000</td>
    </tr>
    <tr>
      <th>max</th>
      <td>10000.000000</td>
    </tr>
  </tbody>
</table>
</div>
By comparing the ratio of the mean and the standard deviation, also known as the [coefficient of variation](https://en.wikipedia.org/wiki/Coefficient_of_variation), we get a sense that the distribution is somewhat dispersed to the right, as prices do not take on negative values. Furthermore we see what looks like an absurd maximum booking price of $10000. Perhaps this is Paris Hilton's gold and sapphire plated Redwood penthouse? We can get a better idea of whats going on by looking at a graphical representation of the distribution:

```python
df["$ Price"].hist(bins = 64,log=True)
```
![histogram of prices]({{ site.url }}/images/treehouse_fig1.png)

We find that there are indeed a few listings priced at $10000 (note that the y axis is on a logarithmic scale), and that they appear to be well separated from the rest of the data. One's imagination can run wild about the possible sorts of things this could mean about these bookings. However, it is useful to constrain one's imagination by taking a look at the data. In `pandas` is it easy to select rows by criteria in the columns:

```python
df[df["$ Price"] > 5000]
```
<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Unique id</th>
      <th>City</th>
      <th>State</th>
      <th>$ Price</th>
      <th># of Reviews</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2819</th>
      <td>2820</td>
      <td>San Francisco</td>
      <td>CA</td>
      <td>10000</td>
      <td>31</td>
    </tr>
    <tr>
      <th>5135</th>
      <td>5136</td>
      <td>San Francisco</td>
      <td>CA</td>
      <td>6000</td>
      <td>11</td>
    </tr>
    <tr>
      <th>8328</th>
      <td>8329</td>
      <td>Chicago</td>
      <td>IL</td>
      <td>6500</td>
      <td>1</td>
    </tr>
    <tr>
      <th>8638</th>
      <td>8639</td>
      <td>Indianapolis</td>
      <td>IN</td>
      <td>5500</td>
      <td>0</td>
    </tr>
    <tr>
      <th>8867</th>
      <td>8868</td>
      <td>Park City</td>
      <td>UT</td>
      <td>10000</td>
      <td>1</td>
    </tr>
    <tr>
      <th>10332</th>
      <td>10333</td>
      <td>Park City</td>
      <td>UT</td>
      <td>6500</td>
      <td>0</td>
    </tr>
    <tr>
      <th>13431</th>
      <td>13432</td>
      <td>Queens</td>
      <td>NY</td>
      <td>6500</td>
      <td>0</td>
    </tr>
    <tr>
      <th>17195</th>
      <td>17196</td>
      <td>Boston</td>
      <td>MA</td>
      <td>5225</td>
      <td>0</td>
    </tr>
    <tr>
      <th>20279</th>
      <td>20280</td>
      <td>New York</td>
      <td>NY</td>
      <td>5600</td>
      <td>0</td>
    </tr>
    <tr>
      <th>23817</th>
      <td>23818</td>
      <td>Park City</td>
      <td>UT</td>
      <td>5500</td>
      <td>0</td>
    </tr>
    <tr>
      <th>26232</th>
      <td>26233</td>
      <td>Park City</td>
      <td>UT</td>
      <td>10000</td>
      <td>0</td>
    </tr>
    <tr>
      <th>28626</th>
      <td>28627</td>
      <td>Miami Beach</td>
      <td>FL</td>
      <td>10000</td>
      <td>0</td>
    </tr>
    <tr>
      <th>35136</th>
      <td>35137</td>
      <td>Miami Beach</td>
      <td>FL</td>
      <td>6000</td>
      <td>0</td>
    </tr>
    <tr>
      <th>38261</th>
      <td>38262</td>
      <td>Miami Beach</td>
      <td>FL</td>
      <td>5750</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
</div>

By looking at additional columns of these expensive listings we get a clue as to what might be going on. Most of these listings have no reviews. While the challenge didn't offer any specific hints as to what one was to learn by computing median prices, the spirit of it suggest that the goal was to provide insight into a "typical" Treefort BnB experience. It stands to reason that listing without reviews have possibly never been booked, so that even though they are in the data set they may not be relevant to the larger questions we are interested in. In contrast, one would imagine that listings with many reviews are at least significant to the Treefort BnB user experience, inasmuchas users have written about them. So what does any of this have to do with prices? We can get a sense for this by doing:

```python
df.plot(x="$ Price",y ="# of Reviews",kind='scatter',
        logx=True,xlim = [9,11000],ylim = [-10,110])
```
![price/reviews scatter plot]({{ site.url }}/images/treehouse_fig2.png)

Notice how there is a clustering of points with zero or one hundred reviews? The points in the zero cluster extend significantly into the high price regime. It is hard to tell from the plots exactly how important these zero review points are to our overall problem, but they at least suggest we should look further into it. To start we can take at look at the same type of scatter plot, restricted to the bookings in Indianapolis, which which had the highest median prices in our initial analysis

```python
df[df.City == "Indianapolis"].plot(x="$ Price",y ="# of Reviews",kind='scatter',
                                   logx=True,xlim = [9,11000],ylim = [-10,110])
```
![price/reviews scatter plot]({{ site.url }}/images/treehouse_fig3.png)

Yikes! It looks like Indianapolis has a bunch of listing with no reviews, and that they probably are a big part of why it came out on top in our naive median price analysis. We can get a quantitative view of the larger picture by separating the reviewed from the unreviewed and taking a peak at some statistics:

```python
is_reviewed = df["# of Reviews"] > 0
df.groupby(is_reviewed)["$ Price"].describe().to_frame().unstack()
```
<div>
<table border="1" class="dataframe">
  <thead>
    <tr>
      <th></th>
      <th colspan="8" halign="left">0</th>
    </tr>
    <tr>
      <th></th>
      <th>count</th>
      <th>mean</th>
      <th>std</th>
      <th>min</th>
      <th>25%</th>
      <th>50%</th>
      <th>75%</th>
      <th>max</th>
    </tr>
    <tr>
      <th># of Reviews</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>False</th>
      <td>16000</td>
      <td>238.025500</td>
      <td>365.393895</td>
      <td>10</td>
      <td>100</td>
      <td>150</td>
      <td>250</td>
      <td>10000</td>
    </tr>
    <tr>
      <th>True</th>
      <td>26802</td>
      <td>156.726252</td>
      <td>168.202838</td>
      <td>10</td>
      <td>85</td>
      <td>125</td>
      <td>185</td>
      <td>10000</td>
    </tr>
  </tbody>
</table>
</div>
We see that about 40% of the total listings are unreviewed with their prices shifted to the right (both in terms of mean and median). Furthermore there are some rather suspicious looking quantiles for these unreviewed listings ($100, $150, $250... what are the odds?!). All of this seems to add credibility to the initial idea that filtering out listings that haven't been reviewed might give a better picture of what to expect when booking.

With this in mind I set out for a better version of the basic result above. I also thought it could be useful to group location with the same median price into the same row with a slightly different formatting, which could then be indexed by a price rank. This lead me to the following lines of code:

```python
refinedsummary = df.get(df["# of Reviews"] > 0) \
                   .assign(Location = df.City.str.title()+df.State.apply(lambda s: ' ('+s+')')) \
                   .groupby("Location")["$ Price"].describe().unstack() \
                   .sort_values("count").tail(100) \
                   .reset_index().groupby("50%")["Location"].apply(lambda x: "%s" % ' / '.join(x)) \
                   .sort_index(ascending=False).to_frame() \
                   .reset_index().rename(columns = {"50%":"median price ($)"})
refinedsummary.set_index(refinedsummary.index+1,inplace = True)
refinedsummary.index.rename("rank",inplace = True)
refinedsummary.head(15)
```
<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>median price ($)</th>
      <th>Location</th>
    </tr>
    <tr>
      <th>rank</th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>300.0</td>
      <td>Carmel (CA)</td>
    </tr>
    <tr>
      <th>2</th>
      <td>225.0</td>
      <td>Healdsburg (CA) / Malibu (CA)</td>
    </tr>
    <tr>
      <th>3</th>
      <td>200.0</td>
      <td>Incline Village (NV) / Laguna Beach (CA) / Truckee (NV)</td>
    </tr>
    <tr>
      <th>4</th>
      <td>199.0</td>
      <td>Hermosa Beach (CA)</td>
    </tr>
    <tr>
      <th>5</th>
      <td>184.5</td>
      <td>Napa (CA)</td>
    </tr>
    <tr>
      <th>6</th>
      <td>179.0</td>
      <td>Park City (UT)</td>
    </tr>
    <tr>
      <th>7</th>
      <td>177.5</td>
      <td>Sunny Isles Beach (FL)</td>
    </tr>
    <tr>
      <th>8</th>
      <td>175.0</td>
      <td>Sonoma (CA)</td>
    </tr>
    <tr>
      <th>9</th>
      <td>165.0</td>
      <td>New York (NY)</td>
    </tr>
    <tr>
      <th>10</th>
      <td>162.0</td>
      <td>Manhattan Beach (CA)</td>
    </tr>
    <tr>
      <th>11</th>
      <td>157.5</td>
      <td>La Jolla (CA)</td>
    </tr>
    <tr>
      <th>12</th>
      <td>150.0</td>
      <td>Sausalito (CA) / Beverly Hills (CA) / Newport Beach (CA) / Venice (CA) / Boston (MA) / Austin (TX) / San Francisco (CA)</td>
    </tr>
    <tr>
      <th>13</th>
      <td>149.0</td>
      <td>Marina Del Rey (CA)</td>
    </tr>
    <tr>
      <th>14</th>
      <td>148.0</td>
      <td>Mill Valley (CA)</td>
    </tr>
    <tr>
      <th>15</th>
      <td>130.0</td>
      <td>Santa Monica (CA) / Miami Beach (FL)</td>
    </tr>
  </tbody>
</table>
</div>

With this we see that Indianapolis has been pushed out of the top ranks, which are now occupied by somewhat more sensible places for expensive treehouses (to the extent that such a thing exists).

There are of course, other aspects of the data we might want to consider. For example, where should you head if you want the widest selection of treeforts? By, now, you may have an idea of how I would go about answering this question...

```python
nlistings = df.get(df["# of Reviews"] > 0) \
         .groupby(["City","State"]).count()["$ Price"] \
         .sort_values(ascending = False).to_frame() \
         .rename(columns = {"$ Price":"total listings"})
nlistings.head(10)
```

<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th></th>
      <th>total listings</th>
    </tr>
    <tr>
      <th>City</th>
      <th>State</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>New York</th>
      <th>NY</th>
      <td>5597</td>
    </tr>
    <tr>
      <th>Brooklyn</th>
      <th>NY</th>
      <td>3017</td>
    </tr>
    <tr>
      <th>San Francisco</th>
      <th>CA</th>
      <td>2455</td>
    </tr>
    <tr>
      <th>Los Angeles</th>
      <th>CA</th>
      <td>1941</td>
    </tr>
    <tr>
      <th>Austin</th>
      <th>TX</th>
      <td>1495</td>
    </tr>
    <tr>
      <th>Chicago</th>
      <th>IL</th>
      <td>875</td>
    </tr>
    <tr>
      <th>Washington</th>
      <th>DC</th>
      <td>836</td>
    </tr>
    <tr>
      <th>Miami Beach</th>
      <th>FL</th>
      <td>790</td>
    </tr>
    <tr>
      <th>Portland</th>
      <th>OR</th>
      <td>586</td>
    </tr>
    <tr>
      <th>Seattle</th>
      <th>WA</th>
      <td>573</td>
    </tr>
  </tbody>
</table>
</div>

At first glance (say, for the first ten seconds) this looks reasonable, although there is a question of whether or not or exactly how Brooklyn is not contained in New York. But let's assume that "New York" means every borough except Brooklyn. The data show that there are 3017 reviewed listings in Brooklyn. Are there over three thousand treehouses in Brooklyn? Granted, there are quite a few trees. But treehouses are sufficiently rare that you [might make it into the New York Times for building one there](http://www.nytimes.com/2011/11/10/garden/a-treehouse-grows-in-brooklyn.html?_r=0). Still, exactly how many is not obvious but...

__WARNING!!! YOU ARE ENTERING THE FERMI ESTIMATION ZONE. HOLD ON TO YOUR POWERS OF TWO AND TEN!__

... to get things started lets say that there are about as many trees as there are people. So, a couple million. Now, one expects that a great deal, say 90%, of these tree are going to be found in public parks where one is likely to have a hard time constructing a treefort, not to say anything about renting it out. We still have hundreds of thousands of trees to work with though. Now imagine you are walking on a street in Brooklyn and spot a bunch of trees (if you hare having a hard time imagining use [Google Street View](https://www.google.com/maps/place/Brooklyn,+NY/@40.6315707,-73.935024,3a,75y,154.54h,71.57t/data=!3m7!1e1!3m5!1sVwjugDSsxLihdoo8BY0A2A!2e0!6s%2F%2Fgeo2.ggpht.com%2Fcbk%3Fpanoid%3DVwjugDSsxLihdoo8BY0A2A%26output%3Dthumbnail%26cb_client%3Dmaps_sv.tactile.gps%26thumb%3D2%26w%3D203%26h%3D100%26yaw%3D97.5%26pitch%3D-7!7i13312!8i6656!4m2!3m1!1s0x89c24416947c2109:0x82765c7404007886!6m1!1e1)). Most of the trees that you see are sort of lining the streets and sidewalks, which also would make for construction difficulties. Let's be generous though and say that 10% of the remaining trees can be found in backyards and other places suitable for treehouse construction. There are ten thousand-ish trees (from tiny saplings to grand oaks) left to work with. And yet, the above table states that there several thousand listings in the same area. Are we to believe that every third tree in Brooklyn has a treehouse on it and all of them are for lease through Treefort BnB!?

Of course we don't, because we know that the data are fake and Treefort BnB does not exist. But I wanted to end with that little Fermi estimation paragraph because it shows how data of this sort usually raises many interesting questions, including those about the accuracy of the data itself, which is something that freelancers should definitely keep in mind when working with a client's data.
