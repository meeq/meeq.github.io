---
layout: post
title: Cloning Flappy Bird for the Nintendo 64 with libdragon (Part 1)
category: Software
tags:
  - Programming
  - Retrogaming
  - Necromancy
---
Yesterday I finally got around to packaging up [a ROM file](https://github.com/meeq/FlappyBird-N64/blob/master/FlappyBird.z64?raw=true) and [releasing the source code to my "port" of Flappy Bird for the Nintendo 64](https://github.com/meeq/FlappyBird-N64), so I wanted to spend a little time to do a post-mortem on its development and look back on the bizarre phenomenon of Flappy Bird in general.

![Flappy Bird for Nintendo 64 Gameplay Video]({{ site.url }}/assets/flappy-n64-gameplay.gif)

If you came here just for the ROM, you may be somewhat disappointed to find that it does not run in most N64 emulators. If you are interested as to why, the short answer is that most emulators don't attempt to accurately represent the Nintendo 64 hardware. The best way to play this ROM is on a real N64 using a flash cart such as [64drive by retroactive](http://64drive.retroactive.be/) or [EverDrive-64 by krikzz](http://krikzz.com/).

This first part will be an introduction into Flappy Bird and Nintendo 64 development to set the stage for a technical deep-dive of the source code.

## Some context on Flappy Bird

![Flappy Bird for iOS Title Screen]({{ site.url }}/assets/flappy-ios-title.png)

On May 24 2013 a game called [Flappy Bird](http://www.dotgears.com/apps/app_flappy.html) was released for mobile phones by Dong Nguyen and quickly took the world by storm. While the game and its graphics were quite simple, its hook was remarkably effective: fly as far as you can, which is not quite as easy as it looks.

![Flappy Bird for iOS Gameplay Screens]({{ site.url }}/assets/flappy-ios-gameplay.png)

Objectively, you play as a bird who is constantly plummeting to the ground and must be kept aloft with continuous flapping to gain height. As you fly forward, you must navigate through narrow gaps between obstructions from above and below until you crash. The game is scored based on how many obstructions you clear, allowing you to compete for a high score and acquire medals for outstanding performance.

![Flappy Bird for iOS Game Over Screen]({{ site.url }}/assets/flappy-ios-gameover.png)

Due to its simplicity, a lot of people (myself included) looked at Flappy Bird and thought "oh, I could do that". While the game lacks depth, it has a certain charm to it. Clearly it struck a chord with the zeitgeist of the internet, as [innumerable](https://flappybird.io/) [clones](https://flappybird.me/) and ["rip-offs"](https://www.digitaltrends.com/mobile/flappy-bird-ripoffs-now-charting/) of the game have been spawned since its creator [removed the original from the App Store on February 10 2014](http://www.telegraph.co.uk/technology/10626852/Flappy-Bird-to-be-taken-down-after-ruining-creators-life.html). The "Flappy" game concept is even being [used by Code.org as a tutorial](https://studio.code.org/flappy/1) aimed at teaching kids to learn programming.

![Flappy Bird Code.org Tutorial Screenshot]({{ site.url }}/assets/flappy-code.png)

## Convergence with retrogaming

As a gaming enthusiast born in the 1980's, I was most-delighted to see that there is a burgeoning scene of homebrew developers who have released versions of Flappy Bird for a wide variety of obsolete game consoles:

  * [Atari 2600 (Flappo Bird)](https://tacsgames.com/2014/02/08/flappo-bird-out-now-for-atari-2600/)
  * [Atari 2600 (Flappy)](https://atariage.com/store/index.php?l=product_detail&p=1038)
  * [Commodore 64](http://sos.gd/flappy64/)
  * [Dreamcast VMU](http://retrogamingmagazine.com/2016/03/28/flappy-bird-vmu-released-for-sega-dreamcast-visual-memory-unit/)
  * [Game Boy](https://www.youtube.com/watch?v=L1arYXOawP8)
  * [Game Boy Advance](https://www.youtube.com/watch?v=SNm9kNV9rnw)
  * [Intellivision (Flapee Bird)](http://atariage.com/forums/topic/234851-collectorvision-official-thread-for-flapee-bird/)
  * [Nintendo Entertainment System](http://www.retrogamenetwork.com/2014/07/13/new-port-of-discontinued-mobile-sensation-flappy-bird-hits-nintendo-entertainment-system/)
  * [SEGA Master System (Flip Flap)](http://pdroms.de/mastersystem/flip-flap-v2-10-master-system-game)
  * [SuperGrafx](https://www.youtube.com/watch?v=2ZhZ7e33zrg)
  * [Super Nintendo (Super Mario World Code Injection)](https://www.youtube.com/watch?v=hB6eY73sLV0)
  * [Text Instruments TI-99/4A](https://www.youtube.com/watch?v=XPXwxc-_Wp4)
  * [Vectrex (Veccy Bird)](http://vectorgaming.proboards.com/thread/890/veccy-bird?page=1)
  * [Virtual Boy (Flappy Cheep Cheep)](http://www.planetvb.com/modules/newbb/viewtopic.php?topic_id=5527)
  * [ZX Spectrum](http://retrogamingmagazine.com/2015/01/24/type-program-flappy-bird-zx-spectrum-available-old-school-method/)

Right around the time of Flappy Bird's rise to fame, I was going through an obsessive phase with my Nintendo 64 and started learning as much as I could about the technical aspects of the system. When it came time to apply my research and create a dead-simple yet compelling demo, it only seemed natural that the N64 should be blessed by its own version of the smash arcade hit game of the year.

## Software development kits

Out there on the internets you can find some amazing stuff. You can even find the official Nintendo Software Development Kit for the N64, which is proprietary, confidential, and disclosed only to official licensed developers. This is super neat, but Nintendo is renowned for aggressively enforcing their copyrights, so I ruled out using the official SDK that nearly all games officially published for the platform used. The fear of litigation has not stopped many other intrepid homebrew developers, so [there is a guide available for getting started down that route](https://n64squid.com/homebrew/n64-sdk/) (at least until the cease & desist letter shows up).

![Official Nintendo 64 Development Kit Documentation]({{ site.url }}/assets/flappy-n64sdk.png)

Since the N64 was developed and released during the mid-1990's, much of the official tooling is still stuck in the past, having been designed around [SGI workstations](https://en.wikipedia.org/wiki/SGI_Indy) and Windows 95. I prefer not to use Windows whenever humanly possible, and trying to [bring an SGI Indy back from the dead](http://www.sgistuff.net/hardware/systems/indy.html) wasn't something I had signed up for. What I was looking for was a modern toolchain that I could run on Linux and OS X. Luckily, such beasts also exist out there on the web.

Way back in 2005 a dedicated soul named [Ryan Underwood (Halley's Comet Software)](http://hcs64.com/n64info.html) took his vast knowledge of reverse-engineering the N64 and released the [Open Source Nintendo Ultra64](https://sourceforge.net/p/n64dev/code/HEAD/tree/trunk/n64dev/), a collection of tools and documentation for building N64 software without the secret Nintendo magic. This project appears to have been abandoned, but it was a fasinating resource and appears to have jump-started the N64 development scene. In 2009 [Shaun Taylor (dragonminded)](https://dragonminded.com/n64dev/) started the [libdragon project](https://github.com/DragonMinded/libdragon), a library for interacting with the Nintendo 64 hardware on top of the [GCC compiler suite](https://gcc.gnu.org/) and [newlib embedded C library](https://sourceware.org/newlib/).

Ultimately, libdragon seemed up to the task, despite its work-in-progress status and limited feature-set compared to the commercial SDK. [Controller input](https://dragonminded.com/n64dev/libdragon/doxygen/group__controller.html) worked, [2D graphics](https://dragonminded.com/n64dev/libdragon/doxygen/group__graphics.html) rendered rectangles and sprites using both software and hardware-acceleration, [audio output](https://dragonminded.com/n64dev/libdragon/doxygen/group__audio.html) appeared to be possible with raw waveforms, and all of it seemed [reasonably well-documented](https://dragonminded.com/n64dev/libdragon/doxygen/modules.html). I appreciated being able to enjoy the benefits of a modern compiler and only had minor hiccups getting everything working on my Macbook Pro mostly related to building GCC.

![libdragon spritetest example]({{ site.url }}/assets/flappy-libdragon-spritetest.gif)

Not using the commercial N64 SDK simultaneously liberated me from the tyranny of closed-source proprietary software, but also restricted the project with the severe caveat that most mature and popular N64 emulators won't be able to run the game.

## The mixed blessing of emulation accuracy

A physical Nintendo 64 performs general computation on the CPU, which is a pretty standard MIPS R4300i processor, and must be emulated in order to do much of anything. However, the CPU must go through the [~~MCP (Master Control Program)~~](http://tron.wikia.com/wiki/MCP) RCP (Reality Co-Processor) in order to interact with the rest of the system. The CPU can off-load vector processing to the RSP (Reality Signal Processor) for high performance calculations performed by "microcode" instructions. Both the CPU and the RSP can generate commands for the RDP (Reality Display Processor) in the form of "display lists" which control its operating state to render shaded, textured, and depth-buffered geometry. The RCP is also responsible for communicating with the VI (Video Interface), AI (Audio Interface), PI (Peripheral Interface), SI (Serial Interface), and RDRAM (Rambus Dynamic Random Access Memory).

![Nintendo 64 Motherboard]({{ site.url }}/assets/flappy-n64-board.png)

All of this means that in order to accurately emulate an N64, one must completely reverse-engineer the RCP, which is a massive and complex undertaking. There are only two emulators currently on the scene that even attempt this effort: [CEN64 aims for *perfect* emulation down to the register-transfer level.](https://cen64.com/) [MAME also intends for their Nintendo 64 driver to have low-level emulation accuracy](http://mamedev.org/). At the moment, neither of these emulators are as mature as the [high-level emulation offerings](http://emulation.gametechwiki.com/index.php/Nintendo_64_emulators), many of which sport extremely high compatibility with commercial N64 titles at full speed.

![UltraHLE Website from 1999]({{ site.url }}/assets/flappy-ultrahle.png)

You may recall [UltraHLE, the first successful N64 emulator](http://www.emuunlim.com/UltraHLE/old/main.htm), which was able to properly play a handful of real commercial games like Super Mario 64 and The Legend of Zelda: Ocarina of Time all the way back in 1999. UltraHLE took a different approach than other emulators of its day -- rather than attempt to accurately mimic the hardware of the console, [High-Level Emulation intercepts the C library calls](http://emulation.gametechwiki.com/index.php/High/Low_level_emulation) to communicate with the hardware and emulates the library function without executing the low-level code that would run on real hardware. [Even emulating hardware a generation older than the Nintendo 64 requires a significant amount of processing power to do accurately.](https://arstechnica.com/gaming/2011/08/accuracy-takes-power-one-mans-3ghz-quest-to-build-a-perfect-snes-emulator/)

Since libdragon's library does not even remotely resemble the official Nintendo SDK, emulators are not able to perform this high-level emulation with open-source homebrew and must accurately emulate the hardware in order to display graphics, play sound, and receive controller input correctly. This puts a small damper on the playability of anything made with libdragon for others, but as an N64 enthusiast armed with an EverDrive-64 I consider the real hardware to be the only target that matters.

## Getting started making stuff

Now that I had my toolchain and what to make, it was time to ~~carefully plan out the project~~ jump right in with coding and figure things out as I go. In part 2 I will break down the graphics, audio, and gameplay code and dig into the technical details of building a game with libdragon.

Thanks for reading!
