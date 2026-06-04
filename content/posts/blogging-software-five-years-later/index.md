+++
title = "The Blogging Software Project, Five Years Later"
date = 2026-06-03T20:30:00
draft = false
categories = ["Computers"]
slug = "blogging-software-five-years-later"
+++

Back in [January 2021](/new-project-blogging-software/) I wrote about a new project: I was going to ditch WordPress and build my own "serverless" blogging software. WordPress was running on an EC2 instance, burning CPU cycles around the clock to serve a blog that, by my own admission, mostly gets visited by bots. I figured I could do better with Lambda and friends, only pay for what I actually used, and have a fun project to keep my skills current along the way.

Then, true to form, I disappeared for five years.

In my defense, I did eventually come back around to it. But the first thing I had to admit was that the 2021 plan was over-engineered. I was about to go write a whole application — functions, an API, a database, the works — to serve what is, when you get right down to it, a stack of HTML files that changes a couple of times a decade. There are no comments here. There are no logins. There's nothing dynamic to speak of. Building custom software for that is like installing a freight elevator to move a single box of books.

The honest answer was a static site generator. My posts are just text and a few photos, so I write them in Markdown and a tool called [Hugo](https://gohugo.io/) renders them into plain HTML. Those files go into an S3 bucket, and CloudFront sits in front to handle HTTPS and caching. That *is* serverless — there's no server to patch, no database to back up, it scales to zero, and it costs me well under a dollar a month (most of which is the DNS zone I was already paying for). As a bonus, the thing that always nagged me about running WordPress — the steady drip of security updates and plugin vulnerabilities — simply goes away. There's nothing to exploit in a static file.

The whole thing is wired up so that publishing is now just `git push`. My content lives in a GitHub repo; pushing to it kicks off a pipeline that runs Hugo and copies the result to S3. All my old WordPress URLs redirect to the new ones, so nothing should break if you followed a link here from somewhere.

Here's the part that made it genuinely a "keep my skills current" project, just not in the way I expected back in 2021. I didn't really build this by hand. I used [Claude Code](https://claude.com/claude-code), an AI coding agent, and mostly just described what I wanted: static files on S3, CloudFront in front, content as Markdown in GitHub, and a pipeline to build and publish it. It poked around my old WordPress site, asked me a handful of genuinely good questions — which generator, how to manage the infrastructure, whether to migrate my old posts or start fresh — and then went and did the work. It pulled every old post out of WordPress and converted it to Markdown, wrote all of the infrastructure as code, stood it up in AWS, and handled the migration. It even caught a busted code snippet in my old kinetic watch charger post, where an `#include <Servo.h>` had quietly lost the `<Servo.h>` part years ago because it looked like an HTML tag. My job was mostly to review, approve, and click a single button to connect it to GitHub.

Full disclosure: it drafted this post too. I'm editing and second-guessing it before it goes live, which feels like the right division of labor.

It's a little surreal. The version of me writing that 2021 post wanted a project to keep up with where technology was heading. It turns out where it was heading was "describe the thing you want and review what an agent builds." The blogging software I was fretting about writing already existed, and most of what was left was plumbing that a machine was happy to handle.

If you're reading this, it means the new setup works. I won't promise the next post will come any sooner than this one did — but at least now there's no idle server sitting around waiting for it. 🙂
