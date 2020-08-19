---
title: "JSON.stringify does not support all types"
date: 2020-08-18T22:05:08+02:00
categories:
tags:
- javascript
- typescript
- node
keywords:
- tech
#thumbnailImage: //example.com/image.jpg
---

We all know that JSON can only contain, arrays, objects, strings, numbers, booleans and null.
But when working with javascript, it is easy to forget
that you have implicitly converted a data structure to JSON and back and in the process
actually modified the data structure.

<!--more-->

## Background

In one of the projects I am working on, we had code that handled keys for [jsonwebtoken](https://www.npmjs.com/package/jsonwebtoken) in a data structure.
The code had been working fine for a long time, but then we added support for RS256 encryption.
The RS256 support worked fine in some parts of the project, but failed with an error message

```
key must be a string or a buffer
```

in other parts of the project. This was due to JSON.stringify.

RS256 keys are passed to jsonwebtoken as a [Buffer](https://nodejs.org/api/buffer.html).
All other key types we had supported before were passed in as a string to jsonwebtoken.

## JSON.stringify a Buffer

If you pass a structure containing a Buffer to JSON.stringify, it returns it as an object:

```plaintext
> JSON.stringify( { key: Buffer.from([0]) } )
'{"key":{"type":"Buffer","data":[0]}}'
```

You can see that there is some reasonable representation of the buffer in the response.
But if you do JSON.parse on the result, you don't get a Buffer back:

```plaintext
> JSON.parse(JSON.stringify( { key: Buffer.from([0]) } ))
{ key: { type: 'Buffer', data: [ 0 ] } }
```

This is the reason for the `key must be a string or a buffer` message from jsonwebtoken.
The key used to be a Buffer, but it got passed through JSON which turned it into an object.

It is easy to convert it back to a Buffer with

```plaintext
> Buffer.from({ type: 'Buffer', data: [ 0 ] })
<Buffer 00>
```

but you have to remember to do it yourself.

## clone

One situation where data is converted to JSON and back is if you have a naive clone implementation:

```plaintext
function clone(data) {
    return JSON.parse(JSON.stringify(data))
}
```

The result of the clone operation is only identical to the input as long as
the input can be represented as JSON.

## serialization

Another situation where data can be converted to JSON and back is
if you have some sort of IPC based on JSON where different processes exchange data.
One example is the [cluster](https://nodejs.org/api/cluster.html) module.
If you send() a message between the master and a worker,
it will be serialized as JSON, which means that Buffers are converted to objects.

## typescript

If you are using typescript, you are to some extent more vulnerable to this problem
since you have learned to trust your types and they tell you everything is ok.
For the serialization problem, you have to do an explicit cast operation
when you receive the data, so it is up to you to handle it there.

Our naive clone function in typescript can look like this:

```plaintext
function clone<T>(data: T): T {
    return JSON.parse(JSON.stringify(data))
}
```

A parameterized function definition. You probably felt good when you wrote that.
The problem is that you have now learned that the definition is wrong - 
clone does not return the same type in all situations.

Fortunately, recent typescript [can express what clone does](https://effectivetypescript.com/2020/04/09/jsonify/). Replace your clone with this instead:

```plaintext
type Jsonify<T> = T extends {toJSON(): infer U}
  ? U
  : T extends object
  ? {
      [k in keyof T]: Jsonify<T[k]>;
    }
  : T;

function clone<T>(data: T): Jsonify<T> {
    return JSON.parse(JSON.stringify(data))
}
```

It will now correctly tell you what type the return value of clone has.
For simple structures, it will be identical to the input type,
but if the input contains more complex types like Buffer, Date, Map or Set,
the output type will be different.
This can help you catch bugs early.