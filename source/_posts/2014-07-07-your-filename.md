---
layout: post
published: false
title: ""
comments: true
categories: misc
---

## Using RxJava in combination with Retrofit

At WunderCar our backend provides a pure REST api as communication interface for the mobile clients. This is of course a very clean and straightforward solution. For the Android app we decided to use Retrofit as communication layer as it provides a very easy and clean way to communicate with a REST interface.
Furthermore, in our app we try to create some kind of realtime experience for the user. For instance, we show the driver on a map as he is approaching the guest to pick him up. Most of the business logic is dealt with on the backend side to be able to change things quickly and to have a consistent experience on all platforms. To be specific, the mobile clients ask the backend for the user state and update the UI accordingly. 

