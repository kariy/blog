+++
title = "I like to prematurely optimize my code"
description = "Sometimes premature optimizing your code is good"
date = 2024-06-21T22:40:48-04:00
draft = true
tags = ["programming"]
+++

Premature optimization is the root of all evil bla bla bla.... If you're a somewhat decent software engineer that has been coding for a while, then you most likely have heard of that saying at some point in your life before.

For most people, they view it as a negative thing.

Something that one should always try to avoid because once you have succumbed to the temptation of pursuing the most optimal 'code', what entails is a long wasted hours that could've been used to do stuff that can actually bring value to the software.

I do believe that if what you're hyper fixating on finding the most 'perfect' code/implementation for a particular problem, that should generally be viewed very negatively. There is no such thing as the 'perfect' code especially if you're working on a software that actually makes money, the most important thing is to just get things done.

Well... not necessarily most of the time.

Personally, I think there is some places or aspects of when you're writing code where you can or should spend a tad bit more effort in optimizing them.

When you're working on a large project where there are other people that will be actively reading/pushing code other than you, then it's important that the code you write can be easily digestible by them.

Yes, you can say skill issue or whatever. But if you want people to actually enjoy working with your code, then you should be hyper optimizing on making your code as simple and straightforward as it can be.

Write comments whenever it make sense. If the project is sufficiently large enough where each people is assigned to different parts of the code base, then it's very likely they don't have the full context of your the PRs that you asked them to review. Usually, reviewing your peers' PRs is good thing. Yes, proper reviews takes time but imo, it's good to have most of the devs working on a project to have at least a high-level overview of the entire code base, including parts where they don't even have a major (if any) responsibility at all. Having the full picture of how each part of the code base is moving and intertwined together can help in making better and more positive value judgement when deciding how certain features/refactors should be done.

Other thing to prematurely optimize, is error handling. Try to narrow down as much as possible places where errors can happen. If there are bunch of `.unwrap`s in your function, then try to return the error instead of resorting to panics.

You should also be hyper fixated on making your code easy to be modified in the future. Software specifications change all the time, usually the implementations that you wrote today are based on the limited use cases that you have for now. Once more ideas have come out, that's when your CTO decides that you should extend your previous code to support a new feature that they are planning. If you have at least spent a significant amount of effort in making sure you previous implementation can be changed easily (which tbf is easier said than done, especially when you don't know what kind of changes you will made), once you (or your colleagues) attempt to refactor that it, you wouldn't have to cursed your past selves for writing such complicated code in the first place.
