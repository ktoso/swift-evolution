# Distributed Remote Call Validation

* Proposal: [SE-NNNN](NNNN-distributed-remote-call-validation.md)
* Authors: [Konrad 'ktoso' Malawski](https://github.com/ktoso)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Implementation: WIP
* Review: 

## Motivation

Distributed actors in Swift enable library developers to build custom remote call procedure handling systems that seamlessly integrate into existing Swift programming patterns and structured concurrency across process boundaries.

Increasingly, these systems are used in security sensitive use-cases and expressing the formal policy requirements on distributed calls is left up to developers inside their distributed functions; much like they would in the past with manually implemented serialization logic.

Before distributed functions were introduced, code would often manually decode requests and validate the credentials before performing the actual service logic:

```swift
// before distributed actors
func handleOpenDoor(request: RequestStruct, reply: (ReplyStruct) -> ()) {
  // Serialization by hand...
  let mappings = request.getField("mappings", as: [Int: Int].self)
  let userID = request.getField("userID", as: Int.self)
  // ... 
  
  // Policy validation by hand...
  guard isAdmin(userID) else {
    reply(.insufficientPermissions)
    return
  }
  
  // Actual logic
  // ...
  
  // Serialize response 
  let response = // ...
  reply(response)
}
```

Distributed functions got rid of the serialization/deserialization, allowing system authors to take complete ownership of this and simplify developers day-to-day code. However, validation still remained in user-implementation code:

```swift
// distributed funcs until today
distributed func openDoor(request: RequestStruct) async -> ReplyStruct {
  // Policy validation by hand...
  guard isAdmin(request.userID) else {
    return .insufficientPermissions
  }

  // Actual logic
  // ...
  return .ok
}
```

While this is an improvement already, thanks to encapsulating the serialization logic in an `DistributedActorSystem`, and also adopting async/await, we found that developers were still forced to express their policy requirements in code.

These manual checks error-prone and annoying to audit, because they require reading through the code, and making sure, by human or tool, that the actual logic and policy is matching the stated policy of a service. It also is usually done after having de-queued the message payload and potentially even after having deserialized it. Sometimes, we might want to perform validation check on a per message (header) basis, however before dequeueing and de-serializaing the rest of the message. 

This proposal focuses on the 90% of cases where a static, well defined policy is applied per distributed function, and validation is performed before deserializing the message payload, which was difficult to achieve previously. The proposed solution also provides superior auditability.

### Intended audience

The Distributed module and language feature always have multiple audience personas in mind. It might be helpful to have a brief reminder of those personas before we explore the proposal in depth. Specifically, most of this proposal is intended for:

- **Actor system library authors** - those developers implement the `DistributedActorSystem` protocol and have knowledge and control over the runtime and networking code the system implements. They may need to transport level security policies, or other domain specific requirements the language feature cannot know about. This is the primary intended audience of many of the details of this proposal.
- **Actor system users** - developers who use implemented actor systems, and get the benefit of not having to worry about networking details already taken care of at the actor system level. This is the largest group of developers, however this persona doesn't necessarily need to know about any of the implementation details of the ActorSystem of their choice.
- **Application auditors** - engineers who do not actively develop neither the actor system, nor the actual applications using them. This persona is an important target audience of this proposal because it is responsible for the overall security and policy of published systems, and needs tools to validate the code efficiently and confidently. This proposal features an auditing section aimed towards this persona, and the general design and focus on "single source of truth" can also be tracked to this point of view.

## Proposed solution

This proposal introduces a way to declaratively express policy requirements on distributed functions.

The end goal is to **allow actor system implementations to provide their own domain-specific validation macros**. For example, you could imagine a simple entitlement (or header) based validation system, like this:

```swift
protocol FishingBoat: DistributedActor where ActorSystem == FishingActorSystem {
  @ValidateRemoteCall(.requireFishingLicense)
  distributed func fishBasic()

  @Entitlement("com.example.fishing.permit")
  distributed func fish()

  @Entitlement(.anyOf("com.example.fishing.permit.big", "com.example.fishing.permit.huge"))
  distributed func bigFish()
}
```

Or, you could imagine expressing a policy using Apple's [LightweightCodeRequirements](https://developer.apple.com/documentation/lightweightcoderequirements) DSL and refer to it from a validation macro, or any other mechanism. Due to the nature of distributed actors in Swift, this design is open ended and open to extension to various use-cases that may have very different shapes, runtime capabilities, and security requirements.

## Detailed design

### The RemoteCallValidator type

Remote call validation is effectively just a type that the actor system invokes before remote calls are performed or executed. It wraps a validator closure keyed on aconcrete `DistributedActorSystem`, which enables type-safety for optional context information to be passed to the validator.

```swift
@available(SwiftStdlib 6.5, *)
public struct RemoteCallValidator<ActorSystem: DistributedActorSystem>: Sendable {

  public init(
    _ check:
      @escaping @Sendable
      (ActorSystem.RemoteCallValidationContext) throws(ActorSystem.RemoteCallValidationFailure) -> Void
  )

  /// Pass-through initializer. Used by the macro plugin so that both
  /// `@ValidateRemoteCall({ ... })` and `@ValidateRemoteCall(.namedRecipe)`
  /// funnel through a single code path.
  public init(_ validator: RemoteCallValidator<ActorSystem>)

  /// Invoke the wrapped validator against `context`; rethrows what the body throws.
  public func check(
    context: ActorSystem.RemoteCallValidationContext
  ) throws(ActorSystem.RemoteCallValidationFailure)
}
```

The validation's check function contract is effectively that it is expected to _throw_ an error whenever the validation has failed. The actual signature is tied to the actor system implementations it is compatible with, and is expressed using two new associated types on the `DistributedActorSystem` protocol:

- `RemoteCallValidationContext` - which determines the context parameter type that can be passed to the validation check,
- and `RemoteCallValidationFailure` - which is the error type expected to be thrown if the validation check fails.

```swift
public protocol DistributedActorSystem<SerializationRequirement>: Sendable {
  // ...

  @available(SwiftStdlib 6.5, *)
  associatedtype RemoteCallValidationContext = Void // could be (ActorID, RemoteCallTarget)

  @available(SwiftStdlib 6.5, *)
  associatedtype RemoteCallValidationFailure: DistributedActorSystemError
}
```

The default value of thrown errors is `DistributedActorSystemError` because that's the protocol of errors that can be thrown "by" the actor system, and not the actual target of the call. Today throws are not typed, so distributed functions become implicitly `throws`, but this sets us up to explore a future where a distributed function throws either:

- user error declared in the function signature: `distributed func tryMe() throws(MyError)`
- or a system error that conforms to `DistributedActorSystemError`

As the system already is allowed to throw `DistributedActorSystemError` validators should fold into this, and just throw "more specific" validation errors.

### The @ValidateRemoteCall macro

Actor system authors have two options to author and express validation policies. 

The simplest approach, is to use the new `ValidateRemoteCall` peer macro which may be attached `distributed` declarations (including `distributed actor`, `distributed func` and `distributed var`):

```swift
@available(SwiftStdlib 6.5, *)
@preservedInInterface  <<<<<<<<<<<<<<<<<
@DistributedValidatorMacro
@attached(peer, names: arbitrary)
public macro ValidateRemoteCall<ActorSystem: DistributedActorSystem>(
  _ validator: RemoteCallValidator<ActorSystem>
) = #externalMacro(module: "SwiftMacros", type: "ValidateRemoteCallMacro")
```

Validators can then be provided via an extension on the `RemoteCallValidator`, like this:

```swift
// FishingActorSystem module
extension RemoteCallValidator where ActorSystem == FishingActorSystem {
  public static var requireFishingLicense: RemoteCallValidator {
    RemoteCallValidator { _ in
      // Check the caller carries a valid fishing license using
      // whatever mechanism FishingActorSystem exposes (e.g. a task-local
      // envelope header). Throw to reject the call.
    }
  }
}
```

Which would be then used by users of that distributed actor system, i.e. developers actually writing protocols and actors, as follows:

```swift
// API module
protocol FishingBoat: DistributedActor where ActorSystem == FishingActorSystem {
  @ValidateRemoteCall(.requireFishingLicense)
  distributed func fish() -> Fish?
}
```

In practice it is the side which enforces the server side validation requirements, that will provide the validators. And the actor system that specifies what kinds of validators can be used.

The implementation details of this macro are out of scope of the proposal, but it's worth mentioning that it effectively generates a peer declaration that uses `@section` to generate a section in which function pointers are emitted which can be later found and referenced by the runtime when validation needs to be invoked. The general mechanism is quite similar to how swift-testing implements the `@Test` macro (detailed in [Runtime-discoverable test content](https://github.com/swiftlang/swift-testing/blob/main/Documentation/ABI/TestContent.md)).

### Declaring third-party validation macros: `@DistributedValidatorMacro`

The mechanism that ties a custom validation macro (like the `@RequireFishingLicense` above) into the inheritance opt-in is a **marker macro**, `@DistributedValidatorMacro`, provided by the standard library. It is attached to a `macro` declaration and generates a marker type conforming to `DistributedRemoteCallValidationMacroIdentifier`. The marker's name is derived by appending `Macro` to the macro declaration's name; the compiler recovers the macro identity by stripping that `Macro` suffix when consulting `InheritMacros<...>`.

```swift
@available(SwiftStdlib 6.5, *)
@attached(peer, names: suffixed(Macro))
public macro DistributedValidatorMacro() =
  #externalMacro(module: "SwiftMacros", type: "RemoteCallValidationMarkerMacro")

@available(SwiftStdlib 6.5, *)
public protocol DistributedRemoteCallValidationMacroIdentifier {}
```

Third-party actor system authors use it when declaring their own validation macros. For example, the fishing actor system's macro would be declared as:

```swift
// FishingActorSystem module

// Attaching @DistributedValidatorMacro generates:
//   public enum RequireFishingLicenseMacro: DistributedRemoteCallValidationMacroIdentifier {}
@DistributedValidatorMacro
@attached(peer, names: arbitrary)
public macro RequireFishingLicense(_ category: FishingLicenceCategory) =
  #externalMacro(module: "FishingMacros", type: "RequireFishingLicenseMacro")
```

The generated marker type is what the actor system lists in its `RemoteCallValidation` typealias to opt into inheriting the macro from protocol requirements onto witnesses:

```swift
final class FishingActorSystem: DistributedActorSystem, @unchecked Sendable {
  // ... other required conformances (ActorID, InvocationEncoder, etc.) ...

  typealias RemoteCallValidation =
    DistributedRemoteCallValidation.InheritMacros<RequireFishingLicenseMacro>
}
```

The standard-library-provided macros `@ValidateRemoteCall` and `@Entitlement` are themselves declared with `@DistributedValidatorMacro`, which is why they have corresponding `ValidateRemoteCallMacro` and `EntitlementMacro` marker types usable in `InheritMacros<...>`. An actor system that lists neither inherits nothing; the compiler will not clone unknown validation attributes from a protocol requirement onto a witness, so a stray `@RequireFishingLicense` on a protocol requirement is silently ignored by any system that has not opted into it. This keeps each actor system in control of exactly which policy vocabulary it understands.

### Configuring supported validation macros

An actor system may configure and validate which validation macros it supports. This will matter the most when we discuss validation macro inheritance in the next sections. 

```swift
// The existing Distributed::DistributedActorSystem type
public protocol DistributedActorSystem<SerializationRequirement>: Sendable {
  // ...

  /// Customization point which allows configuring the remote call validation mechanisms used by this actor system.
  @available(SwiftStdlib 6.5, *)
  associatedtype RemoteCallValidation: DistributedRemoteCallValidationSetting =
    DistributedRemoteCallValidation.InheritMacros<ValidateRemoteCallMacro>
}
```

The actor system may override this associated type with a specific list of validation macros it supports, and even prohibit the default general purpose validation mechanism, and replace it with it's domain specific one, e.g.:

```swift
final class FishingActorSystem: DistributedActorSystem, @unchecked Sendable {
  // ... other required conformances (ActorID, InvocationEncoder, etc.) ...

  typealias RemoteCallValidation =
    DistributedRemoteCallValidation.InheritMacros<RequireFishingLicenseMacro>
}
```



### Validating incoming calls

Distributed actor system implementations always have some form of receive loop, or other mechanism using which they receive messages from the network. They deserialize the invocation identifier, actor identifier, and eventually proceed to invoke the `DistributedActorSystem.executeDistributedTarget(on:target:invocationDecoder:handler:)` function which deserializes the remaining arguments and invokes the target method.

It is the responsibility of the actor system's receive implementation to invoke the remote call validation before the `executeDistributedTarget` is triggered. This is intentionally not automatically performed by the `executeDistributedTarget` because depending on system serialization specifics, this may be too late -- perhaps the system would prefer to validate the incoming calls before even attempting to create the InvocationDecoder instance, assuming that even that could be compromised by an mallicious payload.

A typical implementation would therefore look like this:

```swift
// FishingActorSystem
extension FishingActorSystem {
  func receiveMessage(envelope: Envelope) async throws {
    let target = RemoteCallTarget(envelope.targetIdentifier)
    let actor = try self.knownActor(id: envelope.recipientID)

    // Look up and run all validators attached to this target *before*
    // decoding any arguments or invoking the target method. If a
    // validator throws, the call is rejected. A `RemoteCallValidationLookupError`
    // returned from `validate(...)` signals a programmer error (the target
    // is not bound to this actor system), not a validator rejection.
    let context = (actor.id, target)
    if let lookupError = try self.validate(target: target, context: context) {
      throw lookupError
    }

    // ... only now decode the remaining bytes and dispatch: ...
    var decoder = FishingInvocationDecoder(envelope: envelope)
    try await self.executeDistributedTarget(
      on: actor,
      target: target,
      invocationDecoder: &decoder,
      handler: FishingResultHandler(reply: envelope.reply))
  }
}
```





Under the hood, `validate(target:context:)` looks up the target's validation chain by its mangled identifier and composes the results as **AllOf** - every attached validator must accept for the call to proceed:

```swift
// Pseudo-code for `DistributedValidation.lookup`:
extension DistributedValidation {
  static func lookup<ActorSystem: DistributedActorSystem>(
    targetIdentifier: String,
    using system: ActorSystem.Type
  ) throws(RemoteCallValidationLookupError) -> RemoteCallValidator<ActorSystem>? {
    // Follow the accessible-function record's tagged Flags pointer to the
    // first `swift5_daval` record, then walk `relativeNext` to collect
    // every validator on this target. Multiple records arise from stacked
    // attributes and from cross-module inheritance of a protocol
    // requirement's attribute onto the concrete witness.
    guard let first = firstValidationRecord(for: targetIdentifier) else {
      return nil // no `@ValidateRemoteCall` / `@Entitlement` on this target
    }
    let validators = collectValidators(from: first, using: ActorSystem.self)

    // AllOf composition: the first thrown error short-circuits the rest.
    return RemoteCallValidator<ActorSystem> { context in
      for validator in validators {
        try validator.check(context: context)
      }
    }
  }
}
```



### Single source of truth 

With most client/server applications, the source of truth for the protocol spoken by a service is the actual `protocol` type offered in an API package by the service, for example:

```swift
@Validated...
protocol Door: ValidatedDistributedActor where ActorSystem == HouseSystem {
  @ValidateRemoteCall(.loggedIn)
  distributed func ringDoorBell()

  @ValidateRemoteCall(.loggedIn)
  distributed func openDoor()
 
}

extension Door {
    func validate(target: RemoteCallIdentifier) throws {}
    // validators(identifier) -> Validator
  
      func validatedDesc()-> [String....] {}
}
```



```swift
distributed actor MyDoor: Door {
		// every validate() --->>> 
    distributed func ringDoorBell() // NEEDS VALIDATOIN
}
```



### Validation inheritance

In order to achieve this goal of single-source-of-truth for the validation rules to be on the protocols, we provide an automatic way for those macro rules to be inherited by specific distributed methods because remote call invocations are performed on specific _concrete_ methods, and not on the protocol requirements.

The compiler behavior on macro inheritance is configured by an actor system using a new typealias:

```swift
@available(SwiftStdlib 6.5, *)
extension MyActorSystem {
  
  public typealias RemoteCallValidation =
    DistributedRemoteCallValidationMacros<ValidateRemoteCallMacro, EntitlementMacro>

}
```

### Validation stacking

It is possible to stack multiple validations, both through protocol func validation inheritance, as well as though applying multiple validations to the same function.

In both cases,the combination of those validations is following an "AND" semantic, in the sense that all validations must pass for a method to be invoked. This is the only logical stacking method, as stacking them as an "OR" would allow side-stepping requirements imposed by the protocol authors, which would be a security hole in the model.

The stacking therefore works as follows:

```swift
protocol Boat: DistributedActor where ActorSystem == FishingSystem {
  @ValidateRemoteCall(.fishingLicense)
  distributed func fish() -> Fish?
}

distributed actor CoolBoat: DistributedActor where ActorSystem == FishingSystem {
  @ValidateRemoteCall(.coolFishingLicense)
  // AND (implicitly, since required by the protocol)
  // @ValidateRemoteCall(.fishingLicense)
  distributed func fish() -> Fish? { nil }
}
```

And applying multiple validations on the same method, also requires them all to pass:

```swift
distributed actor CoolBoat: DistributedActor where ActorSystem == FishingSystem {
  @ValidateRemoteCall(.coolFishingLicense)
  // AND
  @ValidateRemoteCall(.fishingLicense)
  distributed func fish() -> Fish? { nil }
}
```

This is useful when some checks are additional and controlled by an implementation, in addition to the protocol enforced requirements.

It also is important to allow multiple requirements to be stated cleanly, if they're using different attributes for example:

```swift
  @Role(.admin)
  // AND
  @ValidateRemoteCall(.fishingLicense)
  distributed func adminFishing() -> AdminFish? { nil }
```



### Optional: remoteCall client-side validation

As this is a security feature, protecting access to restricted methods and resources, the validation of course needs to be performed on the server-side.

However, it may sometimes be useful, perhaps during development builds, to offer more informative errors about why a validation might fail immediately on the client-side. These checks are only "nice to have" and never provide actual security from a service point of view, nevertheless, it is possible to call validators on the client side.

An actor system which offers optional client-side validation, would opt into doing so by issuing the validate call in the `remoteCall` (and `remoteCallVoid`) methods which it implements already to make the remote calls made on distributed functions:

```swift
// existing DistributedActorSystem/remoteCall protocol requirement implementation
public func remoteCall<Act, Err, Res>(
    on actor: Act,
    target: RemoteCallTarget,
    invocation invocationEncoder: inout InvocationEncoder,
    throwing errorType: Err.Type,
    returning returnType: Res.Type
  ) async throws -> Res
    where Act: DistributedActor,
          Act.ID == ActorID,
          Err: Error,
          Res: SerializationRequirement {

    // add this to validate calls on client-side:
    // - if `validate` throws, it was a validator rejection => fail the call
    //   immediately without touching the network
    // - if `validate` returned a `RemoteCallValidationLookupError` instead,
    //   it is a programmer error (e.g. the target is not bound to this
    //   actor system); surface it as a thrown error.
    let context = (actor.id, target)
    if let lookupError = try self.validate(target: target, context: context) {
      throw lookupError
    }

    // ... encode + network round trip ...
  }
```



### Auditing 

A crucial feature of this validation proposal is its **auditability**, both by developers working on introducing new distributed functions, as well as external security reviewers and automated tools.

The existing hand written `guard isAdmin(...)` or `try checkPermissions(...)` based code is hard to audit manually (or even with AI tooling, as there is an additional cost and uncertainty to it).

The metadata necessary for the distributed function validation is retained in swift modules and interfaces, so it is possible to easily audit the produced interfaces and modules if they correctly applied the required policy. This is a vast improvement over looking at the code's implementation and making sure on each commit if the control flow didn't accidentally side-step the checks.

A common concern in these declarative security systems is "**How do we know if policy was not applied?**", therefore this design takes this into account and makes it very simple to build a linter or other tool that will validate produced build artifacts, and if a `distributed func/var` is missing a policy macro attachment, this can be confidently flagged and prevent shipping such mistake in production. 



Example output:

```swift
> swift-inspect distributed audit /tmp/audit-fixture/libBank.dylib  --demangle
Distributed accessible functions in /tmp/audit-fixture/libBank.dylib:
      total: 3
  validated: 2

  func/var                                                                            Validators
  ------------------------------------------------------------------------------------------------
  distributed thunk Bank.Bank.read() async throws -> Swift.String            1  @Entitlement(.anyOf(["admin", "superuser"]))
  distributed thunk Bank.Bank.transfer(amount: Swift.Int) async throws -> Swift.Bool  1  @Entitlement("com.example.transfer")
  distributed thunk Bank.Bank.ping() async throws -> ()                               -
```



## Source compatibility

This design is entirely source additive, and does not change any source level semantics of existing distributed actors.

## ABI compatibility

This proposal is ABI additive. The `Flags` field of the existing accessible-function record is repurposed: bit 0 remains the existing `Distributed` bit, bit 1 is a new `HasValidation` bit, and the remaining bits carry a self-relative offset to this target's first validation record. If `HasValidation` is set, the runtime follows that offset into the new `swift5_daval` section and walks the target's validation-record linked list (each record chains to the next via its `relativeNext` field).

The two new sections are `swift5_daval` for validation records and `swift5_davala` for the macro-emitted validator accessor thunks.

Old runtimes are unaware of these validation records; they only observe bit 0 of `Flags`. The record itself keeps its existing size and layout, so this change is purely ABI additive and does not break existing ABI.

## Future directions

### Source and binary auditing tools

It might be desirable to provide additional auditing tools which can both enforce policy that every method must have a validation attached. These tools could operate either on source level, using sourcekit powered analysis, and power tools like linters to provide development time hints about missing policy enforcement; or, such tools could even be working on binary artifacts to validate produced binaries have indeed not omitted any policy. 

We believe security is multi layered, and providing devleopers a good development time experience as well as auditors good tools to enforce policy are both valuable tools, and not entirely multually exclusive. It also depends on the use case, how much effort and what techniques are used to validate policy like these. And each project needs to decide the level of effort and tooling they are interested in.

## Alternatives considered

### 

## Acknowledgments







### Implementation details: validator section

```swift
@Entitlement("com.example.cross-module")
distributed func openDoor() -> Bool { true }

---

public typealias _DistributedValidationRecord = (
  kind: UInt32,
  reserved: UInt32,
  relativeAccessor: Int32,
  relativeNext: Int32
)

public typealias _DistributedValidationAccessor = @convention(c) (
  _ outValue: UnsafeMutableRawPointer,
  _ type: UnsafeRawPointer,
  _ reserved: UInt
) -> CBool

---

#if objectFormat(MachO)
@section("__DATA_CONST,__swift5_davala")
#elseif objectFormat(ELF) || objectFormat(Wasm)
@section("swift5_davala")
#elseif objectFormat(COFF)
@section(".sw5davala$B")
#endif
@used
@available(*, deprecated, message: "Implementation detail of Distributed. Do not use directly.")
private static let $s48distributed_macro_validation_cross_module_binary6MyHomeC8openDoor11EntitlementfMp_25__daval_openDoor_accessorfMu_: 
    Distributed._DistributedValidationAccessor = { outValue, type, reserved in
  let expected: Any.Type = Distributed.RemoteCallValidator<MyHome.ActorSystem>.self
  guard type.load(as: Any.Type.self) == expected else { // checks if this is the right system/validator
    return false
  }
                                                  
  let validator = Distributed.RemoteCallValidator<MyHome.ActorSystem>({ _ in // pass context if we have any
    // THIS IS @Entitlement's implementation:
    try Distributed.DistributedValidation.evaluateEntitlement( 
      Distributed.EntitlementPolicy.entitlement(
        "com.example.cross-module" // USER provided string into the @Entitlement macro
      )
    )
  }
  outValue.assumingMemoryBound(to: Distributed.RemoteCallValidator<MyHome.ActorSystem>.self)
    .initialize(to: validator)
  return true
)
```



```swift
#if objectFormat(MachO)
@section("__TEXT,__cstring")
#elseif objectFormat(ELF) || objectFormat(Wasm)
@section("swift5_davala_desc")
#elseif objectFormat(COFF)
@section(".sw5davala_desc$B")
#endif
@used
@available(*, deprecated, message: "Implementation detail of Distributed. Do not use directly.")
// TODO: InlineArray...? Will we ever support StaticString here?
private static let 
  $s48distributed_macro_validation_cross_module_binary6MyHomeC8openDoor11EntitlementfMp_25__daval_openDoor_accessorfMu__desc: 
  \(raw: tupleType) = 
    (..., ..., ..., ..., ..., ...) // The whole TEXT of the macro
```



Existing Accessible Function Records:

```swift
/// An "accessible" function that can be looked up based on a string key,
/// and then called through a fully-abstracted entry point whose arguments
/// can be constructed in code.
template <typename Runtime>
struct TargetAccessibleFunctionRecord {
public:
  /// The name of the function, which is a unique string assigned to the
  /// function so it can be looked up later.
  RelativeDirectPointer<const char, /*nullable*/ false> Name;

  /// The generic environment associated with this accessor function.
  RelativeDirectPointer<GenericEnvironmentDescriptor, /*nullable*/ true>
      GenericEnvironment;

  /// The Swift function type, encoded as a mangled name.
  RelativeDirectPointer<const char, /*nullable*/ false> FunctionType;

  /// The fully-abstracted function to call.
  ///
  /// Could be a sync or async function pointer depending on flags.
  RelativeDirectPointer<void *, /*nullable*/ false> Function;

  /// Flags providing more information about the function.
  AccessibleFunctionFlags Flags; // 32 bit, if HAS_VALIDATION ---> rel pointer to ValidationRecord
};
```



## TODO

- **where** to preserve the text

`StringRef preservedArgText;` in `CustomAttr`? or make a new attribute 

- **end-user** extensibility because domain specific macros / DSLs

So we need to enable

```swift
@preservedInInterface <<<< needed to "keep this one in interfaces"
@DistributedValidatorMacro <<< was my idea to emit a type `EntitlementMacro` 
@available(SwiftStdlib 6.5, *)
@attached(peer, names: arbitrary)
public macro Entitlement(_ policy: EntitlementPolicy) =
```

- How to **opt in** which macros to "inherit"

above idea is the `DistributedValidatorMacro` and allow listing them somewhere...

```swift
extension FakeRoundtripActorSystem {
  public typealias RemoteCallValidation =
    DistributedRemoteCallValidationMacros<ValidateRemoteCallMacro, EntitlementMacro>
```

This is just `<each Macro: DistributedRemoteCallValidationMacroIdentifier>`

- **Retain in binary on by default**, probably keep by default, but allow disabling with compiler flag?

- `swift-inspect distributed audit`

```
> swift-inspect distributed audit /tmp/audit-fixture/libBank.dylib  --demangle
Distributed accessible functions in /tmp/audit-fixture/libBank.dylib:
      total: 3
  validated: 2

  func/var                                                                            Validators
  ------------------------------------------------------------------------------------------------
  distributed thunk Bank.Bank.read() async throws -> Swift.String                     1  @Entitlement(.anyOf(["admin", "superuser"]))
  distributed thunk Bank.Bank.transfer(amount: Swift.Int) async throws -> Swift.Bool  1  @Entitlement("com.example.transfer")
  distributed thunk Bank.Bank.ping() async throws -> ()                               -
```
