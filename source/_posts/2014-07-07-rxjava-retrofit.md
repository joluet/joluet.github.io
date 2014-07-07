---
layout: post
date: "2014-07-07 15:15"
published: true
title: RxJava in combination with Retrofit to chain REST calls
comments: true
categories: rxJava
---

## Example
At WunderCar our backend provides a pure REST API as communication interface for the mobile clients. For the Android app we decided to use Retrofit as communication layer since it provides a very easy and clean way to communicate with a REST interface.
Due to our architecture we often have to chain api calls. For example we need to fetch the user state after the user has been successfully logged in. This can lead to ugly nested calls:

```java
api.login(new Callback<ResponseBody>() {
         	@Override
            public void success(final ResponseBody body, final Response response) {
            	// login was successful -> ask for user state
				api.getUserStatus(new Callback<UserStatus>() {
                    @Override
                    public void success(final UserStatus status, final Response response) {
                    	// update UI according to user state
					}

					...
```
RxJava allows us to shorten this piece of code to:
```java
eventAPI.requestRide().
	flatMap(status -> api.getUserStatus()).		// chain calls using flatMap
    subscribe(onComplete, onError);	 			// callbacks onComplete and onError
```

## RxJava
RxJava is an open source library that implements the _[Reactive Extensions(Rx)](https://rx.codeplex.com/)_.
The key class of Rx is the _Observable_ that represents a model object for asynchronous data streams. Observables can be easily composed using different operators that are capable of filtering, selecting, transforming, combining and composing Observables.
Practically, this means that Observables are aimed fulfill asynchronous tasks like e.g. making an http call. The cool thing about Observables is the possibility to compose them to create new more powerful Observables. See the [RxJava Wiki](https://github.com/Netflix/RxJava/wiki) for a detailed introduction.

## Retrofit and RxJava
I won't go into detail about Retrofit here. See [square.github.io/retrofit](http://square.github.io/retrofit/) for an introduction. Retrofit integrates RxJava and one can define api calls that return Observables:
```Jjava
@GET("/user/login.json") 
Observable<SuccessResponse> login();
```
Thus, one doesn 't
