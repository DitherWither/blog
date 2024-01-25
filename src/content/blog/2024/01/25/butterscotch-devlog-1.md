---
title: 'Butterscotch Devlog 1: The first one'
description: 'I got '
pubDate: 'Jan 25 2024'
heroImage: '/images/2024/01/25/butterscotch-devlog-1.png'
---

Now, when you want to get started with making an operating system, you have a ton of choices, regarding where to start, and what platform to use.
I considered using RISC-V and Arm, but in general, x86 tended to have much better docs and resources, so I went with that in the end. I also chose to use Rust, as I felt certain things would be easier to do in rust. I can easily use libraries for some stuff as well, rust has plenty of libraries that work without the standard library

As this was my first time developing an operating system, I went with a tutorial, namely [Philipp Oppermann's blog](https://os.phil-opp.com/) on making an operating system in rust.

The first couple of days of butterscotch's development was relatively simple and uninteresting, as I was just following a tutorial. There were some annoyances though:

 - The lazy static crate isn't really recommended anymore, but the alternative to it (lazycell) isn't as simple to use. I ended up going with `static mut` for stuff that is only written to during initialization
 - For stuff that gets mutated, a mutex usually suffices, along with the `Option` enum
 - The crate that was used for the bootloader had major changes to its structure, and I personally dislike the way the new crate does things
  
As for the bootloader, the `bootloader` crate's newer versions recommend you to make a a separate crate that uses your main kernel crate as a binary dependency (a nightly rust feature), and then use a `build.rs` script to build it. Personally, I think this is unnecessary complexity over what could be a simple `Makefile`, a cli tool with a config file, or a couple of build scripts. I did use the older version of the bootloader crate while I was using the tutorial though (I switched it out later)
  
Once I got memory allocation working, I started to diverge from the tutorial, as that tutorial was practically abandoned after this.

The first order of business was to move to the `talc` allocator instead of the linked list one. (I don't exactly remember why I did this)

Next, I decided to move from the `bootloader` crate to using Limine. Limine is a bootloader and a boot protocol, and the bootloader sets a lot of stuff for you, just like the bootloader crate. Notably, limine doesn't need an assembly stub to boot.

Some other stuff limine is:

 - It always loads the kernel at the higher half of the virtual address space This is useful, as your userspace can then be loaded at the lower half.
 - It loads up into long mode directly, and can set up the framebuffer, map the physical memory into the virtual address space,
 - It sets up the GDT up with sane entries
  
The annoying stuff about it was, however, that my paging implementation relied on the `bootloader` crate's memory map, and in general, I had a bunch of code that pointed to stuff provided by the bootloader crate

Another thing was that I had to completely rewrite the entire console implementation, as limine deprecated VGA text mode. Initially, my implementation directly wrote the pixels to the display, calculating the color for each pixel, and each and every print call. The font crate I was using provided a bitmap with the intensity of each pixel, for an anti aliased font. This meant, however, that I had to use the intensity to calculate the actual display color. I ended up storing the entire character set in a large `Vec`, so that these calculations are only done on boot. I could in theory do this at compile time, but this would mean that I can't configure the colors at runtime, which is something I plan to add later on.

While I was rewriting the framebuffer, I used the serial port for debugging and IO

Limine masks all Interrupts by default, so I had to manually unmask the interrupts.

Also, I removed the old GDT, as it was causing issues with a lot of stuff. Notably, I was getting a double fault for all interrupts, even the ones with handlers. Limine sets this up for us anyways, which works good enough (for now).

Finally, I configured the PIT (Programmable Interrupt Timer), to send an interrupt every millisecond, and used it to implement a `sleep()` function. 

Our kernel is currently single core, and uses the PIC for IO interrupts, instead of the APIC, which is designed to work with multiple cores. I intend to rewrite the kernel to use that instead, but that is a problem for future me.

I also plan to add a simple filesystem driver soon.

I was considering writing a microkernel, but that would make this too complicated, so I'm instead writing butterscotch as a monolithic kernel, and either start from scratch later, rewrite parts of it to make it a microkernel, or maybe make an operating system based on an existing kernel (Sel4 Maybe). I plan on developing butterscotch to the point that it has a basic CLI with secondary storage, and maybe multitasking.

Anyways, this article was written on June 25, the literal day it was published, in less than half an hour. Therefore, it probably has a ton of mistakes, and isn't as detailed as I would like it to be. Next week's devlog should be more interesting, longer, and better written.

Thanks for reading, and bye