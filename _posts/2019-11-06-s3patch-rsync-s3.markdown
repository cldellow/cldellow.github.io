---
title:  "s3patch"
author: cldellow
tags:
  - s3
  - s3patch
---

When Jenn and I were travelling in Europe, we often tinkered with some
side projects. These required uploading large files regularly. The files
were 99% unchanged from upload to upload. For example, a ZIP file
with a node.js app, or a JAR file with a Scala app, where the bulk was
made up of third-party dependencies that never changed.

Still, in our Edinburgh apartment with its 50KB/sec upload rate, this
meant a several minute delay every time we wanted to push a change.
Worse, the Internet was _unusable_ while the upload was happening.

I wondered if there was something like rsync for S3, but couldn't find
anything. So I made [s3patch](https://s3patch.com/). It uses xdelta3
to ship diffs to servers in AWS, which then reconstitute the final thing
and upload it to S3. This tool is probably most helpful to people
working on questionable wifi, so home-based folks and digital nomads
working out of cafes.
