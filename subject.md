# Subject

A Subject is a double nature. It has both the behaviour from an [Observer](/observer.md) and an [Observable](/observable-anatomy.md). Thus the following is possible:

Emitting values

```
subject.next( 1 )
subject.next( 2 )
```

Subscribing to values

```
const subscription = subject.subscribe( (value) => console.log(value) )
```

To sum it up the following operations exist on it:

```
next([value])
error([error message])
complete()
subscribe()
unsubscribe()
```

## Acting as a proxy

A `Subject` can act as a proxy, i.e receive values from another stream that the subscriber of the `Subject` can listen to.

```
import { interval, Subject } from 'rxjs';
import { take } from 'rxjs/operators';

let source$ = interval(500).pipe(
  take(3)
 );
 
const proxySubject = new Subject();
let subscriber = source$.subscribe( proxySubject );

proxySubject.subscribe( (value) => console.log('proxy subscriber', value ) );

proxySubject.next( 3 );
```

So essentially proxySubject `listens` to `source$`

But it can also add its own contribution

```
proxySubject.next( 3 )  // emits 3 and then 0 1 2 ( async )
```

**GOTCHA**  
Any `next()` that happens before a subscription is created is lost. There are other Subject types that can cater to this below.

### Business case

So what's interesting about this?  It can listen to some source when that data arrives as well as it has the ability to emit its own data and all arrives to the same subscriber. Ability to communicate between components in a bus like manner is the most obvious use case I can think of. Component 1 can place its value through `next()` and Component 2 can subscribe and conversely Component 2 can emit values in turn that Component 1 can subscribe to.

```
sharedService.getDispatcher = () => {
   return subject;
}

sharedService.dispatch = (value) => {
  subject.next(value)
}
```

## ReplaySubject

prototype:

```
import { ReplaySubject } from 'rxjs';

new ReplaySubject([bufferSize], [windowSize], [scheduler])
```

example:

```
import { ReplaySubject } from 'rxjs';

let replaySubject = new ReplaySubject( 2 );

replaySubject.next( 0 );
replaySubject.next( 1 );
replaySubject.next( 2 );

//  1, 2
let replaySubscription = replaySubject.subscribe((value) => {
    console.log('replay subscription', value);
});
```

Wow, what happened here, what happened to the first number?  
So a `.next()` that happens before the subscription is created, is normally lost. But in the case of a `ReplaySubject` we have a chance to save emitted values in the cache. Upon creation the cache has been decided to save two values.

Let's illustrate how this works:

```
replaySubject.next( 3 )
let secondSubscriber( (value) => console.log(value) ) // 2,3
```

**GOTCHA**  
It matters both when the `.next()` operation happens, the size of the cache as well as when your subscription is created.

In the example above it was demonstrated how to use the constructor using `bufferSize` argument in the constructor. However there also exist a `windowSize` argument where you can specify how long the values should remain in the cache. Set it to `null` and it remains in the cache indefinite.

### Business case

It's quite easy to imagine the business case here. You fetch some data and want the app to remember what was fetched latest, and what you fetched might only be relevant for a certain time and when enough time has passed you clear the cache.

## AsyncSubject

```
import { AsyncSubject } from 'rxjs';

let asyncSubject = new AsyncSubject();
asyncSubject.subscribe(
    (value) => console.log('async subject', value),
    (error) => console.error('async error', error),
    () => console.log('async completed')
);

asyncSubject.next( 1 );
asyncSubject.next( 2 );
```

Looking at this we expect 1,2 to be emitted right? WRONG.  
Nothing will be emitted unless `complete()` happen

```
asyncSubject.next( 3 )
asyncSubject.complete()

// emit 3
```

`complete()` needs to happen regardless of the finishing operation before it succeeds or fails so

```
asyncSubject( 3 )
asyncSubject.error('err')
asyncSubject.complete()

// will emit 'err' as the last action
```

### Business case

When you care about preserving the last state just before the stream ends, be it a value or an error. So NOT last emitted state generally but last _before closing time_. With state I mean value or error.

## BehaviourSubject

This Subject emits the following: 

* the initial value 
* the values emitted generally
* last emitted value.

methods:

```
next()
complete()
constructor([start value])
getValue()
```

```
import { BehaviorSubject } from 'rxjs';

let behaviorSubject = new BehaviorSubject(42);

behaviorSubject.subscribe((value) => console.log('behaviour subject',value) );
console.log('Behaviour current value',behaviorSubject.getValue());
behaviorSubject.next(1);
console.log('Behaviour current value',behaviorSubject.getValue());
behaviorSubject.next(2);
console.log('Behaviour current value',behaviorSubject.getValue());
behaviorSubject.next(3);
console.log('Behaviour current value',behaviorSubject.getValue());

// emits 42
// current value 42
// emits 1
// current value 1
// emits 2
// current value 2
// emits 3
// current value 3
```

### Business case

This is quite similar to `ReplaySubject`. There is a difference though, we can utilize a default / start value that we can show initially if it takes some time before the first values starts to arrive. We can inspect the latest emitted value and of course listen to everything that has been emitted. So think of `ReplaySubject` as more _long term memory_ and `BehaviourSubject` as short term memory with default behaviour.

