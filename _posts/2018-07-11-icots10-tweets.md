---
layout: post
title: "Battle of the conference hashtags"
tags:
  - event-notes
  - R-code
  - social-media
---

A tweet comparing ICML and useR tweets got me thinking, and I've wanted to use the `twitteR` or `rtweet` package for a while...

<blockquote class="twitter-tweet" data-lang="en">
<p lang="en" dir="ltr">
How does <a href="https://twitter.com/hashtag/ICOTS10?src=hash&amp;ref_src=twsrc%5Etfw">\#ICOTS10</a> measure up? <a href="https://t.co/NUC3k2XAQK">https://t.co/NUC3k2XAQK</a>
</p>
— Kelly Bodwin (@KellyBodwin) <a href="https://twitter.com/KellyBodwin/status/1016743797550485505?ref_src=twsrc%5Etfw">July 10, 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

-   [Number of tweets by conference](#number-of-tweets-by-conference)
-   [How about number of retweets by conference?](#how-about-number-of-retweets-by-conference)
-   [Number of favourites?](#number-of-favourites)
-   [What about percentage of tweets getting favourited or retweeted?](#what-about-percentage-of-tweets-getting-favourited-or-retweeted)
-   [Which tweets saw the most retweets?](#which-tweets-saw-the-most-retweets)
-   [The most favourites?](#the-most-favourites)
-   [Emojis ❤️](#emojis)
-   [Checking if we can scrape the emoji lists](#checking-if-we-can-scrape-the-emoji-lists)
-   [Scraping the emoji lists](#scraping-the-emoji-lists)
-   [Which emoji were the most popular?](#which-emoji-were-the-most-popular)
    -   [Top \#ICOTS10 emojis](#top-icots10-emojis)
    -   [Top \#useR2018 emoji](#top-user2018-emoji)
    -   [Top \#ICML2018 emoji](#top-icml2018-emoji)
-   [More from ICOTS](#more-from-icots)
-   [Vanity time: My best tweet?](#vanity-time-my-best-tweet)

{% highlight r %}
library(twitteR)
library(ggplot2)
library(dplyr)
library(lubridate)
library(stringr)
library(tidyr)
library(RColorBrewer)
library(robotstxt)
library(rvest)
library(tibble)
library(data.table)
library(DT)
library(widgetframe)
{% endhighlight %}

Just one thing to be careful of, you'll need to get your own twitter keys. [This might be quite helpful](https://medium.com/@GalarnykMichael/accessing-data-from-twitter-api-using-r-part1-b387a1c7d3e). I had to reset my keys to get the code to work. Not sure why, but [others have had the same issue](https://github.com/geoffjentry/twitteR/issues/74).

{% highlight r %}
setup_twitter_oauth(consumer_key, consumer_secret, access_token, access_secret)
{% endhighlight %}

    ## [1] "Using direct authentication"

The tweet that inspired this shared a tweet that compared useR (a conference about using the R statistical programming language in Brisbane, Australia in 2018), with ICML (a conference about machine learning in Stockholm, Sweden in 2018). The below chunk gets the data for these two conferences and for the 10th International Conference on Teaching Statistics in Kyoto, Japan.

Also note, the `n =` were chosen by trial and error. Very sophisticated.

{% highlight r %}
icots = searchTwitter('#icots10', n = 1200)
user = searchTwitter('#useR2018', n = 5000)
icml = searchTwitter('#ICML2018', n = 5200)

icots_tweets = twListToDF(icots) %>%
  mutate(Hashtag = "#ICOTS10") %>%
  write.csv("icots.csv", row.names=FALSE)
user_tweets = twListToDF(user) %>%
  mutate(Hashtag = "#useR2018") %>%
  write.csv("user.csv", row.names=FALSE)
icml_tweets = twListToDF(icml) %>%
  mutate(Hashtag = "#ICML2018") %>%
  write.csv("icml.csv", row.names=FALSE)
{% endhighlight %}

There also seemed to be lots of tweets under the ICOTS hashtag that were about Ethereum, which wasn't really related to us. I've filtered these out.

And you might want to look at the dates in a different timezone setting - these conferences are all in different countries and I've just taken the time and date created information directly from the Twitter API results with no transformation.

{% highlight r %}
all_tweets = rbind(icots_tweets, user_tweets, icml_tweets) %>%
  mutate(date = format(created, format="%Y-%m-%d")) %>%
  filter(!str_detect(text, "@EthereumLimited:")) %>%
  filter(!str_detect(text, "Ethlimited"))

all_noRT = all_tweets %>%
  filter(isRetweet == FALSE)
{% endhighlight %}

Let's take a look!

### Number of tweets by conference

{% highlight r %}
all_tweets %>%
  mutate(Date = as.Date(date)) %>%
  group_by(Hashtag, Date) %>%
  summarise(Count = n()) %>%
  ggplot(aes(Date, Count, colour = Hashtag)) +
  geom_line() +
  scale_x_date(date_minor_breaks = "1 day") +
  ggtitle("Battle of the conference hashtags")
{% endhighlight %}

![]({{ site.baseurl }}/images/conferencetweets_files/figure-markdown_github/graph-1.png)

{% highlight r %}
all_noRT %>%
  mutate(Date = as.Date(date)) %>%
  group_by(Hashtag, Date) %>%
  summarise(Count = n()) %>%
  ggplot(aes(Date, Count, colour = Hashtag)) +
  geom_line() +
  scale_x_date(date_minor_breaks = "1 day") +
  ggtitle("Battle of the conference hashtags - retweets removed")
{% endhighlight %}

![]({{ site.baseurl }}/images/conferencetweets_files/figure-markdown_github/graph-2.png)

Can you spot which day ICOTS had a half-day and folks went off exploring Kyoto or to see the deer at Nara instead of tweeting?

<blockquote class="twitter-tweet" data-lang="en">
<p lang="en" dir="ltr">
No matter where he goes, <a href="https://twitter.com/hadleywickham?ref_src=twsrc%5Etfw">@hadleywickham</a> is mobbed by fans <a href="https://t.co/YxlR2HVAVz">pic.twitter.com/YxlR2HVAVz</a>
</p>
— Hilary Parker (@hspter) <a href="https://twitter.com/hspter/status/1017235183664132096?ref_src=twsrc%5Etfw">July 12, 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
### How about number of retweets by conference?

{% highlight r %}
all_tweets %>%
  group_by(Hashtag) %>%
  summarise(Retweets = sum(retweetCount)) %>%
  ggplot(aes(Hashtag, Retweets, fill = Hashtag)) +
  geom_bar(stat="identity") +
  ggtitle("Retweets by conference hashtag")
{% endhighlight %}

![]({{ site.baseurl }}/images/conferencetweets_files/figure-markdown_github/biggest_retweets-1.png)

### Number of favourites?

{% highlight r %}
all_tweets %>%
  group_by(Hashtag) %>%
  summarise(Favourites = sum(favoriteCount)) %>%
  ggplot(aes(Hashtag, Favourites, fill = Hashtag)) +
  geom_bar(stat="identity") +
  ggtitle("Favourites by hashtag")
{% endhighlight %}

![]({{ site.baseurl }}/images/conferencetweets_files/figure-markdown_github/biggest_favourites-1.png)

### What about percentage of tweets getting favourited or retweeted?

I was curious whether support/quality within the hashtags was similar. I think proportion of tweets getting retweeted or favourited might be an interesting measure of the "interestingness" of the tweets, but more likely an indicator of how supportive the group on that hashtag are.

{% highlight r %}
all_noRT %>%
  group_by(Hashtag) %>%
  filter(isRetweet == FALSE) %>%
  summarise(Favourited = sum(favoriteCount > 0), Retweeted = sum(retweetCount > 0), total = n()) %>%
  mutate(`Percentage Favourited`= Favourited/total, `Percentage Retweeted` = Retweeted/total) %>%
  select(Hashtag, `Percentage Favourited`, `Percentage Retweeted`) %>%
  gather(Measure, Percentage, -Hashtag) %>%
  ggplot(aes(Hashtag, Percentage, fill = Measure)) +
  geom_bar(stat="identity", width = 0.5, position = "dodge") +
  ggtitle("Percentage of Original Tweets Favourited or Retweeted by Conference Hashtag") +
  scale_fill_brewer(palette="Accent") +
  scale_y_continuous(labels = scales::percent)
{% endhighlight %}

![]({{ site.baseurl }}/images/conferencetweets_files/figure-markdown_github/percentages-1.png)

The retweets are strong with \#ICOTS10!

### Which tweets saw the most retweets?

{% highlight r %}
toptweet = all_tweets %>%
  group_by(Hashtag) %>%
  filter(retweetCount == max(retweetCount)) %>%
  distinct(Hashtag, .keep_all = TRUE)

topretweeticots = toptweet %>%
  filter(Hashtag == "#ICOTS10")

topretweeticml = toptweet %>%
  filter(Hashtag == "#ICML2018")

topretweetuser = toptweet %>%
  filter(Hashtag == "#useR2018")
{% endhighlight %}

Check out the most retweeted [ICOTS tweet](https://twitter.com/anyuser/status/1017805795109548032), [ICML tweet](https://twitter.com/anyuser/status/1017881934448513024) and [useR tweet](https://twitter.com/anyuser/status/1017918719123771392).

### The most favourites?

{% highlight r %}
topfav = all_tweets %>%
  group_by(Hashtag) %>%
  filter(favoriteCount == max(favoriteCount)) %>%
  distinct(Hashtag, .keep_all = TRUE)

topfavicots = topfav %>%
  filter(Hashtag == "#ICOTS10")

topfavicml = topfav %>%
  filter(Hashtag == "#ICML2018")

topfavuser = topfav %>%
  filter(Hashtag == "#useR2018")
{% endhighlight %}

Check out the most favourited [ICOTS tweet](https://twitter.com/anyuser/status/1017244182035849216), [ICML tweet](https://twitter.com/anyuser/status/1017059295052140544) and [useR tweet](https://twitter.com/anyuser/status/1016860465807257600).

### Emojis ❤️

What better place to explore emojis from than the country that gave us the language that gave us the word! Ya follow? Emoji is a Japanese word that, if a quick Google search is to be believed, means "picture character".

Anna Fergusson, (@annafergussonnz)\[<https://twitter.com/annafergussonnz>\], and I, in our talk on (modern data in a large introductory statistics course)\[<https://annafergusson.github.io/paristergram/>\] based on the awesome work Anna has been driving, tried to quickly show how emoji analysis can be a bit of fun. So let's keep that up here.

This first section is mostly just copied from the work Anna and I did that will be up on the (Parisgram site)\[<https://annafergusson.github.io/paristergram/>\] at somepoint. The emoji handling is not very elegant right now.

There is a table provided by the Unicode organisation that has the emoji, and an additional one that details skintone variations. These also provide a text description and groups the emoji into wider groups, like flags, buildings, types of facial expression etc. It would be ideal to have that information in R to work with directly, so I want to scrape the tables into R. I can check if this seems to be allowed by using the `robotstxt` package. There are a range of other ways to get databases like this, this just seemed easiest for this case.

### Checking if we can scrape the emoji lists

{% highlight r %}
# These are the url with the  list of emoji codes from Unicode
url_base = "https://unicode.org/emoji/charts/full-emoji-list.html"
url_skin = "https://unicode.org/emoji/charts/full-emoji-modifiers.html"
{% endhighlight %}

{% highlight r %}
# We can check out the robots.txt file right from R
robotstxt::get_robotstxt("unicode.org")
{% endhighlight %}

I've removed the output, but when I ran it, the bits we wanted were okay, and the crawl delays were only for certain bots.

The `paths_allowed()` functions will return true if the path is allowed, based on the robots.txt file for the domain.

{% highlight r %}
robotstxt::paths_allowed(url_base)
{% endhighlight %}

    ## [1] TRUE

{% highlight r %}
robotstxt::paths_allowed(url_skin)
{% endhighlight %}

    ## [1] TRUE

Both come back `TRUE`, we are allowed to scrape them.

### Scraping the emoji lists

{% highlight r %}
emoji_base = read_html(url_base) %>%
  html_nodes("table") %>%
  html_table()

emoji_skin = read_html(url_skin) %>%
  html_nodes("table") %>%
  html_table()
{% endhighlight %}

Now to clean up the tables we scraped, so they are nice for us to use later.

{% highlight r %}
emoji_ref_interim1 = as.data.frame(emoji_base) %>%
  select(Smileys.1, Smileys.2, Smileys.14) %>%
  rename(code = Smileys.1, emoji = Smileys.2, short_name = Smileys.14) %>%
  filter(code != "Code") %>%
  mutate(descripID = code == emoji) %>%
  mutate(table = "base")

# The "keycap: \*" emoji caused me no end of headaches and I have no idea why

emoji_ref_interim2 = as.data.frame(emoji_skin) %>%
  select(People.1, People.2, People.14) %>%
  rename(code = People.1, emoji = People.2, short_name = People.14) %>%
  filter(code != "Code") %>%
  mutate(descripID = code == emoji) %>%
  mutate(table = "skin")

emoji_ref_interim = rbind(emoji_ref_interim1, emoji_ref_interim2)

# This part here just lets us get the group descriptions for the emoji from the table and make them a variable
descriptions = as.data.frame(emoji_ref_interim) %>%
  filter(descripID == TRUE) %>%
  tibble::rownames_to_column(var = "number")

times = c(diff(which(emoji_ref_interim$descripID)), length(emoji_ref_interim$descripID) - max(which(emoji_ref_interim$descripID))+1)

group_desc = rep(descriptions$code, times-1)

emoji_ref = emoji_ref_interim %>%
  filter(descripID == FALSE) %>%  
  mutate(group_description = group_desc) %>%
  tibble::rownames_to_column(var = "number") %>%
  mutate(number = as.numeric(number)) %>%
  mutate(character_length = nchar(code)) %>%
  arrange(number)
{% endhighlight %}

One wrinkle is that as far as I can tell, you can't stop the Twitter app from truncating tweets that have been retweeted when using twitteR. This seems possible in rtweet - but I couldn't get the authentication working. More [on the GitHub issues page](https://github.com/mkearney/rtweet/issues/179).

So, I'm just going to try to make this work out by getting all the unique tweets ("RT @username" will be in front of the retweeted ones) and then removing all the tweets that had been truncated (i.e. retweeted). If my logic is right, this should leave us with the untruncated text of all tweets, just some of them will be the retweeted version. This shouldn't effect the emojis.

I am also going to remove the midway summary tweet that I shared with the current top emojis as that would increase the counts for those tweets.

<blockquote class="twitter-tweet" data-lang="en">
<p lang="en" dir="ltr">
Top 5ish emojis from three different conferences (as of 15 mins ago). Ranked 1-5ish, left to right, in ( ) means equal counts.<a href="https://twitter.com/hashtag/ICOTS10?src=hash&amp;ref_src=twsrc%5Etfw">\#ICOTS10</a> 😍👏(😂🐱🎨🎼)<a href="https://twitter.com/hashtag/useR2018?src=hash&amp;ref_src=twsrc%5Etfw">\#useR2018</a>: 📦👌(😍🌧️)<a href="https://twitter.com/hashtag/ICML2018?src=hash&amp;ref_src=twsrc%5Etfw">\#ICML2018</a>: 🔥😂(😁🤔🙌✈️✔️)<br><br>Code: <a href="https://t.co/yixpezWy8p">https://t.co/yixpezWy8p</a><br>Of course, this tweet will mess it up!
</p>
— Liza Bolton (@Liza\_Bolton) <a href="https://twitter.com/Liza_Bolton/status/1017216200697212928?ref_src=twsrc%5Etfw">July 12, 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
{% highlight r %}
unique_icots = icots_tweets %>%
  filter(id != 1017216200697212928) %>%
  distinct(text, .keep_all=TRUE) %>%
  filter(truncated == FALSE)

unique_user = user_tweets %>%   
  filter(id != 1017216200697212928) %>%
  distinct(text, .keep_all=TRUE) %>%
  filter(truncated == FALSE)

unique_icml = icml_tweets %>%
  filter(id != 1017216200697212928) %>%
  distinct(text, .keep_all=TRUE) %>%
  filter(truncated == FALSE)

emoji_list_icots = str_extract_all(unique_icots$text, '[\U{10000}-\U{1FFFF}]|[\U{20A0}-\U{27FF}]')
emoji_icots = data.table::rbindlist(lapply(emoji_list_icots, as.data.frame), id="src", fill = TRUE, use.names = TRUE)

emoji_spread_icots = emoji_icots %>%
  rename(ID = src, emoji = `X[[i]]`) %>%
  group_by(ID) %>%
  mutate(counter = row_number()) %>%
  spread(counter, emoji, fill = NA) %>%
  unite(all, -ID, sep="", remove=FALSE) %>%
  mutate(all = str_replace_all(all, "NA", ""))

emoji_list_user = str_extract_all(unique_user$text, '[\U{10000}-\U{1FFFF}]|[\U{20A0}-\U{27FF}]')
emoji_user = data.table::rbindlist(lapply(emoji_list_user, as.data.frame), id="src", fill = TRUE, use.names = TRUE)

emoji_spread_user = emoji_user %>%
  rename(ID = src, emoji = `X[[i]]`) %>%
  group_by(ID) %>%
  mutate(counter = row_number()) %>%
  spread(counter, emoji, fill = NA) %>%
  unite(all, -ID, sep="", remove=FALSE) %>%
  mutate(all = str_replace_all(all, "NA", ""))

emoji_list_icml = str_extract_all(unique_icml$text, '[\U{10000}-\U{1FFFF}]|[\U{20A0}-\U{27FF}]')
emoji_icml = data.table::rbindlist(lapply(emoji_list_icml, as.data.frame), id="src", fill = TRUE, use.names = TRUE)

emoji_spread_icml = emoji_icml %>%
  rename(ID = src, emoji = `X[[i]]`) %>%
  group_by(ID) %>%
  mutate(counter = row_number()) %>%
  spread(counter, emoji, fill = NA) %>%
  unite(all, -ID, sep="", remove=FALSE) %>%
  mutate(all = str_replace_all(all, "NA", ""))
{% endhighlight %}

How many tweets (not counting retweets) use at least one emoji in this data?

{% highlight r %}
emoji_prop_icots = sum(!is.na(emoji_spread_icots$`1`))/dim(unique_icots)[1]
emoji_prop_user = sum(!is.na(emoji_spread_user$`1`))/dim(unique_user)[1]
emoji_prop_icml = sum(!is.na(emoji_spread_icml$`1`))/dim(unique_icml)[1]
{% endhighlight %}

So 7% of ICOTS tweets had at least one emoji in them, 11% of useR tweets and 3% of ICML tweets.

### Which emoji were the most popular?

This is a not very elegant function to count how many times each emoji in the emoji\_ref dataframe appears in the text you're analysing (in the form of the all columns from the emoji\_spread dataset). I'm also not 100% sure the tryCatch is set up properly - it needs a tryCatch because I keep getting errors for the keycap: \* emoji.

The second function doesn't count repeats of emoji in that same tweet. I know at least one of my tweets had a gratuitous number of hand clap emojis. Will not counting repeats in a tweet change the top used emoji? `str_detect` is quite useful for that.

{% highlight r %}
emoji_counter = function(data_col, emoji_from_ref){
  counts = tryCatch({
    sum(str_count(data_col, emoji_from_ref))
  }, error=function(e)counts=NA, {}
  )
  return(counts)
}

emoji_counter_single = function(data_col, emoji_from_ref){
  counts = tryCatch({
    sum(str_detect(data_col, emoji_from_ref))
  }, error=function(e)counts=NA, {}
  )
  return(counts)
}
{% endhighlight %}

You can short by either ranking (`count` or `rank` include repeats in a tweet, `count_single` and `rank_single` don't include repeats within a tweet)

#### Top \#ICOTS10 emojis

{% highlight r %}
# This can take a bit of time to run
emoji_counts_icots = emoji_ref %>%
  rowwise() %>%
  mutate(counts = NA) %>%
  mutate(counts = emoji_counter(emoji_spread_icots$all, emoji)) %>%
  mutate(counts_single = emoji_counter_single(emoji_spread_icots$all, emoji)) %>%
  arrange(number) %>%
  mutate(rank = round(rank(-counts), 0)) %>%
  mutate(rank_single = round(rank(-counts_single), 0)) %>%
  select(-c(character_length, number)) %>%
  filter(counts > 0) %>%
  arrange(desc(counts)) %>%
  select(emoji, short_name, counts, rank, counts_single, rank_single)

frameWidget(DT::datatable(emoji_counts_icots))
{% endhighlight %}

<iframe src="http://blog.dataembassy.co.nz/images/conferencetweets_files/figure-markdown_github/widgets/widget_icots_emoji.html" width="100%" height="500" style="border: none;">
</iframe>
#### Top \#useR2018 emoji

{% highlight r %}
emoji_counts_user = emoji_ref %>%
  rowwise() %>%
  mutate(counts = NA) %>%
  mutate(counts = emoji_counter(emoji_spread_user$all, emoji)) %>%
  mutate(counts_single = emoji_counter_single(emoji_spread_user$all, emoji)) %>%
  arrange(number) %>%
  mutate(rank = round(rank(-counts), 0)) %>%
  mutate(rank_single = round(rank(-counts_single), 0)) %>%
  select(-c(character_length, number)) %>%
  filter(counts > 0) %>%
  arrange(desc(counts)) %>%
  select(emoji, short_name, counts, rank, counts_single, rank_single)

frameWidget(DT::datatable(emoji_counts_user))
{% endhighlight %}

<iframe src="http://blog.dataembassy.co.nz/images/conferencetweets_files/figure-markdown_github/widgets/widget_user_emoji.html" width="100%" height="500" style="border: none;">
</iframe>
#### Top \#ICML2018 emoji

{% highlight r %}
emoji_counts_icml = emoji_ref %>%
  rowwise() %>%
  mutate(counts = NA) %>%
  mutate(counts = emoji_counter(emoji_spread_icml$all, emoji)) %>%
  mutate(counts_single = emoji_counter_single(emoji_spread_icml$all, emoji)) %>%
  arrange(number) %>%
  mutate(rank = round(rank(-counts), 0)) %>%
  mutate(rank_single = round(rank(-counts_single), 0)) %>%
  select(-c(character_length, number)) %>%
  filter(counts > 0) %>%
  arrange(desc(counts)) %>%
  select(emoji, short_name, counts, rank, counts_single, rank_single)

frameWidget(DT::datatable(emoji_counts_icml))
{% endhighlight %}

<iframe src="http://blog.dataembassy.co.nz/images/conferencetweets_files/figure-markdown_github/widgets/widget_icml_emoji.html" width="100%" height="500" style="border: none;">
</iframe>
More from ICOTS
---------------

Most of my tweets from this conference are in the below thread and I also set up a [list of resources](http://blog.dataembassy.co.nz/icots10/) to check out after the conference based on people's recommendations.

<blockquote class="twitter-tweet" data-lang="en">
<p lang="en" dir="ltr">
Delighted to be at <a href="https://twitter.com/hashtag/icots10?src=hash&amp;ref_src=twsrc%5Etfw">\#icots10</a> with an awesome crew from <a href="https://twitter.com/statsauckuni?ref_src=twsrc%5Etfw">@statsauckuni</a>, and almost 500 other delegates from 59 countries. Ft. <a href="https://twitter.com/annafergussonnz?ref_src=twsrc%5Etfw">@annafergussonnz</a> <a href="https://twitter.com/rhysauck?ref_src=twsrc%5Etfw">@rhysauck</a> <a href="https://twitter.com/Emmawonwon?ref_src=twsrc%5Etfw">@Emmawonwon</a> <a href="https://t.co/nnKnezTfsu">pic.twitter.com/nnKnezTfsu</a>
</p>
— Liza Bolton (@Liza\_Bolton) <a href="https://twitter.com/Liza_Bolton/status/1015879520576942080?ref_src=twsrc%5Etfw">July 8, 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
Vanity time: My best tweet?
---------------------------

I kinda want to know which of my tweets did the best too...

{% highlight r %}
mytweets_noRT = all_noRT %>%
  filter(Hashtag == "#ICOTS10") %>%
  filter(screenName == "Liza_Bolton")

mytweets = all_tweets %>%
  filter(Hashtag == "#ICOTS10") %>%
  filter(screenName == "Liza_Bolton")

myprop_noRT = dim(mytweets_noRT)[1]/dim(filter(all_noRT, Hashtag == "#ICOTS10"))[1]
myprop = dim(mytweets)[1]/dim(filter(all_tweets, Hashtag == "#ICOTS10"))[1]

myfavtweets = mytweets_noRT %>%
  arrange(desc(retweetCount)) %>%
  select(id, text, retweetCount, favoriteCount) %>%
  filter(retweetCount == max(retweetCount)) %>%
  filter(favoriteCount == max(favoriteCount))
{% endhighlight %}

[Turns out this one was pretty good](https://twitter.com/anyuser/status/1016130460697522177), with 8 retweets and 31 favourites. Thanks guys!

In earlier versions of this post I calculated what proportion of the \#ICOTS10 tweets were my tweets but used ALL ICOTS tweets, including all the many retweets. I tweeted 6% of the \#ICOTS10 tweets including retweets (my own and others'), but 15% of the original \#ICOTS10 tweets (i.e. not counting retweets). What a loudmouth!
