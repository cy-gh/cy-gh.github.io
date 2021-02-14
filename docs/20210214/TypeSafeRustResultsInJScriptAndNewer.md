# Type-safe Rust-like Result in VSCode for JScript and newer

For my multi-threaded hasher project, I needed something more flexible than simple exceptions/errors which JScript allows ([which roughly corresponds to JS 1.5/ES3](https://johnresig.com/blog/versions-of-javascript/)). On top of it the lack of a better runtime like modern browsers, thus the lack of a debugger and stacktrace of exceptions, i.e. no `e.stack`, made me think hard. Never underestimate the value of a debugger and how that ties your hands if you have none. Last but not least, in particular for JScript, sooner or later your code must run in Windows Scripting Host (WSH) JScript environment and your .js must be parseable by WSH. These are big obstacles.

Luckily, the combo VSCode+JSDoc+TSC help us immensely at design-time. And the solution below works in any modern JavaScript environment with or without a debugger as well and the reasons why I prefer Result to Exception/Error is explained below in detail.

In short, I found no easy way of making VSCode/TSC & ESLint warn me of potential exceptions which might be raised and I should put try-catch around a function call. This was **the** big showstopper for me because I am used to languages from my day job automatically warn me that I **have to** handle exceptions for my own good. With Result object, VSCode forces me to adjust my code as soon as I adjust a low-level method from `@return {simpleType}` or `@return {successType|false}` to `@return {Result.<successType, errorType>}`. Now we can enforce a tiny little bit more static checks in a highly dynamic and marvelous language like JS without having to set up an overkill infrastructure with unit tests, code coverage, etc.

![](TypeSafeRustResultsInJScriptAndNewer-ModernSolutions.jpg)

All JScript talk left aside, this works in ECMAScript and TypeScript as well, with the same benefits.

## Short answer

- Create `NewFile.d.ts` file with the interface definition:

```javascript
interface Result<T, E> {
    ok: T;
    err: E;
    new <T, E>(okValue: T, errValue: E): Result;
}
```

- Put the link to `NewFile.d.ts` at the top of your JScript, JavaScript, etc.

```javascript
///<reference path="./NewFile.d.ts" />
// @ts-check
```


- Implement the Result constructor in your .js:

```javascript
/**
 * Generic Result object
 * @param {any} okValue value on success
 * @param {any} errValue value on error/failure
 */
function Result(okValue, errValue) {
    this.ok    = okValue;
    this.err   = errValue;
}
// note this one does not allow any falsy value for OK at all
Result.prototype.isOk      = function () { return this.ok; };
// note this one allows falsy values - '', 0, {}, []... - for OK - USE SPARINGLY
Result.prototype.isValid   = function () { return !this.err; };
Result.prototype.isErr     = function () { return !!this.err; };
Result.prototype.toString  = function () { return JSON.stringify(this, null, 4); };
/**
 * Result wrapper
 * @param {any=} okValue
 * @returns {Result}
 */
function ResultOk(okValue) {
    return new Result(okValue||true, false);
    // or to allow falsy values
    // return new Result(okValue, false);
}
/**
 * Result wrapper
 * @param {any=} errValue
 * @returns {Result}
 */
function ResultErr(errValue) { return new Result(false, errValue||true); }
```


- Now you can use it, eg by defining the Result types in your JSDoc.

```javascript
/* @returns {Result.<number, string>} number on success, error string on error */
function doStuff() {
    // note the @returns {Result.<number, string>}
    var preRequisite = true; // hard-coded value for brevity
    if (!preRequisite) return ResultErr('Some error happened (...details...)');
    var successResult = 42; // hard-coded value for brevity
    return ResultOk(successResult);
}
function caller() {
    var res = doStuff();
    if (res.isErr()) { /* react to it */ }
    // now use res.ok as you wish, eg res.ok.toPrecision(2)
    // VSCode can infer res.ok's type automatically, in this case 'number'.
    // if you change the return type of ok in doStuff() in the future
    // VSCode will automatically warn you here in this function
    // and that toPrecision() is not available.
}


// it also works for any custom type, i.e. for custom classes you define
/** @param {number} num */
function Thread(num) {
    this.id = num;
}
/** @returns {Result.<Thread, true>} */
function otherDoStuff() {
    // if (...) return ResultErr(); // true will be auto-assigned to err property
    return ResultOk(new Thread(new Date().getTime()));
}
function otherCaller() {
    var res = otherDoStuff();
    // typing res.ok. and pressing Ctrl-Space will show Thread's id field automatically
    // console.log(res.ok.id);
}

/** @returns {Result.<string, string>} */
function yetAnotherDoStuff(value) {
    // note the {Result.<string, string>}
    // you can not do this with @returns {type1|type2} alternative,
    // the success/failure information is now decoupled from the values
    // and tied to 2 distinct fields in Result.
    // you do not have to define constants, hard-coded values, enums and so on
    // to signify one value as success, another one as error.
    var check = true; // hard-coded value for brevity
    if (check) { return ResultOk('value passed the check'); }
    else { return ResultErr('value failed the check'); }
}
```


- Win! WSH will have no clue what's going on under its nose :D



VSCode uses the .d.ts and JSDoc information to infer the types in `result.ok` and `result.err` automatically. It works for JS-builtin objects like boolean, string, etc. as well as custom classes you define. Compared to the simpler alternative `@returns {successType|false}` this is much more standardized, and if you prefer you can also use falsy values as `ok`. Also in the simpler alternative, your `err` value must be always distinct from any `ok` value, whereas with Result you can also use the same type in both `ok` and `err`, eg both strings or both booleans, which is a huge advantage.


## Long answer

The short answer covers everything if you're only interested in Result in JScript, JS or ES. Read on if you are bored :)

### First attempt

Since there is no stack trace in DOpus, I wanted to know where exactly the error occurred. It had been a while since I developed in JavaScript and learned that JavaScript does not allow to find out in which method we are. Autohotkey for example has a magic `A_ThisFunc` variable, which always knows whichever function or class method we are in, no such thing in JS. A very old and obsolete `arguments.caller` does not work in JScript either.

So I developed this:

```javascript
var funcNameExtractor = (function () {
    var cache       = {},
        reExtractor = new RegExp(/^function\s+(\w+)\s*\(/);
    /**
     * I know that is is a very ugly solution, but it works
     * @param {function} fnFunc
     * @returns {string}
     * @throws
     */
    return function(fnFunc) {
        if (cache[fnFunc]) return cache[fnFunc];
        if (typeof fnFunc !== 'function') throw new Error('...');
        var matches = fnFunc.toString().match(reExtractor),
            out     = matches ? matches[1] : 'anonymous func';
        cache[fnFunc] = out;
        return out;
    };
}());
function sayMyName(param) {
    var fnName = funcNameExtractor(sayMyName);
    // prints out 'My name is: sayMyName()...'
    DOpus.Output('My name is: ' + fnName + '(), and param is: ' + param);
}
var obj1 = {
    read: function() {
        // this does not work
        var fnName = funcNameExtractor(obj1.read);
        // fnName = 'anonymous func'
    }
};
var obj2 = (function() {
    function fnRead() {
        // the problem with obj1 can be easily fixed like this
        // and this does work but code has redundancy now.
        // this has legitimate uses such as data encapsulation
        // but often I don't need encapsulation.
        var fnName = funcNameExtractor(fnRead);
        // fnName = 'fnRead'
    }
    return { read: fnRead };
}());
```

Problem is this makes the code verbose and does not work for anonymous functions, like callbacks or methods like in obj1 above.

Then I gave up hope and tried to get more help from JS Exceptions/Errors.

### JS Exceptions

#### Advantages

* The advantage of exceptions are: In a bigger call-chain, you don't have to catch them at every step, they are automatically bubbled up, but have their price, performance and coding-wise.

#### Disadvantages

* In JScript/WSH, Exceptions do not carry a StackTrace in them, you have no idea where they might have been raised. In order to track it, you can insert a "where" property manually where you raise them or put the same info in the message.

  I tried that and it makes the code uglier and bloated; although this might be unavoidable if detailed callstack is needed.

  ```javascript
  function entryPoint() {
      // should I put try-catch around it or not? no idea until I manually check everything below.
      outerWrapper();
      // -> innerWrapper() -> onDOpusOnTheFlyCalculateAndExport() -> NotImplementedYetException!
  }
  function outerWrapper() { return innerWrapper(); }
  function innerWrapper() { return calculateAndExport(); }
  function calculateAndExport() {
      throw new NotImplementedYetException('I had no time yet', calculateAndExport);
  }
  // ...
  function anotherEntryPoint() {
      try {
          someMethod();
      } catch(e) {
          // this catch never executes if the throw below has been disabled
          // would print out 'FileReadWriteException @someMethod: something bad happened'
          DOpus.output(e.name + ' @' + e.where ': ' + e.message);
      }
  }
  function someMethod() {
      // DISABLED: throw new FileReadWriteException('something bad happened', someMethod);
  }

  /**
   * I needed something like an interface, where all Exceptions implement the same interface
   * so unfortunately I had to implement this kinda ugly solution.
   * Don't blame me, blame JScript.
   * And I hope you realize what the consequences of lack of a debugger and a callstack trace are.
   * @param {function} fnCaller
   * @param {string} message
   * @param {string} where
   * @constructor
   */
  function UserException(fnCaller, message, where) {
      this.name    = funcNameExtractor(fnCaller);
      this.message = message;
      this.where   = typeof where === 'string'
          ? where
      	: typeof where === 'function'
          	? funcNameExtractor(where)
      		: where;
  }

  /** @constructor @param {string} message @param {string} where */
  function NotImplementedYetException(message, where) {
      this.message='';
      this.name='';
      this.where='';
      UserException.call(this, NotImplementedYetException, message, where);
  }
  /** @constructor @param {string} message @param {string} where */
  function FileReadWriteException(message, where) {
      this.message='';
      this.name='';
      this.where='';
      UserException.call(this, FileReadWriteException, message, where);
  }
  // ... other exceptions ...

  ```



* But even with the where info, there is no way of knowing which callstack an exception goes through, between the point where they were raised and the point where they are caught. For small programs it's not a big deal to keep an overview, but if scripts grow and some common methods are called at every possible hierarchy level, we lose track.

  All because of not having a debugger and stack trace :/

* Also, some exceptions are easily recoverable but without a catch block and more importantly **a complete rework of all JSDocs** from all methods which raise an exception to all their callers, and their callers' callers, there is no way of knowing if a method call on a higher level should expect an exception.


This can be simply "fixed" though if all top level methods wrap sub-calls within try-catch blocks, i.e. we handle exceptions only on higher levels, not intermediate/deeper levels, etc.

  Unfortunately unlike in other programming languages like Java, neither VSCode nor ESLint/JSHint warn you of missing try-catch around a call which might raise an exception and so **we might end up putting unnecessary try-catch's everywhere**.

### To the rescue: Rust Result's

Is there a better solution? I believe so.

I've been dabbling in [Rust](https://www.rust-lang.org/) on and off for some time. Though I'm still a newbie, I know it better than I do TypeScript. Some of Rust's concepts are fascinating, like the [Result](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html) enums. Simple yet effective. Results have 2 possible values: Ok or Err. Whenever a method call might return a recoverable error, you can check if either its Ok or Err field is populated and proceed. Both fields are type-safe.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}

use std::fs::File;
use std::io;
use std::io::Read;
fn read_username_from_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");
    let mut f = match f {
        Ok(file) => file,
        Err(e) => return Err(e),
        // alternative:
        // Err(e) => panic!("Problem opening the file: {:?}", e),
    };
    let mut s = String::new();
    match f.read_to_string(&mut s) {
        Ok(_) => Ok(s),
        Err(e) => Err(e),
    }
    // f is the file handle on success or io::Error
    // Rust has more concise syntax, this is only for demonstration
}
```



To simulate Rust's `panic!` we can still use `throw new Error()` and just let it bubble up but for recoverable errors Result is superior option to JS exceptions IMO.

As mentioned above, in VSCode or ESLint there is no help for missing try-catchs where they are necessary. At first I thought it would be never possible to use them in JScript of all languages but turns out it was much easier than I feared. **The type-safe definition comes from .d.ts file, and the method return definitions come from JSDoc parts, and from WSH's point of view, it's all valid JScript.** Obviously type-safety is guaranteed only in design-time through this trick. If you handle external input in outermost layers of your code, you still have to validate them before passing them to inner layers to ensure runtime safety but it's something we have to do in JS/ES anyway.

The big personal benefit is **Results force you to revisit every piece of the call chain** as soon as you change low-level functions, because VSCode starts complaining that you cannot use `result` directly in the caller as before, so you revisit every caller and add one ore more of the `result.ok, result.err, result.isOk(), result.isErr()`. Then once you adjust the own @return parameter of the caller, VSCode warns that the caller's caller cannot use `result` directly and so on.

You might ask:

>  "Do I need this paradigm shift and all this effort? I don't use archaic stuff like JScript and I already have debugger & stacktrace."

Not so fast. In fact, this technique works with EcmaScript as well. Of course the same benefit with enforcing a try-catch alike handling applies in ES as well.

#### JScript-only variation with stack trace (not automatically of course, don't be ridiculous :D)

With some modifications to the Result constructor, the callstack can be simulated as well. Every time you "catch" a `result.err`, we can raise a new `ResultErr` using the previous result object and append it to an internal array attribute in the result. Of course this could be done with exceptions and try-catch as well, because It's basically catching and re-throwing.

Mind you, the above solution is more universal, the one below applies to mainly to rather unique DOpus+JScript combination.

Change the methods as follows. Normally only the `ResultErr` should be changed, since we usually don't need a stack on success but both functions are adjusted for sake of completeness and OCD:

```javascript
/**
 * Generic Result object
 * @param {any} okValue value on success
 * @param {any} errValue value on error/failure
 */
function Result(okValue, errValue) {
    this.ok    = okValue;
    this.err   = errValue;
    this.stack = [];
}
/**
 * wrapper for Result
 * @param {any=} okValue
 * @param {any=} addInfo
 * @returns {Result}
 */
function ResultOk(okValue, addInfo) {
    var res = okValue instanceof Result ? okValue : new Result(okValue||true, false);
    if (addInfo) res.stack.push(addInfo);
    return res;
}
/**
 * wrapper for Result
 * @param {any=} errValue
 * @param {any=} addInfo
 * @returns {Result}
 */
function ResultErr(errValue, addInfo) {
    var res = errValue instanceof Result ? errValue : new Result(false, errValue||true);
    if (addInfo) res.stack.push(addInfo);
    return res;
}
// putting the stack handling in the wrapper is more preferable
// than putting them in Result itself, which would over-complicate the constructor.
```

Now if you have the following, he top caller will have all the call chain:

```javascript
function main() {
    var tmpErr = outerErrWrapper();
    DOpus.output(JSON.stringify(tmpErr, null, 4));
    /* prints out: {
        "ok": false,
        "err": "Something bad happened",
        "stack": [
            "lowLevelErrFunction",
            "innerErrWrapper",
            "outerErrWrapper"
        ]
    }*/
}
function outerErrWrapper() {
    return ResultWithStackErr(innerErrWrapper(), 'outerErrWrapper');
}
function innerErrWrapper() {
    return ResultWithStackErr(lowLevelErrFunction(), 'innerErrWrapper');
}
function lowLevelErrFunction() {
    return ResultWithStackErr('Something bad happened', 'lowLevelErrFunction');
}
```

Chances are you will never need it, nor do I. It's there because I could :)
