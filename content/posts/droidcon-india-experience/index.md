---
title: "My experience at Droidcon India 2019 🇮🇳"
summary: Modern app distribution
date: 2019-11-10
weight: 1
tags: ["Droidcon", "Conference", "Android", "Community"]
cover:
  image: images/header.webp
  
  caption: "Shot on my Pixel 3a XL"
  hiddenInList: false
---

Attending Droidcon was in my wish list until recently I attended Droidcon India. It was great catching up with like-minded folks and connecting with them in person. This post is all about my experience at Droidcon India, so without further ado let’s get started.


## Day 0 (Coz we are devs 😜)
The first day started with a welcome address by [Adnan](https://twitter.com/AdnanM0123). Organizing an event this big is not an easy task. He welcomed us warmly despite him and his organizing team being tired after a few sleepless nights. He mentioned two things that really struck me. The conference is not about attending as many talks as you can, rather it’s fine to skip a few and connect with people. The other thing he mentioned was, make 3 new friends whenever you go to the conference and over a period of time you will end up creating a healthy community around you. Next up was opening keynote.

### Keynote — Inhibiting the impostor
It was all about impostor syndrome, how to detect whether you have it and ways to overcome it.

> Impostor syndrome (also known as impostor phenomenon, impostorism, fraud syndrome or the impostor experience) is a psychological pattern in which an individual doubts their accomplishments and has a persistent internalized fear of being exposed as a “fraud”.
- Wikipedia

I think there couldn’t be a better topic than this for an opening keynote. It helped me to overcome my impostor syndrome of being an attendee and not a speaker. Often people doubt themselves when they attend any conference and feel they don’t know anything and that’s why they are there to gain knowledge. But every one of us knows something that others don’t.

It helped me and I am sure others too, to feel comfortable as an attendee throughout the conference.

### Talks

I had already decided which talks I was gonna attend prior to the conference. As planned on the first day I attended all of them except the last two.

> All talks were awesome so please don’t consider talks mentioned here are good ones. These are just my personal favorites and you can decide which one to watch once recordings are out.

#### Reimagining Android Productivity with TCR(test && commit || revert)

This talk by [Ragunath Jawahar](https://twitter.com/ragunathjawahar) was about writing test and production code first and then if the test passes the code gets checked-in automatically and if it fails it will revert test as well as production code. This way TCR model kinda forces you to ship features that are rock solid. Less bug means less time being wasted on fixing those and more time for developing something hence the title.

#### Let’s Stream that Video — an ExoPlayer Starters Guide

This talk was about ExoPlayer for streaming video. The fact that it makes streaming simple and does lots of heavy lifting for you makes it an obvious choice for streaming video. The talk covers basics of ExoPlayer like ExoPlayer API and how to use it followed by how to customize UI like media control and action buttons in the player.

#### Zero to Hundred — Building an App that scales

For B2C scaling is very important. This talk was about how PhonePe addressed it using a layered architecture having a clean separation of concerns. They took inspiration from the Android framework itself to come up with robust architecture.

#### Multi modularization scaling app

At [BookMyShow](https://in.bookmyshow.com/) we are working on modularization so I was more curious about how Grab is doing this. This was about the advantages that you get if you modularize your app and how to structure your modules. The talk also covers how you can leverage Dagger to communicate between feature modules.

The first day ended with lots of learnings and networking.

## Day 1 (Day 2)

The second day kicked off with a great keynote by [Amrit Sanjeev](https://twitter.com/amsanjeev) (Dev advocate at Google). He started with 10 things in Android 10 that you should care about to give the best experience to your users. After that, he switched the gear to the current situation around developers covering the following points.

- Indian devs learn new technologies very fast but are reluctant to try it in production and wait for others to poke holes in new technologies.
- Indian devs heavily depend on libraries and there is nothing wrong with this but we should also know how to build one if the situation arises in the future.

He also talked about how technology is evolving rapidly and we should have the breadth of knowledge on various technologies(React Native, Flutter, etc) but at the same time master one technology and be an expert in that.

### Talks

Again only covering those that I attended and out of them, those I was awake in 🤦‍♂

#### @inject basics

Everyone loves Dagger.

{{< iframe src="https://giphy.com/embed/7TcdqHUCFIfEaRJzKn" width="480" height="270" frameBorder="0" class="giphy-embed" >}}

This talk covers the basics of dependency injection and how you can achieve it without any dependency framework and then how frameworks like Dagger, Koin, etc achieve it with code generation or reflection.

#### Coding with your hands tied — A session on developing Android Libraries

Unlike app developers, library devs don’t have much liberty in terms of using other libraries. In this talk [Darshan Pania](https://twitter.com/i_m_Pania) (Developer at CleverTap) covered how they are addressing this problem. One thing I really liked about this talk was it was interactive talk in the sense that he took opinion from the audience on what CleverTap can do better to enhance their SDK. Finally at the end talk covered how to publish your library on JCenter through JFrog Bintray.

In the end, there was a closing keynote which I couldn’t attend but definitely check out once the recording is available.

My 2nd day was more about networking and less on talks as you can figure out from the number of talks I covered here for 2nd day.

## Parting thoughts

You must be wondering why everything is rainbows and sunshine here after all we are talking about a tech event without technical issues. There were few regarding LED panels showing wrong colors and slides were skewed. Also, 2nd day started a bit late and they had to shorten breaks in between to compensate that.

Apart from these two issues overall It was a great event and I learned a lot. Topics were a healthy mixture of everything that Android devs can benefit from and speakers were also chosen from all around the world.

I got the chance to meet a few folks I follow on twitter but never met them in person. Made some new friends and lots of memories. Kudos to [Adnan](https://twitter.com/AdnanM0123) and the entire organizing team for pulling this off 👏. I would highly recommend next Droidcon India 👍

Finally, I want to close this with the event link so that you can go through the talk schedule and have a look at all talks that are not covered here.

https://www.in.droidcon.com/

That’s all about my experience at Droidcon India 🙂

Until next time 👋

> *This was originally posted on [Medium](https://medium.com/android-news/my-experience-at-droidcon-india-2019-5e4360c647d5)*
