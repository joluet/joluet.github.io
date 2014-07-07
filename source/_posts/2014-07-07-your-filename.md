---
layout: post
published: true
title: ""
comments: true
categories: rxJava
---

## Using RxJava in combination with Retrofit to chain api calls

At WunderCar our backend provides a pure REST api as communication interface for the mobile clients. For the Android app we decided to use Retrofit as communication layer since it provides a very easy and clean way to communicate with a REST interface.
Furthermore, in our app we try to create some kind of realtime experience for the user. For instance, we show the driver on a map as he is approaching the guest to pick him up. Most of the business logic is dealt with on the backend side to be able to change things quickly and to have a consistent experience on all platforms. To be specific, the mobile clients ask the backend for the user state and update the UI accordingly. Due to this architecture we often have to chain api calls. For example we fetch the user state from our backend after certain calls have been answered. This can lead to ugly nested calls:

```java
api.requestRide(new Callback<UserStatus>() {
         	@Override
            public void success(final UserStatus status, final Response response) {
            	// api call was successful -> ask for user state
				api.getUserStatus(new Callback<UserStatus>() {
                    @Override
                    public void success(final UserStatus status, final Response response) {
                    	// update UI according to user state
					}

					...
```

