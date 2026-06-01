+++
title = "Remote Control For Radio"
date = 2015-10-01T19:55:59
draft = false
categories = ["Computers", "Ham Radio"]
slug = "remote-control-for-radio"
aliases = ["/2015/10/remote-control-for-radio/"]
+++

Seen a few articles about people controlling their radios remotely lately and thought what most others were doing was a bit too complicated. It could be done a lot easier, at least in my opinion of what’s easier.

First, rig control. Linux has the hamlib libraries that can control many different radios. I had built a simple CI-V serial interface for my ICOM IC-706 a while back. It’s not perfect, but can be made to work at 300 baud. The rigctl command line program is pretty simple to use. The manual page for it has all of the commands listed. Many don’t apply for my particular radio, but I can switch frequency, mode, memory channel, and a few other key operating parameters. I can read all of those parameters as well to make sure I’m where I think I am.

Second, the sound card interface. I got a Signalink USB a few months ago with the appropriate cable to hook to the back of the IC-706. Experimented around a bit with the levels and VOX control settings to come up with something that seemed to work well for me. This also keys up the transmitter so it’s nearly plug and play.

Third, I need a way to get my voice to the radio and get the audio back to my remote location. I see many people using Skype for this, which seems to be overkill to me. I thought there must be some easy point-to-point VoIP program I could use on my home network and potentially route outside the network if I wanted. I found a program called Linphone in the Debian repositories and it was really easy to set up. No need to set up an account or anything like that and it has a smart phone client as well. Installed it on my radio computer and on my laptop, set up the radio computer to automatically answer and use the Signalink USB for audio, then call it from the laptop. That’s all the setup that was needed. Some quick testing and I was able to get two way communication going.

Final test was checking into a net. My local simplex net tonight was run by an operator that I usually have trouble hearing and I had my squelch set too high, so he was cutting in and out pretty bad. That’s one of the settings that aren’t available through the CI-V interface on the IC-706, so I quickly ran into the shack and turned that down a little. I was able to hear him better after that, but I had already missed my turn by then. I switched over to the repeater net a few minutes later and confirmed my check in over there, no problems at all.

Not bad for a couple of hours of tinkering. Most articles I read include installing Ham Radio Deluxe, two Skype accounts, and remote desktop control software. Compare with rigctl, Linphone with no accounts needed, and simple SSH. I think it’s easier anyway.
