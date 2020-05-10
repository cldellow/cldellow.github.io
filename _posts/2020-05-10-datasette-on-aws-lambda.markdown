---
title:  "Running Datasette on AWS Lambda"
author: cldellow
tags:
  - datasette
  - aws
  - lambda
  - mangum
  - apigateway
---

I love software tools that (a) are opinionated and (b) get out of the developer's way. These seem like principles that are in tension with each other, but they don't have to be.

Consider the problem of exposing a data layer to a web stack. We're spoiled for choice:

<center>
<div><a href='//sketchviz.com/@cldellow/e10de81267c04a3996572620a02ad77d'><img src='https://sketchviz.com/@cldellow/e10de81267c04a3996572620a02ad77d/0ecc15e6811749d1fa8ad64f6e4333abd02dacfa.sketchy.png' style='max-width: 100%;'></a><br/><span style='font-size: 80%;color:#555;'>Hosted on <a href='//sketchviz.com/' style='color:#555;'>Sketchviz</a></span></div>
</center>

By _opinionated_, I mean that the tool has a clear vision on how to accomplish the task at hand. This usually means the authors have spent many hours thoughtfully considering the problem and making judicious tradeoffs.

By _in the way_, I mean the tool's vision doesn't match your own. :)

In the particular case of data access libraries, I'm a huge fan of systems that view the power of SQL, window functions, common table expressions and all, as a positive, not a negative to be hidden from the user. This can allow very rapid prototyping for people comfortable with SQL.

For Postgres, there's [PostgREST](http://postgrest.org/en/v7.0.0/). For SQLite, there's [Datasette](https://github.com/simonw/datasette).

Since [my love of SQLite](https://cldellow.com/2018/06/22/sqlite-parquet-vtable.html) is no secret, I've recently been spending a lot of time with Datasette. In particular, I hacked up the CloudFormation glue to make it possible to run Datasette fully serverlessly, using Amazon API Gateway, AWS Lambda, S3 and [the excellent mangum adapter for ASGI](https://github.com/erm/mangum).

You can see the results in the [datasette-lambda](https://github.com/code402/datasette-lambda) repository for now. At some point, I'll package it into a publish plugin for Datasette itself.
