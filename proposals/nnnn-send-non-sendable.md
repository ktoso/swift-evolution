# Send Non-Sendable

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Joshua Turcotti](https://github.com/jturcotti)
* Review Manager: TBD
* Status: **Awaiting implementation** or **Awaiting review**
* Vision: *if applicable* [Vision Name](https://github.com/apple/swift-evolution/visions/NNNNN.md)
* Roadmap: *if applicable* [Roadmap Name](https://forums.swift.org/...))
* Bug: *if applicable* [apple/swift#NNNNN](https://github.com/apple/swift/issues/NNNNN)
* Implementation: available on public Github: [SendNonSendable.cpp](https://github.com/apple/swift/blob/main/lib/SILOptimizer/Mandatory/SendNonSendable.cpp)
* Upcoming Feature Flag: `SendNonSendable`
* Review: ([pitch](https://forums.swift.org/t/pitch-safely-sending-non-sendable-values-across-isolation-domains/66566))

## Introduction

Swift Concurrency splits memory into the "isolation domains" of various actors and tasks. Computations isolated to distinct domains can execute concurrently, so to prevent data races it is vital that no mutable state is simultaneously accessible from multiple domains. The Swift type system ensures this separation property by allowing only references to deeply immutable values to be communicated between isolation domains. This is enforced through `Sendable` checking; only deeply immutable values can safely conform to the `Sendable` protocol, and only values that conform to the `Sendable` protocol can cross isolation domains.

Unfortunately, requiring `Sendable` conformance for all values that cross isolation domains is very restrictive in practice. For example, mutable objects cannot be constructed by one actor and then sent to another, even if they are never accessed except to be constructed then sent. A flow-sensitive analysis that determines whether values of arbitrary type can be safely sent without introduing data races would allow this pattern, and thus provide a large increase in the expressivity of Swift concurrency.

The `SendNonSendable` pass, currently available as an experimental feature, implements such a flow-sensitive analysis. It tracks all non-`Sendable` values within function scopes, allowing them to be sent between isolation domains but marking them as "transferred" when they are. Transferred values can henceforth not be accessed. This ensures that any potential data races resulting from concurrent access in the sending and receiving domains are avoided. The crux of this analysis is grouping values that could alias or reference each other into static "regions". Sending a value should transfer away the value itself and all other aliasing or referencing values in its region, as accessing those values after the send could also yield a race.

This pass allows for greater flexibility of programming with Swift concurrency, without sacrificing data-race freedom or ergonomics.

## Motivation

The following code demonstrates a situation in which:

- `NearestNeighbors` is a class that cannot safely be made `Sendable` due to the presence of cycles
- `NearestNeighbors.init` is a very expensive operation
- Ultimately, a `NearestNeighbors` instance needs to be displayed to the user via an `addToDisplay` call

The code illustrates a reasonable attempt to use the `NearestNeighbors` class to model and display location data.

```swift
// A representation of location data that associates each point with `numNeighbors`
// of its nearest neighbors in a dataset. This yields connected clusters of
// points, one root of each of which is stored in `rootPoints`.
class NearestNeighbors { 
  class DataPoint {
    // point to the `numNeighbors` nearest neighbors of this point
    var nearestPoints : [DataPoint]
    ...
  }
  
  let numNeighbors : Int
  var rootPoints : [DataPoint]
  ...
}
	
// Build the nearest neighbors graph, associating each point in the dataset `data`
// with its `numNeighbors` nearest neighbors, and storing a point from each resulting
// cluster in `rootPoints`.
// EXPENSIVE operation
func computeNearestNeighbors(data : LocationData, numNeighbors : Int = 10) -> NearestNeighbors { ... }

// display a nearest neighbors graph to the user
// UI operation - so must be `@MainActor`
@MainActor func addToDisplay(neighbors : NearestNeighbors) { ... }

// take location data, build a nearest neighbors graph for it, and display it to the user
func computeAndDisplayData(data : LocationData) async {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors) // warning: passing argument of non-sendable type 'NearestNeighbors' into @MainActor-isolated context may introduce data races
}
```

Unfortunately, current Swift strict concurrency does not allow this code. Since `computeAndDisplayData` is a `nonisolated` function, and `addToDisplay` is `@MainActor`-isolated, the call `await addToDisplay(neighbors)` crosses isolation domains, and thus is not permitted to pass non-sendable values. To make the compiler accept this code, there are three options:

- Disable strict concurrency checking (undesirable)
- Make `NearestNeighbors` `Sendable` (not possible because can't create a cyclic graph with only `let` bindings)
- Make `computeAndDisplayData` `@MainActor`-isolated (undesirable because this would cause the main thread to hang while computing the nearest neighbors graph)

There is thus currently no way to build and display this graph-like `NearestNeighbor`s object without giving up either safety (data-race freedom) or performance. 

## Proposed solution

To a human programmer, the function `computeAndDisplayData` is clearly safe. Swift concurrency prevents `neighbors` from being sent across isolation domains to the main actor because of the potential for the main actor to race with the remainder of `computeAndDisplayData` over access to `neighbors`. But `computeAndDisplayData` doesn't access `neighbors` again after the call, and doesn't allow it to escape. What is needed is a pass that can tell the difference between safe functions like:

```swift
// SAFE:
func computeAndDisplayData(data : LocationData) async {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors)
  // no subsequent access to `neighbors`
}
```

and unsafe functions like:

```swift
// UNSAFE:
func computeAndDisplayData(data : LocationData) async {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors)
  neighbors.addNewData(receiveNewData()) // racy access to `neighbors`
}
```

or:

```swift
// UNSAFE:
func computeAndDisplayData(data : LocationData) async -> NearestNeigbors {
  let neighbors = computeNearestNeighbors(data: data)
  await addToDisplay(neighbors)
  return neighbors // allowing `neighbors` to escape
}
```

Current Swift sendable checking is not flow-sensitive, so it cannot determine whether implementations access non-sendable values after they're sent across isolations. This forces it to ban sending non-sendable values across isolations outright.

This motivates the development of the `SendNonSendable` pass, currently available as `-enable-experimental-feature SendNonSendable`. The `SendNonSendable` pass treats non-sendable values linearly: allowing them to be freely used up until they're sent across isolations, at which point they're treated as *transferred* and subsequent usages of them yield diagnostics. This linear checking allows the `SendNonSendable` pass to determine that the safe version of `computeAndDisplay` above is race-free, and suppress any sendable diagnostics that would be thrown about it, and to determine that the unsafe versions are race-prone, and emit informative diagnostics about the potential races.

### Regions

The above unsafe examples are clearly racy because a single value, `neighbors`, is sent across isolations then accessed or escaped. There are also examples in which races could arise where it's slightly less obvious that accessing the sent value could race with accessing the kept value. 

```swift
func computeAndDisplayLargest(datasets : [LocationData]) async {
  var listOfGraphs : [NearestNeighbors]
  for data in datasets {
    listOfGraphs.append(computeNearestNeighbors(data : data))
  }
  
  let largestGraph = listOfGraphs.max(by: { $0.numRootPoints() > $1.numRootPoints() })
  let smallestGraph = listOfGraphs.min(by: { $0.numRootPoints() > $1.numRootPoints() })
  
  await addToDisplay(largestGraph)
  
  smallestGraph.decreaseClusterSize(1)
}
```

In this function, `Largest` attempts to display the largest `NearestNeighbor` graph generated from a list of datasets passed to the function. No single value appears to be both sent across isolations and subsequently accessed. However, there is no guarantee that `largestGraph` and `smallestGraph` are distinct values - they could be the same value! So the access and mutation of `smallestGraph` via `smallestGraph.decreaseClusterSize(1)` could race with the access to `largestGraph` granted to the main actor via `await addToDisplay(largestGraph)`. 

Strict linearity would solve this by preventing aliasing to ever arise in the first place by banning obvious aliasing (e.g. `let x = y`), and as a result preventing implementations of functions like `min` and `max` that provide first class references to objects still present elsewhere in a datastructure. In short - strict linearity enforces "one first class reference at a time" semantics. This is an unnatural programming model, and in addition to outright preventing certain APIs from being implemented it is awkward and cumbersome to work with (try googling "how to write a doubly linked list in Rust"). 

The solution that the `SendNonSendable` pass chose to implement in place of strict, value-level linearity, is *linear regions*. Regions are collections of values that could alias or reference each other, split up so that two values in separate regions *cannot* ever alias or reference each other. Operations such as cross-isolation method calls that would transfer ownership of values instead transfer ownership of *entire regions* at a time - allowing aliasing to arise but preventing it from leading to races. For example, in the above `computeAndDisplayLargest` function, the values `listOfGraphs`, `largestGraph`, and `smallestGraph` would all be considered in the same region - allowing the `SendNonSendable` code to realize that the access  `smallestGraph.decreaseClusterSize(1)` is potentially racy because `smallestGraph`'s region was already transferred to a different isolation. 

To summarize, the proposed `SendNonSendable` pass accomplished the goal of allowing non-sendable values to be sent across isolations while maintaining data-race freedom by:

- Tracking all **non-sendable values** in scope at all program points, partitioning them into *regions* that are permitted to contain arbitrarily aliased values, but are guaranteed not to alias each other
  - For example, track that `listOfGraphs`, `largestGraph` and `smallestGraph` are all in the same region above
- At program points where **non-sendable values** cross isolations, transfer away the region containing that value
  - For example, mark that the region containing `largestGraph` as transferred after the call to `addToDisplay`
- Produce diagnostics upon access to any **non-sendable** values in regions that have already been transfered away
  - For example, prevent the access to `smallestGraph` above because that value is in a transferred region

Note the emphasis passed on **non-sendable values** - values of sendable type are ignored by this pass, and subject to typechecking exactly as currently implemented.

## Detailed design

This proposal does not add new syntax to Swift, nor change any types. The only change it makes to existing behavior is to prevent the emission of the `non_sendable_call_argument` diagnostic that is emitted when a call that is determined to cross isolation domains has non-sendable arguments. Preempting this diagnostic is what allows non-sendable values to now be passed across isolation domains, such as from a nonisolated context to a main actor isolated context as shown above. 

Instead of emitting the `non_sendable_call_argument` diagnostic, the `TypeCheckConcurrency` Sema pass now decorates `ApplyExpr`s with whether they represent an application that crosses isolations. The `SendNonSendable` mandatory SIL pass is then responsible for emitting diagnostics around the subset of isolation-crossing applications that are could yield races, but only that subset. 

The key question then becomes: how does the SIL pass determine which isolation-crossing applications could yield races? 

### <a name="regionrules"></a> At a high level: simple rules for regions

All non-sendable values within a function body are tracked to determine what region they reside in. Some simple rules regarding value initialization:

- If the initializer for a non-sendable value takes any non-sendable arguments, then the regions of all its arguments will be merged together so that they all occupy a single, shared region, after the initializer returns. The initialized value will also reside in that region.
-  If the initializer is passed no arguments, or is passed only sendable arguments, then a fresh region shared with no existing values will be created for the initialized value. 

It is important to keep in mind that these regions are a purely static abstraction, so allocating fresh regions means only choosing a new identifier for that region, not performing a dynamic allocation of any sort. The following code points out region-creation behavior.

```swift
class NonSendable {
  var x : SendableType
  var y : OtherNonSendableType?
  
  init(_ x : SendableType, _ y : OtherNonSendableType? = none) {
    self.x = x; self.y = y
  }
}

// starting region partition: none

let x = SendableType() // x does not have a region because it is sendable
let y = OtherNonSendableType() // y gets a fresh region
let ns1 = NonSendable(x) // ns1 gets a fresh region
let ns2 = NonSendable(x, y) // ns2 gets the same region as y

// ending region partition: {ns1}, {y, ns2}
```

Functions in general actually exhibit the same semantics: non-sendable arguments are merged into a common region, and non-sendable results come from that region or a fresh one in the case of no non-sendable arguments. This is necessary because, without further information, we have to assume the called function creates references between its passed non-sendable arguments.

```swift
func compareBoxContents(_ box0 : Box, _ box1 : Box) {
  if (box0.contents == box1.contents) {
    print("the same")
  } else {
    print("different")
  }
}

func reassignBoxContents(_ box0 : Box, _ box1 : Box) {
  box0.contents = Contents()
  box1.contents = box0.contents
}

// upon initialization, box0 and box1 occupy separate regions
let (box0, box1) = Box(), Box()

compareBoxContents(box0, box1) // this call merges the regions of box0 and box1

// this transfers the region of box0
Task {
  box0.contents.increment()
}

// this is an error, because the region of box1 was transfered
// (same region as box0)
Task {
  box1.contents.increment()
}
```

This function convention is illustrated above. Though the above code is actually perfectly safe, as the call to `compareBoxContents` does NOT introduce aliasing or referencing between `box0` and `box1`, the function convention has to be safe with respect to implementations like `reassignBoxContents`, that have the same static type as `compareBoxContents`, but DO introduce aliasing between the contents of `box0` and `box1`. If  `reassignBoxContents` were emplaced in the above code, then the creation of the two tasks would indeed yield a data race, so the code must be marked unsafe.

### Summary : simple rules for regions

In summary, regions are "created" when non-sendable values such as classes are intialized without any non-sendable arguments. Regions expand in the following ways:

- `let y = x.f`: reading a non-sendable-typed field of `x` yields a value `y` in the same region as `x`
- `let y = x`: creating an alias `y` yields a value in the same region as `x`
- `y.f = x`: creating a reference from a non-sendable value `y` to a non-sendable value `x` merges their regions
- `z = { ... x ... y ...}`: capturing non-sendable values `x` and `y` in a closure merges their regions, and yields a new closure `z` in the same region as `x` and `y`
- `if/for/while/switch` (control flow): if `x` and `y` are in the same region in any predecessor basic block of a function, then they will be merged into the same region in all successors of that basic block

The result of this is that at the point of a send of a non-sendable value between isolation domains, all other values that could potentially be aliased or referenced from/by it, even along just one control flow path to the send, will be known to be in the same region as it, and will be marked "transferred " and be rendered inaccessible after the send.

### Crossing Isolation Domains

The function conventions outlined above apply to function calls that will execute synchonrously in the same isolation domain as the caller. Calls that can cross isolation domains have slightly different semantics: instead of just being merged, the regions of all non-sendable arguments to a cross-isolation call are *transferred away* as well. The reason this is necessary is different for the two types of calls that pass values across isolation domains.

#### Calls into actors

When a non-sendable value is passed to an actor-entering call, i.e. a call to an actor method from any isolation except that actor's own isolation, the value could "escape" into the storage of that actor. So even after the (necessarily async) call returns, the actor could process a new request concurrently with the caller executuing more code. Since the originally passed value is now accesssible to the actor through its storage, the caller must be banned from accessing it to prevent a data race. Thus actor-entering calls must transfer their arguments. The following code illustrates this:

```swift
actor NationalPark {
  var visitors : [Visitor]
  
  func admitVisitor(_ visitor : Visitor) {
    visitors.append(visitor)
  }
  
  func admitAndGreetVisitor(_ visitor : Visitor) {
    // this call does not cross isolations,
    // so it does not transfer `visitor`, it just merges its region
    // with the region of `self`
    admitVisitor(visitor) // callsite 1 - safe
    
    // so access to `visitor` is permitted
    visitor.greet()
  }
}

func visitParks(_ parks : [NationalPark], _ visitorName : String) async {
  let visitor = Visitor(visitorName)
  
  for park in parks {
    // this call enters the `park` actor, so it
    // transfers `visitor`
    // accessing `visitor` in a loop is thus an error
    await park.admitVisitor(visitor) // callsite 2 - ERROR
  }
}
```

Note that the call to `admitVisitor` within the actor method `admitAndGreetVisitor` labelled `callsite 1` does NOT transfer its argument, and continued access after the callsite is permitted. This is because that call is NOT a cross-actor call, so there is no risk of another actor holding a reference to the argument.

On the other hand, the call to `admitVisitor` in the nonisolated function `visitParks` labelled `callsite 2` enters an actor, so it DOES transfer its argument. This makes the written code unsafe, and indeed it yields an error. If the code were allowed, then multiple actors could concurrently reference the same `visitor` object from their storage, racing on it. 

#### Task creation

Special functions that create new tasks, such as `Task.init` or `Task.detached`, or functions that otherwise cause passed closures to execute concurrently with the caller such as `MainActor.run`, must also transfer their argument. In particular, the argument will be a closure, so any values captured in the closure will be in the same region as that closure, and transferring it will transfer those values. This transfer prevents races on those values between the caller and the executor of the passed closure. This prevents the following possible racy code from being expressible as well:

```swift
func visitParksParallel(_ parks : [NationalPark], _ visitorName : String) {
  let visitor = Visitor(visitorName)
  
  for park in parks {
    Task {
      await park.admitVisitor(visitor) // will error - closure creation transfers `visitor`
    }
  }
}
```

User-defined functions that takes non-sendable closures should not, in general, exhibit isolation-crossing semantics and transfer their arguments, so it will be necessary to inform the `SendNonSendable` analysis of the special functions such as `Task.init`, `Task.detached`, and `MainActor.run` that execute their passed closure under different isolation than or concurrently with the caller. Two possible approaches for this are:

1. Hard-code a list of such functions - possible but likely considered bad practice
2. Mark such functions with an annotation such as `@IsolationCrossing` at the source level where they're defined. 

Additionally, to achieve the desired level of expressivity, it will be necessary to *remove* the source-level `@Sendable` annotation on the types of the arguments to these functions. Without the introduction of the `SendNonSendable` pass, race-freedom is attained for functions such as `Task.detached` only by ensuring they only take `@Sendable` args. This prevents entirely the passing of closures that capture non-sendable values to these functions (as such closures are necessarily `@Sendable`). By implementing the `SendNonSendable` pass, sendable closures will still be able to be passed freely to these functions, but non-sendable closures will not be outright banned - rather they will be subject to the same flow-sensitive region-base checking as all other non-sendable values passed to isolation-crossing calls: if their region has not been transferred the call will be allowed, and otherwise it will throw an error. This change (removing `@Sendable` from the argument signatures of these functions) is necessary.

### Diagnostics

The easiest way to generate diagnostics for the `SendNonSendable` pass would be to note each code site at which a value is accessed, but is known by the pass to be in a transferred region. These diagnostics would read something like:

```swift
func giveBoxToActor(a : MyActor) async {
  let b = Box()
  
  await a.accessBox(b)
  
  if (b.contents >= 7) { // warning: value of non-sendable type `Box` accessed here, but could've been sent to concurrently executing code above, yielding a potential raace
		....
}
```

Entirely correct semantics for this pass could be implemented with this diagnostic style, i.e. diagnostics could be thrown iff the pass discovers an error, but this style of diagnostics is potentially confusing to programmers. Although it is not the site at which the error is discovered, the site at which the isolation-crossing send actually took place is likely a much more logical place to put the warning. This is likely because the calls that cross isolations are trivially known to be sites at which concurrency, and concurrent communication, comes into play. The sites at which values in transferred regions are accessed, however, could be any valid AST node in the Swift language. At a high level, it seems reasonable to expect programmers to think the most carefully about the values being sent at the points concurrent communciation is performed. Thus, although a race arises from the combination of a value sent at such a point with a value in the smae region accessed later in the function, highlighting the point at which the value sent allows the programmer to continue to focus their debugging efforts largely on the points at which concurrency communication is performed. In line with this reasoning, the `SendNonSendable` implementation instead emits diagnostics like the following:

```swift
func giveBoxToActor(a : MyActor) async {
  let b = Box()
  
  await a.accessBox(b) // warning: passing argument of non-sendable type 'Box' from nonisolated context to actor-isolated context at this call site could yield a race with accesses later in this function (1 access site displayed)
  
  if (b.contents >= 7) { // note: access here could race
		....
}
```

If there are multiple sites at which values in the region of the sent value are accessed, they will all be displayed:

```swift
func compareListElems(a : MyActor) async {
  let myList : [NonSendableElem] = genList()
  
  let elem0 = myList.min(by : {$0 < $1})
  let elem1 = myList.max(by : {$0 < $1})
  
  await a.displayElem(elem0) // warning: passing argument of non-sendable type 'NonSendableElem' from nonisolated context to actor-isolated context at this call site could yield a race with accesses later in this function (2 access sites displayed)
  
  print("min: \(elem0)") // note: access here could race
  print("max: \(elem1)") // note: access here could race
}
```

There is one other category of diagnostic emitted by this pass. As described in the [section above](#regionrules), the default function convention assumes that functions do not transfer the regions of their arguments (including `self`). Thus making an isolation-crossing call passing any non-sendable values in the same region as a non-sendable `self` value, or any non-sendable args, will be an error.

<a name="passtoactor"></a>

```swift
func passToActor(a : MyActor, v : NonSendableValue) async {
  await a.foo(v) // warning: call site passes `self` or a non-sendable argument of this function to another thread, potentially yielding a race with the caller
}
```

As the diagnostic message indicates, this warning is necessary because function conventions allow code that continues using non-sendable arguments after they are passed to a non-isolation-crossing call.

```swift
func genAndPassTo(a : MyActor) async {
  let v = NonSendableValue()
  
  // call does NOT transfer v
  await passToActor(a, v)
  
  // access here allowed
  print(v)
}
```

To allow the function `passToActor` above to typecheck, a `transferring`-style annotation is needed. See the section [Transferring args](#transferringargs) below.

## Source compatibility

All previously valid, well-typed Swift code will still be valid, well-typed Swift code. 

## ABI compatibility

No currently planned ABI changes, except as indicated above to potentially change the signatures of functions like `MainActor.run`, which should be possible to do without an ABI break. In the future, we could possibly have to add more information to function signatures to support more general operations on the region partition, such as ensuring that result values from functions come from fresh regions or the regions of arguments; or allowing two arguments to a class method to come from a region distinct from `self` but the same as each other.

## Implications on adoption

The `SendNonSendable` pass, if adopted, would encourage the development of libraries that pervasively use Swift concurrency, but do not enforce sendability on all types communicated between isolation domains. This is a large expressivity win, but could fundamentally change the way libraries are written, and this have a large impact if the feature were adopted then rolled back. 

## Future directions

### <a name="transferringargs"></a>Transferring args

In the current implementation of `SendNonSendable`, functions cannot transfer their arguments. This greatly constricts allowed programming patterns. To send a value to another isolation domain, that value *must* have been initialized locally, not read from an argument or from `self`'s storage. Allowing values from `self`'s storage to be safely sent to other threads is the subject of the `iso` fields extension (discussed below)[#iso] and is a bit involved, but allowing arguments to be sent (i.e. transferred), is much simpler. All that is necessary is to add a `transferring` annotation to function signatures that indicates that certain arguments could be transferred by the end of the function body. As a simplest example, the following code shows the behavior of the `transferring` annotation:

```swift
func passToActorTransferring(a : MyActor, transferring v : NonSendableValue) async {
  await a.foo(v)
}

func genAndPassTransferring(a : MyActor) async {
  let v = NonSendableValue()
  
  // call transfers v
  await passToActorTransferring(a, v)
  
  // access here NOT allowed
  print(v)
}
```

Unlike the prior function `passToActor` (defined above)[#passtoactor], `passToActorTransferring` *does* typecheck now, but `genAndPassToTransferring` does not - the inverse situation of the non-transferring functions.

`transferring` parameters are a very natural programming pattern. Without them, it is only possible for non-sendable values to ever make a single hop between domains. They are also very easy to implement, and might even be done so before the acceptance of this proposal. The largest difficulty with the addition of this feature is ergonomic interplay with the existing `transferring` annotation that exists in the Swift language. The existing `transferring` keyword focuses on non-copyable types - and specifies ownership conventions for handling refcounts and deallocation. This is related to the idea of `transferring` needed by `SendNonSendable`(let's call it `region-transferring` for now), and in fact, any case in which a parameter is `region-transferring`, it should also be `transferring` (in the existing Swift, non-copyable, sense). Unfortunately, the converse does not hold. For example, parameters to initializers and setters are usually `transferring`, but they should not be `region-transferring`. The reason for this is that the region-based typechecking of `SendNonSendable` is able to track the fact that by passing values to a setter or initializer, ownership has been transferred, but to a known target: the region of `self` of the called method. Thus by not marking such parameters as `region-transferring`, they can still be used after being passed to a setter or initializer as long as the set or initialized value is not transferred. If all such methods' parameters were `region-transferring`, then even if `self` were not transferred, the arguments to the methods would not be accessible after the call.

It is worth noting that for any methods meant to be called only in isolation-crossing contexts, it is a strict gain of expressivity to mark their non-sendable arguments as `transferring`; at callsites, transferance is enforced anyways, and within the function body, the freedom to transfer the arguments again by passing to a third isolation domain would be attained. This provides further evidence that exposing it as an annotation would be useful.

To concretely illustrate the semantics of this extension, `transferring` parameters would:

- transfer the regions of their non-sendable arguments at callsites *whether or not* the callsite is isolation-crossing (in contrast to non-`transferring` parameters that only transfer their arguments' regions if the callsite is isolation-crossing)
- be allocated separate regions from `self` and all other arguments in the region partition used at the entry to function bodies.

It is of note that there is technically an even more general approach that expands on the second point of semantics above by allowing two `transferring` parameters to be assumed to come from the same region as each other. Without this feature, `transferring` parameters would always have to be passed values known to be in a separate region from all other arguments at the callsite. With this feature, two values that are possibly aliases of each other, or possibly reference each other, could be passed to two `transferring` arguments. The benefits of this further extension are less concrete that `transferring` itself, so it is not likely this will be introduced soon.

### Returning fresh

Another way that function signatures could be made more expressive is through their results. In the current `SendNonSendable`, there is no way to return non-sendable values from an isolation-crossing function. In fact, the diagnostics produced for obtaining non-sendable results across isolation boundaries that arise from `TypecheckConcurrency` are not even suppressed. This makes code such as the following impossible:

```swift
func generateFreshPerson(_ name : String, _ age : Int, _ ancestryMap : AncestryMap) -> Person {
  let person = Person(name, age)
  if (ancestryMap.containsChild(person)) { ... /* do some logic */ }
  
  return person
}
```

This code basically wraps an initializer with extra logic, and aims to return a non-sendable result to the caller in a fresh region. Unfortunately, the current `SendNonSendable` pass does not allow its result to be used. A useful extension would allow functions with non-sendable results to allow those results to be accessible to cross-isolation callers in the following cases:

- For methods that are not actor methods, the result of isolation-crossing calls to those methods are always available to the caller. 
  - By default, results are provided in the same region as `self` and any other arguments (except `transferring` arguments (see above)[#transferringargs])
  - If the `fresh` keyword is placed on the result type in the function signature (e.g. `func generateFreshPerson(...) -> fresh Person`), then the result is provided in a fresh region
- For actor methods:
  -  If the result is not annotated with the `fresh` keyword, then it is never accessible to cross-isolation callers, as the result is in the same region as actor-isolated storage, which is not accessible to the caller.'
  - If the result is annotated with `fresh`, then it is provided to the caller in a fresh region just as for non-actor methods.

Running the `SendNonSendable` pass now involved also ensuring that any values used as `fresh` non-sendable results do indeed come from a region distinct from the region of self and any args.

### <a name="iso"></a>`iso` fields

In the system outlined up to this point, a notable weak point lies in that regions tend to grow very large. In fact, any data structures, including actors and all their stroage, will comprise a single region. As shown in the rest of this proposal, this still allows a large variety of concurrent programming patterns currently impossible in Swift, but it also bans many more. For example, it prevents actors from sending parts of their state to other actors and replacing it with fresh values, such as the following:

```swift
actor IndecisiveBox {
  var contents : NonSendable
  
  func setContents(_ val : NonSendable) {
    contents = val
  }
  
  func swapWithOtherBox(_ otherBox : IndecisiveBox) async {
    let otherContents = await otherBox.contents
    await otherBox.setContents(contents) // warning: call site passes `self` or a non-sendable argument of this function to another thread, potentially yielding a race with the caller
    contents = otherContents
  }
}
```

This could be a perfectly safe pattern, but in the current `SendNonSendable` pass will produce diagnostics as shown above. The issue is that it is not statically known that it is safe to send `contents` to another thread while continuing to access the rest of the actor's storage. To this end, the system could introduce the `iso` keyword. 

The `iso` keyword, when placed on fields, for example as `iso var contents : NonSendable` above, would indicate that instead of tracking the source and target of the reference (here, `self` and `self.contents`) as necessarily belonging to the same region, they could belong to *different* regions. This allows the above code to typecheck, but comes at the cost of greater complexity to the analysis. For more details see (the PLDI paper)[https://www.cs.cornell.edu/andru/papers/gallifrey-types/].

### Global actors

Global actors can have state in the form of actor-isolated global variables. Unlike non-isolated global variables, global actor isolated variables will be able to safely be non-sendable, but they need to be modelled as residing in a `self`-like region dedicate to that global actor in functions isolated to the global actor. This will ensure that non-sendable values read from global actor isolated state cannot be sent to another isolation domain, and, via the region mechanim, that any values that could reference or alias global actor isolated state cannot be sent to another isolation domain

### Async let, and task completion

When structured tasks or statements executed with `async let` complete, not just their possibly non-sendable result but any non-sendable values captured by them should become accessible to the caller. This is not yet implemented.

## Alternatives considered

### Leaving sendability requirements as is

### Relying on move-only (i.e. purely linear) types

### Non-inferred regions

### A finer capabilities lattice

## Acknowledgments

This proposal is based on joint work with Mae Milano and Andrew Myers, published in the PLDI 2022 paper (A Flexible Type System for Fearless Concurrency)[https://www.cs.cornell.edu/andru/papers/gallifrey-types/]. Doug Gregor and Kavon Farvardin assisted with the development of the `SendNonSendable` implementation and this proposal as well.
