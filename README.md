# Orthogonal Class Member Syntax
By Mark S. Miller (@erights) and Allen Wirfs-Brock (@allenwb)

**ECMAScript Class Syntax: A Proposal to clarify orthogonal concerns**
## Summary
[Various](http://tc39.github.io/proposal-decorators/) [distinct](https://tc39.github.io/proposal-class-public-fields/) [proposals](https://github.com/tc39/proposal-private-fields) suggest syntax and semantics for new [*ClassBody*](https://tc39.github.io/ecma262/#prod-ClassBody) elements. However, up to now, there has not been any guidance for choosing the syntax for new *ClassBody* extensions. This document presents a common syntactic model for *ClassBody* member elements.  This model can be used to align syntactic design decisions in existing and future proposals.

A "class member" is an object property or private field that is defined as part of a *ClassBody*.  We define class members in terms of three orthogonal dimensions:   

  1. Visibility: Public property or a private field?
  1. Kind: A variable or a method?
  1. Placement: Is the member part of the class constructor, the prototype, or each class instances?

Each of these dimensions is encoded in the syntactic structure of a [*ClassElement*](https://tc39.github.io/ecma262/#prod-ClassElement).   **Visibility** is encoded by the syntax of the member name: either a [PropertyName](https://tc39.github.io/ecma262/#prod-PropertyName) or a `#`[*BindingIdentifier*](https://tc39.github.io/ecma262/#prod-BindingIdentifier) sequence designating a private field name.  **Kind** is encoded by using either the syntax of an initialized binding or the syntax of a [MethodDefinition](https://tc39.github.io/ecma262/#prod-MethodDefinition).     **Placement** is encoding by the initial keyword of the *ClassElement*: `static`prefixes members of the class constructor,  `own`prefixes instance object members , and the absence of a keyword indicates a member of the prototype object.

A *ClassElement* is either a single *MethodDefinition* or a list of initialized bindings.

```js
//A kitchen sink example
class Foo {
  //instance members
  own x=0, y=0;  // two data properties
  own #secret;   // a private field
                 // initial value undefined
  own *[Symbol.iterator](){yield this.#secret}
                 // a generator method
  own #callback(){}  //a private instance method  
  //class constructor members               
  static #p=new Set(), q=Foo.#p;
                // a private field and a property
                // of the class constructor                     
  static get p(){return Foo.#p} //accessor method     
  //prototype methods                
  setCallback(f){this.#callback=f}
  constructor(s){
     this.#secret = s;
  }
}
```

The most important part of this proposal is the concept of syntactic orthogonality. The key idea is that instead of arbitrarily assigning syntax to various _ClassElement_ constructs, there is a set of simple orthogonal  syntactic units that represent composable semantic concepts. If a JS programmer understand what `static`, #, and the binding list syntactic forms mean then they will understand what `static #foo; ` or `own #bar;` means even if they haven’t specifically been taught about private fields on constructors or own private instance fields.  Orthogonality does introduce some syntactic forms that have limited utility. But that’s ok as the orthogonal consistently makes such rarely used cases understandable. The important thing is that all of the allowed cases obey the non-surprising orthogonality meaning of the syntactic tokens. A few special case restrictions are ok, but too many would loose the orthogonality and degenerates into a big bag of arbitrary rules.

## Details

### Non-Orthogonal Goals
  1. Compatible with existing ECMAScript class definition syntax
  1. Express private fields and public properties
  1. Easy to understand
  1. Hard to misunderstand
  1. Initialization syntax that does not invite confusion with assignment
  1. Enable an annotation to annotate many fields and/or properties together
  1. Express private methods and private static methods
  1. Be understandable in a "shallow wide" sense
  1. Invite understanding to become "deeper and narrower"
  1. Don't cut off sensible choices without good reason
  1. Avoid combinatorial explosion of ad hoc syntaxes to express many sensible choices
  1. Preserve investment in existing proposals -- stay similar to them
  1. Avoid semantic confusion arising from semantics of `private`/`public` in other langauges that use a similar syntax for class definitions.

### Orthogonal Dimensions of Declarative Class State

#### Placement: Is the member part of the class constructor, prototype, or class instances?

##### static

If the initial keyword of a <i>ClassElement</i> is **`static`** then the element defines members  on  the class/constructor. This is compatible with existing proposed uses of **`static`**.

##### own

If the initial keyword of a <i>ClassElement</i> is **`own`** then the element defines members on the instance objects created by the class constructor. This differs from current proposals for initializing state on the instances. Besides the general benefits of orthogonality, using **`own`** for this has specific advantages over these proposals without **`own`**:
 1. The syntax `x = 9;` looks like an assignment and will be misunderstood as an assignment. The syntax `own x = 9;` looks like a variable declaration and initialization, making DefineProperty semantics less surprising.
 1. Grouping multiple declarations enables them to be annotated together. However, several people said they find the syntax `x = 9, y, z = 10;` confusing in this position. The syntax `own x = 9, y, z = 10;` is not surprising, again, because it follows the existing syntactic pattern of variable declarations and initializations.
 1. The syntax supports the definition of method forms on instance objects.     


##### Neither static nor own

In the absence of either the **`static`** or **`own`** initial keywords, the element defines members on the prototype object. When used with <i>MethodDefinition</i> class elements, this is compatible with existing syntax.

Used with the binding list form of class elements , this would recreate the confusions we just enumerated above with `x = 9, y, z = 10;` together with the further confusion that this would be initialized on the prototype. Also while the conciseness of no leading placement keyword makes sense for prototype method definitions (which are the most commonly used  form of <i>ClassElement</i>) it makes little sense for rarely needed prototype data properties or prototype level private fields. Instead, we disallow the binding list forms of class elements that do not have either a **`static`** or **`own`** initial keyword. For those writing code, this violation of orthogonality may be a rude surprise. But for those reading code which is not statically rejected, there is no such violation or rude surprise.

For reasons discussed below, we also disallow prototype placement of private methods.

#### Visibility: Public property vs private field

In the existing [private state proposal](https://github.com/tc39/proposal-private-fields) a class element prefixed with a  **`#`** sigil defines a private field of a class instance.  The existing private state proposal postpones the issue of how one would express private instance methods, private prototype methods or private static methods. In this proposal any class element that defines a member name that is prefixed with a  **`#`** sigil defines a private field. The instance, prototype, or class/constructor placement is orthogonally determined based upon the placement keyword.  `static` *MethodDefinition*, where the member name in the *MethodDefinition* is prefixed with **`#`**, is a method accessed as a private field of the class/constructor object, i.e., the method is the initial value of a private field. Similarly,`own` *MethodDefinition* containing a **`#`** sigil is a method accessed as a private field of instance objects created by the class/constructor. Each such private method field of instance objects is initialized with a new function object as part of the instance initialization process. (Or perhaps the initial function objects are determined at class definition time and initially shared by all instances. That decision is in the realm of the private state proposal.)

In the WeakMap-like way of defining private state, these additional cases are specified the same way: the private names are in scope over the same body of code. But rather than using instances as the keys of the weakmap-like collection named by those names, the key would be either the prototype or the class/constructor. Of course, implementations can implement by any means that is not observably different from that.

#### Kind: A variable or a method?

Each member is defined either via a *MemberDefinition* or as an element of a member binding list.  While both sorts of members are normally mutable we characterize members defined via binding lists as "variables" because they are typically used to store state rather than to invoke object behavior.

A *MethodDefinition* can define a normal method, a generator method, an async function method, an accessor, or the constructor. Accessors members  are not about the initial value of the member, but rather about the nature of the member itself. This only makes sense for properties. A private accessor field is not meaningful and hence is not allowed.

### The constructor method  

_Preliminary: There are known issues in this section but they don't impact the rest of the proposal_

The normal constructor declaration serves two purposes: To define the behavior of the class/constructor object and to initialize the `"constructor"` property of the prototype to point at this class/constructor. If treated orthogonally, we would continue to have a MethodDefinition for the name `"constructor"` define the behavior of the class/constructor itself. However, what objects are initialized to point at this constructor would be determined orthogonally. Again most of these choices may rarely be useful but there is no reason to violate orthogonality to surprising disallow them. In addition, one case is hugely useful:

Classes will often be defined where the authority provided by holding an instance of a class should not necessarily confer the authority to invoke the constructor and make new instances. This came up most recently with WeakRefs but naturally occurs in many places. I have written much Java code whose correctness depends on the instances providing less power than their class object provides. In this design, if the `"constructor"` method definition is made static, then it is initialized to point at itself. If made private, no matter where it is placed, only code inside the class can follow this pointer back to the class/constructor. Clients of the instances have no such access and so cannot so navigate.

### No Private Methods on Prototypes
A "private method" is simply a private field that is initialized using a method definition. The ability to define a private methods on a class prototype initially sounds like useful functionality. However, when the proposed private field access semantics is examined, prototype placement of private methods turns out to be pretty useless. The basic problem is that  there is no convenient way to reference such methods as instance objects don't inherit access to private fields placed on the prototype. Consider:

```js
class X {
  #helper() {};  //private method on prototype
  leader() {
    this.#helper(); //error because instances don't have a #helper field
    #helper();      //error because this means the same as  this.#helper()
    this.__proto__.#helper();  //error if invoked on a subclass instance
    X.prototype.#helper();    // works (assuming no rewiring) but ugly
                  //better to use a static private method:
                  //static #helper(self) {};
                  // X.#helper(this);
  }
}
```
Class scoped lexical function declarations appear to be a better solution for most private method on prototype use cases. These will be presented in a separate proposal.

### Summary of Disallowed Member Definitions

A member definition is either a element of a data member binding list or one of the <i>MethodDefinition</i> forms. Most of the orthogonal combinations of method placement and visibility permit the use of any member definition form.  The forms that are syntactically disallowed are summarized in the following table.


position/visibility| public property | private field |
-------------------|-----------------|---------------|
<b>own</b> | | ~~accessor method~~|
<b>static</b> | | ~~accessor method~~
<b>(prototype)</b> | ~~data property~~ | ~~data field~~<br>~~<i>MethodDefinition</i>~~|

###Proposed Syntax

The following is the proposed syntactic changes introduced by this proposal. Grammar parameter which are not directly relevant to this proposal have been elided and will have to be reintroduced for the final specification text.

#### [14.5 Class Definition](https://tc39.github.io/ecma262/#sec-class-definitions) Changes

<pre>
<i>ClassElement</i> :
&nbsp;&nbsp;&nbsp;<i>MemberElement</i>
&nbsp;&nbsp;&nbsp;<i>Annotation</i> <i>MemberElement</i>
&nbsp;&nbsp;&nbsp;<b>;</b>
</pre>
*Annotation* is defined by the [Decorators Proposal](http://tc39.github.io/proposal-decorators/).

<pre>
<i>MemberElement</i> :
&nbsp;&nbsp;&nbsp;<i>ConstructorElement</i>
&nbsp;&nbsp;&nbsp;<i>PrototypeElement</i>
&nbsp;&nbsp;&nbsp;<i>InstanceElement</i>

<i>ConstructorElement</i> :
  &nbsp;&nbsp;&nbsp;<b>static</b> <i>MemberBindingList</i> <b>;</b>
  &nbsp;&nbsp;&nbsp;<b>static</b> <i>MethodDefinition</i><sub>[Private]</sub>

<i>PrototypeElement</i> :
  &nbsp;&nbsp;&nbsp;<i>MethodDefinition</i>

  <i>InstanceElement</i> :
  &nbsp;&nbsp;&nbsp;<b>own</b> <i>MemberBindingList</i> <b>;</b>
  &nbsp;&nbsp;&nbsp;<b>own</b> <i>MethodDefinition</i><sub>[Private]</sub>

 <i>MemberBindingList</i> :
  &nbsp;&nbsp;&nbsp;<i>MemberBinding</i><sub>[Private]</sub>
 &nbsp;&nbsp;&nbsp;<i>MemberBindingList</i> <b>, </b> <i>MemberBinding</i><sub>[Private]</sub>

<i>MemberBinding</i><sub>[Private]</sub> :
 &nbsp;&nbsp;&nbsp;<i>PropertyName</i>  <i>Initializer</i><sub>opt</sub>
 &nbsp;&nbsp;&nbsp;<small>[+Private]</small> <i>PrivateBindingIdentifier</i>  <i>Initializer</i><sub>opt</sub>

<i>PrivateBindingIdentifier</i> :
 &nbsp;&nbsp;&nbsp;<b>#</b> <i>BindingIdentifier</i>
 </pre>

#### [14.3 Method Definitions](https://tc39.github.io/ecma262/#sec-method-definitions) Changes
<pre>
<i>MethodDefinition</i><sub>[Private]</sub> :
&nbsp;&nbsp;&nbsp;<i>MethodName</i><sub>[?Private]</sub><b> ( </b><i>UniqueFormalParameters</i><b> ) { </b><i>FunctionBody</i><b> }</b>
&nbsp;&nbsp;&nbsp;<i>GeneratorMethod</i><sub>[?Private]</sub>
&nbsp;&nbsp;&nbsp;<i>AsyncMethod</i><sub>[?Private]</sub>
&nbsp;&nbsp;&nbsp;<b>get</b> <i>PropertyName</i><b> ( ) {</b> <i>FunctionBody</i><b> }</b>
&nbsp;&nbsp;&nbsp;<b>set</b> <i>PropertyName</i><b> ( </b><i>PropertySetParameterList</i><b> ) { </b><i>FunctionBody</i><b> }</b>

<i>MethodName</i><sub>[Private]</sub> :
 &nbsp;&nbsp;&nbsp;<i>PropertyName</i>   
 &nbsp;&nbsp;&nbsp;<small>[+Private]</small> <i>PrivateBindingIdentifier</i>   
</pre>

#### [14.4 Generator Function Definitions](https://tc39.github.io/ecma262/#sec-generator-function-definitions) Changes
<pre>
<i>GeneratorMethod</i><sub>[Private]</sub>:
&nbsp;&nbsp;&nbsp;<b>*</b>  <i>MethodName</i><sub>[?Private]</sub><b> ( </b><i>UniqueFormalParameters</i><b> ) { </b><i>GeneratorBody</i><b> }</b>
</pre>

#### [14.6 Async Function Definitions](https://tc39.github.io/ecma262/#sec-async-function-definitions) Changes
<pre>
 <i>AsyncMethod</i><sub>[Private]</sub>:
&nbsp;&nbsp;&nbsp;<b>async</b> <small>[no <i>LineTerminator</i> here]</small> <i>MethodName</i><sub>[?Private]</sub><b> ( </b><i>UniqueFormalParameters</i><b> ) { </b><i>AsyncFunctionBody</i><b> }</b>

</pre>
