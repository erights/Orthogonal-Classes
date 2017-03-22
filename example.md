
Orthogonal Class Member Syntax examples derived from [Public Field proposal examples](https://github.com/tc39/proposal-class-public-fields/issues/46#issuecomment-239031422)



Basic example:

``` js

class Counter {
  own count = 0;

  increment() {
    ++this.count;
  }
}

let counter = new Counter;
counter.increment();
console.log(counter.count); // 1
```
In all of examples in this document any _IdentifierName_ used to define a property using a `own` or `static` field declaration can be replace with a private field name. However, such private fields can not be directly accessed from outside the containing class declaration. For example, the above basic example can be rewritten using private fields as:

``` js

class Counter {
  own #count = 0;

  increment() {
    ++this.#count;
  }
}

let counter = new Counter;
counter.increment();
console.log(counter.#count); // Early Error: #count not within a class definition
```

Lots of syntax examples:

``` js
// Basic syntax

class C {
  own a = 0;
}

class C {
  own a;
}

class C {
  static a = 0;
}

class C {
  static a;
}


// ASI examples

class C {
  own = 0
  b(){}
}

class C {
  own a
  own b
}

//or

class C {
  own a, b
}

class C {
  own a
  *b(){}
}

class C {
  own a
  ['b'](){}
}

class C { // a property named 'a' and a property named 'get'
  own a
  own get
}

//or

class C { // a property named 'a' and a property named 'get'
  own a. get
}

class C { // a single static property named 'a'
  static
  a
}

class C { // a property named 'a' and a property named 'static'
  own a
  own static
}

//or

class C { // a property named 'a' and a property named 'static'
  own a, static
}


// ASI non-examples / errors

class C { own a b } // There is no line terminator after 'a', so ASI does not occur

class C { // '*' may follow 0 in an AssignmentExpression, so no ASI occurs after 0
  own a = 0
  *b(){}
}

class C { // '[' may follow 0 in an AssignmentExpression, so no ASI occurs after 0
  own a = 0
  ['b'](){}
}

class C { // 'a' may follow 'get' in a MethodDefinition, so no ASI occurs after 'get'
  own get
  a
}


// Non-examples

class C { // a getter for 'a' installed on C.prototype; this is existing syntax
  get  
  a(){}
}
```


Examples, not in current public field examples

```js
class C { //multiple public fields on one line
  own a=1, b, c=3;
  // a=1, b; c=3;  //with current public fields
}

//own and static declarations can define both public and private fields
class C { //multiple public and private fields on one line
  own a=1, #b, c=3, #d=4;
}

class C { //multiple static fields on one line
  static a=1, b, c=3;
  // static a=1, static b; static c=3;  //with current public fields
}

class C { //numeric property names and ADI
  own 0
  own 1
  own 2
  own length=3
//or
  own 0, 1, 2, length=3
}

class C { //per instance accessor properties accessor properties
  get p() {return getSomethingExpensiveToCompute(this)};
  own get p() {
    Object.defineOwnProperty(this."p",{value: super.p, configurable:true});
    return this.p
  };
  own set p(v) {Object.defineOwnProperty(this."p",{value: v,configurable:true}});}
}

class C {//decorating multiple fields}
  @nonenumerable @nonconfigurable
  own a=1, b=2, c=3;

// compare to:
//
// @nonenumerable @nonconfigurable
// a=1;  
// @nonenumerable @nonconfigurable
// b=2;  
// @nonenumerable @nonconfigurable
// c=3;  
}
```
