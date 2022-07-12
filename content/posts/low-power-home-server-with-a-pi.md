---
layout: post
title: Low power home server without a RaspberryPi
date: 2022-05-16 00:00:00 +0000
categories:
 - Hardware
 - Home Automation
---
Let's face it; electricity is expensive; in the UK alone, the cost per kWh has doubled over the last year, and reflecting on what we're running in our homes 24/7 like home servers feels like a problem worthy of attention.

## Where we begin
As it stood in my home, Frigate (CCTV NVR), Home Assistant (Home Automation), Plex (Media Server) & NAS were all running from an HP ProLiant DL380p Gen8. When I acquired this server, it provided a rock-solid base to run everything from and be damned the power consumption; electricity was cheap. Spec'd up with 2x Xeon E5-2670 V2 10 Core CPUs, 64GB DDR3 Memory, 6x 4TB Seagate IronWolf hard drives & Dual 750W Gold PSU. The DL380p was and still is a joy to play with, but with a power consumption of 120w, it posed an exciting challenge; what's the lowest power consumption we can achieve?

Yes, I know 120W isn't a great deal, but over a day, that is 2.8kWh, and over a year, 1051.2kWh or £273 (@26p kWh), not an insubstantial saving could be achieved.  

## The challenge
  * Keep power consumption below 10% of the HP DL380p (Roughlty 12w) 
  * Run 6 CCTV cameras through [Frigate](https://frigate.video) with Object Detection
  * Run [Home Assistant](https://www.home-assistant.io) (and MQTT) for home automation
  * Run [Plex](https://plex.tv) which only has to cope with 1-2 streams for our house hold
  * Provide Network Attached Storage
  * Do all this during the 2022 silicon shortage

## Selecting a CPU
A convenient feature of PassMark's CPU Benchmarks is the [Mega List](https://www.cpubenchmark.net/CPU_mega_page.html), letting you filter on a range of criteria. Filtering TDP by 5w to 10W & sorting by CPU Mark gave me a hit list of CPUs to hunt for.

In this case, most of these CPUs were considered laptop/embedded or low-end desktop devices. Which means they were most likely to be soldered directly to a motherboard. Running Frigate object detection on these lower-spec chips without something like a [Coral AI](https://coral.ai) accelerator was going to tax the CPU and push power consumption well above the 12w limit.

At the time of writing, the USB version of the Coral AI was almost impossible to find, but the less popular M.2 A+E Key versions were reasonably plentiful. So I needed a board with an M.2 A+E slot, which isn't too difficult as this type of slot tends to be used for WiFi cards. The other goods news, these accelerators consume only around 2W.

![Coral AI M.2 A+E](/img/coral-ai-m2-ae.jpg)

The hunt began... scrolling to the bottom of the CPU benchmark listing gave me some attractive options, but many were just not available. ASRock makes a great range of ITX boards like the [ASRock J5040-ITX](https://www.asrock.com/mb/Intel/J5040-ITX/index.asp), but they were just impossible to find. So working further down the list and googling for prebuilt machines, I found these...

* Lenovo IdeaCentre 310S (Intel Pentium Silver J5005 & 4GB DDR4)
* HP Slim S01 (Intel Pentium Silver J5005 & 4GB DDR4)

The Lenovo was in plenty of supply on eBay for less than £150, and it had the required M.2 A+E slot as well as one PCIe 16x and one 1X sized expansion slot. With the onboard GPU able to give some support for Frigate and Plex transcoding, the only downside is it lacked enough SATA ports, and its built-in power supply limited the number of drives it could support to just 2. Though luckily, the PCIe slot and some helpful parts from the crypto mining community would solve that for me. We'll come back to that later.

![Lenovo 310s](/img/lenovo-310s.jpg)

## First build

![Low power first build](/img/low-power-first-build.jpg)

In this configuration, there is a Crucial 120GB SSD acting as the boot device, the 1TB HDD that came with the Lenovo working as the storage drive for CCTV footage, and the Coral AI accelerator installed (There is also an M.2 NVME disk on a PCIe riser, but that was removed for testing power consumption)

Measuring the power consumption at the wall, with everything running (but plex not actively streaming), I'd failed already at 12.9W of consumption. Though this was easy to explain, that spinning platter of rust of a hard drive is spinning all the time, storing streamed CCTV footage from 6 cameras. These drives will consume anywhere from 4-6W to do this, which blew the budget pretty quickly.

## Attempt two

![Low power attempt two](/img/low-power-attempt-two.jpg)

Switching to an SSD for CCTV storage should reduce the consumption by 2-4w. So on attempt two, and now all neatly mounted in a case (again with NVMe and external fans disconnected), I achieved my goal of 9W consumption at the wall! With 3W left in the budget, I had yet to install the 4 HDDs in the machine, power them and somehow connect them with no SATA ports left. 

## What's going on with the NMVe disk

The thought at the time was to sacrifice NAS storage space in favour of lower power consumption. The spare NVME disk here would have added fractions of a Watt in standby, keeping this whole build still well under 10w. In this case, though, the cost of acquiring an NVMe SSD big enough to replace the 16TB disk array from the HP DL380p was just going to be too expensive.

So back on the hunt... I now needed to power four disks, and I needed a PCIe to 4x SATA card, which wasn't going to add too much in the way of power consumption.

![Dell H310](/img/dell-h310.jpg)

In my possession, I had a Dell PERC H310 patched to work in IT mode, perfect for connectivity but with a nominal power consumption of [6.4W](https://www.servethehome.com/lsi-host-bus-adapter-hba-power-consumption-comparison/) I'd have blown the budget again before even reconnecting the drives. At this point, I need to reference [Matt Gadient](https://mattgadient.com/8-port-sata-on-a-pcie-1x-lane-looking-at-the-pce8sat-m01-expansion-card/) who got there well before me on this whole low power subject. Taking notes from his posts, I ordered a four-port Marvel 9215-based card, hoping for a similar 2W increase in power once installed, taking me to 11w with the added 0.5W per hard disk that brings me to precisely 12W.

Lastly with I needed to work out how to power these four drives. The board is powered by a 65W power supply; if all four drives were spinning up, that could consume up to 48W, leaving very little headroom, so I need a better solution.

## Attempt three

![Low power attempt three](/img/low-power-attempt-three.jpg)

Adding in the 4x 4TB levels Seagate IronWolf drives and another power supply poses another small problem. We need the second power supply to turn on when the motherboard turns on. This turns out to be a simple fix with an Add2PSU board, and these seem to be used quite frequently by crypto miners for using multiple power supplies when running large numbers of graphics cards.   

![Add2Psu](/img/add2psu.jpg)

After some configuration for drive power management, this ended up giving a consumption of 20W. I tested the SATA controller on its own without the second power supply finding that it only added around 1W. This means that each drive was consuming around 2.5W when spun down, which seemed relatively high considering the drives themselves (in theory) should be consuming <1W. Assuming the worse case of 1W per drive, that's around 6W that can only be coming from the addition of a second power supply.

## Accepting defeat for now

Others that have attempted this build have been able to 
optimse to near-total idle usage, because this machine is processing 6 video streams, doing object detection and persisting streams to disk, it averages around 15% constant CPU utilisation. Along with the overhead of multiple power supplies, I ran out of further avenues to reduce power consumption right now without spending a lot more money. These are not excuses, just acceptance that, for now, I am defeated, but I might revisit this.

## What could be done differently

Reusing a motherboard like this was a great way around the availability problem of other options. But the ideal would be finding a board with 6-8 SATA ports, M.2 A+E ports, and most importantly, powered via standard ATX, removing the overhead of two power supplies. A few do exist, and there are some tempting AliExpress options that could be fun to play with. 

I also did a brief prototype powering the motherboard by stepping up the 12v rail of an ATX power supply, meaning I could power the motherboard and drives from a single power supply. This worked but would never be something I'd put into production using cheap off-the-shelf boost converters. A more exciting option could be to replace the 65W external power supply used by the motherboard with a 120W version and derive 5V and 12V rails from that independently from the motherboard to power the drives. This could be a fun new avenue to return to and build.  

## Final Build

  * Intel Pentium J5005
  * 8GB DDR4
  * 120GB SSD (Boot)
  * 120GB SSD (CCTV)
  * Marvel 9125 - 4 port SATA Controller
  * 4x Seagate IronWold 4TB Drives
  * Coral AI Accelerator
