# Understanding Functional Reactive Programming

Proper event handling is hard, not only because you have to manage the state transitions but you also have to balance multiple different events, occuring in tandem. What I will show is an approach called FRP or (Functional Reactive Programming). It simplifies most event based programs and avoids **injected functions** completely, thus allowing better reasoning about the program.

## The basics


Imagine you are making a space shooter, in this game you have the following mechanics.

* Every 200 ms update the postion
* Every 0.5 sec spawn a meteor in a random direction
* When the user clicks space we fire a missile

![3 meteors and a space ship (in space)](./space.png)

Now lets say that the user fires a missile 530 ms into the game.

This will give us the following event timelines

| Timeline |0 ms| 100 ms| 200 ms| 300 ms | 400 ms | 500 ms | 530 ms | 600 ms | 700 ms | 800 ms | 900 ms | 1 sec |
|-------|----|-------|-------|--------|--------|-|--------|--------|--------|--------|-------|-|
| Postion | | | **P**| | **P**| ||  **P** | | **P** | | **P** | |
| Meteor | |  || |  | **M** || | | |    |  **M** |
| Click | | | | | | | **C** | | | | |  |


If we see each timeline a list of points `(Time, Event)`, this gives us a time series for when a significant event occurs inside the game. Because time is monotonically increasing (never decrease), we can lazily merge the series at a very low cost.

Merging algorithm (In Pseudo C#)
### Pseudo
```csharp

static IEnumerator<(Time,TEvt)> merge(IEnumerator<(Time,TEvt)> A,
                                      IEnumerator<(Time,TEvt)> B) {

    (time_a, a) = A.take(1);
    (time_b, b) = B.take(1);

    if (time_a >= time_b then) {
        yield (time_a, a);
        yield (time_b, b);
    } else {
        yield (time_b, b);
        yield (time_a, a);
    }
}
```

***NB:** This algorithm is only meant to given an understanding, it won't actually work as it assumes the two series swap between producing an event, which is obviously not true. (Real algorithm will be given in the end)*


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

 Notice that in our timeline Position update at time step 1 sec, occurs before the spawning of meteors, this is because our merging algorithm is left biased. This goes back to what I said about understanding and handling events in tandem.

With this merged event series. For each event we want to perform a state transtion of our game.

Example main loop

### C#
```csharp
var state = new State();
for timedEvt in events {

    if timedEvt.evt == GameEvents.Positon {
        state.UpdatePosition();
    } else if timedEvt.evt == GameEvents.Meteor {
        state.SpawnMeteor();
    } else if timedEvt.evt == GameEvents.Click {
        state.FireMissile();
    }

}
```

### Rust
```rust
let mut state = State::new();
for &(_, evt) in &events {
    match evt {
        Position => update_position(&mut state),
        Meteor => spawn_meteor(&mut state),
        Click => fire_missile(&mut state),
    };
}
```









