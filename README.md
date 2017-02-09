# Orthogonal Class Member Syntax 
Proposed EcmaScript Class Syntax clarifying orthogonal concerns
## Summary
Various distinct proposals suggest syntaxes and semantics for new [*ClassBody*](https://tc39.github.io/ecma262/#prod-ClassBody) elements. However, up to now there has not been any guidance for choosing the syntax for new *ClassBody* extensions. This document presents a common syntactic model for *ClassBody* member elements.  This model can be used to align syntatic design decisions in existing and future proposals. 

A "class member" is an object property or private field that is defined as part of a *ClassBody*.  We define class members in terms of three orthogonal dimensions:   

  1. Visibility: Public property vs private field
  1. Kind: A variable or a method?
  1. Placement: Is the member part of the class constructor, prototype, or class instances?

Each of these diminsions is encoded in the syntactic structure of a [*ClassElement*](https://tc39.github.io/ecma262/#prod-ClassElement).   **Visibility** is encoded by the syntax of the member name: either a [PropertyName](https://tc39.github.io/ecma262/#prod-PropertyName) or a `#`[*BindingIdentifier*](https://tc39.github.io/ecma262/#prod-BindingIdentifier) sequence designating a private field name.  **Kind** is encoded by using either the syntax of an initialized binding or the syntax of a [MethodDefinition](https://tc39.github.io/ecma262/#prod-MethodDefinition).     **Placement** is encoding by the initial keyword of the *ClassElement*: `static`prefixes members of the class constructor,  `own`prefixes instance object members , and the absence of a keyword indicates a member of the prototype object.

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
  static get p(){return this.#p} //accessor method     
  //prototype methods                
  setCallback(f){this.#callback=f}
  constructor(s){
     this.#secret = s;
  }
}
```
##Details
                    
### Non-Orthogonal Goals
  1. Compatible with existing EcmaScript classes
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
  
### Orthogonal Dimensions of Declarative Class State
  
#### Placement: Is the member part of the class constructor, prototype, or class instances?

##### static

If the initial keyword of a <i>ClassElement</i> is **`static`** then the element defines members  on  the class/constructor. This is compatible with existing proposed uses of **`static`**.

##### own

If the initial keyword of a <i>ClassElement</i> is **`own`** then the element defines members on the instance objects created by the class constructor. This differs from current proposals for initializing state on the instances. Besides the general benefits of orthogonality, using **`own`** for this has specific advantages over these proposals without **`own`**:
 1. The syntax `x = 9;` looks like an assignment and will be misunderstood as an assignment. The syntax `own x = 9;` looks like a variable declaration and initialization, making DefineProperty semantics less surprising.
  2. Grouping multiple declarations enables them to be annotated together. However, several people said they find the syntax `x = 9, y, z = 10;` confusing in this position. The syntax `own x = 9, y, z = 10;` is not surprising, again, because it follows the existing syntactic pattern of variable declarations and initializations.
  3. The syntax supports the definition of method forms on instance objects.     


##### Neither static nor own

In the absence of either the **`static`** or **`own`** initial keywords, the element defines members on the prototype object. When used with <i>MethodDefinition</i> class elements, this is compatible with existing syntax. 

Used with the binding list form of class elements , this would recreate the confusions we just enumerated above with `x = 9, y, z = 10;` together with the further confusion that this would be initialized on the prototype. Also while the conciseness of no leading placement keyword makes sense for prototype method definitions (which are the most commonly used  form of <i>ClassElement</i>) it makes little sense for rarely needed prototype data properties or prototype level private fields. Instead, we disallow the binding list forms of class elements that do not have either a **`static`** or **`own`** initial keyword. For those writing code, this violation of orthogonality may be a rude surprise. But for those reading code which is not statically rejected, there is no such violation or rude surprise.

For reasons discussebelow, we also disallow prototype placement of private methods.

#### Visibility: Public property vs private field

The presence of the **`#`** sigil prefixing a member name in a class element defines a private field. The existing private state proposal uses this specifically for a private instance field, which we would write `own # ...`. The existing private state proposal postpones the issue of how one would express private instance methods, private prototype methods or private static methods. Here, the instance and static cases simply falls out of using the `static` and `own`placement keyword. `static # MethodDefinition` is a method accessed as a private field of the class/constructor object, i.e., the method is the initial value of a private field. Similarly,`own # MethodDefinition` is a method accessed as a private field of instance objects created by the class/constructor. Each such private method field of instance objects is initialized with a new function object as part of the instance initialization process. 

In the WeakMap-like way of defining private state, these additional cases are specified the same way: the private names are in scope over the same body of code. But rather than using instances as the keys of the weakmap-like collection named by those names, the key would be either the prototype or the class/constructor. Of course, implementations can implement by any means that is not observably different from that.

#### Kind: A variable or a method?

Each member is defined either via a *MemberDefinition* or as an element of a member binding list.  While both sorts of members are normally mutable we characterize members defined via binding lists as "variables" because they are typically used to store state rather than to invoke object behavior.

A *MethodDefinition* can define a normal method, a generator method, an async function method, an accessor, or the constructor. Accessors members  are not about the initial value of the member, but rather about the nature of the member itself. This only makes sense for properties. A private accessor field is not meaningful and hence is not allowed. 

### The constructor method  
The normal constructor declaration serves two purposes: To define the behavior of the class/constructor object and to initialize the `"constructor"` property of the prototype to point at this class/constructor. If treated orthogonally, we would continue to have a MethodDefinition for the name `"constructor"` define the behavior of the class/constructor itself. However, what objects are initialized to point at this constructor would be determined orthogonally. Again most of these choices may rarely be useful but there is no reason to violate orthogonality to surprising disallow them. In addition, one case is hugely useful:

Classes will often be defined where the authority provided by holding an instance of a class should not necessarily confer the authority to invoke the constructor and make new instances. This came up most recently with WeakRefs but naturally occurs in many places. I have written much Java code whose correctness depends on the instances providing less power than their class object provides. In this design, if the `"constructor"` method definition is made static, then it is initialized to point at itself. If made private, no matter where it is placed, only code inside the class can follow this pointer back to the class/constructor. Clients of the instances have no such access and so cannot so navigate.
  
###Proposal

Starting with [Grammar Summary, Functions and 
Classes](https://www.ecma-international.org/ecma-262/7.0/#sec-functions-and-classes):

<code>
<i>ClassElement</i> :
 &nbsp;&nbsp;&nbsp;<i>MemberElement</i>
&nbsp;&nbsp;&nbsp;<i>Annotation</i> <i>MemberElement</i> 
&nbsp;&nbsp;&nbsp;<b>;</b>

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
 &nbsp;&nbsp;&nbsp;<small>[+Private]</small> <i>PrivateBindingIdentifer</i>  <i>Initializer</i><sub>opt</sub> 

<i>PrivateBindingIdentifer</i> : 
 &nbsp;&nbsp;&nbsp;<b>#</b> <i>BindingIdentifer</i>  
 
<i>MethodDefinition</i><sub>[Private]</sub> :
&nbsp;&nbsp;&nbsp;<i>MethodName</i><sub>[?Private]</sub><b> ( </b><i>UniqueFormalParameters</i><b> ) { </b><i>FunctionBody</i><b> }</b>
&nbsp;&nbsp;&nbsp;<i>GeneratorMethod</i><sub>[?Private]</sub>
&nbsp;&nbsp;&nbsp;<i>AsyncMethod</i><sub>[?Private]</sub>
&nbsp;&nbsp;&nbsp;<b>get</b> <i>PropertyName</i><b> ( ) {</b> <i>FunctionBody</i><b> }</b>
&nbsp;&nbsp;&nbsp;<b>set</b> <i>PropertyName</i><b> ( </b><i>PropertySetParameterList</i><b> ) { </b><i>FunctionBody</i><b> }</b>

<i>MethodName</i><sub>[Private]</sub> : 
 &nbsp;&nbsp;&nbsp;<i>PropertyName</i>   
 &nbsp;&nbsp;&nbsp;<small>[+Private]</small> <i>PrivateBindingIdentifer</i>   

<i>GeneratorMethod</i><sub>[Private]</sub>:
&nbsp;&nbsp;&nbsp;<b>*</b>  <i>MethodName</i><sub>[?Private]</sub><b> ( </b><i>UniqueFormalParameters</i><b> ) { </b><i>GeneratorBody</i><b> }</b>

 <i>AsyncMethod</i><sub>[Private]</sub>:
&nbsp;&nbsp;&nbsp;<b>async</b> <small>[no <i>LineTerminator</i> here]</small> <i>MethodName</i><sub>[?Private]</sub><b> ( </b><i>UniqueFormalParameters</i><b> ) { </b><i>AsyncFunctionBody</i><b> }</b>


  </code>


This syntax directly expresses the desired orthogonality. However, in the orthogonal matrix, there are some cases that, if allowed, would violate some of our goals. So the actual proposal sacrifices orthogonality in banning these cases. The above grammar is therefore a cover grammer where the rejection is the post-parse check. The rejected cases are

```
Empty ElementVisibility BindingList ";"
ElementPlacement # get PropertyName "(" ")" "{" FunctionBody "}"
ElementPlacement # set PropertyName "(" PropertySetParameterList ")" "{" FunctionBody "}"
```
Aside from these rejections, the rest of the proposal preserves the remaining orthogonality.




### Open questions

#### Distributing Visibility over BindingList

The above grammar has all elements of a BindingList share one Placement and one Visibility. For Placement, the syntax looks intuitive. For visibility, `own # x = 9, y = 10;` can be misunderstood as declaring a private `#x` field and a public `y` property. It seems that one possibility would be to change the grammar so that this is indeed allowed but would actually have that meaning. IOW, the Visibility sigil would be present or absent separately for each member of a BindingList. However, this would be hostile to sharing an annotation across all the members of one BindingList. Instead, we can have the sigil per element, but require that all elements of the same BindingList have the same visibility. Either all have the sigil and are private fields, or none have the sigil and are public properties. As with the other restrictions above, this may surprise writers with rejections they do not expect, but will minimize misundertanding of those reading code that was not statically rejected.
```
Aside from these rejections, the rest of the proposal preserves the remaining orthogonality.


### Placement

#### static

If the initial keyword is **`static`** then the declaration initializes state on  the class/constructor. This is compatible with existing proposed uses of **`static`**.

#### own

If the initial keyword is **`own`** then the declaration initializes state on the instance. This differs from current proposals for initializing state on the instance. Besides the general benefits of orthogonality, using **`own`** for this has specific advantages over these proposals without **`own`**:
  1. The syntax `x = 9;` looks like an assignment and will be misunderstood as an assignment. The syntax `own x = 9;` looks like a variable declaration and initialization, making DefineProperty semantics less surprising.
  1. Grouping multiple declarations enables them to be annotated together. However, several people said they find the syntax `x = 9, y, z = 10;` confusing in this position. The syntax `own x = 9, y, z = 10;` is not surprising, again, because it follows the existing syntactic pattern of variable declarations and initializations.

By orthogonality `own MethodDefinition` defines a method on the instance. This might rarely be useful, but banning it would be surprising.

#### Neither static nor own

In the absence of either the **`static`** or **`own`** initial keywords, the declaration initializes state on the prototype. When coupled with MethodDefinition, this is compatible with existing syntax. 

Coupled with the BindingList form, this would recreate the confusions we just enumerated above with `x = 9, y, z = 10;` together with the further confusion that this would be initialized on the prototype. Instead, we statically reject this one case. For those writing code, this violation of orthogonality can be a rude surprise. But for those reading code which is not statically rejected, there is no such violation or rude surprise.

### Visibility

The presence of the **`#`** sigil indicates a private field. The existing private state proposal uses this specifically for a private instance field, which we would write `own # ...`. The existing proposal postpones the issue of how one would express private methods or private static methods. Here, those other cases simply falls out. `# MethodDefinition` is a private method definition, i.e., the method is the initial value of a private field of that name on the prototype. `static # MethodDefinition` defines a private static method, i.e., the method is the initial value of a private field on the class/constructor.

In the WeakMap-like way of defining private state, these additional cases are specified the same way: the private names are in scope over the same body of code. But rather than using instances as the keys of the weakmap-like collection named by those names, the key would be either the prototype or the class/constructor. Of course, implementations can implement by any means that is not observably different from that.

### Kind -- MethodDefinition or BindingList

The MethodDefinition can define a normal method, a generator method, an accessor, or the constructor. Accessor properties are not about the initial value of the property, but rather about the nature of the property itself. This only makes sense for public properties. A private accessor field is not meaningful.

The normal constructor declaration serves two purposes: To define the behavior of the class/constructor object and to initialize the `"constructor"` property of the prototype to point at this class/constructor. If treated orthogonally, we would continue to have a MethodDefinition for the name `"constructor"` define the behavior of the class/constructor itself. However, what objects are initialized to point at this constructor would be determined orthogonally. Again most of these choices may rarely be useful but there is no reason to violate orthogonality to surprising disallow them. In addition, one case is hugely useful:

Classes will often be defined where the authority provided by holding an instance of a class should not necessarily confer the authority to invoke the constructor and make new instances. This came up most recently with WeakRefs but naturally occurs in many places. I have written much Java code whose correctness depends on the instances providing less power than their class object provides. In this design, if the `"constructor"` method definition is made static, then it is initialized to point at itself. If made private, no matter where it is placed, only code inside the class can follow this pointer back to the class/constructor. Clients of the instances have no such access and so cannot so navigate.

## Open questions

### Distributing Visibility over BindingList

The above grammar has all elements of a BindingList share one Placement and one Visibility. For Placement, the syntax looks intuitive. For visibility, `own # x = 9, y = 10;` can be misunderstood as declaring a private `#x` field and a public `y` property. It seems that one possibility would be to change the grammar so that this is indeed allowed but would actually have that meaning. IOW, the Visibility sigil would be present or absent separately for each member of a BindingList. However, this would be hostile to sharing an annotation across all the members of one BindingList. Instead, we can have the sigil per element, but require that all elements of the same BindingList have the same visibility. Either all have the sigil and are private fields, or none have the sigil and are public properties. As with the other restrictions above, this may surprise writers with rejections they do not expect, but will minimize misundertanding of those reading code that was not statically rejected.
