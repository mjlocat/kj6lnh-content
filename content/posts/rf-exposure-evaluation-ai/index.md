---
title: "Taming the Beast: Using AI to Solve the New FCC RF Exposure Evaluations"
date: 2026-06-17T20:00:00-07:00
categories: ["Ham Radio"]
tags: ["FCC", "Regulations", "Tech", "DIY", "HF", "VHF"]
draft: false
---

If you’ve been keeping up with your licensing requirements lately, you already know that the FCC’s grandfathering rules for RF exposure evaluations are a thing of the past. Every amateur station in the US—no matter if you are running QRP in the backyard or a full legal-limit shack—is now required to perform and maintain an RF exposure evaluation.

When I first sat down to audit my three active home transmitters, I dreaded the process. If you’ve ever tried to do these calculations manually or even plugged numbers into a standard web calculator, you know exactly what a pain it can be. 

## The Old Way: Math, Loss Charts, and Frustration

Evaluating a station the traditional way feels less like a hobby and more like a tedious engineering homework assignment. You have to hunt down formulas for Maximum Permissible Exposure (MPE) power density, look up the specific frequency blocks to determine your environment’s threshold, and then figure out your True Average Power.

To get that average power, you have to find the attenuation specs for your exact feedline, calculate the precise loss across your exact cable length, factor in the mode duty cycle (like FT8’s continuous blocks vs. the rapid bursts of AX.25 packet), and then guess your own operating duty cycle over a strict 30-minute regulatory window. 

One mistake with a decimal point or a dB-to-linear conversion ratio, and your safe boundary calculations are completely skewed. Even the dedicated web calculators usually require you to know all of your end-state variables upfront, leaving you to guess at things like antenna gain values for heavily modified, shortened homebrew verticals.

## Enter AI: Chatting Your Way to Compliance

Instead of fighting with spreadsheets and formulas, I decided to treat the problem like a technical consultation and ran the entire evaluation using an AI assistant. The difference was night and day. 

Instead of demanding a perfect string of variables, the AI simply asked me a few conversational questions about my station:
* *What gear are you running?*
* *What does the physical property line look like?*
* *How often are you actually transmitting?*

I just described my everyday operating realities. For my main HF station, I explained that I’m running an Icom IC-718 into a 20M Hamstick right outside the garage, with an insulated counterpoise running horizontally to a side fence line where a steep hill allows physical access. I noted that I usually "hunt and pounce" on FT8 at 90W, but occasionally do a CQ run. 

From just that description, the AI did the heavy lifting:
1. It calculated the exact 84% inherent duty cycle of an FT8 transmission window.
2. It factored in a conservative 50% worst-case operating duty cycle for a heavy CQ run.
3. It calculated the exact feedline loss for my 25 feet of RG-8X at 14 MHz.
4. It spit out the final continuous time-averaged EIRP (**34.8 Watts**) and mapped out my exact safety boundaries (**1.6 feet** for a controlled environment, **3.5 feet** for the public).

## Evaluating the Rest of the Shack

Once the first baseline was established, replicating it for the rest of my gear was a breeze. We knocked out a second scenario for a 40M Hamstick restricted to 40W due to ALC limits, which pulled the public safety buffer back to a comfortable 2.8 feet.

Next, we tackled my VHF setups:
* **The VHF Packet Station:** An IC-706 pushing 30W into a roof-mounted discone via 75 feet of LMR-400, used mostly for overnight Winlink cron jobs. The math showed that because packet radio transmits in rapid, sparse bursts, the true time-averaged power drops to a minuscule **6.5 Watts EIRP**. With the antenna 15 feet up on a single-story roof apex, vertical separation makes it completely, automatically compliant.
* **The APRS Weather Node:** A Yaesu FT-5100 running 25W into an N9TAX rollup J-pole hanging right on the fence. Even accounting for a highly conservative 3 dB loss from an old, spliced piece of RG-8X, a 0.3-second weather burst sent once every 5 minutes means an operating duty cycle of just **0.33%**. The resulting average power is only **67 milliwatts**, shrinking the required safety boundary to less than 2 inches.

## The Takeaway

By handing the calculations over to an AI, I didn't just get raw numbers; I got a practical, real-world translation of how those numbers interact with my physical property line. It flagged the areas where I needed a basic mitigation plan (like ensuring the fence line is clear or utilizing insulated wire to prevent RF burn risks) and validated the spots that were already perfectly safe.

If you’ve been putting off your station evaluation because you don't want to dig through old ARRL tables or spend an evening punching numbers into a scientific calculator, give an AI a try. Describe your station like you're talking to a fellow ham, and let technology do the math for you. 

My complete evaluation is now neatly formatted and ready to go if ever asked. Compliance checked, hassle avoided.
