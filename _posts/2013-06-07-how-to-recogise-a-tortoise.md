---
group: blog
title: How to Recognise a Tortoise
layout: post
---

Computer vision is an area which I had a distant interest in for quite some time. I appreciate that the task of vision is a complex one and always imagined that there must be some great algorithms behind the task. The process of computer vision, like many areas of artificial intelligence, is one where it is so easy to view the actions of a computer as those of a person. It was this lingering interest, with a touch of reverence, that prompted me to choose a vision project in my final year of university. It completely changed the way I think about artificial intelligence.

My [final year project](http://bitbucket.org/iwillspeak/turf) performs statistical object recognition using *Grassmannian Manifolds*. It is an interesting technique. I enjoyed learning more and more about the field of object recognition throughout the process. The downside however was the more I learnt, the more areas I exposed to the light of knowledge the fewer areas there were for that great secret of artificial intelligence to hide. The secret of course is a disappointing one: it is all a sham. There is a man behind the curtain.

I don't mean to belittle the achievements of artificial intelligence researchers here, there is a lot of great research going on out there. The problem I have is more one of perception. The more I learnt about the area the more the realisation dawned that I didn't consider the process intelligence. The computers were just following an algorithm, carrying out a pre-determined process. Some would argue that the human brain is just a computer carrying out an incredibly complex program, however I feel that there is some form of difference. Until the algorithm, if there is one, that determines our actions is known the mist surrounding our actions will always separate my perception of humans and computers.

## What About the Tortoise?

Intelligent or not I am proud of my application. It performs recognition using a statistical representation of images. If we want the computer to recognise an example object, such as *George* the tortoise, we first need to make sure that the program knows about tortoises. We do this using a set of images of several stock tortoises. The images of each tortoise are grouped together into a Maifold, a representation of them in a higher dimensional space.

When we want to recognise George we take some pictures of him too. With these images we compute his manifold representation too. From this representation we can decide what type of object George is. This stage is known as classification.

In my project a simple classifier is used that simply picks an object type based upon the closest objects that the program knows about in the higher dimensional space to the one it is trying to classify. This is referred as the *K-Nearest Neighbour* classifier by those in the know. It is simple, but effective. My application was ~88-92% accurate with the test data sets I used.

## Can I See a Picture of George?

Sure can: ![George, the mightiest of tortoises][george]

Cute isn't he? Until next time, have fun!

[george]: {{ site.url }}/img/posts/george_the_tortoise.jpg