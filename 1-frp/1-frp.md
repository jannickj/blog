# Understanding Functional Reactive Programming

Contrary to popular belief Frp does not require a functional language nor does the code have to functional.
 As such all code examples will be in C# that there can be no doubt. The only requirement is that events and the time they occur are **immutable**. In most systems this is the case as events are simply just messages, small fragments of state.

The issues most pressing that Frp solves are as follows:

* **Desync:** Self correcting in nature, preventing state coming out of sync  *(better stability)*
* **Avoid Hooks:** Inversion of control, avoids the need for injected state changing functions *(better reasoning)*
* **Histroic playback:** (Requres: **Deterministic transitions**)  Any replay system must be Frp  *(better overview)*

## So what is Frp?

The key characteristic of any Frp system is that **Events** are associated with a **Timestamp**, and that these are **immutable** meaning their **state** is never changes.

A lot of systems we use are Frp in nature and it is what allows them to be powerful.

The most obvious example is **banking**, in the banking world every transaction is associated with a timestamp and a the amount to move from one account to another.

A less obvious example is `git` a tool most developers use everyday. In `git` every `commit` is associated with the changes to the code along with a timestamp, `git` guarantees the purity of state by using a checksum hash for the commit.

With that said lets get into the nitty gritty details of Frp.

## The basics

Imagine you are making a space shooter, in this game you have the following mechanics.

* Every 200 ms update the postion of all objects
* Every 0.5 sec spawn a meteor in a random direction
* When the user clicks spacebar we fire a missile

![3 meteors and a space ship (in space)](./space.png)

Now lets say that the user fires a missile 530 ms into the game.

This will give us the following event timelines

| Timeline |0 ms| 100 ms| 200 ms| 300 ms | 400 ms | 500 ms | 530 ms | 600 ms | 700 ms | 800 ms | 900 ms | 1 sec |
|-------|----|-------|-------|--------|--------|-|--------|--------|--------|--------|-------|-|
| Postion | | | **P**| | **P**| ||  **P** | | **P** | | **P** | |
| Meteor | |  || |  | **M** || | | |    |  **M** |
| Click | | | | | | | **C** | | | | |  |


If we see each timeline as a list of points `(Time, Event)`, this gives us a time series for when a significant event occurs inside the game. Because time is monotonically increasing (never decrease), we can lazily merge the series at a very low cost O(1) for each  event.

`Merge` algorithm (In C#)
```csharp

static IEnumerator<(Time,TEvt)> merge<TEvt>(
    this IEnumerator<(Time,TEvt)> A,
         IEnumerator<(Time,TEvt)> B) {

    var hasA = A.MoveNext();
    var hasB = B.MoveNext();

    // Go through A and B simultaniously always yielding the yongest event
    while(hasA && hasB) {
        var (time_a, a) = A.Current;
        var (time_b, b) = B.Current;
        if (time_a >= time_b then) {
            yield A.Current;
            hasA = A.MoveNext();
        } else {
            yield B.Current;
            hasB = B.MoveNext();
        }
    }

    // Yield any remaining events in either A or B
    if (hasA) {
        do {
            yield A.current;
        } while(A.MoveNext());
    } else if (hasB) {
        do {
            yield B.current;
        } while(B.MoveNext());
    }

}
```

Given that each merged series produce a new series we can apply merge multiple times.

```csharp
var merged = PositionEvents
                .merge(ClickEvents)
                .merge(MeteorEvents)
```

Merging produces the following timeline

| Timeline | 200 ms|  400 ms | 500 ms | 530 ms | 600 ms | 800 ms |  1 sec | 1 sec |
|-|-|-|--------|--------|-------|-|-|-|
| Merged | **P**| **P** | **M** | **C** |  **P**  | **P**  | **M** | **P** | |

 **Notice:** In our timeline Position update at timestep 1sec occurs after the spawning of meteors, this is because our `merge` algorithm is left biased.

With this merged event series. For each event we want to perform a state transtion of our game. Given that we have a merged sequence of events that is now easy.

Example update state function

```csharp
static void UpdateState(GameState gameState,
                        IEnumerator<(Time,GameEvents)> merged) {
    var events = merged.reverse();
    for (time, evt) in events {
        if evt == GameEvents.Positon {
            gameState.UpdatePositions();
        } else if evt == GameEvents.Meteor {
            gameState.SpawnMeteors();
        } else if evt == GameEvents.Click {
            gameState.ship.FireMissile();
        }
}
```
Notice that we play events in reversed order, this is an important and easy to miss. However we want to playback from oldest to newest event (obviously).

With this we have an event playback system, however there are still some things that are left unclear.
* Generating re-occuring events
* Real time systems
* How to deal with user input

## Re-occuring events

In Frp it is important to distingush between **Predictable events** and **Unpredicatable events**.

**Unpredicatable events** are outside the control of the internal system and as such can occur at any time. The **Click spacebar** is an example of this type of events as it is entirely up to the player when to press spacebar.

**Predictable events**  occur at a regular interval. The **meteors spawn** and the **Postion update** are examples of predictable events. These are often re-occuring, and will be our focus.

**Question:** If you have an event that occurs every hour, and the time is currently *15:13*, how many minutes is it since the last event occured? **13 minutes?**

**Unfortunately** that is wrong, as we never specified when the events began we have nothing to base our predictions on.

However lets for the purpose assume it started at *10:30* then we know that the last event must have occured at *14:30* and predicting the time since it occured is **trivial**, and can be written with the following formula.

```
Last = Now - (Now + First) % Interval
```

So to answer our question

```
- Now       = 15:13 = 913 min
- First     = 10:30 = 630 min
- Interval          = 60 min

Last = 913 - (913 + 630) % 60 = 870
```
870 min / 60 = 14.5 = 14:30 **(Correct)**

In most Frp systems, especially real time ones, you are able to control when your events began. As such if we define an **origin time** a time in which all events started occuring.

This **origin time** should be **_Zero_** for simplicity, allowing us to write the formula as.


```
Last =  Now - Now % Interval
```

### Generating events

With our newfound ability to predict occurance of events we can create an event generator.

```csharp
static IEnumerator<Time> every(long now, long interval) {
    var last = now - now % interval;
    while(last > 0) {
        yield last;
        last = last - interval;
    }
}
```

Running it with `every(1000, 200)` gives us

| Timeline |0 ms| 100 ms| 200 ms| 300 ms | 400 ms | 500 ms | 600 ms | 700 ms | 800 ms | 900 ms | 1 sec |
|-------|----|-------|-------|--------|--------|-|--------|--------|--------|--------|-------|
| () | | | ()| | ()| |  () | | () | | () |

If we map our time series for our events given in the game example we get
```csharp
Position = every(1000, 200).select(t => (t, Events.Position))
Meteor = every(1000, 500).select(t => (t, Events.Meteor))
```

| Timeline |0 ms| 100 ms| 200 ms| 300 ms | 400 ms | 500 ms | 600 ms | 700 ms | 800 ms | 900 ms | 1 sec |
|-------|----|-------|-------|--------|--------|-|--------|--------|--------|--------| -|
| Positon | | | **P**| | **P**| |  **P** | | **P** | | **P** |
| Meteor | |  || |  | **M** || | |    |  **M** |

**Tadaaaa!!.. magic**, we now have the same identical event time series that we had before.

An import thing to notice is that the `every` produces a lazy sequence meaning `every(time,interval).take(1)` runs at `O(1)` time and not `O(time/interval)`. This becomes really important when time is set to current unix timestamp.

## Real time playback

In the examples shown so far time has been a static constant that was set from the start. However for real time systems this won't do, instead we view time as a window and we only update our state based on events inside that window. *Confused*? Don't worry all will become clear soon.

Remember the function `updateState` we defined earlier?

```csharp
var gameState = new GameState();

var now = 1000;
var position = every(now, 200).select(t => (t, Events.Position));
var meteor = every(now, 500).select(t => (t, Events.Meteor));
var merged = position.merge(meteor);

updateState(gameState, merged);
```

This piece of code will give us the state after 1000 ms of events.

Let say `421 ms` passes and `now = 1421`, how would we go about updating the `gameState`? If we update our state with all events from 1000 to 0 again then our state would have been played with the same events mutiple times. As such we must cut the events to only be the window of events between now and last window.

`Window = ]Last, Now]`
Meaning that Now is included and events exactly at Last is excluded, as those were included in the prior window.

```csharp
var last = 1000;
var now = 1421;
var position = every(now, 200).select(t => (t, Events.Position));
var meteor = every(now, 500).select(t => (t, Events.Meteor));
var merged = position
                .merge(meteor)
                .takeWhile((t,_) => t > last); //All events until time of last

updateState(gameState, merged);
```
**Window** \]1000, 1421\]

| Timeline | > 1 sec | 1200 ms | 1300 ms | 1400 ms | 1421 ms |
|-------|---------- |-------|- |- |- |
| Postion | | **P**|  | **P**| |
| Meteor | |  | | | |

As we can see in this window `gameState` will only be updated with two position updates, the meteors are not spawned because that update occurs outside the window.

Now lets says another 90 ms. How will the next window look?

**Window** \]1421, 1511\]

| Timeline  | > 1421 ms | 1500 ms   | 1511 ms   |
|-------    |---------- |-------    | -         |
| Postion   |           |           |           |
| Meteor    |           | **M**     |           |

In this window it is only the meteors that gets updated.

### Putting it all together!

To make it work continuously, we run it in a while loop, and store the time of the `last` update step.

```csharp
var last = DateTime.Now.TotalMilliseconds;

while(True) {
    var now = DateTime.Now.TotalMilliseconds;
    var position = every(now, 200).select(t => (t, Events.Position));
    var meteor = every(now, 500).select(t => (t, Events.Meteor));
    var merged = position
                    .merge(meteor)
                    .takeWhile((t,_) => t > last);

    updateState(gameState, merged);
    last = now;

    draw(gameState);
}
```

**Question** Lets say that the `draw` function completely lags out when `now = 482`, and the next `now = 1232`, what will happen to `gameState` at `1232`? will it miss all the events that occured between *1231* and *482* **!?!?**

**Answer** No the window size is expanding and contracting depending on how fast the computer is able to run the update loop. This because Frp is **insentive** to the resolution at which it does **event sampling**. In prior works I've refered to this as **Sampling Resolution Insensitivity**.
And with we see why Frp is:

* Self correcting in nature, preventing state coming out of sync  *(better stability)*

## User input

If you've come this far, you should now be well versed in how Frp works, and it shouldn't be much of a surprise when I say that User input events are just another Event time series.

Most legacy window framework, are completely based around injected functions, however with just a few tweaks it is fairly easy to change it to Frp design.
```csharp

var asyncEvents = AsyncList();

window.onClickSpaceBar(_ => asyncEvents.add(Click.spacebar));
...

while(True) {
    ...
    // Copy click events from the window frame work,
    // and clear for next window.
    var clickEvents = asyncEvents.CopyThenClear()
                        //attach all events with a time
                        .select(c => (Now, c));

    // use them as any other event
    var merged = position
                    .merge(meteor)
                    .merge(clickEvents)
                    .takeWhile((t,_) => t > last);
    ...
}
```

As you might notice the time the user input occur is just what ever the current time window is at, you can spend time and energy on making it be the time at which the user actually clicked space. However the update windows are in general very small, such that the actual game appears smooth, given this whether we say the user clicked **0.1-0.2ms** faster has no actual impact.

Using this solution we are able to completely avoid any state update inside an injected function. And instead we have inverted the control back to us of when and how user input changes our state thus fulfilling the second promise of Frp:

* **Avoid Hooks:** Inversion of control, avoids the need for injected state changing functions *(better reasoning)*


## Historic playback

Finally we arrive at historic event playback. Whether you're making a game replay system or just want have better debugging power. In Frp historic playback is trivial.

We simply add a step of saving all the events **(Yeah it's that easy!)**
```csharp
var last = DateTime.Now.TotalMilliseconds;
var history = List<(Time, Events)>()

while(True) {
    var now = DateTime.Now.TotalMilliseconds;
    var position = every(now, 200).select(t => (t, Events.Position));
    var meteor = every(now, 500).select(t => (t, Events.Meteor));
    var merged = position
                    .merge(meteor)
                    .takeWhile((t,_) => t > last);
    // Store the events
    history.AddMany(merged.reverse());
    ..
}
```

With this event history we can save it on our computer and play it back at a later time. To see what actually happend. An important thing to note that if you planning to playback the events, you should be certain that your `updateState` function is **deterministic** otherwise the is no guarantee you will get the same result.

With this we conclude the last and final promise of Frp:

* **Histroic playback:** (Requres: **Deterministic transitions**)  Any replay system must be Frp  *(better overview)*


## Advanced: Monadic time series












