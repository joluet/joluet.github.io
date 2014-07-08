---
layout: post
date: "2014-07-07 15:15"
published: true
title: Using RxJava and Retrofit to compose REST calls in an elegant way
comments: true
categories: rxJava
---

## Example
At WunderCar our backend provides a pure REST API as communication interface for the mobile clients. For the Android app we decided to use Retrofit as communication client since it provides a very easy and clean way to communicate with a REST interface.
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
Note: I use lambda expressions in the RxJava code snippets. This makes the code much more readable. Unfortunately they are only available in Java 8. To be able to use them in Android with Java 7 you have to include the retrolambda library like explained [here](http://zserge.com/blog/android-lambda.html).

## RxJava
RxJava is an open source library that implements the _[Reactive Extensions(Rx)](https://rx.codeplex.com/)_.
The key class of Rx is the _Observable_ that represents a model object for asynchronous data streams. Observables can be easily composed using different operators that are capable of filtering, selecting, transforming, combining and composing Observables.
Practically, this means that Observables are aimed fulfill asynchronous tasks like e.g. making an http call. The cool thing about Observables is the possibility to compose them to create new more powerful Observables. See the [RxJava Wiki](https://github.com/Netflix/RxJava/wiki) for a detailed introduction.

## Retrofit and RxJava
I won't go into detail about Retrofit here. See [square.github.io/retrofit](http://square.github.io/retrofit/) for an introduction. The point is Retrofit integrates RxJava and one can easily define api calls that return Observables using Retrofit:
```java
@GET("/session.json") 
Observable<SuccessResponse> login();

@GET("/user.json") 
Observable<UserState> getUserState();


```
Thus, we don't even need to create our own Observables. We can just compose the Observables provided by Retrofit. There are many possible compositions as described in the [wiki](https://github.com/Netflix/RxJava/wiki/Observable). In our case the _flatMap_ as well as the _combineLatest_ operators have been very useful:

[`Observable<R> flatMap(Func1<? super T,? extends Observable<? extends R>> func)`](http://netflix.github.io/RxJava/javadoc/rx/Observable.html#flatMap(rx.functions.Func1)) creates an Observable by applying a function that you supply to each item emitted by the original Observable. The function intself is an Observable. After applying the function the items are emitted.
A very simple example where the supplied function doesn't use the items emitted by the original Observable.
```java
api.signUp(signUpData).
	flatMap(successResponse -> eventAPI.emailSignIn(signUpData.email, signUpData.password))
```
The original Observable does the sign up. After the successful sign up the second Observable tries to sign in the user. So the flatMap operator is used to chain the two network calls.

[`<T,R> Observable<R> combineLatest(java.util.List<? extends Observable<? extends T>> sources, FuncN<? extends R> combineFunction)`](http://netflix.github.io/RxJava/javadoc/rx/Observable.html#combineLatest(java.util.List,%20rx.functions.FuncN)) creates an Observable that emits the latest items that have be emitted by the source Observables. This is helpful when you want to execute multiple REST calls and only update the UI when all calls have finished.