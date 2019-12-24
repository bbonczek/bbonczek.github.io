---
layout: post
title:  "polling with ngrx"
date:   2018-03-01 17:15:00 +0200
categories: jekyll update
---
So, you want to do polling to a server using `NgRx`? I assume you want to wait for each server response `before sending new request`, and also you want your code to be readable and preferably testable? Perhaps you want to implement long polling? Here is the solution.

### disclaimer

Let me begin with a confession - I'm new to state management stuff. I've been using `NgRx` for about 2 months now, so everything below should be taken with a grain of salt.

### problem

For the sake of this example, let's assume that our application is displaying a list of devices that are in our subnet. At any given time, somebody could add or remove a device. Since we have no clue if something like this happened, we have to ask a server for the current state every x seconds.

Our request is a GET to this address: `http://localhost:5000/api/subnet`.
Response from the server may look something like this:

{% highlight javascript %}
[
  { ip: "192.168.1.1", type: "router", name: "TP-Link AC1750" },
  { ip: "192.168.1.2", type: "PC", name: "home-pc" },
  { ip: "192.168.1.3", type: "unknown", name: "raspberryPI" }
]
{% endhighlight %}

Let's assume that time our server needs to scan subnet is 1 - 3 seconds, and a cache is stored for 5 seconds. It means that there is no point in sending a request more often than at least 6 seconds.

The naive approach would be to send a request every 8 seconds (caching time + maximum scanning subnet time). The problem occurs, when something bad happens to our server - `browser will keep on sending new requests`, even though the server is not responding.

![interval-polling]({{ site.url }}/assets/interval-polling.png){:class="center-image"}

### solution

What if instead of setting interval between two requests, we set it between response and request?

In other words, let's send a request, wait for a response and after it arrives wait for 5 seconds, send another request and so on. Below there is a diagram that shows, what we want to do.
![interval-polling-after-response]({{ site.url }}/assets/after-response-interval-polling.png){:class="center-image"}

Let's try to implement that. We will start with something easy, and write our model and actions:

#### `subnetEntry.model.ts`
{% highlight ts %}

export interface SubnetEntry {
  ip: string;
  type: string;
  name: string;
}

{% endhighlight %}

#### `subnet.actions.ts`
{% highlight ts %}

import { Action } from '@ngrx/store';

export enum SubnetActionTypes {
  GetSubnetDevicesRequested = '[Subnet] Get Devices Requested',
  GetSubnetDevicesSucceded = '[Subnet] Get Devices Succeded'
  GetSubnetDevicesFailed = '[Subnet] Get Devices Failed',
  StartPollingSubnetDevices = '[Subnet] Start Polling',
  StopPollingSubnetDevices = '[Subnet] Stop Polling'
}

export class GetSubnetDevicesRequested implements Action {
  readonly type = SubnetActionTypes.GetSubnetDevicesRequested;
}

export class GetSubnetDevicesSucceded implements Action {
  readonly type = SubnetActionTypes.GetSubnetDevicesSucceded;

  constructor(public payload: {entries: SubnetEntry[]}) {}
}

export class GetSubnetDevicesFailed implements Action {
  readonly type = SubnetActionTypes.GetSubnetDevicesFailed;

  constructor(public payload: {error: string}) {}
}

export class StartPollingSubnetDevices implements Action {
  readonly type = SubnetActionTypes.StartPollingSubnetDevices;
}

export class StopPollingSubnetDevices implements Action {
  readonly type = SubnetActionTypes.StopPollingSubnetDevices;
}

export type SubnetActionsUnion =
  | GetSubnetDevicesRequested
  | GetSubnetDevicesSucceded
  | GetSubnetDevicesFailed
  | StartPollingSubnetDevices
  | StopPollingSubnetDevices;

{% endhighlight %}

Next thing we will do is write our reducer. It's super easy as well.

#### `subnet.reducer.ts`
{% highlight ts %}

import * as SubnetActions from './subnet.actions.ts';
import SubnetEntry from './subnetEntry.model.ts';

export interface State {
  entries: SubnetEntry[];
  error: string;
}

export const initialState: State {
  entries: [],
  error: ''
};

export function reducer(
  state: State = initialState,
  action: SubnetActions.SubnetActionsUnion
): State {
  switch (action.type) {
    case SubnetActions.SubnetActionTypes.GetSubnetDevicesSucceded:
      return {
        entries: action.payload.entries,
        error: ''
      };

    case SubnetActions.SubnetActionTypes.GetSubnetDevicesFailed:
      return {
        entries: [],
        error: action.payload.error
      };

    default:
      state;
  }
}

export const getEntries = (state: State) => state.entries;
export const getError = (state: State) => state.error;

{% endhighlight %}

Pretty standard reducer, huh?

Now let's get to the meat of the problem.

#### `subnet.effect.ts`
{% highlight ts %}
import { Injectable } from '@angular/core';
import { of } from 'rxjs';
import { Effect, Actions, ofType } from '@ngrx/effects';
import { map, switchMap, catchError, takeWhile, delay } from 'rxjs/operators';
import * as SubnetActions from './subnet.actions.ts';
import SubnetEntry from './subnetEntry.model.ts';

@Injectable()
export class SubnetEffects {
  constructor(
    private actions$: Actions<SubnetActions.SubnetActionsUnion>,
    private http: HttpClient
  ) {}

  private isPollingActive = false; // a flag that indicates whether polling is active or not

  // This effect starts polling. There is no other way of initializing polling that calling StartPollingSubnetDevices action.
  @Effect()
  startPolling$ = this.actions$.pipe(
    ofType(SubnetActions.SubnetActionTypes.StartPollingSubnetDevices),
    map(() => this.isPollingActive = true), // switch flag to true
    switchMap(() => {
      return this.http.get<SubnetEntry>('http://localhost:5000/api/subnet').pipe(
        switchMap(entries => new SubnetActions.GetSubnetDevicesSucceded({ entries })),
        catchError(error => of(new SubnetActions.GetSubnetDevicesFailed({ error })))
      ),
    }),
  );

  // This effect stops polling. There is no other way to stop polling that calling Stop PollingSubnetDevices action.
  @Effect()
  stopPolling$ = this.actions$.pipe(
    ofType(SubnetActions.SubnetActionTypes.StopPollingSubnetDevices),
    map(() => this.isPollingActive = false) // switch flag to false
  );

  // this effect repeats only if time since last response is at least 5 seconds.
  @Effect()
  continuePolling$ = this.actions$.pipe(
    ofType(
      SubnetActions.SubnetActionTypes.GetSubnetDevicesSucceded,
      SubnetActions.SubnetActionTypes.GetSubnetDevicesFailed
    ),
    takeWhile(() => this.isPollingActive), // do this only as long as flag is set to true
    switchMap(() => {
      return this.http.get<SubnetEntry>('http://localhost:5000/api/subnet').pipe(
        delay(5000),
        switchMap(entries => new SubnetActions.GetSubnetDevicesSucceded({ entries })),
        catchError(error => of(new SubnetActions.GetSubnetDevicesFailed({ error })))
      );
    })
  );
}


{% endhighlight %}

### small effects

I find this way of composing a feature from multiple small effects very useful. If you don't want to keep any private variable in an effect (`isPollingActive`), it could easily be stored in the state, and accessed via `withLatestFrom` operator.

