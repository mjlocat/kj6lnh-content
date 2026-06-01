+++
title = "New Project – Blogging Software"
date = 2021-01-23T19:22:16
draft = false
categories = ["Computers"]
slug = "new-project-blogging-software"
aliases = ["/2021/01/new-project-blogging-software/"]
+++

I’ve been away from the blog for some time now. Was doing the side-gig startup thing for a while. That didn’t work out and ended up getting a little burned out for a while, so I was more focused on hiking since then. Now, I’m ready for another project. Don’t care to do anything that requires a business plan, this is more for fun, keep some skills current, and do something that’ll help me out a little.

At the time of this writing (January 2021), I’m hosting this blog using WordPress on an Amazon EC2 instance. WordPress is pretty popular software that requires some kind of persistent service running. I haven’t looked at statistics, but I doubt this blog gets any kind of traffic other than bots, given that I rarely post anything. That’s a lot of idle CPU time. While the smallest EC2 instances aren’t very expensive, I can probably do better.

I work with Amazon Web Services a lot at my day job. EC2 (virtual servers) is just one of hundreds of services they offer. Many of these services offer on-demand compute, storage, and data services. Used together, this is known as “serverless” architecture. For example, Lambda is a Functions as a Service (FaaS) offering that allows you to run a small piece of code. As soon as it’s done running, you’re not charged for it until the next time it runs. The service provider manages provisioning resources, starating it up, and releaseing the resources when done. This really brings down the cost of running a low traffic site like this one.

WordPress isn’t designed to work in that way though. I’ve seen a couple of blogs describing ways to coax WordPress to run this way, but it seems pretty brittle and unmaintainable to me. So, if not WordPress, what? I searched around for a while looking for something that was built like this, but came up with nothing. So sounds like it could be a fun project.

I’ll plan to post more once I get a little further along.
