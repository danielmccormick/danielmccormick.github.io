---
layout: post
title: "Embedded Systems - Learning and Growth (WIP)"
categories:
  - Blog
last_modified_at: 2019-07-18T13:15:02-05:0
---

This school term, I'm taking embedded microprocessor systems (ECE 224), and instrumentation and prototyping (ECE 298). While at times they have been challenging, overall, I have learned a lot and ultimately got to deal with interesting challenges taught in class. 

## Rule One: Printf is not your friend

Many people, including me loathe to use a debugger as the first line of debugging. Generally speaking, maybe we'd prefer to print out a value to figure out why we're getting a segfault, or to validate things are going right. Maybe we'd use printf to display our results in a fashion to get the code. 

In ECE 298, our processor (MSP430FR4133) didn't support printing floating point numbers, because it used too much power. This resulted in issues because we couldn't see what we were actually doing. Additionally, for performance, our display didn't have a printf, so we had to arbitrarily write bytes to the buffer. Finally, once we resolved this by finding a reasonable print function (to display all of six characters)< it turns out representing a floating point number as 6 characters is _weird_. Ultimately, I came up with a [workaround](https://github.com/danielmccormick/playground/tree/master/ftoc/source), but in embedded systems, displaying information is both expensive and painful. 

In ECE 224, our labs were entirely about high performance programming, and printf caused even more issues than previously though. When it comes to high performance computing, if your immediate code needs to be performant, one very important rule is pretty simple: **you must not use blocking functions**. If your code needs to respond *right away*, and you suddenly need to lock a mutex, you're in big trouble. At best, you can spinlock, but that's still very much slower. While we typically don't think about it, printing to a console can absolutely wreck your outputs. If you don't know this, you'll get some confusing and curious results.   
  
For example, in one case, when trying to do performance sensitive stimulus response (ie responding within a period of <=750 cycles), every other test would straight up fail. We were able to pass any individual test case, we were able to pass every other test case, but no matter how we incremented, every other test case we tried failed. Looking through our code wasn't super useful because it _was_ correct, it just wasn't performant enough. Eventually, after some careful thinking, we solved it.   
  
Not only is printf expensive in power, it can easily ruin your program if called at a bad moment. 

## Rule Two: Do not trust the compiler

In one of our labs, we had a fairly simple piece of busy work to do to benchmark results. It seemed simply enough, and surely would be meaningful enough to approximate roughly the right amount of work:  
  
    int background() {
        int j;
        int x = 0;
        int grainsize = 4;
        int g_taskProcessed = 0;
        for(j = 0; j < grainsize; j++) {
            g_taskProcessed++;
        }
        return x;
    }

Ok. Compiled using ARM gcc 8.2 it seems pretty reasonable:

    background():
        str     fp, [sp, #-4]!
        add     fp, sp, #0
        sub     sp, sp, #20
        mov     r3, #0
        str     r3, [fp, #-16]
        mov     r3, #4
        str     r3, [fp, #-20]
        mov     r3, #0
        str     r3, [fp, #-12]
        mov     r3, #0
        str     r3, [fp, #-8]
    .L3:
        ldr     r2, [fp, #-8]
        ldr     r3, [fp, #-20]
        cmp     r2, r3
        bge     .L2
        ldr     r3, [fp, #-12]
        add     r3, r3, #1
        str     r3, [fp, #-12]
        ldr     r3, [fp, #-8]
        add     r3, r3, #1
        str     r3, [fp, #-8]
        b       .L3
    .L2:
        ldr     r3, [fp, #-16]
        mov     r0, r3
        add     sp, fp, #0
        ldr     fp, [sp], #4
        bx      lr

Of course, there's other factors that could contribute to performance, but it seems fairly reasonable to use as a benchmark. However, since we assumed the compiler would know things we didn't, we used optimization. As a result, we got bafflingly high performance. Obviously it doesn't take a houdini in hindsight to figure this out after the fact, since the code was equivalent to 

    int background() {
        return 0;
    }

Which translates into

    background():
        mov     r0, #0
        bx      lr

Of couse, we wouldn't have figured this out without benchmarking. In the other case, we had a compiler move a print statement to an inopportune position of performance sensitive code which resulted in really _weird_ sounds for our WAV music player ("Our printfs aren't even in the critical section!")   

Luckily, when tested thoroughly, the compiler optimization greatly enhanced the performance of our WAV player and was absolutely invaluable in fixing. I'm not complaining about what the optimizer does, just giving a heads up for those thinking about it. Of course, if we were allowed to modify our code, we simply could've declared all the variabeles volatile. In hindsight, perhaps saying "do not trust the compiler" might be a bit inflammatory.

## Rule 3: Consider the hardware

Since printf is too slow, we were going to toggle some memory mapped LEDs to observe the results in real time. One would on first glance presume you could write your code along the lines of 

    uint16_t led_status = IORD(LED_BASE,0);
    led_status ^= 0x0002;
    IOWR(LED_BASE,0,led_status);

Which gave strange and unexpected results. With further reading, it turns out the LED was write-only. With this knowledge in mind, it's easy to see the issue, however you just treat the memory address as a data port like any other, this isn't immediately obvious. This also is the cause of other issues.

To be continued?
