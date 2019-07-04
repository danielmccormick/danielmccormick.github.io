---
layout: post
title: "Learning From Mistakes - Concurrency, Packing and PNGs"
categories:
  - Blog
last_modified_at: 2019-07-04T14:15:02-05:00
---

I'm a firm believer than you learn more from failures than from successes. From that perspective, for our [concurrency class](https://github.com/jzarnett/ece252) labs I must have learned a ton.  

In one of our labs, a challenge we had was that we needed to implement the producer/consumer problem for PNGs.
We needed to download the PNGs from a webserver (as a _producer_) into a finite size buffer, then move the data into the appopriate slots (as a _consumer_) and concatenate them all into a final `all.png` 
While that sounds fairly reasonable all things considered, we can get into a few interesting issues.

### System Design

It was reccomended we use something like a circular queue being mutexed and accessed between the producers and consumers. The idea with this would be that the consumers could chase the tail of the producers (and then have individual locks on each element in the queue). This design, all things being considered has a few flaws:  

1. There's no guarantee the last element would finish last, resulting in undesired gaps between producing and consuming  
2. There's implementation issues when there's more producers/consumers than the size of the queue - how to you decide who gets which slots  
3. Scheduling how each worker moves: Does the first to finish just lock the queue and grab the first available slot? Does each worker get a predetermined slot?  

Regardless, there was a design that made more sense to us. 
The idea here is that the buffer would be random access, and each index would be stored inside one of two stacks (either the producer stack or the consumer stack). When the producer gets a signal that it's stack has an index, it grabs the index, downloads the data and adds the index to the consumer stack.  

The producer pattern was roughly:  

    * wait(producer_semaphore)  
    * lock(producer_mutex)  
    * get_index()  
    * unlock(producer_mutex)  
    * produce_data()   
    * lock(consumer_mutex)   
    * insert_index()  
    * unlock(consumer_mutex)  
    * post(consumer_semaphore)  

And the consumer pattern looked like: 

    * wait(consumer_semaphore)  
    * lock(consumer_mutex)  
    * get_index()  
    * unlock(counsmer_mutex) 
    * consume_data()  
    * lock(producer_mutex)  
    * insert_index()  
    * unlock(producer_mutex)  
    * post(producer_semaphore)  

This starts with all locks unlocked, the producer stack/semaphore full, and the consumer stack/semaphore empty (as nothing is produced to begin). While lots of people had deadlocks and issues, since we considered the design pattern carefully, we didn't have conflicts with this issue. The reasons this were successful were pretty simple: we never nested mutexes, so we couldn't have one lock a resource the other would need. The semaphore was an appropriate barrier since it'd let a worker through to take data if and only if data was available. While I do like concurrency, I'm glad I didn't have too many issues debugging it. 

### Struct Packing and Cryptic Error Messages

At one point, we had everything right but one checksum: the [CRC](https://en.wikipedia.org/wiki/Cyclic_redundancy_check) (cyclic redunancy check) for our IHDR (the header chunk) was incorrect. Our CRC was always valid for the other components and always invalid for the header. Since it ws the same function, and we were quite confident about the spacing, it seemed very likely something was wrong about the header. 

For reference, our struct for the IHDR components were declared something like   

    typedef struct {  
        uint32_t length;  
        uint32_t type; //0x49484452  
        uint32_t width;  
        uint32_t height;
        uint8_t bit_depth;  
        uint8_t colour_type;  
        uint8_t compression_method;  
        uint8_t filter_method;  
        uint8_t interlace method;  
        uint32_t crc;  
    } IHDR_t;`

In additional debugging, we also found we got all other members correct - the only variable wrong in the header was the CRC. This bug is not obvious at first, but it makes a reasonable amoutn of sense.  

The way we really called the crc calculation was something like `crc(&IHDR.type, (3*4)+(5*1))`because we were calculating the CRC over three four-byte variables and five one-byte variables. We also knew each variable in the struct was corect, the size of each variable was correct, and the CRC was correct. If you'd like to figure out why yourself, you should stop reading now. 

Fortunately for us, a hex dump of the header yielded some more insight. It turns out the culprit in this case was [struct padding](https://stackoverflow.com/questions/4306186/structure-padding-and-packing). While in hindsight, this makes a lot of sense, at the time it's easy to neglect that c structs don't guarantee there's contiguous memory by default. 

The fix from here was compiler specific, but pretty straightforwards:  

    typedef struct __attribute__(packed) {  
        ...    
    } IHDR_t;    

This problem also seemed specific to us, since a lot of our peers stored the variables in a "data" member, and to calculate the CRC they copied it all into a buffer. Fortunately, we prevailed and produced the correct result.


### Cleanup and Debug: 

Since we were using multiple processes concurrently, over school networks, we ended up using shared messages.
It wasn't too long before the server was spitting out an errormessage alone the line of `shmget: No space left on device.  
  
This wasn't so cryptic, but it was almost comical looking at who hoarded the memory (on university servers).  
  
`ipcs -a > log.out; wc -l log.out; grep -c "_username_" log.out` 

`3409 log.out 1293`

Any given server, the same one person used roughly a quarter of all possible shared memory. Everyone was guilty of leaking at a point, but some seem to have a bit more of an issue than others. It's also worth nothing that while shared memory unhooks from processes naturally, it doesn't deallocate naturally (or even with `shmdt`).  This _was_ discussed in class, but it seems like many people forgot or their code prematurely terminated before the cleanup could occur.

If anyone is dealing with shared memory for the first time, it's worth remembering to include `shmctl(shmid,IPC_RMID,NULL);` at the end of their program to avoid funny surprises like this. I guess this is why it's important to consider the impact your code has on others.

Finally, when debugging, to try printing the buffer treating the data as a big pile of characters and printing the data out char by char. This results sometimes in obnoxious outputs however, like `!▒▒j▒▒{▒s▒▒rt▒s▒m[rΘm▒B▒I]w▒i▒)j        ▒`9▒▒▒m[`.  
  
Obviously, this isn't an ideal or easy way to debug, and it's hard to recognize what bytes physically are. An easy fix solution would be to print it out chunk by chunk as an integer, but there's an even better solution. Printing it was a hex value. Either dumping a finished file with xxd or printing out using %x. This check is invaluable for when you need to diff non text files or trying to figure out how much you get wrong. 


Mistakes are natural, and in my opinion better in a lab environment where the scope is rather limited (constraining the causes of the bug), to learn the causes before there's a litany of factors to consider concurrently. To quote Alexander Suvorov, "Train hard, fight easy. 

The main takeaways I had from this were:

1. Take some time to _really_ think about your design, or you'll spend far more time in the long run on the design.  
2. Consider what you really are accomplishing, because abstractions tend to be leaky.  
3. Always choose the best debugging tools for the job - you'll otherwise just maker it harder on yourself.

