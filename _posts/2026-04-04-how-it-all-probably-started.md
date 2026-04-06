---
layout: post
title: "How it all (probably) started: Dad vs Son (Blue vs Red)"
date: 2026-04-04 13:37 +0200
categories: [Personal]
tags: [personal]
---

No story can start without a famous Arnold Schwarzenegger quote "None of us can make it alone". Especially when you are in elementary school and you don't have money to buy a computer. Fortunately, my dad did and he decided to upgrade from an Atari to a Pentium 200 MMX. What a beast that was! You could actually have everything stored on a HDD, and not loaded directly from a floppy disk.

But my father soon realized that his teenage son does not possess virtuous traits of patience and self-control, as my eyes were bleeding from computer usage in the first few weeks. He also noticed that when he was away from home, the first thing I did was jump on the computer to play video games. Yes, video games. I did not make computer virus at age 11 that brought down Wall Street. So he did what any parent would do - add some protections.

At first, these protections were simple. He would take the power cable with him at work. But, you could always "find" an extra cable in school or borrow it from a friend. Then, when he went to work, I would plug my "borrowed" cable and played until I would see him arriving back. As he was a school professor and had his timetable printed out at home, I could easily see when he would come back from work. Of course, I always wanted to play until the last minute so I would often get up from playing the game, looking out the window to see if he was coming back.

Unlike his son, my father was smart, and he knew that something fishy is going on. Instead of asking me to tell the truth, he  trusted his instincts (that I would lie), and called the company from which he bought the computer to ask them what can he do to prevent / control this unwanted behavior. As putting me in a ZOO was not an option, he decided to set a pre-boot BIOS password. Next day when my father went to work, I immediately jumped at the computer and was literally in shock when I saw:

![Thanks, Dad.](/assets/img/1/award-bios-password.jpeg)

I started to enter some passwords and soon reached the maximum number of password attempts (don't remember the exact value). I was frustrated. WHAT KIND OF SORCERY IS THIS? So, I would wait till the next day and try some other passwords - but no luck. And then, something magical happened. During the weekends I could play the computer for a few hours and I learned that you can actually send and receive something called "electronic email". While poking around the computer I discovered the username and password for POP3 authentication in a folder my father used to store his data. Was it possible that he used the same password for the BIOS? Should I try it now to check or wait until the next opportunity? No dear patience, you will not win. So, I immediately rebooted the computer, entered the password that my father used for POP3 auth to the BIOS prompt and BOOM! Ha! I still remember being so excited that playing games over that weekend was not fun at all, as it would be during the working week when my father is away.

I was using this password for quite a few weeks, but as my father had decades of experience with students who were doing no-good, he knew something is wrong. He sensed that once again. And he changed the BIOS password. I was fucked. I did not manage to guess the password and all of my attempts to look at the password he was typing in when he arrived home failed, because he would kick me out of the room at those moments. This traumatic childhood experience lasted for a few months, until my friend told me that he heard there is something called a "master password", that allows you to bypass any user-set password. So I've Yahoo'ed (yes, Yahoo) to find it, and it worked. Writing this post I could not remember which was it, but I've Googled it around to find [this](https://gist.github.com/Yousha/b6ed8edf8961f028140a563718ea929c), and I am pretty sure the password for my Award BIOS was BIOSTAR.

My father knew I used the computer. I was not sure how at the moment, but I knew he tried changing BIOS passwords - not knowing that I was using the master password to bypass whatever he set it to. Then I caught him touching the CRT monitor to feel if it was warm! So I adapted - before he came home from work I waved with a book over the monitor until it completely cooled down.

And lastly, ISDN came. Pixel loading of mature websites became much more tolerable. But Internet bills started getting bigger and bigger (my fault, of course), so my father called the ISP company to see what can be done about it. He then subscribed to an ISP service which allows you to block all outgoing calls (out-of-city, Internet, mobile phone networks) and allow you to call only inter-city land-line numbers while the protection is in place. This worked in a way that you would dial a series of codes, then the unblock / block code (0/1), and then ending with a 4-digit PIN and a hash (#). Challenge accepted, Dad! As my father had enrolled me in a primary music school, I had good enough hearing to break this. First, I would tap the redial button and listen to the sound of the numbers he entered on the phone next to the computer (to block all outgoing calls), and then on the other home phone I would decipher the PIN digits one by one by listening and repeating the dial tones. Red team wins!

As you can see, I was creative. Like children are. I just had a particular interest in breaking restrictions my father put in place. Funny thing is - once my father removed all the restrictions, I actually started using computer / Internet like a normal person would.

All of it was there 30 years ago — reconnaissance, obtaining hardware to bypass restrictions, brute-forcing, password reuse, credential harvesting, shoulder surfing, hardcoded credentials, acoustic side-channel attack (this is a thing apparently), and anti-forensics.

This post is for you, Dad.

Hack the planet.

Ivan
