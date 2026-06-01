+++
title = "Kinetic Watch Charger"
date = 2015-05-02T15:27:19
draft = false
categories = ["Electronics"]
slug = "kinetic-watch-charger"
aliases = ["/2015/05/kinetic-watch-charger/"]
+++

When Carrie and I got married, she gave me a very nice watch as a wedding present. It’s a kinetic watch (i.e. it is wound up by movement). I haven’t used it since the band broke a long time ago. Finally got the band fixed, but since it hasn’t been used, it’s not wound up at all right now. I can either wear a watch that’s one right twice per day for a while until it’s wound up, or I could put the <a href="http://learn.parallax.com/KickStart/900-00008" target="_blank" title="Parallax Continuous Rotation Servo">continuous rotation servo</a> I got a while ago to good use.

Just needed to wire up power, ground, and data lines between the servo and an Arduino and write the following sketch:

```cpp
#include <Servo.h>
Servo myServo;
int waittime = 1000; // Let each rotation happen for 1 second

void setup()
{
  myServo.attach(7);
}

void loop()
{
  myServo.writeMicroseconds(1700); // rotate counter-clockwise
  delay(waittime);
  myServo.writeMicroseconds(1300); // rotate clockwise
  delay(waittime);
}
```

Uploaded the sketch to the Arduino and the servo started spinning. Just needed to connect the watch to the servo (scotch tape is quick and dirty) and plug the whole thing into a USB battery. Seems to be working pretty well.
