Building a capable star tracker on the STM32H743VIT microprocessor.

# Week 0: Planning and Requirement Gathering

## why do I want to build a star tracker?
It seems like a fun challenge that will help apply and learn about what goes into designing and validating software and firmware. I initially picked up the interest for star trackers at an internship and the processing pipelines and simulations really fascinated me and I think about it fairly often. Enough to make me want to try and build my own! I will mainly be focusing on the firmware and compute side of things, and I'll be doing it on a very constrained system (relatively speaking).
## what makes a star tracker work?
The inner workings of how a star tracker functions can be simplified to a couple high level steps that each require important design decisions to be made depending on the targets you want to meet. The way I understand a star tracker is by a deceivingly simple pipeline.

![[StarTracking.drawio.png]]

Okay fine it may not be that simple. And some foreshadowing, it gets wayyy more complex as we dissect each step. Thankfully from a high level it isn't too bad... We get an image in, we want to pre-process it to separate the stars from the background, etc. Essentially making the image suitable for our star detection algorithm to pick out what might be a star and what might not be. Then we identify what parts of our image are stars and which ones are not. The next 3 steps are meant to take our flat layout of stars in our image, usually specified by what row and what column of the image they are in, and turn it into a 3D unit vector so that we can compute certain attributes that will allow us to compute attitude. Generally star trackers also have 2 operating modes: Lost In Space (LIS) or Tracking mode. A star tracker will always start in LIS mode then if implemented it may switch to tracking mode once an attitude is determined. In LIS mode the star tracker has no idea which way its facing. It will take our best star candidates in unit vector form and compute the direction cosines between them and search the whole catalogue for a specific match. This can take a fair bit of time since the cosines must be compared to the majority of the catalogue. But if we already have an attitude we can focus on only a section of catalogue since we know that our next lookup will be somewhere nearby to our previous lookup. Once we have identified a match, we do some validation and perform yet another transformation to find out what quaternion our match corresponds to. That's it... right? Fuckkkkk no. But we will work through it. 

> Most of the information I use throughout my explanations comes from previous professional experience (shoutout Adam, Elliot, Martin and Hamed if you are reading this you guys are awesome) and this gem of a book on celestial navigation using Stars as a reference (star tracking lol) [Star Identification: Methods, Techniques and Algorithims](https://link.springer.com/book/10.1007/978-3-662-53783-1)
## designing

Right off the bat we can start making decisions, this is great, probably. Since our processing pipeline is fairly independent of hardware, so we can model it in a more tool rich environment like a Linux machine. But there is a constraint that we must absolutely pay attention to during modelling. 

Our MCU has limited RAM, we have about 1MB total to work with, but it is distributed in banks. This is important because some banks are in the same clock domain as the CPU (D1, or the AXI bus), some are in a separate clock domain, where all our peripherals are (D2, AHB), and some are on a lower power bus (D3). The biggest constraint here will be that we can't fit an entire image in our D2 domain SRAM, or even on the AXI SRAM for that matter. 

Now you may ask, surely an image can't be that big? I have thousands of images on my phone and I can access them fairly quickly? To that my answer is: a 1920×1080 8-bit grayscale image occupies 2,073,600 bytes (~2MB). The H743 has no single contiguous SRAM region that large, and its total usable SRAM is ~900KB after accounting for stack and peripheral buffers, so storing a full frame in RAM is impossible. Storing it in internal flash is feasible for a simulation bench (the H743 has 2MB of dual-bank flash) but internal flash reads through the AXI interface introduce wait states at 480MHz (the clock which im running the core system at), we set that at 3 wait states, so thats 3 wait states per 64-bit word access without the prefetch cache. Streaming row by row with the prefetch unit enabled is viable, but it bounds throughput in a way that SRAM access does not. Not only that but flash memory is page accessed, it makes for accessing images internally more complex than simple RAM access.

Since we are also using Ethernet as our communications method, and the primary way images will get onto our star tracker, we will want RAM that is accessible to the Ethernet DMA controller for performance reasons 

![[Pasted image 20260327110709.png]]
Memory & Bus architecture from the Reference Manual, you can see the different domains and which peripherals have access to what. Take note of the different SRAM banks. available. In terms of size, the AXI SRAM is certainly the largest weighing in at 512kb, while the other banks are a quarter of that size. 

>  This diagram is from the [RM0433](https://www.st.com/en/microcontrollers-microprocessors/stm32h743-753/documentation.html) the link is to the main documents page, you may need an account to download the document. Big surprise, you'll see me *reference* this a lot during the embedded design process. I may also complain about it a lot since it likes to be vague and not cover all the features that the STM32H743 is capable of, and then im stuck relying on HAL comments.

Okay so we don't have enough space for entire image, end of project... NAHHHH. We just have to change the way we process the image. We don't actually need the whole image at once to be able to determine an attitude. We can instead take a streaming approach. We split the image into chunks of rows, get the information we need for that chunk (which will be significantly more compact) and proceed to the next chunk. This way we only need to store a fraction of the image in RAM at a time. 

So when we model our pipeline we will just need to implement it using this chunked approach so that we can easily map it into our firmware later. 

### initial embedded design
While the star tracking fundamentals are still soaking in my brain, I wanted to get started on setting up the infrastructure for the star tracker. Mainly choosing between RTOS or bare metal, organising memory and choosing an initial communications protocol. 

#### RTOS or bare-metal

> Only a fool would develop using an RTOS, too much abstraction
> -some old fortran wizard probably

I went with an RTOS off the rip since we will be dealing with a few concurrent tasks and managing that on bare-metal is not something I'm particularly inclined on spending my time on. We will have at least 3 tasks at the beginning, one for managing commands coming in from the communications peripheral. Another for managing outgoing commands/telemetry. And one for the processing pipeline, which we may want to split into a few tasks so that in between computations we can do some command processing. 

## Week 1: Modelling

> honestly i feel like this should've been one of the earliest steps. If i have the ability

Before we hit any potential road blocks like not having enough memory to run our pipeline or processing power to provide real time results, we should definitely model our processing pipeline. Since most of the work that will be occurring on the embedded device is hardware agnostic we can model almost the entire processing pipeline on my Linux machine.

Once we've modelled our pipeline we can benchmark it, and profile it to get an idea of if our hardware will be able to keep up. If not, we will try and optimise for time, or space, (or try both) to get it to work on our target hardware. And in the worst case we will upgrade our hardware to something beefier, (a multi-media processor would be cool like an SoC with ARM A9 arch).



