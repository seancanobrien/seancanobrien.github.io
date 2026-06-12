---
title: ArXiv weekly
---

# Weekly email summaries of ArXiv submissions

I like the [daily emails](https://info.arxiv.org/help/subscribe.html) that the ArXiv has for new submissions.
However, submissions are abundant and most are not immediately relevant.
There is a nice [API](https://info.arxiv.org/help/api/index.html) for searching articles and downloading metadata, and I use this API to generate weekly summaries of submissions based on keyword filters, which I send to myself each Monday.

The code for this is on [github](https://github.com/seancanobrien/arxiv_weekly_summary).

### What this does
The main input is a filter file, which specifies repositories, authors and search strings.
It strictly matches repositories, and is more lenient with other search criteria.
```
example_filter.txt
--------------------
Anything before the first header (RePOSITORIES: etc) is ignored.
We can put an email here, for use in another script.

# Comment lines are ignored
REPOSITORIES:
math.gr
math.at

AUTHORS:
Terrence Tao
Leonhard Euler
# Hyphens and potentially other symbols should be substituted for underscores.
Jean_Pierre Serre

# These headers are case sensitive and must contain the colon
KEYWORDS:
artin
coxter
braid
maths is cool
```
The main script (ideally scheduled by Cron) does the following:

- The python script pulls relevant articles using the API and makes a summary which is a `.html` file.
- [Mutt](http://www.mutt.org/) sends this `.html` file to my email.
I use a [Zoho mail](https://www.zoho.com/mail/) to send these emails.
Zoho provides an SMTP server, which plays nicely with Mutt.

These emails look like this:

<div class="email-preview" markdown="0">

<h1>Arχiv Weekly Update</h1>
<h4>Mon 2026-04-20 to Sun 2026-04-26</h4>
<h3>Search Criteria</h3>
<ul>
<li><strong>Subject categories</strong>: <a href="https://arxiv.org/list/math.GR/recent">math.GR</a>, <a href="https://arxiv.org/list/math.AT/recent">math.AT</a>, <a href="https://arxiv.org/list/math.GT/recent">math.GT</a></li>
<li><strong>Match authors</strong>: Terrence Tao, Leonhard Euler, Jean_Pierre Serre</li>
<li><strong>Match title or abstract</strong>: artin, hyperplane arrangement, raag, coxter, braid, hurwitz</li>
</ul>
<hr />
<h3><a href="http://arxiv.org/abs/2306.10519v3">Handle decompositions and Kirby diagrams for the complement of plane algebraic curves</a></h3>
<p><strong>Authors:</strong> Sakumi Sugawara</p>
<p><strong>Article updated (V3)</strong>: 2026-04-23 02:45:01. <strong>Originally published</strong>: 2023-06-18 10:48:43</p>
<p><strong>Categories:</strong> <u>math.GT</u>, math.AG</p>
<p><strong>Abstract:</strong></p>
<p>The complement of plane algebraic curves are well studied from topological and algebro-geometric viewpoints. In this paper, we will describe the explicit handle decompositions and the Kirby diagrams for the complement of plane algebraic curves. The method is based on the notion of braid monodromy. We refined this technique to obtain handle decompositions and Kirby diagrams.</p>
<hr />
...
<p>Thank you to arXiv for use of its open access interoperability.</p>
<p>Link to its <a href="https://info.arxiv.org/help/api/index.html">API</a>, which this makes use of.</p>

</div>

---

**Get in touch if you have any questions.**
