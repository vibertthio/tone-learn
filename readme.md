# tone-learn

This repo serves as a documentation of me learning Tone.js. My goal of learning is to understand the design, the architecture, and the implementation. It's based on [Tone.js v14.7.17](https://tonejs.github.io/docs/14.7.77/index.html).

# Start it from the ground

Tone.js uses [`standardized-audio-context`](https://github.com/chrisguttandin/standardized-audio-context) to replace to the native [`AudioContext`](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext). It helps Tone.js abstracts away from browser imeplementations of Web Audio API. In my opinion, the most important contribution of Tone.js is the timekeeper class `Transport`.

```
Tone
├── component/
├── core/
├── effect/
├── event/
├── instrument/
├── signal/
├── source/
├── classes.ts
├── fromContext.test.ts
├── fromContext.ts
├── index.test.ts
├── index.ts
└── version.ts
```

Above is the source code directory of Tone.js, `Tone/`. Let's go straight into the entry point `Tone/index.ts`.

The first thing I noticed is that it imports and exports the `getContext` and `setContext` from `Tone/core/Global.ts`:

```typescript
// Tone/index.ts

export { getContext, setContext } from "./core/Global";
```

This method is written based on the assumption that we have only one global audio context. The context is the only access point to the Web Audio API, so almost everything in Tone.js is related to this object, expect `TODO`. But, how is this global context created? There are several files in `Tone/core/context/` related to the abstractions over the `standardized-audio-context`, including `AudioContext.ts`, `BaseContext.ts`, `Context.ts`, `OfflineContext.ts`, and `DumyContext.ts`.

`AudioContext.ts` is the contact point to the `standardized-audio-context`. It provides the methods create 2 different kinds of audio contexts, the regular one and the [offline one](https://developer.mozilla.org/en-US/docs/Web/API/OfflineAudioContext). The offline context only renders audio signal as fast as possible to an `AudioBuffer` in the memory instead of to the speaker. It could be used for analysis, testing, and save audio data without worrying about real-time audio rendering.
Native Web Audio API includes a class `BaseAudioContext` as the base class of this 2 kinds of contexts. The file `BaseContext.ts` is the Tone.js version of it. It extends the Tone's `Emitter` instead of the native [`EventTarget`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget)and it removes some unused or deprecated methods.

```typescript
// Tone/core/context/BaseContext.ts

export type ExcludedFromBaseAudioContext =
	| "onstatechange"
	| "addEventListener"
	| "removeEventListener"
	| "listener"
	| "dispatchEvent"
	| "audioWorklet"
	| "destination"
	| "createScriptProcessor";

export type BaseAudioContextSubset = Omit<
BaseAudioContext,
ExcludedFromBaseAudioContext
>;

export abstract class BaseContext
	extends Emitter<"statechange" | "tick">
	implements BaseAudioContextSubset {
    ...
}
```

`Context.ts` is the meat of the Tone's context implementation. I want to use the next section to cover it. It extends `BaseContext` and implements most of the methods of native audio context. Besides, it also includes some important member variables, such as `transport`, `destination`, `_ticker`,  `draw`, and `lookAhead`. 

`OfflineContext.ts` is the offline context implementation of Tone.js. One interesting thing is that it inherit the class [`Tone.Context`](https://tonejs.github.io/docs/14.7.77/Context) rathe than `BaseContext`. It's a little bit different from the native Web Audio API. I am not sure about the reason, but I guess it's just convenient. Tone also provides an [`Offline`](https://tonejs.github.io/docs/14.7.77/fn/Offline) API for using `OfflineContext`. You can checkout the document page to see how to use it. One interesting thing to notice is how `OfflineContext` render: it needs to call the native API  `startRendering` along with `_renderClock` because there are events bind to the "clock" of the `Context` assigned by Tone.js. Inside, `_renderClock`, the following line is how to manually move the clock of `Context` forward:

```typescript
// Tone/core/context/OfflineContext.ts

// invoke all the callbacks on that time
this.emit("tick");
```

I will discuss about the mechanism behind `emit` in the next part.

`DumyContext.ts` is used in the `Tone/core/Global.ts` for preventing errors from Node when importing Tone.js. It's just a class extending `BaseContext` and implement the methods with no-op functions. As far as I know, Tone.js doesn't work on Node.js at this moment, but who knows what will happen. 

# Context.ts

Why do I want to start from `Context`? Isn't it just a wrapper around the native API? It's not.

 `Tone.Transport` is the main timekeeper as the wiki says, but we can still trigger and schedule future triggers oscillators and synthesizers in Tone.js without starting `Transport`. It's just calling directly to the native API and schedule it precisely. A lot of the features of `Transport` are implemented in the code of `Context`, such as the `_ticker` in `Context`.

We mentioned all the Tone.js codes share a single `Context`. Where is it created? It's in the function `getContext` in `Global.ts` and it will be called on line 33 in `Tone/index.ts`.

## Initialization

Where are the `Tone.Transport` and the `Tone.destination` of the `Tone.context` created? They are both created in the initialization process of the `Tone.context`. By the way, the capitalization of the three objects are not typos, it was a weird convention of Tone.js. 

 `Tone/core/context/ContextInitialization.ts`.

It's included in `Context.ts` and called in the private method `initialize`. This file provides functions and variables in order to aggregating all the necessary callbacks to create and closed a context. The reason why they are not created in the constructor is that not every context needs these objects, such as `OfflineContext`. There are totally four different member variables are initilized by this mechanism:

1. `context.transport`
2. `context.destination`
3. `context.listener`
4. `context.draw`

You can find all of them by searching `onContextInit` in the source code directory. Also, all these objects are exposed in the core functions: `Tone.getContext`, `Tone.getDestination`, `Tone.getDraw`, `Tone.getListener`, and `Tone.getTransport`.



# Transport

`TODO`



# Future Topics

- Events and `ToneEvent`
- class Tone
- `Tone.Signal`
- Source
- Component
- Instrument
- Effects
- Play with microphone inputs and other `StreamMedia`
- Unit







