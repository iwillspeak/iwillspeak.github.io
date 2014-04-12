---
group: blog
title: Secure for the Common Case
layout: post
---

"I'll have to type in that thing again" my dad moaned. "They gave me a new laptop and I haven't used it on the wireless yet." he explained. The "thing" he was referring to of course was the wireless key. A completely incomprehensible string of numbers and letters that took up three lines on the back of the junk-mail envelope in his hand. I began wondering exactly how secure the house wireless password was.

The key itself was written on pieces of paper and haphazardly taped to the bottom of the router. It's not really _that_ much of a secret, provided you have access to the house already. It is not the ever so slight reduction in security that really matters here though. Someone's ability to access the internet is the last thing you care about if they have broken into your house. Having a password so random and complex that you _have_ to write it down everywhere, a password that you dread having to type in in case you make a mistake, is a level of complexity that is not needed. It is just an annoyance. 

It all boils down to two questions. Is `9a05bbaf5c827459ae7711ce05` really more secure than `Hundr3dw3ight Axolotl Tr33`? Does the security gain justify the added pain of using the "random" one?

I would suggest that the answer to these questions is "possibly" and "probably not". Not very definitive I know, it is hard to give definitive answers to these questions. You never really know [what kind of threats][xkcd_comic] you are going to be up against. Perhaps the first string is "more random", whatever that means. Perhaps it contains more information. Perhaps it would be harder to attack with a brute force approach. Perhaps not. Susceptibility to brute force attacks depends upon the assumptions made by the attacker. Had they not realised that the password contained spaces, as many people's don't, then the second password would take a _very_ long time to crack.

In computer science there is a saying "optimise for the common case". It is used to argue that it is only worth optimising the bits of applications that people will use most often. Any time spent optimising other parts is not worth the gains. It is worth more to save a tenth of a second for instance in a browsers page load time than it is to save several seconds in it's bug reporting screens.

There is more to it though. The phrase also discourages optimising some parts of the program at the expense of others, especially those parts that are used most often. You can't optimise everything, but you can at least make the most common things as fast as possible.

Seen like this the most important feature of a good password, for WiFi or anything, is that it should be memorable. The "common case" for password use surely has to be remembering it and typing it in. You can't make your password uncrackable, it just needs to be more difficult to break than any of those nearby. The barrier you have to beat [doesn't seem to be that high][sophos_vid].

With security in mind all your wireless password needs to do in the most common case is to keep out the next-door neighbours who have run out of bandwidth on their own connection and attempt to piggyback yours. After all, if GCHQ wants to read your emails your wireless password isn't going to stop them; it'll just ask the people at the NSA. 

[sophos_vid]: http://www.youtube.com/watch?feature=player_embedded&v=NaPdSiX0Aho
[xkcd_comic]: http://xkcd.com/538/