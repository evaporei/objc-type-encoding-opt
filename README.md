# objc-type-encoding-opt

> Objective-C [Type Encoding](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100) optimization

## Background

The Objective-C runtime handles all method dispatches via [objc_msgSend](https://developer.apple.com/documentation/objectivec/1456712-objc_msgsend) during the execution of the program. For this dispatching mechanism to work properly, all methods have a type signature which is encoded following [this specification](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100). Example `bool myMethod(int a, int b)`/`- (bool)myMethod:(int)paramA:(int)paramB` becomes `Bii`.

## Objective

Even though the Objective-C runtime is highly optimized, the indirection mechanism isn't zero-cost. This repository tries to experiment with the dispatching mechanism to gain **performance at execution time**.

## Premises

- Require **no code change** from the developer's perspective;
- Do the optimization at **compile time**;
- Make the dispatching mechanism of **all methods faster**.

## Hypothesis

One of the methods involved in setting the dispatching mechanism in place is the [class_addMethod](https://developer.apple.com/documentation/objectivec/1418901-class_addmethod), which receives a `types` parameter of type `const char *`/`UnsafePointer<CChar>?`. This is basically a pointer (`*const usize`) to a sequence of N bytes (`u8` for Rust devs 🦀). So in the case of the `Bii` type signature, this would require a C call similar to `class_addMethod(objc_getMetaClass("CoolClass"), @selector(...), MyMethodParamAParamB, "Bii")`.

For the Objective-C runtime to actually call the correct method, it will need to do some sort of "pattern matching" on the `"Bii"` type signature, aka compare strings until it finds the correct 3 bytes sequence that maches. This can be made more efficient by converting the multiple byte comparison with a simple enum/integer one.

In practice the optimization is to:

1. At compile time store all the possible method type signatures used by the source code in an enum as distinct variants;
  - Eg: compiling a program that has three functions with two distinct signatures (one of them is repeated), will generate an enum with two possible variants.
2. All of the places that use the current Type Encoding as a byte sequence should use this new unsigned integer enum instead, so the comparisons are faster and performance is (hopefully) improved at execution time.

## Experiment

1. The runtime should be more deeply studied to really find out how the type signatures are matched and choosen at a regular program execution;
2. The hypothesis should be implemented;
3. Benchmarks should be executed before and after the optimization;
  - In a simple scenario: loop with a few different method calls;
  - In a real program.
4. Then compare the results before and after the optimization is set in place.

## Resources

- [runtime source](https://opensource.apple.com/source/gcc/gcc-5482/libobjc/)
- [Objective-C Runtime talk](https://www.youtube.com/watch?v=G1780anbyM8)
