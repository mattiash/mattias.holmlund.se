---
title: "Waiting for stuff in tests"
date: 2020-09-06T17:10:00+0200
categories:
tags:
- javascript
- typescript
- node
keywords:
- tech
#thumbnailImage: //example.com/image.jpg
---

When writing tests, you sometimes need to wait for something to happen.
This article describes different methods of doing this, including a new
tryUntil() construct available in [purple-tape](https://www.npmjs.com/package/purple-tape).

<!--more-->

## fake time

In many cases, you can use [sinon](https://www.npmjs.com/package/sinon)
to [fake the progress of time](https://sinonjs.org/releases/latest/fake-timers/).

This is useful if what you are waiting for is an interval or timeout in the code that you are testing.
By faking the progress of time, you can make your test run fast even though it has to wait
for a long timeout.

If the thing that you are waiting for happens in a separate process (such as an external web-server, database or other system), then Sinon cannot be used since it cannot influence the time it takes for the other process to perform the necessary action.

## sleep

The most straight-forward method to wait for an external event is to sleep in the test:

```plaintext
await sleep(2000)
t.true(await checkCondition(), 'condition shall be true')
```

This works if you know the maximum amount of time that you have to wait for the condition to be true.
It however has the unfortunate side-effect of making your test take longer to execute.
It also means that you have to sleep for the longest possible time that it may take for the condition to be true
every time you run the test.
For example, if it may take between 1 and 30 seconds before the condition is true,
you always have to wait for 30 seconds.
This can slow down your tests a lot if the longest possible time is long.


## check condition repeatedly

If you know that the time it takes for the the condition to be true varies,
you can check the condition repeatedly:

```plaintext
let success = false

for(let n=0; n<10; n++) {
    if(await checkCondition()) {
        success = true
        break
    }
    else {
        await sleep(1000)
    }
}

t.true(success, 'condition shall be true')
```

This method is however slightly inelegant.
It requires a lot of code, and you have to check the condition in one place (`if(...)`)
but do the test-assertion in another place (`t.true(...)`)

## tryUntil

To allow you to write your tests in a more elegant way,
purple-tape has a `tryUntil()` construct to let you run code over and over again
while waiting for something:

```plaintext
t.tryUntil( async () => {
    t.true(await checkCondition(), 'condition shall be true')
}, 30000)
```

The first parameter for the tryUntil method is the function that shall be run.
The test is regarded as a success if it runs at least one assertion method (ok, equal, pass etc)
and all assertions return an ok status.

tryUntil will run the supplied function over and over again until the all the tests in the supplied function pass, or until 30 seconds has passed.

The output from the test will only the result of the last invocation of the test-function which is the one where all assertions passed:

```plaintext
ok 1 condition shall be true
```

or the last failed run if the timeout occurs:

```plaintext
not ok 1 condition shall be true
```

tryUntil passes a single parameter to the function which is the t-object itself.
The following code has the exact same result as the previous example:

```plaintext
async function testFn(t) {
    t.true(await checkCondition(), 'condition shall be true')
}

test( 'Test', async (t) => {
    t.tryUntil( testFn, 30000)
})
```

tryUntil takes a third, optional parameter that decides how long the test-code shall sleep between attempts. It defaults to a "reasonable" time that is dependent on the timeout value.
