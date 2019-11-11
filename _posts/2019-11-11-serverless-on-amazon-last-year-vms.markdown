---
title:  "Serverless on Amazon: Is 2019 the Last Year I'll Have to Deal With VMs?"
author: cldellow
canonical_url: 'https://dev.to/cldellow/serverless-on-amazon-is-2019-the-last-year-i-ll-have-to-deal-with-vms-2a7'
tags:
  - serverless
  - aws
  - lambda
  - devops
---

The last few months have seen me flirt with Amazon API Gateway and AWS Lambda, aka "serverless computing". I come from a traditional ops background. I’m very comfortable running my own VMs, nginx load balancers, Varnish caches, etc. That stack delivers amazing performance at a great price point, but it also comes with significant ops burden. I had a few projects I wanted to build, but the thought of the ongoing ops effort to keep them running was holding me back.

Serverless seems like the perfect answer, right? We can all picture an overly-enthusiastic Amazon Developer Evangelist saying things like:

- No server configuration boilerplate, just focus on your business logic!
- We'll handle security patches for you!
- Don't worry about scaling to handle traffic spikes, it "just works"!
- Only pay when your code is running!

And maybe you've heard the counter arguments from the old-school developers:

- Lambda cold starts mean users get frustratingly inconsistent response times.
- Debugging a serverless app requires duct-taping no fewer than 17 Rube Goldberg machines together, just to see a message informing you that your log files are in another castle.
- The tooling, in general, is just really immature.

So, who's right, the hypers or the haters? For me, the answer is "it depends." (I can hear you groaning from here. Sorry.) I tried serverless for three different workloads:

1. Cron jobs (**A+**, would 100% recommend)
2. APIs (**B**, probably a good fit, but not always)
3. A full-blown website (**D**, probably not a good fit)

Let’s dive in to each.

# Cron jobs

Although [cron](https://en.wikipedia.org/wiki/Cron) originally referred to the 45-year-old tool made by Bell Labs, it has become a generic term for any tool that lets you schedule work to happen at regular intervals.

For example, maybe you need a reporting job that runs each day to query your database, summarize new sales, and e-mail a report to your team.

In the old days, you'd pay for a server to be on 24/7, you'd deploy your code to it somehow, and you'd put some [cryptic entry](https://crontab.guru/) in its crontab file. If all went well, the job would run regularly. If the job failed, you’d get an email with its output. Maybe. If you configured anything wrong, it might just fail silently. And even though your job only took 10 minutes to run, you're still on the hook to pay for the server for the other 99.3% of the day.

In the serverless model, you write an AWS Lambda function and then [configure a CloudWatch Events Rule that triggers on a schedule](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/Create-CloudWatch-Events-Scheduled-Rule.html).

Under the old model, you'd pay $4.30/month for the cheapest EC2 instance, a t3a.nano, with an 8GB SSD drive. You'd have to keep it patched and secured against hackers.

Under the serverless model, you’d...need a spreadsheet to figure out what it costs. Your Lambda bill has two parts: a fixed per-request price, and a variable per-request price that goes up with how long your function takes to run (rounded to the nearest 100ms), and how much memory it uses. The [Lambda pricing page](https://aws.amazon.com/lambda/pricing/) has a table of prices that would make an accountant go cross-eyed. Confused yet? I was, too, so here’s a worked-out equation of what this example would cost:

31 requests * ($0.0000002/request + 10 minutes/request * 60 seconds/minute * 10 100ms blocks/1 second * $0.000000834/100ms for 512 MB) =  $0.16

That's **94%** cheaper, and you have practically no traditional sysadmin chores to do!

Why _wouldn't_ you want serverless?

Well, it's true about cold starts: each day, when your reporting job runs, it'll be delayed for several seconds as Lambda downloads your code, unpacks it, and prepares to run it for the first time that day. But do you really care if you get your report at midnight exactly, or a few seconds after? Didn't think so.

Lambda also has limited storage - you can store files in `/tmp`, but it's a paltry 512 MB. If you need a lot of temporary storage, that's a no go.

There are [other limits](https://docs.aws.amazon.com/lambda/latest/dg/limits.html), too. If your function takes a long time to run, you'll need to figure out how to break it down so each invocation is no longer than 15 minutes.

Logging can get pricey. Lambda uses CloudWatch Logs, which charges an eye-watering $0.50 per GB of logs. And that's just to receive them! You'll also pay to store them (although, thankfully, you can set aggressive retention policies to minimize this cost) and to query them.

My specific cron jobs fit well within these constraints. I loved the experience of using serverless for cron, and should have switched a long time ago. **A+**

_(The cron jobs were for Code 402 Crawls, a service that makes it easy to [search the Common Crawl](https://code402.com/crawls). Shameless plug: check it out!)_

# APIs

The next project was inspired by my experiences trying out serverless. I had been working with Lambda and it was great. Change something, package the code, upload it, test it. Lather, rinse, repeat. And then I moved to an apartment with. Really. Slow. Internet. Each time I uploaded a 30-100 MB package for Lambda, the Internet was unusable for several minutes. It was a short-term rental, so upgrading the Internet wasn't possible. Sure, I could have used [AWS Lambda Layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) or [AWS CodeBuild](https://aws.amazon.com/codebuild/). But I'm old. I'm set in my ways. I already had a system. And to add insult to injury, each upload was _logically_ very small: sure, it was a big file, but the changes from the previous version were really small.

For each upload, this happened:

[![Me, sad at my slow 50MB upload](https://thepracticaldev.s3.amazonaws.com/i/gk1mxpt2jk896ds6pvvo.png)](https://sketchviz.com/@cldellow/dfa592bb3c91b10106bf6fcf8222a2fd/0c2341b3b5667adab7521d923802b00188caf061)

But I knew something like this should be possible:

[![Me, happy at my fast 50KB upload](https://thepracticaldev.s3.amazonaws.com/i/o7vrmanb1tc78l6drb2z.png)](https://sketchviz.com/@cldellow/f885fbb3db183704ea11f952e24fa301/2d6ea1fd9304012c527a86f607a4b2182fa36465)

Spoiler: it _was_ possible!

Except that to make this tool available for everyone, I needed not just one Lambda, but 16! If I didn't have a Lambda in each region where a user might have an S3 bucket, I'd have to pay [Amazon’s extortionate data transfer fees](https://19x50e48lpyz2s9tzz3qjjsn-wpengine.netdna-ssl.com/wp-content/uploads/2019/09/datatxfr-nov2019@2x.png).

_And_ I wanted to have an API Gateway in each region, so that I could give people a command-line tool that invoked the service with a plain old HTTP request, instead of having to use the AWS SDK and embed my credentials in the command-line tool.

...and this is when it became obvious that maintaining this by hand would lead to madness. Instead, I wrote an [AWS CloudFormation template](https://aws.amazon.com/cloudformation/) that could script it all, from registering domain names and requesting SSL certificates to configuring logging and publishing the Lambda function.

So, how did deploying this as a serverless system compare to a traditional VM-based approach?

Pretty well! The same cost savings applied, except now they were multiplied by the 16 regions where my Lambdas ran.

The main concern was cold start times. Because most people don't spend their whole day uploading code, it was pretty common for a use of the service to trigger a cold start and its 3-4 second delay. But this turned out to not be a big deal. After all, the alternative was a several minute upload.

I think for a lot of APIs, the same reasoning would apply. If the API is being consumed by an automated process or is replacing an otherwise very slow operation, the cons of serverless are well worth the benefits. **B**

_(You can use this service, too. I call it s3patch, but I think of it like [rsync for S3](https://s3patch.com/).)_

# A full-blown website

Drunk with power, I figured it was time to put Lambda to the ultimate test: run a whole user-facing web site on it.

And this is where things came unglued.

Remember how I said I'm old and stuck in my ways? Although you can write Lambdas that have [sub-second cold start times](https://mikhail.io/serverless/coldstarts/aws/), it requires very carefully limiting the third-party dependencies that you use. Meanwhile, I've used expressjs. I like expressjs. I wanted to run my website on expressjs, using the very slick [Apex Up serverless framework](https://github.com/apex/up). Here's what a deploy looks like:

```
     build: 6,307 files, 24 MB (1.868s)
     deploy: production (commit b957007) (59.374s)    
     endpoint: https://xxx.execute-api.us-east-1.amazonaws.com/production/
```

Yup. **6,307** files, adding up to **24 MB**. A cold start for this Lambda reliably ran into the 7-8 second range, no matter how much RAM ([and thus how much CPU](https://epsagon.com/blog/how-to-make-aws-lambda-faster-memory-performance/)) I allocated to the Lambda.

Think about the last time you waited 8 seconds for a web page to load. I don't mean to finish loading, I mean 8 seconds before there was any proof of life. You can't, right? Because you'd never stick around that long.

But whatever, maybe it wouldn't be that bad. I wrote the website, published it, and spent a few days using it to see if the cold starts were even noticeable.

It was atrocious.

I ended up throwing a bag of tricks at it to make it tolerable: prerendering things and serving them via CloudFront origin groups, aggressive caching, and prewarming. That's a whole other blog post. I think I spent more time mitigating cold starts than working on the site itself. **D**

_(That site was Sketchviz, which makes it super easy to create [Graphviz diagrams with a hand-drawn feel](https://sketchviz.com/new). Let me know how I did — does it feel fast?)_

# Tools

Starting from scratch, these are the tools that I found really useful while developing serverless applications on AWS:

- [AWS CloudFormation](https://aws.amazon.com/cloudformation/), to do all the fiddly resource creation and wiring up (When possible, I prefer to use a vendor's own tools, so I avoided [Terraform](https://www.terraform.io/), but I've heard good things about that, too.)
- [Apex Up](https://github.com/apex/up), for running an expressjs app with zero friction (I also tried [claudia.js](https://claudiajs.com/), but it just wasn't as smooth. The [Serverless framework](https://serverless.com/) sounds promising, but I'd rather let it get a few more years of battle scars before committing to it.)
- [s3patch](https://s3patch.com/), for faster upload-test iterations
- [saw](https://github.com/TylerBrock/saw), for monitoring CloudWatch Logs in real-time (Sure, at 9 MB, it's 150 times the size of the venerable tail command, but someone's got to keep the hard drive manufacturers in business!)

No doubt a veteran practitioner could quintuple this list of tools, but my read on the serverless space is that the tooling ecosystem is still churning a lot. I'd hesitate to invest significantly in a serverless-specific toolchain until we have more time to see who the winners and losers are.

# Conclusion

Overall, I've been satisfied with the hands-off experience serverless has enabled for these three projects. It's still early days for me personally, and I don't feel like I've yet used serverless “in anger.”

Even so, regardless of the stability of the tooling ecosystem, serverless-the-platform is definitely ready for prime time. Start thinking about what workloads you have that would be good fits for moving to Lambda...but maybe keep websites running on VMs and containers for now.

What about you? What workloads are you running serverlessly, and how has it gone? Will 2019 be the last year you run bare-metal VMs?
