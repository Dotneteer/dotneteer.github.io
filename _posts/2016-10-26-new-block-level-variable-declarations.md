---
layout: post
title:  "ES 2015 — New Block-Level Variable Declarations"
date:   2016-10-26 13:20:00 +0100
categories: javascript 
abstract: >-
  In contrast to other curly brace programming languages like C, C++, Java, or C#, JavaScript shows some unusual behavior. In this
  article, you will learn about the new ES2015 block declarations.
---

In contrast to other curly brace programming languages like C, C++, Java, or C#, JavaScript shows some unusual behavior. Take a look at this code snippet:

```JavaScript
// Exercise-01-01.js

function getWiggin(childIndex) {
  if (childIndex === 1) {
    var wiggin = 'Peter';
  }
  else if (childIndex === 2) {
    var wiggin = 'Valentine';
  }
  else if (childIndex === 3) {
    wiggin = 'Ender';
  }
  return wiggin;
}

console.log("The First:", getWiggin(1));
console.log("The Second:", getWiggin(2));
console.log("The Third:", getWiggin(3));
console.log("Unknown:", getWiggin(4));
```

The multiple declarations of the `wiggin` variable in `getWiggin` suggests that the instance within the `childIndex === 1` branch is different from the other created in the `childIndex === 2` section. Your instinct whispers that both of them are visible only within their hosting blocks. Nonetheless, the function uses a single instance of `wiggin`, as the code output shows:

```
The First: Peter
The Second: Valentine
The Third: Ender
Unknown: undefined
```

### Variable Hoisting
You are deceived by a JavaScript feature, variable hoisting. While the engine parses the code, it hoists the `wiggin` variable declaration to the top of the `getWiggin` function as if you wrote this:

```javascript
// Exercise-01-02.js

function getWiggin(childIndex) {
  var wiggin;

  if (childIndex === 1) {
    wiggin = 'Peter';
  }
  else if (childIndex === 2) {
    wiggin = 'Valentine';
  }
  else if (childIndex === 3) {
    wiggin = 'Ender';
  }
  return wiggin;
}

// ...
```

### Global Variables

Variables declared in the global scope may cause surprises, too. Since a `var` declaration in the global scope creates a variable, which is a property on the global object—the `windows` object in browsers—, you can override an existing global. This action may lead to unexpected behavior. Take a look at this code:

```javascript
// Exercise-01-03.js

var Buffer = "Battle School";
console.log(window.Buffer);
```

The declaration of `Buffer` is in the global scope. Thus, assigning a string value to this variable overrides the `window.Buffer` property. You may not know that `Buffer` is a predefined function in the global scope. When the code runs in the browser, the second line of the console output—the browser writes this log item and not the exercise code—clearly indicates that redefining `Buffer` has unexpected side effects:

```
Battle School
Uncaught TypeError: e.Buffer.isBuffer is not a function
```

### For-Loop Variables

Although seasoned developers know that a for-loop variable’s scope is not constrained to the body of the loop, novices are not necessarily aware of this fact:

```javascript
// Exercise-01-04.js

var wiggins = ['Peter', 'Valentine', 'Ender'];

for (var i = 0; i < wiggins.length; i++) {
  console.log(i, wiggins[i]);
}

console.log('The loop variable is still valid:', i);
```

The last line of the code output points out that the i variable still holds its value, though the execution flow has left the block inside the for-loop:

```
0 'Peter'
1 'Valentine'
2 'Ender'
The loop variable is still valid: 3
```

This behavior cannot cause such issues as the accidental override of a global variable, but still, instinctively, you might find it controversial.

### Block-Level Declarations

Many curly-brace languages support block scopes, and so does ECMAScript 2015. You can use the `let` declaration with the same syntax as `var`, but in contrast to `var`, `let` limits the particular variable’s scope to the current code block. Rewriting the `getWiggin` function with `let` declarations would close the `wiggin` variable into code blocks:

```javascript
// Exercise-01-05.js

function getWiggin(childIndex) {
  if (childIndex === 1) {
    let wiggin = 'Peter';
  }
  else if (childIndex === 2) {
    let wiggin = 'Valentine';
  }
  else if (childIndex === 3) {
    let wiggin = 'Ender';
  }
  return wiggin;
}

console.log("The First:", getWiggin(1));
console.log("The Second:", getWiggin(2));
console.log("The Third:", getWiggin(3));
console.log("Unknown:", getWiggin(4));
```

As you run this code, the first call of `getWiggin` raises a `ReferenceError` instance telling you that `wiggin` is not defined. The three `let` declarations create three separate variables, each visible only within its hosting block—between the opening and closing curly braces of the corresponding `if`. The `return wiggin` statement references to an undeclared variable—none of the previously used instances of wiggin is visible at that point—, and this is why the JavaScript engine raises an error.

### The let Declaration

The following code uses the `commander` variable in three different `var` declaration:

```javascript
// Exercise-01-06.js

var useSpare = true;

var commander = 'Ender Wiggin';
if (useSpare) {
  var commander = 'Bean';
}

console.log(commander);

var commander = 'Petra Arcanian';
```

The `var` keyword allows you to redeclare the same variable multiple times. Due to variable hoisting, you have only one `commander` variable here, which is valid within the entire context of this sample code. This little JavaScript snippet outputs “Bean”.

With `let`, you can hide a variable in an outer block. Should you change the first two occurrences of `var` in the previous code snippet to `let`, the output would display “Ender Wiggin”:

```javascript
// Exercise-01-07.js

var useSpare = true;

let commander = 'Ender Wiggin';
if (useSpare) {
  let commander = 'Bean';
}

console.log(commander);

// var commander = 'Petra Arcanian';
```

Although the execution flow goes into the `if` block, the `commander` variable declared there is a separate one from other that sets its value to “Ender Wiggin”. While in the inner block, the second declaration hides `commander` in the outer block. As soon as the execution flow leaves the `if` block, the inner `commander` goes out of the scope. It does not hide the outer variable anymore, and thus we get a different output.

The `let` declaration prevents you from redefining the same variable in the same scope.

Observe that the third `commander` declaration in the last line is commented out. Should you uncomment this line, you would get a `SyntaxError` with a message of “Identifier ‘commander’ has already been declared”.

The `let` declaration mends the for-loop variable issue you saw in `Exercise-01-04.js`. As this code sample demonstrates, the `i` loop variable is valid only within the loop’s body:

```javascript
// Exercise-01-08.js

var wiggins = ['Peter', 'Valentine', 'Ender'];

for (let i = 0; i < wiggins.length; i++) {
  console.log(i, wiggins[i]);
}

// Raises a ReferenceError: 'i is not defined':
console.log('The loop variable is not valid:', i);
```

###No Hoisting with let

While the `var` declaration hoists the variable to the top of its declaring context—global or function—, `let` does not. The following code snippet raises an error (`ReferenceError`) with the `typeof child` condition, because, at the point where the expression is used, the `child` variable is not defined yet:

```javascript
// Exercise-01-09.js

var wiggins = ['Peter', 'Valentine', 'Ender'];

for (let i = 0; i < wiggins.length; i++) {
  console.log(typeof child);
  let child = wiggins[i];
}
```

When the JavaScript engine finds a block, it scans it for variable declarations before processing the statements within the block. If the engine finds a `var` declaration, it hoists the variable to the top of the corresponding context (function or global scope). When it finds a `let` (or `const`) declaration, it moves the variable into a temporal store—often mentioned as _Temporal Dead Zone_, or _TDZ_. While processing the block statements, any access attempt to a variable in TDZ raises a _ReferenceError_. When the engine reaches the `let` or `const` declaration, it removes the variable from TDZ, and thus excepts the subsequent references to the variable.

### The const Declaration

In tandem with `let`, ES 2015 introduces another declaration, `const`. Its syntax is similar to `let`. The JavaScript engine takes the variables declared with `const` into account as constants, and thus it does not allow assigning a new value to them:

```javascript
// Exercise-01-10.js

const army = "Dragon";
console.log("Ender's army: ", army);

army = "Salamander";
```

Because `const` forbids reassignment, the last code line raises a `TypeError` with the “Assignment to constant variable” message.

_NOTE: When you define a const, you need to initialize it immediately. If you omit the initialization, the engine raises a `SyntaxError`._

Just as `let`, `const` is a block-level declaration:

```javascript
// Exercise-01-11.js

var battleIsComing = true;

if (battleIsComing) {
  const commander = 'Ender';
  console.log(commander, 'leads us');
}

console.log(commander);
```

The last line of this code raises a `ReferenceError` because `commander` is not visible outside its declaring block, the true condition branch of `if`.

### Using const with Objects

The `const` declaration does not prevent you from changing the properties of an object that is assigned to a `const` variable:

```javascript
// Exercise-01-12.js

const petraStats = {
  name: 'Petra',
  wins: 12,
  rank: 4
};

petraStats.wins = 14;
petraStats.rank = 3;

console.log(petraStats);
```

As the code shows, you can assign new values to the properties of `petraStats`. JavaScript assigns object references to variables. Because the reference to `petraStats` does not change when you set the `wins` and `rank` properties of the object, this construct is entirely valid.

However, a new object assignment violates the `const` rule, and so the second assignment in this code raises a `TypeError`:

```javascript
// Exercise-01-13.js

const petraStats = {
  name: 'Petra',
  wins: 12,
  rank: 4
};

petraStats = {
  name: 'Petra',
  wins: 14,
  rank: 3
};

console.log(petraStats);
```

### Using const in Loops

When you apply `const` in loops, there are a few things you should be aware of. The following code snippet—in which the `child` variable is set in each iteration of the for-loop—is valid:

```javascript
// Exercise-01-14.js

var wiggins = ['Peter', 'Valentine', 'Ender'];

for (let i = 0; i < wiggins.length; i++) {
  const child = wiggins[i];
  console.log(child + ' is a Wiggin');
}
```

Because the lifetime of `child` is bound to a single iteration and its initial value is set only once in each iteration, this construct works as you expect.

You may think that you can change the `let` declaration of `i` to `const`, but this approach does not work:

```javascript
// Exercise-01-12.js

var wiggins = ['Peter', 'Valentine', 'Ender'];

for (const i = 0; i < wiggins.length; i++) {
  console.log(wiggins[i]);
}
```

After one iteration (that writes “Peter” to the output) this code raises a `TypeError` with the “Assignment to constant variable” message. The reason for this behavior is that the `i++` expression is about to change the value of `i`.

Semantically, the engine represents the for-loop as two nested blocks. The outer block contains the loop variable initialization, the condition check, and the update of the loop variable. The inner block is the body of the for-loop that executes in each iteration.

Because the loop variable is in the outer block, you cannot update it, and thus the engine raises the `TypeError` as the loop is about to carry out the second iteration.

There are two other loop constructs in JavaScript, the for-in-loop, and the for-of-loop, respectively. Interestingly, you can use `const` declaration in them. This sample demonstrates the for-in-loop:

```javascript
// Exercise-01-16.js

const petraStats = {
  name: 'Petra',
  wins: 12,
  rank: 4
};

for(const key in petraStats) {
  console.log(key, '=', petraStats[key]);
}
```

Here, the code does not reassign the key iteration variable. The scope of the for-in-loop’s iteration variable is the iteration block. Internally, the JavaScript engine initializes key to the corresponding string value in each iteration.

_NOTE: ES 2015 has a new construct, the iterator. The for-of-loop works with iterators, as you will learn about it in a future article._

### Mending For-loops with let

JavaScript closures often make a game on developers when using in a loop:

```javascript
// Exercise-01-17.js

let names = ['Salamander', 'Rat', 'Rabbit', 'Dragon']
let armyMakers = [];

populateArmyMakers();
for (var i = 0; i < armyMakers.length; i++) {
  console.log(armyMakers[i]());
}

function populateArmyMakers() {
  for (var i = 0; i < names.length; i++) {
    armyMakers.push(function() {
      return names[i] + ' army';
    })
  }
}
```

The `populateArmyMakers` function creates an array of five functions that seem to return “… army” strings. However, when you run this code, it displays “undefined army” five times. The reason for this behavior is that `i` is a captured by the army-name-creator function, and is evaluated only when the generated function runs. Each function stored in `armyMakers` uses the final value of `i` after completing the loop, namely 4. Because the length of `names` is 4, `names[4]` retrieves an undefined value.

Seasoned developers know the remedy: instead of a simple function, an immediately invoked function expression (IIFE) should be used:

```javascript
// Exercise-01-18.js

let sqrFunctions = [];

populateSqrFunctions();

for (var i = 0; i < sqrFunctions.length; i++) {
  console.log(sqrFunctions[i]());
}

function populateSqrFunctions() {
  for (var i = 0; i < 5; i++) {
    sqrFunctions.push(
      (function(n) {
        return function() {
          return n*n
        };
      })(i));
  }
}
```

As this short code snippet shows, IIFEs are not easy to read. There are too many braces and parentheses that make it difficult to follow the code.

The `let` declaration makes this hocus-pocus unnecessary. When you run this code, the output displays the expected army names:

```javascript
// Exercise-01-19.js

let names = ['Salamander', 'Rat', 'Rabbit', 'Dragon']
let armyMakers = [];

populateArmyMakers();
for (var i = 0; i < armyMakers.length; i++) {
  console.log(armyMakers[i]());
}

function populateArmyMakers() {
  for (let i = 0; i < names.length; i++) {
    armyMakers.push(function() {
      return names[i] + ' army';
    })
  }
}
```

Because the scope of `i` is the for-loop, the engine creates a clone of `i` in each iteration, and the newly created function captures this cloned value. As `i` changes, so does the value of the captured clone. Due to the `let` declaration, the output of this code is:

```
Salamander army
Rat army
Rabbit army
Dragon army
```

### Functions and the TDZ

Function hoisting and block-level variables sometimes may deceive you. For example, you may think the following code snippet displays “Ender”:

```javascript
// Exercise-01-20.js

function setCommander() {
  commander = 'Bean'
}

let commander = 'Ender';
setCommander();
console.log('The commander is', commander);
```

However, it displays “Bean”. After dealing so much with `let` and block-level variable declarations, you unconsciously may think the commander variable used within the `setCommander` function is a local—and a block-level—variable. So, setting it to “Bean” does not affect `commander` declared with let right after the function.

Well, `commander` used in the function is the same variable declared outside of it. Do not let it deceive you!

When the JavaScript engine parses this code, it first scans for variables and then hoists functions. By the time `setCommander` is analyzed, the engine knows `commander`. During code execution, when the flow reaches the `let` declaration, `commander` is removed from the TDZ. The invocation of `setCommander` assigns “Bean” to the variable.

_HINT: Try what output this code snippet produces when you change the lines of variable declaration and `setCommander()` invocation._

### How to Move from var to let and const

With ECMAScript 2015, the `let` declaration works as `var` should have always behaved. Thus, it seems to be a good idea to change all `var` declarations to `let` as you adopt ES2015. Nonetheless, due to the semantic differences between `var` and `let` may prevent your modified code to work after changes.

Provided, you have automated tests, you can mitigate this risk. However, if you feel that the test coverage is not sufficient, be careful with such changes. Change `var` to `let` in small steps and check that the affected code still works.

There is another—even bolder—approach. When you face with a `var` declaration that has an initialization expression, change it to `const`. If the code reassigns a value to it, you will get a `TypeError` with the “Assignment to constant variable” message. Should you get this message, you would know that you cannot apply `const` but `let`.

Many developers of the ES 2015 camp have already tried this approach. The feedback from JavaScript communities points out that this practice is viable and useful. Most variables do not change their values after the initial assignment, thus using `const` can be an excellent tool to prevent unexpected changes that would lead to bugs.

### Summary

The new `let` and `const` declarations allow you to create variables with a block-level scope. In contrast to `var`, these do not hoist variable declarations to the top of their scope (function or global), and thus eliminate a couple of annoying behaviors of `var`.

With `const`, you declare variables that are assigned to an initial value only. When you try to reassign them to a new value, the JavaScript engine raises a `TypeError`. Be careful with object values: `const` does not prevent you from changing a property of a const-assigned object.

With `let` and `const`, you can declare your variables in the narrowest scope—the innermost block—where you need them, and it may help you avoid unexpected features and bugs coming from the floppy behavior of `var`.
