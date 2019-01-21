date: 2018-02-27

# Digital pendulum
## Constant acceleration

My intuition is that moving motor with constant speed from a start point to
an end point is not the best thing to do. Moved objects have inertia, so
moving objects with constant acceleration and than decceleration seems more
reasonable.

I extended my code for driving stepper motor with a function that does
just that.

I calculate an array of delays and use it to drive the motor:

```forth
25 constant motor.move-profile-size
800 constant motor.min-delay
10000 constant motor.max-delay

create motor.move-profile motor.move-profile-size 1+ cells allot
: init-profile ( ratio1 ratio2 min-delay max-delay -- )
  motor.move-profile cell+
  dup motor.move-profile-size cells +
  swap do
    dup i !
    2over */
    2dup > if
      i motor.move-profile - 2 arshift motor.move-profile !
      leave
    then
  1 cells +loop
  2drop 2drop
;
9 10 motor.min-delay motor.max-delay init-profile
```

It really helps! With delay 0.8 ms between half-steps motor misses steps,
but when I start from 10 ms delay and than gradually lower it to 0.8 ms
it moves without problems.

It's not a constant acceleration though... Do you know why?
[*Answer is here*](004-Constant-acceleration-revisited).

## Pendulum
Let's try something more complicated. Nice and natural movement is a pendulum movement.

A position in time can be described by the following equation:

$$x = r \sin{\omega t}$$

What we need is not a position in time, but a timestamp for each position. So we need to inverse the equation:

$$t = {1 \over \omega} \arcsin{x \over r} $$

What we are really interested in is a derivative of this function. The derivative tells how long we should wait in each position:

$$t' = {1 \over \omega r} { 1 \over \sqrt{1-({x \over r})^2} } $$

Calculating this with `mecrisp-stellaris` was a challenge for a novice like me, but here it is:

<iframe width="560" height="315" src="https://www.youtube.com/embed/tZ4Z8J8wuLw?rel=0" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

Full [source code is on GitHub](https://github.com/tocisz/forthplay/blob/master/stepper/pendulum.fs).
