---
layout: post
date: "2014-07-07 15:15"
published: true
title: Neatly Composing REST Calls Using Retrofit and RxJava
comments: true
categories: "android,rxjava,retrofit"
---

This article shows how to use [Retrofit](http://square.github.io/retrofit/) together with [RxJava](https://github.com/Netflix/RxJava) and emphasizes the possiblity to combine and chain REST calls using rxObservables to __avoid ugly callback chains__. It gives you a nice example of using RxJava in order to write less and cleaner code.
<!--more-->
## Example
At [WunderCar](http://www.wundercar.org) our backend provides a pure REST API as communication interface for the mobile clients. For the Android app we decided to use Retrofit as communication client since it provides a very easy and clean way to communicate with REST interfaces.
Due to our architecture we often have to chain api calls. For example we need to fetch the state of a user _after_ the user has been successfully logged in. Using callbacks, this can lead to ugly nested callback chains:

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
eventAPI.login().
	flatMap(status -> api.getUserStatus()). // chain calls using flatMap
    subscribe(onComplete, onError); // action callbacks onComplete and onError
```
Note: I use lambda expressions in the code snippets. This makes the code much more readable. Unfortunately, they are only available in Java 8. To be able to use them in Android with Java 7 you can include the retrolambda library like explained [here](http://zserge.com/blog/android-lambda.html).

## RxJava
RxJava is an open source library that implements the _[Reactive Extensions(Rx)](https://rx.codeplex.com/)_.
The key class of Rx is the _Observable_ that represents a model object for asynchronous data streams. Observables can be easily composed using different operators that are capable of filtering, selecting, transforming, combining and composing Observables.

Practically, this means that Observables are aimed to fulfill asynchronous tasks like e.g. making an http call. One cool thing about Observables is the possibility to compose them to create new and more powerful Observables. See the [RxJava Wiki](https://github.com/Netflix/RxJava/wiki) for a detailed introduction.

## Retrofit and RxJava
I won't go into detail about Retrofit here. See [square.github.io/retrofit](http://square.github.io/retrofit/) for an introduction. The important point is, Retrofit integrates RxJava and one can easily define http calls that return Observables using Retrofit:
```java
@GET("/session.json") 
Observable<LoginResponse> login();

@GET("/user.json") 
Observable<UserState> getUserState();
```
Thus, we don't even need to create our own Observables. We can just compose the Observables provided by Retrofit. There are many possible compositions as described in the [wiki](https://github.com/Netflix/RxJava/wiki/Observable). In our case the _flatMap_ as well as the _combineLatest_ operators have been very useful:

[flatMap(func)]( http://netflix.github.io/RxJava/javadoc/rx/Observable.html#flatMap\(rx.functions.Func1\) ) creates an Observable by applying a function that you supply to each item emitted by the original Observable. The function intself takes the items emitted by the original Observable as input and returns a new Observable.
A very simple example where the supplied function doesn't use the items emitted by the original Observable:
```java
api.signUp(signUpData).
	flatMap(successResponse -> api.emailSignIn(signUpData.email, signUpData.password))
```
In this example the original Observable does the sign up. After the successful sign up the second Observable tries to sign in the user. So the flatMap operator is used to __chain two network calls and execute them one after another__.

[`combineLatest(sources, combineFunction)`]( http://netflix.github.io/RxJava/javadoc/rx/Observable.html#combineLatest\(java.util.List,%20rx.functions.FuncN\) ) creates an Observable that emits the latest items that have be emitted by the source Observables. This is helpful when you want to execute multiple REST calls asynchronously and update the UI when _all_ calls have finished.
```java
Observable.combineLatest(api.fetchUserProfile(), api.getUserState(), 
(user, userStatus) -> new Pair<>(user, userStatus));
```
This simple example executes two asynchronous http calls. __When both calls have returned, their results are emitted as a pair__.

## Threads and Lifecycle
In order to deal with the items that are emitted by an Observable you need to subscribe to it and provide two actions:
```java
login().subscribe(onComplete, onError)
```
Actions are callback functions that are called in case the Observable has finished successfully or has emitted an error.

If you don't assign threads to subscribe or observe on, Observables will just execute their code and deliver the results in the main thread. This is often not what you want. So be sure to call `subscribeOn()` on the Observable to define a scheduler that determines the thread to subscribe on, e.g. `subscribeOn( Schedulers.from( AsyncTask.THREAD_POOL_EXECUTOR ) )`. In addition you should define the thread for observation by calling e.g.  `observeOn( AndroidSchedulers.mainThread() )`.

In Android we also have to think about the activity or fragment lifecycle. Often we want to change the UI when the an Observable emits new items. This is of course only possible if the fragment or activity is still alive. So we have to be sure to unsubscribe from the Observable when the fragment or activity is destroyed at the latest.

Actually RxJava provides a convinience helper class that can prevent some lifecycle issues:
```java
AndroidObservable.bindActivity(Activity activity, Observable<T> source)
```
> This helper will schedule the given sequence to be observed on the main UI thread and ensure that no notifications will be forwarded to the activity in case it is scheduled to finish.
>
> You should unsubscribe from the returned Observable in onDestroy at the latest, in order to not leak the activity or an inner subscriber.
>
> -- <cite>excerpt from the associated source code comment</cite>

So my reccomendation is to use the `AndroidObservable` helper to bind subscriptions to fragments or activities and to make sure that subscriptions are unsubscribed from in `onDestroy` at the latest.
