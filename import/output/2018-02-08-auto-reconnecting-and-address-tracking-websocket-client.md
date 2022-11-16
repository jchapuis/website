---
title: "Auto-reconnecting and address-tracking WebSocket client"
date: "2018-02-08"
categories: 
  - "rx"
  - "ts"
coverImage: "rx-2.png"
---

A common requirement for any web app taking advantage of WebSockets is to maintain the link open, even in occurrence of errors or disconnection. It's also often the case that we want to allow for dynamic configuration of the server address, possibly even by user interaction.

## `WebSocketReactiveClient`

We are going to define such a simple reusable WebSocket client in TypeScript: it accepts an `Observable<string>` addresses stream as well as a reconnection delay, and provides a connection status `Observable<boolean>` stream as well as a `send(request: TReq): Observable<TResp>` method to send out requests to the server. Here's the interface for it:

```
export interface ReactiveClient<TReq, TResp> {
    getConnectionStatus$(): Rx.Observable<boolean>
    send(request: TReq): Rx.Observable<TResp>
}
```

## Example usage

Below a dummy example usage of a `WebSocketReactiveClient` class that implements this interface:

```
const address$ = Rx.Observable.from(["ws://foobar.com:1234", "ws://barfoo.com:1234"]).zip(Rx.Observable.timer(10000), (s, _) => s)
const service = new WebSocketReactiveClient<string, string>(address$, 1000);
service.send("hello").subscribe(console.log)
```

We are simulating a change of address after 10 secs, and sending out a "hello" request and logging the answer.

## Let's implement `WebSocketReactiveClient`

Let's start by declaring the class and its private members:

```
import * as Rx from "rxjs";

interface Response<TReq, TResp> {
    request: TReq;
    response: TResp;
}

export class WebSocketReactiveClient<TReq, TResp> implements ReactiveClient<TReq, TResp> {
    private status$: Rx.Subject<boolean> = new Rx.Subject<boolean>();
    private outgoing$: Rx.Subject<TReq> = new Rx.Subject<TReq>();
    private incoming$: Rx.Subject<Response<TReq, TResp>> = new Rx.Subject<Response<TReq, TResp>>();
}
```

We are actually first defining a private `Response` type: we'll indeed need to find out to which request each response corresponds, and we'll do that by including the original request in the response. In this simple version we are identifying responses by matching requests by equality, but a more robust mechanism would be to assign unique ids for each request and use that as an envelope instead.

We use three internal subjects which allow both subscribing and inserting elements: - `status$`: tracks the connection status - `outgoing$`: outgoing requests stream - `incoming$`: incoming responses stream (precisely of `Response` type)

### Constructor

In the constructor we are defining the Rx pipeline which drives the entire socket operation, including closing/opening connections after an address change and reconnecting in case of error:

```
constructor(private address$: Rx.Observable<string>, private reconnectionDelay: number) {
   address$
     .filter(address => address !== "")
     .switchMap(address => this.connectWebSocket(address))
     .map(msg => msg as Response<TReq, TResp>)
     .subscribe(this.incoming$);
}
```

As we'll see below, `connectWebSocket()` provides an abstraction over the WebSocket stream in the form of an `Observable`. Thanks to `switchMap()`, we are creating a new connection for each address in the `address$` stream, and closing the previous one. JSON is parsed internally by RxJS's `WebSocketSubject` and so we can just cast the incoming message as our Response type (obviously, for a real production implementation we should introduce some degree of safety and more error checking). Subscribing the `incoming$` subject to this pipeline quick-starts the whole operation.

### Connection

As mentioned above we are using RxJS `WebSocketSubject` to create the WebSocket proper, as you can see below in the implementation of `connectWebSocket()`:

```
private connectWebSocket(address: string): Rx.Observable<any> {
    console.log(`Connecting websocket to ${address}`);
    const openSubject = new Rx.Subject<Event>();
    openSubject.map(_ => true).subscribe(this.status$);
    const closeSubject = new Rx.Subject<CloseEvent>();
    closeSubject.map(_ => false).subscribe(this.status$);
    const ws = Rx.Observable.webSocket<any>({
        url: address,
        openObserver: openSubject,
        closeObserver: closeSubject
    });
    return Rx.Observable.using(
        () => this.outgoing$.subscribe(ws),
        () => ws.retryWhen((errs) => errs.switchMap(() => {
            this.status$.next(false);
            console.log(`Connection down, will attempt reconnection in ${this.reconnectionDelay}ms`);
            return Rx.Observable.timer(this.reconnectionDelay);
        }).repeat()));
}
```

`Observable.webSocket()` allows passing in a settings object that we take advantage of to register open and close event handlers in order to track the connection status in our `status$` stream.

In the following call we are subscribing the `ws` to the requests stream `outgoing$`, and making use of `Observable.using(resourceFactory, observableFactory)` to ensure that this subscription will be cleaned up when the returned observable is unsubscribed from.

The second argument to `using()` (`observableFactory`) defines the observable proper, where we chain the `ws` stream with a call to `retryWhen()` which affords us the reconnection behavior. We also add a final call to `repeat()` to deal with the case of the server closing the connection as a result of normal operation (without occurrence of an error), in which case (in this supposed context of continuous link) we also want to re-establish the connection.

## Complete code

You'll find below the complete code. Note that we have just defined a simple request/response scenario, but the protocol could be extended to deal with asynchronous push messages from the server (pub/sub).

```
import * as Rx from "rxjs";

interface Response<TReq, TResp> {
    request: TReq;
    response: TResp;
}

export class WebSocketReactiveClient<TReq, TResp> {
    private status$: Rx.Subject<boolean> = new Rx.Subject<boolean>();
    private outgoing$: Rx.Subject<TReq> = new Rx.Subject<TReq>();
    private incoming$: Rx.Subject<Response<TReq, TResp>> = new Rx.Subject<Response<TReq, TResp>>();

    constructor(private address$: Rx.Observable<string>, private reconnectionDelay: number) {
        address$
            .filter(address => address !== "")
            .switchMap(address => this.connectWebSocket(address))
            .map(msg => msg as Response<TReq, TResp>)
            .subscribe(this.incoming$);
    }

    public getConnectionStatus$(): Rx.Observable<boolean> {
        return this.status$;
    }

    public send(request: TReq): Rx.Observable<TResp> {
        return Rx.Observable.create((observer: Rx.Observer<TResp>) => {
            console.log(`Sending outgoing request ${request}`);
            const responseSub = this.incoming$
                .filter(r => r.request === request)
                .take(1)
                .map(r => r.response)
                .subscribe(observer);
            this.outgoing$.next(request);
            return responseSub;
        });
    }

    private connectWebSocket(address: string): Rx.Observable<any> {
        console.log(`Connecting websocket to ${address}`);
        const openSubject = new Rx.Subject<Event>();
        openSubject.map(_ => true).subscribe(this.status$);
        const closeSubject = new Rx.Subject<CloseEvent>();
        closeSubject.map(_ => false).subscribe(this.status$);
        const ws = Rx.Observable.webSocket<any>({
            url: address,
            openObserver: openSubject,
            closeObserver: closeSubject
        });

        return Rx.Observable.using(
            () => this.outgoing$.subscribe(ws),
            () => ws.retryWhen((errs) => errs.switchMap(() => {
                this.status$.next(false);
                console.log(`Connection down, will attempt reconnection in ${this.reconnectionDelay}ms`);
                return Rx.Observable.timer(this.reconnectionDelay);
            }).repeat()));
    }
}
```
