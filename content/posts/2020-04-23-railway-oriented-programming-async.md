+++
title = "Asynchronous Functions and the Railway Oriented Programming Pattern"
[taxonomies]
tags = [ "functional programming", "railway oriented programming", "reactive programming" ]
+++

I've always had a fascination with functional programming ever since taking a class on functional programming using the OCaml programming language when I was in undergraduate school. Expressive type systems, higher order functions, immutability, and the ability to be explicit about where functions have side effects can make it easier to write correct programs. Although, there's no denying that there can be a steep learning curve for those of us who have gotten used to writing code in an imperative style.

As I continue to put into practice functional programming concepts I try to learn about some of the different patterns bright minds have come up. One such pattern I was reading about a while ago that intrigued me is dubbed "Railway Oriented Programming" by Scott Wlaschin.

So what is "Railway Oriented Programming" or ROP? Railway Oriented Programming is a pattern for handling branches in logic within your program in a clean and concise way. Instead of nesting if statements or using exceptions you use the power of function composition and a static type system to chain functions together. This allows you to choose which branch of logic to go to next and to skip over any remaining functions in the chain when an error is encountered.

Mr. Wlaschin uses the anology of of railway switches to visualize how this control flow pattern works. I think this pattern is particularly useful for modeling the error handling for your domain logic. It allows handling of errors in a type safe way and helps the developer to think about both the happy and not so happy paths of the application logic. It's important to note this isn't meant to be a replacement for exceptions but it can be used as a different way to think about and model the control flow of your domain logic.

If you are not familiar with this concept you will probably find it helpful to read these posts first to get some context as Mr. Wlaschin can explain the concept better than I can:

- [Railway Oriented Programming - Scott Wlaschin (I highly recommend watching the video of his talk)](https://fsharpforfunandprofit.com/rop/)
- [Against Railway Oriented Programming - Scott Wlaschin](https://fsharpforfunandprofit.com/posts/against-railway-oriented-programming/)

So you've read these articles and you agree that is this is cool idea and want to start using it where it makes sense in your code base. This pattern works great if your switch functions are all synchronous and don't need to wait for an HTTP call, file system operation or database query. You simply start with your initial input and pass that input through your switches along your proverbial railway. But this is the real world and most of us have to interact with systems outside of our program. I didn't real find much in my reading to see how anyone who used this pattern handled asynchronous operations outside of F#. I attempted to use this pattern with TypeScript recently and with TypeScript since it is JavaScript under the hood we need a way to handle asynchronous functions.

So lets explore what happens when you throw asynchronous functions in the mix.

Let's use the example validating user input for an API. We'll use the functional programming helper library [True Myth](https://github.com/true-myth/true-myth) for demonstration purposes as it includes an implementation of the Result type and helper functions to wrap/unwrap values and chain together function calls.

Let's consider this example:

```typescript
import { Result, Ok, Err } from 'true-myth';

enum UserValidationError {
    USERNAME_EMPTY = 'Username cannot be empty!'
    USERNAME_INVALID_CHARS = 'Username must only contain alphanumeric chars and dashes/underscores',
}

class User {
    constructor(
        readonly username: string,
        readonly firstName: string,
        readonly lastName: string,
    ) {}
}

const VALID*USERNAME_REGEX = /^[A-Za-z0-9*-]+$/;

function validateUsernameNotEmpty(input: User): Result<User, UserValidationError> {
    if (input.username.length === 0) {
        return Result.err(UserValidationError.USERNAME_EMPTY);
    }
    return Result.ok(input);
}

function validateUsernameHasValidChars(input: User): Result<User, UserValidationError> {
    if (!input.username.matches(VALID_USERNAME_REGEX)) {
        return Result.err(UserValidationError.USERNAME_INVALID_CHARS);
    }
    return Result.ok(input);
}

function printValidUser(input: User) {
    console.log('User is valid: ', input);
}

function printError(error: UserValidationError) {
    console.log('User is invalid: ', error);
}

const newUser = User('', 'Jon', 'Minter');
Result.ok(newUser)
    .andThen(validateUsernameNotEmpty)
    .andThen(validateUsernameHasValidChars)
    .match({
        Ok: user => printValidUser(user),
        Err: error => printError(error),
    });
```

We can see how this pattern might be useful for a few reasons (some of these are not limited to this pattern):

- Encourages us to write small, focused functions and chain them together, these small functions are easier to test
- We can use descriptive names for each function and makes it easy to see what our code is supposed to do by looking at functions that are chained together
- We can use use pattern matching to handle failures in the same way regardless of where in the chain of operations we diverted to our failure path
- Since we are using a language that doesn't provide checked exceptions this provides us a way to handle errors and know the type of the error, where using exceptions or Promise error callbacks we just get _some_ error back. We have no way of knowing the error's type
- If an exception occurs we know it was truly from something exceptional and we can let that exception bubble up to our main exception handler

So this is all well and good, but what if for example we need to validate that the username isn't in use by another user? We'd probably have to make a call to an API or run a query against a database to check if the username is unique. And that call is going to be an asynchronous call that returns a promise. Why is this an issue? Let's consider this addition to our code:

```typescript
...

async function validateUsernameIsUnique(input: User): Promise<Result<User, UserValidationError>> {
const userCount = await getCountOfUsersWithUsername(input.username);
if (userCount !== 0) {
    return Result.err(UserValidationError.USERNAME_NOT_UNIQUE);
}
return Result.ok(input);
}

...

Result.ok(newUser)
    .andThen(validateUsernameNotEmpty)
    .andThen(validateUsernameHasValidChars)
    .andThen(validateUsernameIsUnique)
    .match({
        Ok: user => printValidUser(user),
        Err: error => printError(error),
    });
```

So what's the problem here? Remember when we create an async function in JavaScript this is syntactic sugar for converting the function into a function that returns a promise. So if you tried to compile this code it would fail compilation since it no longer returns `Result<User, UserValidationError>` but `Promise<Result<User, UserValidationError>>` so it cannot be included in the chain of functions.

So you might say why not just use promises instead of this Result type? Well we could but since a promise doesn't enforce a failure type the TypeScript compiler can only guarantee the types along the happy path and not the failure path. A promise failure is just _some_ error type, it could be a string, it could be an Error object, it could be a number, or anything.

Another alternative is use the Result type and then every time we get to a point where we need to perform an async operation switch to using promises and then back to our ROP pattern.

Example:

```typescript
...

async function doValidation(newUser: User) {
let validationResult = Result.ok(newUser)
.andThen(validateUsernameNotEmpty)
.andThen(validateUsernameHasValidChars);

    if (validationResult.isOk()) {
        validationResult = await validateUsernameIsUnique(validationResult.value);
    }
    validationResult
        .match({
            Ok: user => printValidUser(user),
            Err: error => printError(error),
        });

}
```

This works but it's messy and forces us use a different pattern for synchronous vs asynchronous logic. This is a fairly simple and contrived example, but imagine trying to maintain a large codebase having to do this switching between sync/async logic. We don't have our clean railway pattern anymore.

Also, what if we wanted to have the ability to collect a list of functions to chain together at runtime and some or all of those function are asynchronous? We aren't able to do that either.

```typescript
const validators = [
  validateUsernameNotEmpty,
  validateUsernameHasValidChars,
  validateUsernameIsUnique,
];

validators.reduce((result, validator) => {
  return result.andThen(validator); // Oops can't do this if one of these return a Promise
}, Result.ok(input));
```

So how could we solve this? Is there a way to model this so we can handle both async and sync functions and still use this ROP pattern?

Let's think about what's happening when we chain these functions together. When we're working with synchronous functions every time we call True Myth's `andThen` function we pass it a switch function and it applies that function to the current `Result` object we have at the time. When we're working with asynchronous functions we aren't returning an actual `Result` object but a promise that there will be a `Result` object at some point in the future. So we need some way to queue up the functions along our railway and only execute them when the previous function's promise has resolved.

So we can write up a simple helper class that abstracts that logic for us and provides a nice fluent interface. Keeping with our railway analogy we'll call this helper `AsyncRailway`:

```typescript
// Let's create a type alias for a Promise of a Result just to save us some typing
type AsyncResult<S, E> = Promise<Result<S, E>>;
// ...alias type for our async switch functions
type AsyncSwitch<S, E> = (input: S) => AsyncResult<S, E>;
// ...and finally an alias for our synchronous switch functions
type Switch<S, E> = (input: S) => Result<S, E>;

function convertAsync<S, E>(syncSwitch: Switch<S, E>): AsyncSwitch<S, E> {
  return async (input: S) => {
    return Promise.resolve(syncSwitch(input));
  };
}

class AsyncRailway<S, E> {
  private switches: Array<AsyncSwitch<S, E>> = [];
  constructor(private readonly input: Result<S, E>) {}

  static leaveTrainStation<S, E>(input: Result<S, E>) {
    return new AsyncRailway(input);
  }

  andThen(switchFunction: AsyncSwitch<S, E>) {
    this.switches.push(switchFunction);
    return this;
  }

  async arriveAtDestination(): AsyncResult<S, E> {
    return this.switches.reduce(async (previousPromise, nextSwitch) => {
      const previousResult = await previousPromise;
      return previousResult.isOk()
        ? await nextSwitch(previousResult.value)
        : previousResult;
    }, Promise.resolve(this.input));
  }
}
```

So with this helper class we can now do this:

```typescript
await(
  AsyncRailway.leaveTrainStation(Result.ok<User, UserValidationError>(someUser))
    .andThen(convertAsync(validateUsernameNotEmpty))
    .andThen(convertAsync(validateUsernameNotEmpty))
    .andThen(validateUsernameIsUnqique)
    .arriveAtDestination()
).match({
  Ok: (user) => printValidUser(user),
  Err: (error) => printError(error),
});
```

Awesome that works! However, what if our functions are transforming the initial input from one type to another? For example suppose we want to add a function to the chain that saves the user account to our database and returns the inserted user's identifier:

```typescript
...
enum UserSaveError {
DB_SAVE_FAILED = "Failed saving user to the database!"
}
type UserError = UserValidationError | UserSaveError;

class SavedUserAccount {
constructor(readonly newUserId: string) {}
}

async function saveUser(input: User): Promise<Result<SavedUserAccount, UserError>> {
// Let's pretend we made a call to our user database to save the user and
// this is the new user's user ID
return Promise.resolve(new SavedUserAccount(uuid4()));
}

...

await (AsyncRailway
    .leaveTrainStation(Result.ok<User, UserError>(someUser))
    .andThen(convertAsync(validateUsernameNotEmpty))
    .andThen(convertAsync(validateUsernameNotEmpty))
    .andThen(validateUsernameIsUnqique)
    .andThen(saveUser) // Compilation error here because we're returning Result<SavedUserAccount>
    .arriveAtDestination())
    .match({
        Ok: user => printValidUser(user),
        Err: error => printError(error)
    });
```

Whoops! This doesn't compile anymore because our `AsyncRailway` class only allows us to collect and use switch functions that act on and return the same type.

So how could we solve this? There are ways we could implement this with overloaded function definitions and create something that would allow us create a pipeline of transformation functions but there is a simpler solution. We can instead harness the power of RxJS the reactive extensions framework for JavaScript and create a custom operator that would allow us to achieve this same result and still allow us to transform inputs to a different output type with just a few lines of code.

If you aren't familiar with RxJs or reactive programming check out these links for some context:

- [What is Reactive Programming - Andre Stalz](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)
- [Learn RxJs](https://www.learnrxjs.io/)

Here's an implementation idea using a single custom RxJs operators to allow this:

```typescript
import { of, OperatorFunction, pipe } from "rxjs";
import { flatMap } from "rxjs/operators";

type AsyncSwitchTransform<I, O, E> = (input: I) => AsyncResult<O, E>;

function asyncAndThen<I, O, E>(
  railwaySwitch: AsyncSwitchTransform<I, O, E>
): OperatorFunction<Result<I, E>, Result<O, E>> {
  return pipe(
    flatMap((x) => {
      return x.isOk() ? railwaySwitch(x.value) : of(x as Err<any, E>);
    })
  );
}
```

Let's break down what this is doing.

First, we define an type alias so we have a shorthand type for defining our switch functions that can optionally transform the input type to a different output type.

Second, we define an RxJs operator by creating a function that returns an `OperatorFunction` that RxJs can use to transform the `Observable`. This function uses the RxJs `pipe` function and uses the `flatMap` operator with a function that will unwrap the `Result` object, check if there was an error and if not pass the value to the next switch function in the railway sequence. However if the `Result` object contains an error then we short circuit and return the `Error`. This is almost identical to the promise based version above.

Why use `flatMap` instead of the RxJs `map` operator? Remember since we're working with promises the switch function is returning a promise of a future `Result` object rather than the `Result` object itself. So we end up with an observable item that contains a promise of a `Result` and the `flatMap` operator will handle the flattening for us so that at the end of all our operations we end up with an `Observable<Result<User, UserValidationError>>` instead of `Observable<Promise<Result<User, UserValidationError>>>`.

So here's how we would use this new operator:

```typescript
const johnPublic = new User("johnqpublic", "John", "Public");
of(Result.ok(johnPublic))
  .pipe(
    asyncAndThen(convertAsync(validateUsernameNotEmpty)),
    asyncAndThen(convertAsync(validateUsernameHasValidChars)),
    asyncAndThen(validateUsernameIsUnqique),
    asyncAndThen(saveUser)
  )
  .subscribe((result) => {
    result.match({
      Ok: (user) => printSavedUser(user),
      Err: (error) => printError(error),
    });
  });
```

So now that we've defined this operator we can use our railway pattern and use it with functions that transform the input and still have the power of TypeScript's type system to ensure for example we've put our switch functions in the right order and that they can only be used with the types they are defined for.

Another benefit of using RxJs is that we can now run our sequence of switch functions on a stream of items allowing us to use the same sequence of transformations whether we are working with a single item or many items. For example you could use the same logic to import a batch of users into your user database that you use to create a single user.

So that's it, I hope someone finds this useful or thought provoking. You can see a [complete code example](https://www.github.com/jonminter/railway-oriented-programming-async) on my github.
