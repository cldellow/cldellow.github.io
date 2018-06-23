---
layout: post
title:  "Graphing Age Distribution by Gender of Oscar Nominees"
date:   2018-01-06 12:36:10 -0500
author: cldellow
tags:
  - vega-lite
---

_You may also want to read [scraping Wikipedia using bash]({% post_url 2018-01-06-scraping-wikipedia-using-bash %})._

That there are no roles in Hollywood for women over 40 is a common cliche. I was curious to test the truth of it, so I collected some data on the ages of nominees for Best Actor and Best Actress. In the boxplot below, half of the nominees in a given decade fall in the age range delimited by the coloured box.

<!-- Import Vega 3 & Vega-Lite 2 (does not have to be from CDN) -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/vega/3.0.7/vega.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/vega-lite/2.0.3/vega-lite.js"></script>
<!-- Import vega-embed -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/vega-embed/3.0.0-rc7/vega-embed.js"></script>

<div id="vis"></div>

<script type="text/javascript">
  var spec = "https://gist.githubusercontent.com/cldellow/3fca75a008eaf9348f65204efb32a0a9/raw/1ae3d3e0c7c077920fc5b61179ab833b63a31c02/oscar-genders.json";
  vegaEmbed('#vis', spec).then(function(result) {
    // access view as result.view
    }).catch(console.error);
</script>

Of course, this isn't _actually_ measuring the number of roles portraying women over 40, it's only measuring number of roles played by women over 40, and then, only for a critically acclaimed subset.

Still, you'd expect that people up for Best Actor would skew old on the theory that older people are more experienced, and thus more likely to attract award-winning roles. Despite this, the median age of nominees for Best Actress is 34 (vs 40 for Best Actor).

Oh, and the inaugural winners? Emil Jannings, 45 and Janet Gaynor, 22.
