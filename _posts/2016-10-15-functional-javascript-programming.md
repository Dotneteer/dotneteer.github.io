---
layout: post
title:  "Functional JavaScript Programming — Part #1: Introduction"
date:   2016-10-15 13:20:00 +0100
categories: javascript
abstract: >-
  Functional programming is an awesome thing. There are many functional languages out of the space, so if you want to cultivate your knowledge in an artistic level, you can turn to a significant number of them including Haskell, OCaml, F#, ML, Clojure, Erlang, Lisp, and many more.
---
Functional programming is an awesome thing. There are many functional languages out of the space, so if you want to cultivate your knowledge in an artistic level, you can turn to a significant number of them including Haskell, OCaml, F#, ML, Clojure, Erlang, Lisp, and many more. However, in this series, I will teach you the fundamentals of functional programming in JavaScript. As of this writing, JavaScript means rather ES 2015 (in its maiden name, ES6, or EcmaScript 6) than the good old ES5 — thus I will incorporate ES 2015 features into the samples and explanations whenever I can.

### Why Be Functional?

The proper use of functional programming features makes you code shorter, more expressible, and less buggy than the traditional imperative style. I cannot force you to believe this bold statement but, hopefully, you will understand it while we take this journey together.

Take a look at this short JavaScript sample (__Example-01-01__):

```javascript
let dives = [
    { site: 'Shaab El Erg', depth: 36 },
    { site: 'Thislegorm Wreck', depth: 105 },
    { site: 'Shark Reef', depth: 44 },
    { site: 'Claudia', depth: '28'},
    { site: 'Rosalie Moeller Wreck', depth: 138}
];

let deepDives = [];
for (let i = 0; i < dives.length; i++) {
    if (dives[i].depth >= 90) {
        deepDives.push(dives[i]);
    }
}

console.log(JSON.stringify(deepDives, null, 4));
```

We have an array, `dives`, which contains scuba dive log records with dive site names and depth values (given in feet). The imperative style we apply uses a for-loop to extract those dive log entries into `deepDives` where the depth was greater than or equal 90 feet.

With the help of the filter higher order function, we can eliminate the for-loop and describe the same task in a clearer way (__Example-01-02__):

```javascript
let dives = [
    { site: 'Shaab El Erg', depth: 36 },
    { site: 'Thislegorm Wreck', depth: 105 },
    { site: 'Shark Reef', depth: 44 },
    { site: 'Claudia', depth: '28'},
    { site: 'Rosalie Moeller Wreck', depth: 138}
];

let deepDives = dives.filter(function(dive) {
    return dive.depth >= 90;
});

console.log(JSON.stringify(deepDives, null, 4));
```

We passed a function to `filter` to determine whether a specific item in the array should be included in the filtered result:

```javascript
function(dive) {
    return dive.depth >= 90;
}
```

With the help of functional programming tools, we split the imperative approach of __Example-01-01__ into two functions. First, `filter` described the for-loop logic that iterates through the elements of the array and using a predicate to decide whether it needs to include the particular item in the result. Second, the anonymous function passed to `filter` declared the predicate.

If we did not have the filter function in the standard JavaScript library, we could implement it, and use the functional approach (__Example-01-03__):

```javascript
let dives = [
    { site: 'Shaab El Erg', depth: 36 },
    { site: 'Thislegorm Wreck', depth: 105 },
    { site: 'Shark Reef', depth: 44 },
    { site: 'Claudia', depth: '28'},
    { site: 'Rosalie Moeller Wreck', depth: 138}
];

function filter(array, predicate) {
    let result = [];
    for (let i = 0; i < array.length; i++) {
        if (predicate(array[i])) {
            result.push(array[i]);
        }
    }
    return result;
}

function isDeep(dive) {
    return dive.depth >= 90;
}
let deepDives = filter(dives, isDeep);

console.log(JSON.stringify(deepDives, null, 4));
```

### Divide and Conquer

Splitting an imperative task into smaller parts and applying the functional approach makes program maintenance and modification easier and shorter. Let’s assume that we need to modify our original example so that we can extract the shallow dives, too. If the shallow dive requirement came later — long after we already implemented the selection of deep dives — we might repeat the same for-loop pattern for creating a `shallowDives` array as we used earlier for deepDives (__Example-01-04__):

```javascript
let dives = [
    { site: 'Shaab El Erg', depth: 36 },
    { site: 'Thislegorm Wreck', depth: 105 },
    { site: 'Shark Reef', depth: 44 },
    { site: 'Claudia', depth: '28'},
    { site: 'Rosalie Moeller Wreck', depth: 138}
];

let deepDives = [];
for (let i = 0; i < dives.length; i++) {
    if (dives[i].depth >= 90) {
        deepDives.push(dives[i]);
    }
}

// --- Other code lines
// ...

let shallowDives = [];
for (let i = 0; i < dives.length; i++) {
    if (dives[i].depth < 90) {
        shallowDives.push(dives[i]);
    }
}

console.log(JSON.stringify(deepDives, null, 4));
console.log(JSON.stringify(shallowDives, null, 4));
```

Here, not only the double application of the for-loop is redundant, but also the second predicate function we used to create `shallowDives`. If a dive is not deep, it is shallow, and vice versa. With the functional approach, we do not need to declare two distinct predicates (__Example-01-05__):

```javascript
let dives = [
    { site: 'Shaab El Erg', depth: 36 },
    { site: 'Thislegorm Wreck', depth: 105 },
    { site: 'Shark Reef', depth: 44 },
    { site: 'Claudia', depth: '28'},
    { site: 'Rosalie Moeller Wreck', depth: 138}
];

function isDeep(dive) {
    return dive.depth >= 90;
}

let deepDives = dives.filter(isDeep);

// --- Other code lines
// ...

let shallowDives = dives.reject(isDeep);

console.log(JSON.stringify(deepDives, null, 4));
console.log(JSON.stringify(shallowDives, null, 4));
```

In contrast to `filter`, `reject` uses a predicate to decide which items to exclude from the input array. Thus, `dives.reject(isDeep)` returns those dive log entries that are not considered deep dives.

We have just barely scratched the surface of Functional JavaScript programming. In the next part, we discuss why we call filter, reject, and their companions *higher order functions*.