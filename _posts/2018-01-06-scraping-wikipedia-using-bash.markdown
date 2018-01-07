---
layout: post
title:  "Scraping Wikipedia using Bash"
date:   2018-01-06 12:36:10 -0500
categories: bash
---

_You may also want to read [graphing age distribution by gender of Oscar nominees]({% post_url 2018-01-06-graphing-age-distribution-by-gender-of-oscar-nominees %})._

__Source code:__ The code for this experiment can be found at [cldellow/wiki-actors](https://github.com/cldellow/wiki-actors)

Are older women less likely to be successful in Hollywood than older men?

<div style='float: right' markdown='1'>
![Casey Affleck]({{ site.url }}/assets/img/casey-affleck-infobox.png)
</div>


This was a question I wanted to answer recently. To make it more tractable, I scoped it to: is there a meaningful age difference between Best Actor and Best Actress nominations?

To answer this, I needed some data. Wikipedia conveniently has a list of [Best Actor](https://en.wikipedia.org/wiki/Academy_Award_for_Best_Actor) and [Best Actress](https://en.wikipedia.org/wiki/Academy_Award_for_Best_Actress) nominations going back to the Academy Award founding in 1928.

The table of data also links to the actor's page, and most actor pages have what Wikipedia calls an "infobox" with vital statistics about them, including their birthday. 

This is looking pretty promising! We can get a list of actors, then get their age at the time of the awards ceremony. We just have to scrape the data first.

# Tools

Now, we could do this with a modern language and a purpose-built library, say something like Python and [scrapy](https://scrapy.org/)...but where's the fun in that? Instead, we'll do it in bash.

* `bash` - process management + orchestration
* `curl` - HTTP client to fetch things from Wikipedia
* [`pup`](https://github.com/ericchiang/pup) - tool to select HTML elements from the command line via CSS selectors
* `awk` - an all-purpose text processing tool
* `date` - a tool to manipulate time itself
* `ministat`  - statistical analysis on sets of numbers

# Process

I built this iteratively. My main idea was to build a set of composable
functions in the [`wp`](https://github.com/cldellow/wiki-actors/blob/master/wp) script:
a function to get the infobox for an actor, a function to parse the birthdate from the
infobox, a function to compute age given two dates, etc.

This made the development cycle very natural -- bash has a killer REPL. It's called bash. :)

# Scaffolding

The `wp` script starts pretty dumb:

```bash
"$@"
```

That is, it takes the arguments you provided at the command line (which are stored
in the `$@` metavariable), and tries to invoke them like a normal command.

This lets us expose new commands by simply defining a function.

# Infobox

To be a polite Internet user, I wrote a script to cache URLs -- if the URL is
not present locally, it'll download it. Otherwise, it serves the local copy.

Because we don't care about most of the contents of the page, it also permits
taking a CSS selector. When a CSS selector is provided, only that portion of
the page is retained.

Let's wire that up:

```bash
function infobox {
  bin/fetch "$1" .infobox
}
```

`./wp infobox George_Clooney` now spews out the HTML of his infobox.

# Birthday

I started with the simplest task: given an actor, get their birthdate in YYYY-MM-DD format.

By inspecting the output of [`./wp infobox George_Clooney`](https://gist.github.com/cldellow/c0fd24d4a91e28c127827863a2961869), we see this:

```html
<span>(<span class="bday">1961-05-06</span>)</span>May 6, 1961<span class="noprint ForceAgeToShow">(age 56)</span>
```

That's pretty promising! Could this really be as easy as?

```bash
$ ./wp infobox George_Clooney | pup '.bday text{}'
1961-05-06
```

...yes! So we'll add a birthday command:

```bash
function bday {
  infobox "$@" | pup '.bday text{}'
}
```

# Age

Now we need to know how old the person is. For our purposes, a rough approximation
is sufficient--we don't need to worry about leap years and timezones.

`date` shines here. We can pass it a format string so that it formats a date
in seconds since the epoch:

```bash
$ date +%s --date 1961-05-06
-273182400
```

Don't worry too much about the negative number: George Clooney is older than
UNIX, so his birthday starts before 0.

Now we can use bash arithmetic to figure out Clooney's age in years:

```bash
function age {
  from=$(bday "$1")
  to=${2:-$(date +%Y-%m-%d)}

  from_seconds=$(date +%s --date "$from")
  to_seconds=$(date +%s --date "$to")

  echo $(( (from_seconds - to_seconds) / 60 / 60 / 24 / 365 ))
}
```

Note that `to` is an optional YYYY-MM-DD parameter. This lets us ask for the person's
age as of a given date (which will be useful, since there are 89 different
Academy Award ceremony dates). If not provided, we use the `:-` syntax to provide
a default of today's date.

The final line uses bash's arithmetic expansion operation of `$(( ... ))` and translates
from seconds to years.

```bash
$ ./wp age George_Clooney
56

$ ./wp age George_Clooney 1970-07-20
9
```

# Awards

It gets a bit gnarly now. We have this table:

![Best Actor table]({{ site.url }}/assets/img/best-actor-table.png)

They've done a nice job of formatting the table for visual display by making the
year column span all the rows with the nominees for that year. Unfortunately,
that's going to make it hard to mechanically parse: from the point of view
of the HTML, the year is only attached to the first row (with the winner),
not the subsequent rows (with the nominees).

We can work around this by parsing the table line by line and remembering
which year we're working on.

First, let's figure out what CSS selector to use to grab the table:

```bash
$ bin/fetch Academy_Award_for_Best_Actor '.sortable:not(.plainrowheaders)' |
  pup
[...spew of HTML...]
```

This lends itself to making a little state machine in awk:

```awk
# Every row should have an actor.
/<tr>/ {
  found_actor=0
}

# Extract the year from the "1927_in_film" link.
match($0, /([0-9]+)_in_film/, result) {
  year=result[1]
  mode="won"
}

# The first link that isn't to a year or an Academy Awards page is the actor.
!found_actor && !/_in_film/ && !/_Academy_Awards/ && match($0, /href="\/wiki\/([^"]+)"/, result) {
  print year, result[1], mode
  mode="nominated"
  found_actor=1
}
```

We'll add these as the `best_actor` and `best_actress` functions:

```bash
$ ./wp best_actor | head -n5
1927 Emil_Jannings won
1927 Richard_Barthelmess nominated
1928 Warner_Baxter won
1928 George_Bancroft_(actor) nominated
1928 Chester_Morris nominated
```

# Age, redux

We can glue our existing functions together to annotate nominees with their age at nomination:

```bash
function ages {
  "$@" | while read -r year slug result; do
    echo "$year $slug $result $(age "$slug" "$year-01-01")"
  done
}
```

```bash
$ ./wp ages best_actor | head -n5
1927 Emil_Jannings won 42
1927 Richard_Barthelmess nominated 31
1928 Warner_Baxter won 38
1928 George_Bancroft_(actor) nominated 45
1928 Chester_Morris nominated 26
```

Note that we're assuming that the award ceremony is held on the first of the year. This is wrong, but good enough.

# Finally, an answer

We finally have the data we need! But how do we compare several hundred numbers? [`ministat`](https://www.freebsd.org/cgi/man.cgi?ministat) is your friend!

```bash
$ ./wp ages best_actor > actor-ages
$ ./wp ages best_actress > actress-ages
$ ministat -s -C 4 actor-ages actress-ages
x actor-ages
+ actress-ages
+--------------------------------------------------------------------------+
|                         +                                                |
|                         +                                                |
|                      +  +                                                |
|                     ++  +                                                |
|                     ++x +                                                |
|                     ++x +                                                |
|                     ++x +  x x                                           |
|                   + ++x +  x x      x                                    |
|                +  + ++x +  *xx      x                                    |
|                +  + ++*++  *xx      x                                    |
|                +  + ++*++x *xxx     x                                    |
|                +  + ++*++* *xxxx    x                                    |
|                +  + ++**+* **xxx  x x                                    |
|                +  ++++**+****xxx xx x                                    |
|                ++ ++++*******x*x xx x                                    |
|               +++++++********x*x xx x                                    |
|               ++++++*********x*x xx x                                    |
|               ++++++*********x*x xx x+x                                  |
|               +++++**********x*x xx x*x                                  |
|              ++++++**********x*x *x+**x    xx x                          |
|             +++++++************x *x+**x  x xx x                          |
|            +++++++*************x**x***xx xxxx x x                        |
|            +++++*+****************x***xx xxxxxx x x                      |
|            +++++*+*********************xxx*xxxx xxx                      |
|           ++++++***********************xxx*xxxxxxxx++  +                 |
|           +++++**************************x*xxxxx**x*+x + x               |
|           ++*++**************************x*xx*xx**x*+x * * x x    +      |
|*   +     x++*****************************x***********x *** ***+xx *+    +|
|                      |________M_A_________|                              |
|                |________M_A__________|                                   |
+--------------------------------------------------------------------------+
    N           Min           Max        Median           Avg        Stddev
    x 437             8            78            40       42.1373      10.88427
    + 441             8            84            34       36.0839     11.491686
    Difference at 95.0% confidence
      -6.0534 +/- 1.48084
        -14.3659% +/- 3.51433%
          (Student's t, pooled s = 11.1935)
```

Women nominated for best actress are much younger than men nominated for best actor.
