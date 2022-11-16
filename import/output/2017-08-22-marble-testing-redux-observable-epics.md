---
title: "Marble testing redux-observable epics"
date: "2017-08-22"
categories: 
  - "react"
  - "rx"
  - "redux"
  - "testing"
---

One great benefit of defining the sequence of redux actions as an observable is the ability to mock the scheduler to unit-test the asynchronous behavior of the application, redux-observable _epics_ in particular. One can even take advantage of the _marble testing_ approach of RxJS to make for particularly precise descriptions of the action stimuli and the expected outcome.

# Epic definition

We can illustrate this with the classical example of debouncing search input. To setup such a behavior with redux-observable, we will define two epics: a first epic concerned with debouncing search input and issuing `search` actions, and a second epic in charge of carrying out the actual search.

```
// This epic drives the as-you-type experience, throttling the input every 100ms to avoid overloading the server
export const searchOnInputChangedEpic =
    (action$: ActionsObservable<Actions>, store: MiddlewareAPI<SearchState>, services: IServices): Rx.Observable<Actions> => {
        return action$
            .actionsOfType<InputChangedAction>(InputChangedActionType)
            .map(a => a.payload)
            .debounceTime(100, services.scheduler)  
            .distinctUntilChanged()
            .map(search);
    };

// This second epic drives the search, submitting searches to the service, dealing with responses and cancelling in-flight searches if no longer relevant
export const searchEpic =
    (action$: ActionsObservable<Actions>,
     store: MiddlewareAPI<SearchState>, services: IServices): Rx.Observable<Actions> => {
        return action$
            .actionsOfType<SearchAction>(SearchActionType)
            .map(a => a.payload)
            .switchMap(input =>
                input.length > 0 ?
                    services.search.doSearch(input)
                        .map(p => new ContentResult(p))
                        .catch((error: Error) => 
                             Rx.Observable.of<SearchResult>(new ErrorResult(error.message))) 
                    : Rx.Observable.of<SearchResult>(EmptyResult.Instance))
            .map(searchFulfilled);
    };
```

Let's now look more in detail at each component of these epics, starting with the signatures:

## Epic Signature

```
const searchOnInputChangedEpic = (action$: ActionsObservable<Actions>, store: MiddlewareAPI<SearchState>, services: IServices): Rx.Observable<Actions>
```

You might have noticed that we are "injecting" dependencies through an `IServices` interface, which provides the services required by in the epic logic, as well as the Rx scheduler. Epic middleware indeed allows passing an object as the third parameter of an epic. Here's how this interface is defined:

```
import { ISearchService } from "services/search";
import { IScheduler } from "rxjs/Scheduler";

export interface IServices {
    readonly search: ISearchService;
    readonly scheduler: IScheduler;
}
```

The actual `IServices` instance is configured at the middleware creation, like so:

```
const services = { 
   search: new SomeSearchService(),
   scheduler:  Rx.Scheduler.async // setup the default rx scheduler for epics as the async scheduler, to be overriden in tests
};
const epicMiddleware = createEpicMiddleware(rootEpic, { dependencies: services });
```

When unit testing, this interface is mocked with e.g. a mock search service and Rx's `TestScheduler` instead of the standard one.

## Action selection

The following lines only lets input\_changed and search actions filter through, respectively:

```
action$.actionsOfType<InputChangedAction>(InputChangedActionType)
```

```
action$.actionsOfType<SearchAction>(SearchActionType)
```

`actionsOfType<T>()` is a helper operator which both calls redux-observable built-in `ofType()` operator and typecasts the action to the proper type:

```
export function actionsOfType<A>(this: ActionsObservable<any>, type: string): Observable<A> {
    return this.ofType(type).map(t => (t as A));
}

// Add the operator to the Observable prototype:
ActionsObservable.prototype.actionsOfType = actionsOfType;
```

## Input processing

The next part of the input\_changed epic "pipeline" performs input throttling, and only emits searches if the input actually changed between debounces:

```
.map(a => a.payload)
   .debounceTime(100, services.scheduler)  // 100ms throttling delay - important: we specify the scheduler here, to allow for testing
   .distinctUntilChanged()
```

Note that we are passing our injected scheduler to the `debounceTime` operator. This is essential, since it will allow us to unit test debouncing behavior by injecting a `TestScheduler` instance rather than the standard asynchronous scheduler.

## Launch the actual search

The second `searchEpic` launches a search query for each search action emitted by the first epic and deals with the response, also cancelling any prior search that would still be in-flight (thanks to the `switchMap` operator).

```
.switchMap(input =>
                input.length > 0 ?
                    services.search.doSearch(input)  // delegating to the injected search service interface here the actual request emission
                        .map(p => new ContentResult(p))
                        .catch((error: Error) => Rx.Observable.of<SearchResult>(new ErrorResult(error.message))) 
                    : Rx.Observable.of<SearchResult>(EmptyResult.Instance))
```

It is important to note here that we are not emitting requests directly, but rather using our `ISearchService` abstraction. This will make unit testing easier, since epic tests will not have to deal will the additional concern of mocking http requests (e.g. via libraries such as _nock_).

## Action creators

The final line `.map(searchFulfilled);` concludes the epic, by mapping the search result with the `searchFulfilled` action creator function, which is defined like so:

```
export const searchFulfilled = (response: SearchResult): SearchFulfilledAction => createAction(SearchFulfilledActionType, response);
```

You might have noticed that actions are typed. `SearchFulfilledAction` is defined in the following way:

```
export type SearchFulfilledAction = TypedAction<typeof SearchFulfilledActionType, SearchResult>;
```

The `TypedAction<T, P>` is a helper type, with the accompanying `createAction` function

```
export interface TypedAction<T, P> extends Action {
    readonly type: T;
    readonly payload: P;
    error?: boolean;
    meta?: {};
}

export function createAction<T extends string, P>(type: T, payload: P): TypedAction<T, P> {
    return ({type, payload});
}
```

You might have noticed also that our epic signature is typed with an `Actions` type. This is a discriminated union allowing for a common type for all actions:

```
export type Actions =
    InputChangedAction
    | SearchAction
    | SearchFulfilledAction
    | ...
```

# Unit testing

With our epics now defined, let's proceed to to unit testing. We will be taking advantage of Rx's `TestScheduler` as well as a mocking library to mock our services (in our case we are using [TypeMoq](https://github.com/florinn/typemoq)).

## Testing debouncing behavior

Here's the test for `searchOnInputChangedEpic`:

```
 test("searchOnInputChangedEpic debounces input", () => {
        // Create test scheduler
        const testScheduler = createTestScheduler();

        // Mock services instance, passing in our test scheduler
        const servicesMock = TypeMoq.Mock.ofType<IServices>();
        servicesMock.setup(m => m.scheduler).returns(() => testScheduler);
        const services = servicesMock.object;

        // Define marbles
        const inputActions = {
            a: inputChanged("r"),
            b: inputChanged("rx"),
            c: inputChanged("rxjs")
        };
        const outputActions = {
            b: search("rx"),
            c: search("rxjs")
        };
        const inputMarble =     "-ab-----------c";
        const outputMarble =    "------------b-----------c";

        // Mock input actions stream
        const action$ = createTestAction$FromMarbles<Actions>(testScheduler, inputMarble, inputActions);

        // Apply epic on actions observable
        const outputAction$ = searchOnInputChangedEpic(action$, store, services);

        // Assert on the resulting actions observable
        testScheduler.expectObservable(outputAction$).toBe(outputMarble, outputActions);

        // Run test
        testScheduler.flush();
    });
```

Let's break this down into its parts:

### Test scheduler

The first line

```
const testScheduler = createTestScheduler();
```

creates Rx `TestScheduler` instance. It is wrapped in a helper function, since this is used recurrently in all the tests. Here's its definition:

```
import * as Rx from "rxjs";
export const createTestScheduler = () => new Rx.TestScheduler((actual, expected) => expect(actual).toEqual(expected));
```

The `TestScheduler` supports writing "[Marble Tests](https://github.com/ReactiveX/rxjs/blob/master/doc/writing-marble-tests.md)", as we will see below. It allows specifying the deep equality used for comparing values in the marbles: we are using the standard equality in our case.

### `IServices` mocking

With the following block

```
const servicesMock = TypeMoq.Mock.ofType<IServices>();
servicesMock.setup(m => m.scheduler).returns(() => testScheduler);
const services = servicesMock.object;
```

we are mocking the `IServices` instance which our epic relies upon. We first grab a `Mock` instance typed with `IServices` from TypeMoq, which allows setting up the behavior for the mock with a fluent API. In our case, we set it up so that a call to the `scheduler` property returns our `testScheduler` (the `search` property is left out for now, since this test will not be looking at the search proper). Finally we obtain the actual `IServices` instance by means of TypeMoq's \`.object' operator.

### Input and output marbles

Here's the interesting part of the test and where the magic happens:

```
const inputActions = {
    a: inputChanged("r"),
    b: inputChanged("rx"),
    c: inputChanged("rxjs")
};
const outputActions = {
    b: search("rx"),
    c: search("rxjs")
};
const inputMarble =     "-ab-----------c";
const outputMarble =    "------------b-----------c";
```

`inputActions` is an object defining what actions we want to stimulate the epic with, and `inputMarble` indicates when and in what order these actions will be emitted. Similarly, `outputActions` and `outputMarble` represent the expected output sequence. This makes for a visual representation of what is the expected behavior. In these diagrams, a slot (`-` or an actual value, e.g. `a`) represents a unit of time of 10ms. For instance, "-ab" represents an initial delay of 10ms, then a rapid succession of "r" and "rx" input strings (within a 20ms window). The expected outcome is that "r" is debounced, and a `search` action with payload "rx" at exactly 130ms (since we debounce with a 100ms delay). Marbles support more semantics than the simple ones illustrated here, for more information check out "[Marble Tests](https://github.com/ReactiveX/rxjs/blob/master/doc/writing-marble-tests.md)".

Note that such tests can lose clarity for real-life delays (typically 200ms for debouncing, or seconds for timeouts). In such cases, instead of including dashes for each 10ms unit, one can use the following helper method:

```
export function frames(n: number, unit: string = "-"): string {
    return n === 1 ? unit : unit + frames(n - 1, unit);
}
```

The marbles now become:

```
const inputMarble =  "-ab" + frames(10) + "-c";
const outputMarble = "--" + frames(10) + "b" + frames(10) + "-c";
```

which also have the advantage of outlining more clearly the action of the debouncing delay.

### Action stream mocking

The following line in the test generates the action stream from the inputMarble definition:

```
const action$ = createTestAction$FromMarbles<Actions>(testScheduler, inputMarble, inputActions);
```

This makes use of a `createTestAction$FromMarbles<T>` helper method which wraps this logic, common to all tests:

```
import { TypedAction } from "./typed-action";
import { ActionsObservable } from "redux-observable";
import { TestScheduler } from "rxjs/Rx";
import * as Rx from "rxjs";

export function createTestAction$FromMarbles<A>(testScheduler: TestScheduler, marbles: string, values?: any) {
    return new ActionsObservable<A>(testScheduler.createHotObservable(marbles, values));
}
```

### Epic actuation

Epics are actually functions, and we now have the required arguments to invoke it and grab the resulting output actions stream:

```
const outputAction$ = searchOnInputChangedEpic(action$, store, services);
```

### Assertion

We can finally assert on the expected marble, like so:

```
// Assert on the resulting actions observable
testScheduler.expectObservable(outputAction$).toBe(outputMarble, outputActions);
// Run test
testScheduler.flush();
```

Flushing the scheduler actually performs the test, in some sense virtually incrementing the time and carrying out scheduled tasks in order. That allows for defining tests of asynchronous behavior potentially spanning a very long time that run instantaneously!

## Testing search behavior

To test `searchEpic`, we will define two unit tests, one looking at behavior in the case of a actual result, and the other looking at error handling.

### Testing search with result

Here's the full test:

```
test("searchEpic with content", (done) => {
    const searchInput = "rxjs";
    const searchResult = "Reactive programming";

    // Mock services 
    const servicesMock = TypeMoq.Mock.ofType<IServices>();
    const searchMock = TypeMoq.Mock.ofType<ISearchService>();
    searchMock.setup(s => s.doSearch(searchInput)).returns(() => Rx.Observable.of(searchResult));
    servicesMock.setup(s => s.search).returns(() => searchMock.object);
    const services = servicesMock.object;

     // Mock input actions stream
    const action$ = createTestAction$<Actions>(search(searchInput));

    // Apply epic on actions observable
    const outputAction$ = searchEpic(action$, store, services);

    // Assert on the resulting actions observable   
    outputAction$.subscribe(
        action => expect(action).toEqual(searchFulfilled(new ContentResult(searchResult))), 
        error => fail(error), 
        done
    );
});
```

We are not using marbles in this case as this test is not concerned with the asynchronicity but rather the search operation. We are thus not configuring the scheduler, since the epic doesn't involve any time-related Rx operator. Instead, we are now mocking the search service. We are using TypeMock to setup the service so that when receiving a call to `doSearch` with the input `rxjs`, we return an observable simply containing the "Reactive programming" string as unique immediate value:

```
searchMock.setup(m => m.doSearch(searchInput)).returns(() => Rx.Observable.of(searchResult));
```

Asserting on the result now involves subscribing the the output actions stream: we expect a `searchFullfilled` action with a `ContentResult` instance wrapping the "Reactive programming" string. Occurrence of an error in this context indicates a test failure:

```
outputAction$.subscribe(
    action => expect(action).toEqual(searchFulfilled(new ContentResult(searchResult))), 
    error => fail(error), 
    done
);
```

As mentioned previously, the advantage of mocking the search service is that this test does not need to burden itself with additional aspects of the actual http requests carrying out the search. This is a separate concern which is covered separately in the unit tests of the search service itself (which might involve usage of more intrusive mocking mechanisms such as `<a href="https://github.com/node-nock/nock" target="_blank">nock</a>`)

### Testing search with error

Here's the equivalent test covering the error scenario:

```
test("searchEpic with error", (done) => {
    const searchInput = "rxjs";
    const searchError = "Some error";

    // Mock services 
    const servicesMock = TypeMoq.Mock.ofType<IServices>();
    const searchMock = TypeMoq.Mock.ofType<ISearchService>();
    searchMock.setup(s => s.doSearch(searchInput)).returns(() => Rx.Observable.throw(new Error(searchError)));
    searchMock.setup(s => s.search).returns(() => searchMock.object);
    const services = servicesMock.object;
     // Mock input actions stream
    const action$ = createTestAction$<Actions>(search(searchInput));
    // Apply epic on actions observable
    const outputAction$ = searchEpic(action$, store, services);
    // Assert on the resulting actions observable   
    outputAction$.subscribe(
        action => expect(action).toEqual(searchFulfilled(new ErrorResult(searchError))), 
        error => fail(error),
        done
    );
});
```

Note that we are now using the `throw` operator to fake `doSearch()` throwing an exception. Our expectation in this case is that the search epic itself does not terminate with an exception, but rather wraps the error into an `ErrorResult` instance which can be further dealt with accordingly by the app. Terminating the epic would actually be fatal, since it would destroy the search `pipeline` - not the behavior we want in this case.

# Conclusion

We have demonstrated how to take advantage of RxJS's marble tests to unit test asynchronous behavior of redux-observable epics. In the course of this, we have introduced various helpers, including some allowing for a fully typed experience when defining the epic and tests. We have also demonstrated how to limit epic tests concern to actual epic behavior, and not extraneous aspects such as http request emission, by introducing interfaces for the epic dependencies and mocking them with the TypeMock library.
