title: Reactive Programming
date: 2013-02-01 21:15:54
tags: code
comments: yes
---

Most programming languages or techniques nowadays are sequential. When
executing a statement `c = a + b`, `c` is the result of the addition of the
current values of `a` and `b`. If either one of those variables change their
value, `c` is not changed.

```js
var a = 5;
var b = 10;
var c = a + b;
console.log(c); // logs 15
a = 20;
console.log(c); // still logs 15
```

<!-- more -->

## Electronics

In electronics however, this isn't usually the case. The "drain" of a FET
transistor always (under normal circumstances) reacts on the so called "source".
If we put some transistors together we can make a NAND gate:

![NAND Gate](/assets/NAND-gate.png)
<br><small>Image taken from <a href="http://wikipedia.org">Wikipedia</a></small>

If you would describe the behavior of a NAND gate in simple JavaScript code,
it would look as follows:

```js
var Q = !(A && B);
```

But here is the thing: if you now change either `A` or `B`, `Q` is immediately
changed too (if we neglect the propagation delay). You could say the output
reacts on the input.

When some NAND gates are *wired* together, complete ALUs (Arithmetic Logic
Units), or even processors, can be built. At some point, those logic
networks become too difficult to create manually by just creating and solving
some Karnaugh maps. That is why certain tools are created, to simulate,
synthesize and program logical networks. One of those is *VHDL*, which is a
Hardware Description Language. How it works is pretty similar to physically
wiring components together.

## VHDL

In VHDL a component like a NAND consists out of two parts. The first part is
called the `ENTITY`, where the input and output ports are defined. The
second part is the `ARCHITECTURE`, or usually called the behavior. This defines
the logic of the component. For example see this implementation of an `AND`
gate:

```
-- this is the entity
entity ANDGATE is
  port (
    I1 : in std_logic;
    I2 : in std_logic;
    O  : out std_logic);
end entity ANDGATE;

-- this is the architecture
architecture RTL of ANDGATE is
begin
  O <= I1 and I2;
end architecture RTL;
```

<small>Example code taken from <a href="http://wikipedia.org">Wikipedia</a></small>


Eventually all those components can be wired together in VHDL. Now when all
those lines exist between components, what should happen when a line changes?
To solve this, a `SIGNAL` statement can be defined, which connects lines to the
component so they can be used in the `PROCESS`. If a line changes, the behavior
part is executed again. This causes that all changes are propagated through the
network.

## Back to our world: JavaScript

Since you are reading this, and I am doing a lot of JavaScript, I assume you
do that a lot as well. So what can we learn from those electronics and VHDL?

The concept that a variable changes if one of the inputs are changed, is nothing
new under the sun. It is called reactive programming.

Since I am studying Embedded Systems at the TU Delft, I learned a little bit
about VHDL and read other stuff, shortly after that I read this blogpost about
[Bacon.js](https://github.com/raimohanska/bacon.js), which made me think if I
could bring this concept to the JavaScript world by implementing a very small
JavaScript module.

The result of that endeavor is this GitHub repository which contains a little
more than 50 lines of JS: [reactive](https://github.com/arian/reactive).

## Reactive

It works as follows:

```js
function nand(a, b){
    return !(a && b);
}

// This implements the behavior of a NAND.
var A = rx(true);
var B = rx(true);
// Q uses A and B as inputs. The current values of those inputs
// are passed into the "nand" function.
var Q = rx().use(A, B, nand);
console.log(Q.get()); // logs "false"

// Create second NAND with input K and L.
var K = rx(false);
var L = rx(false);
var R = rx().use(K, L, nand);

// Wire the outputs of the previous 2 NANDs into a third NAND.
var S = rx().use(Q, R, nand);

// The final result of a two layer NAND network
// an OR gate with 4 inputs!
console.log(Q.get()) // true

// And it is reactive:
A.set(false); B.set(false);
// All inputs are now false, so result of S is:
console.log(S.get()) // false

// set one input to true
K.set(true);
// So output of S is
console.log(S.get()) // true
```

## Theory is nice, but what's the point?

Responsive design is hot today! "Responding to something" is more or less the
same as "reacting to something". Other topics today are modules, creating less
spaghetti code or loosely coupled code.

When we use MooTools, we can combine DOM Events with reactive in a few lines of
code:

```js
[Document, Window, Element].invoke('implement', 'addEventRx', function(event, fn){
    var r = rx();
    var listener = r._eventListener = function(event){
        var result = fn.call(this, r, event);
        if (result !== undefined) r.set(result);
    };
    this.addEvent(event, listener);
    return r;
});
```

With this put in place, we could add an event with `addEventRx`, just like
`.addEvent` (if you're used to MooTools) or `.on` (some other libs). The
returned value is a `rx` object.

```js
var checkboxes = $$('input[type="checkbox"]').addEventRx('click', function(r){
    return this.checked;
});
```

This code returns an array (because of `$$`) of rx objects. Those objects can
be wired into somewhere else, for example into a component that indicates if
the first two checkboxes are checked:

```js
rx().use(checkboxes, function(){
    if (Array.slice(arguments, 0, 2).every(function(checked){
        return checked;
    }) && arguments.length >= 2){
        $('result').set('text', 'Good Job!');
    } else {
        $('result').set('text', 'Try to check the first two checkboxes');
    }
});
```

The text is automatically updated if the constraint that the first two boxes
should be checked is met, or not.

Another use case could be some complex UI, like an e-mail or messages page. It
probably has some count of unread messages. However it could create a `rx`
object for that so the UI with the unread messages is automatically updated if a
message is opened.

## Should you use it?

I'm not sure if you should use the code in my own
[reactive](https://github.com/arian/reactive) repository. However there are more
sophisticated libraries:

- [RxJS](http://reactive-extensions.github.com/RxJS/)
- [Bacon.js](https://github.com/raimohanska/bacon.js)

I have not yet been able to test any of those, but I'm pretty sure those are
much more complete and battle hardened.

Furthermore I believe this concept could solve some problems in bigger
and complex (front-end) UI applications. Some module could export a `rx` object
that keeps the state of some component (say a list of messages). Another part
of the page (maybe somewhere in a toolbar), displays the number of messages.
This toolbar could grab the loose wire from the messages component, by requiring
the messages `rx` object, and wire it into the toolbar. If the state of the
messages is changed, this state is automatically propagated to the messages
count in the toolbar.
