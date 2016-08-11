---
layout:     post
title:      Meccano Rubik's Shrine
summary:    The story of Rubik's Cube solvers continues!
categories: hacking
published:  false
comments:   true
---

<iframe src="https://www.youtube.com/embed/C9rCBjLGxJs" width="100%" height="360px" frameborder="0" allowfullscreen mozallowfullscreen webkitallowfullscreen></iframe>

# References

* <span class="red">**Order a replica, or a DIY kit**</span> at [Meccano Kinematics](https://meccanokinematics.com) (in development)
* Solving algorithm implementation: [GitHub](https://github.com/muodov/kociemba)
* Other Wilbert's machines: [YouTube channel](https://www.youtube.com/user/Meccanokinematics)

# Introduction

Although our [FAC Solver](/hacking/2015/08/18/fac-rubik-solver/) has gathered 120K+ views
on YouTube, we felt that this was only the beginning. By the time we finished it,
we had a bunch of ideas in mind that we were eager to realize. While I was trying to
figure out more efficient Arduino-Raspberry setup, Wilbert's
tireless genius came up with another 3-gripper machine, this time assembled with
old-school (yet well-known) [Meccano](http://www.meccano.com/) construction set.

While FAC Solver was more of an experimental creation, this time we had lots
of experience, so we could address fundamental weaknesses of the FAC Solver,
at the same time focusing on the look and feel of the machine, making it more
friendly and appealing. We didn't want to make yet another hacky DIY machine,
but something that would combine technology with design, something nice to
build, to see, and to own.

# Mechanical Design: Less Wiring, More Art

Having several decades of Meccano experience, Wilbert managed to design the machine
that is much more compact than the FAC solver. To achieve the cleanest look possible,
all the wiring and unnecessary details are hidden inside the metal body.
If necessary, the PCB board and other electronics are
easily accessible through the door on the back side. Moreover, the whole machine is
assembled using normal Meccano parts that are available on the internet.

![Some overall photos](img)

Wilbert was inspired by the Dark Tower from the Lord of The Rings,
so the look of the Shrine is meant to be quite pompous :)

![LOTR tower vs Rubik's Shrine](img)

Being very careful about the details, Wilbert made sure all parts matched
each other: all original rounded shiny Meccano strips, with 3-layered custom airbrush
finish on the plates. Brass gears put the final touch to the appearance of the machine.
The fine-grained assembly was also very important: the distance between some
parts was just a couple of millimeters.

![Some closeup photos](img)

Wilbert tells more about the mechanical part on [his website](http://wiswin.nl/Meccano Rubik Cube Solver.htm).

# Internals: What can we do better?

We were very satisfied with our previous project, the FAC Solver, but we also
knew we could do better. We learned a lot about hardware parts like stepper
motors and drivers, microcontroller boards, and faced all kinds of unexpected
issues on the way. This experience helped us a lot when we were building this Meccano
Rubik's Shrine.

## Microcontrollers: Use the right tool for the job

In our previous project, the [FAC Solver](/hacking/2015/08/18/fac-rubik-solver/),
(almost) everything was controlled by Raspberry Pi, including solving algorithm, color
recognition, movement scheduling, and pulse generation for the stepper motors.
We used Arduino only for reading the analog signal from LDRs in scanner device.

While it did work quite well eventually, it turned out to be tricky when it comes
to controlling the motors. On the lower level, you can't precisely control the time that
processor gives to your process on RPi. Due to the presence of other processes,
and, actually, the operating system itself, your program might be given different
chunks of CPU time, which makes the generated pulse train unstable. For lower speeds,
it is not a problem, but if the motor speed is high enough, it can be killing for
performance: motors start hesitating.

![Arduino and Teensy closeup photos](img)

To overcome this problem, we decided to use an Arduino board for the actual motor
control. We chose [Arduino Pro Mini](https://www.arduino.cc/en/Main/ArduinoBoardProMini)
because it is probably the most compact one. We made the first working prototype
with it, and later replaced it with the [Teensy 3.2](https://www.pjrc.com/teensy/),
which looks and feels almost identical, but has better performance.

Arduino gave us much more stable pulse train, which allowed to reach higher
speeds. We also used the great [AccelStepper](http://www.airspayce.com/mikem/arduino/AccelStepper/)
library that made movements smoother and more reliable. As before, I used the
[MIN protocol](https://github.com/interactive-matter/MinProtocol) for RPi-Arduino
communication.

## Firmware: The sky is the limit

Since software part was my main responsibility in this project, the firmware was
the very first thing I cared about. Pure RPi/Python implementation of FAC Solver
proved to have a number of nasty caveats, so I wanted to do it right this time.
I braced myself and prepared for the challenge of embedded systems programming,
ready to implement things from scratch for saving a few bytes of RAM.

![Arduino compile result](img)

Probably the most difficult part was the movement optimization. The solving algorithm
produces a solution sequence on Raspberry Pi, which then needs to be translated into
a sequence of gripper movements. Then this data has to be transferred to the Arduino
board because actual electrical pulses are generated there. I wanted to make
the algorithm generic so that I could use it in other machine configurations
(with 4 or even 6 grippers). This added to the complexity of the problem.

Minimizing the overall solving time is not obvious. For example,
in a 3-gripper setup like this, to rotate the Front side of the cube we need to
rotate the whole cube first. We can rotate it around the vertical or horizontal axis.
Whichever is better (leads to faster solution) depends on the current state of
the grippers and on subsequent moves.

![cube movement illustration](img)

Generally, all grippers can move simultaneously. To do this, we need to have
multiple parallel execution flows. Since Arduino is single-threaded,
it has to be done with [coroutine](https://en.wikipedia.org/wiki/Coroutine)-like mechanism.
On the other hand, due to the mechanical construction, some movements require
the other grippers to be in specific positions, which means that parallel workers
need to be able to block each other. Average solving process requires about 200
blocking operations.

[Arduino Pro Mini](https://www.arduino.cc/en/Main/ArduinoBoardProMini),
which we used initially, has only 2KB of RAM. After initializing
objects for serial communication and motor control, there are approximately 200 bytes
(!) left for local variables in actual business logic. With such limited
resources, I could afford to store only absolutely vital data. Everything else had
to be buffered in Raspberry Pi and pushed to Arduino via serial connection
during the run. Pure serial communication is quite expensive, so it was causing
an additional slowdown of the solving process.

It is possible to write C++ code for Arduino, but it is very limited,
and in most cases it is a bad idea to use things like dynamic allocation. For
spoiled developers like me, it is really tough to refrain from using abstract classes
with a bunch of virtual methods and dynamically allocated arrays. It is really hard
(I mean _freaking hard_) to debug because you never see the exceptions: it never
fails explicitly, it just starts working weird at some point, and you can only
guess what went wrong, especially if you have memory-related problems. I ended up
with quite hacky virtual method implementation based on switch statements and
static variables, which was lame, but straightforward.

Later, we changed the Arduino Pro Mini with [Teensy 3.2](https://www.pjrc.com/store/teensy32.html)
which was a great improvement. Not only it is _way faster_ (thanks to ARM processor),
and has _**64KB**_ of RAM, but it has exactly the same form-factor as Arduino Pro Mini.
We didn't even need to change the PCB connectors!

![Arduino Pro Mini vs Teensy 3.2](img)

Well, 64K ought to be enough for anybody, at least in Arduino world :) With all this
memory available, I could easily store information about the whole movement sequence at once.
No additional serial communication + increased speed = faster and smoother solving.

## Camera: Come on, it's the 21st century!

Apart from the motor controlling, the biggest technical change in Rubik's Shrine,
compared to the FAC Solver, was the scanning part. While it was fun to play with
the low-end scanner made from LDRs and LEDs, it was
[quite](/hacking/2015/08/18/fac-rubik-solver/#capacitor-based-scanner) a
[challenge](/hacking/2015/08/18/fac-rubik-solver/#arduino-controlled-scanner)
to deal with ambient light and do the color balancing. It did fit well the brutal
nature of the machine but obviously was not the most modern approach.

This time, we went fancy and used the Raspberry Pi Camera for color scanning the cube.
Since it automatically adapts to the lighting conditions, it solved pretty much
all of those color recognition problems. All I needed to do was to carefully pick
the average colors from specific regions on the photos.

![scan photo](img)

As a bonus, each run is recorded on video and available on the Archive page in the
touch interface. It was also quite handy during debugging.

![video player photo](img)

## Pi Display: the Final Touch

I just couldn't resist. Not so long before we started this project, Pi Foundation
[announced](https://www.raspberrypi.org/blog/the-eagerly-awaited-raspberry-pi-display/)
an awesome 7-inch touch screen for Raspberry Pi that perfectly fitted our model.
I was pleasantly surprised that it just worked: after a couple of nights I had
a nice-looking touch UI written in [Kivy](https://kivy.org/).

![Touch UI (Patterns screen)](img)

Having a touch screen, I was able to add some extra functionality to the machine.
In addition to a nice presentation of the scanning and solving results, it is
possible to make some pretty [patterns](https://ruwix.com/the-rubiks-cube/rubiks-cube-patterns-algorithms/).
I also made a special debug screen for manual control that practically made it
unnecessary to use a laptop for testing.

![Touch UI (Debug screen)](img)

# Public demonstrations and final words

When the Rubik's Shrine was almost finished, we presented it on [SkegEx 2016](http://www.skegex.nmmg.org.uk/),
the annual Meccano exhibition in Skegness, England. We even managed to win the
5th prize, which is quite something considering that SkegEx is generally more
about traditional mechanics rather than robotics and electronics.

![SkegEx photo](img)

But the biggest prize for us were the gleaming eyes of kids visiting the
exposition. It was wonderful to see them pulling their parents to our table
over and over, just to scramble the cube and start the machine once again.

All in all, we are very satisfied with the result. Apart from all the hands-on experience,
for me, as a software developer, it was very fun to work on something physical,
something that I could see and touch. And I like to think that, thanks to Wilbert,
we could push engineering a little bit towards art. Because this is what it is all about.
