---
theme: seriph
background: https://source.unsplash.com/collection/94734566/1920x1080
class: text-center
highlighter: shiki
lineNumbers: false
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
drawings:
  persist: false
transition: slide-left
title: Cover Slide
mdc: true
---

# Parallel JavaScript execution using Dedicated Web wokrers

A guide on how to elevate client side performance with Worker APIs and custom Worker managers.

Beertalk by CuddlyBunion341
@ Renuo AG

---
layout: image-right
image: https://media.istockphoto.com/id/696935130/de/foto/komplexe-mathematische-formeln-auf-whiteboard-mathematik-und-naturwissenschaften-mit.jpg?s=2048x2048&w=is&k=20&c=0gtlcarZJ7kzQhyMs5GXaqDTjeFiU3xbyjfaKxb8RiI=
---

# When should you care about parallel execution?

- Complex Mathematical Calculations
- Big data processing on the client
- Expensive network calls
- Video compression / encoding
- Real time data streaming
- Text analysis / processing

---

# My Problems and Solutions
Challanges I encountered when implementing multithreaded world generation in my [minecraft clone](https://github.com/CuddlyBunion341/tsmc2).

## Render Performance
**Problem:** Frametimes were greatly hindered while world was generating.

**Solution:** Move terrain generation to dedicated worker.

## Generation time
**Problem**: It took > 30s to generate chunks in render distance of player

**Solution:** Manage multiple worker instances in a worker pool

---
layout: image-right
image: https://images.pexels.com/photos/1872903/pexels-photo-1872903.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2
---

# Multithreaded Dish

**Recipe Title:** Deliciously Responsive Web Soup with Dedicated Web Worker Croutons

## Serves
Web application in need of a performance boost.

## Ingredients
- At least one dedicated or shared web worker
- A pinch of JavaScript (JS) or TypeScript (TS) logic
- API endpoints or data sources, finely chopped
- Complex calculations or algorithms, to taste
<!-- - Asynchronous tasks, seasoned with Promises or async/await -->
- Optional: Transfer objects and Managers according to preference

---
transition: slide-up
layout: quote
---

# What is a Web Worker?

"Web Workers makes it possible to run a script operation in a background thread separate from the main execution thread of a web application. The advantage of this is that laborious processing can be performed in a separate thread, allowing the main (usually the UI) thread to run without being blocked/slowed down."

https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API
---
layout: two-cols-header
---

# Difference between Workers
https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API


::left::
## Dedicated / Shared Workers
- run in a separate execution context
- don't support direct DOM manipulation
- are used for simple background tasks
- die as soon as page is closed
- support ES modules
- don't support caching

::right::
## Service Workers
- run in a separate execution context
- don't support direct DOM manipulation
- act as a proxy between application, browser and network
- are long lived
- support ES modules
- support caching

---

# Basic example

```js
// my-worker.js

self.onmessage = (message) => {
  // perform expensive calculation on data
  postMessage(result)
}

// index.js

const worker = new Worker('my-worker.js')
worker.postMessage(message)

worker.onmessage = (message) => {
  // do something with processed data
}
```

---

# Terrain Generation Example

```ts
// Chunk.ts
class Chunk {
  ...
  prepareGeneratorWorkerData() {
    const payload = {
      chunkX: this.x,
      chunkY: this.y,
      chunkZ: this.z,
      chunkWidth: this.chunkData.width,
      chunkHeight: this.chunkData.height,
      chunkDepth: this.chunkData.depth,
      terrainGeneratorSeed: this.terrainGenerator.seed
    }

    const transferable = [this.chunkData.data.data.buffer] // not used at the moment

    const callback = (payload: { data: ArrayBuffer }) => {
      this.chunkData.data.data = new Uint8Array(payload.data)
    }

    return { payload, transferable, callback }
  }
  ...
}
```

---

<Transform :scale="0.95">

```ts
// TerrainGenerationWorker.ts
import { ChunkData } from '../ChunkData'
import { TerrainGenerator } from '../TerrainGenerator'

self.onmessage = (message: any) => {
  const { data } = message
  const { chunkX, chunkY, chunkZ, chunkWidth, chunkHeight, chunkDepth, terrainGeneratorSeed } = data

  const terrainGenerator = new TerrainGenerator(terrainGeneratorSeed)
  const chunkData = new ChunkData(chunkWidth, chunkHeight, chunkDepth)

  for (let x = -1; x < chunkWidth + 1; x++) {
    for (let y = -1; y < chunkHeight + 1; y++) {
      for (let z = -1; z < chunkDepth + 1; z++) {
        const block = terrainGenerator.getBlock(
          x + chunkX * chunkWidth,
          y + chunkY * chunkHeight,
          z + chunkZ * chunkDepth
        )
        chunkData.set(x, y, z, block)
      }
    }
  }

  const arrayBuffer = chunkData.data.data.buffer

  postMessage(arrayBuffer, [arrayBuffer])
}
```

</Transform>

---

# Potential improvements at first glance

1. Implement Chunk serialization / deserialization in Chunk class.
1. Extract Worker logic into separate class?
1. Reduce memory allocation by using existing Buffer objects.
1. Initialize Worker with world seed.

![image](https://images.pexels.com/photos/3854816/pexels-photo-3854816.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2)

---
transition: slide-up
---

# Before / After
Note that chunk meshing is not parallelized, only terrain generation is.

<div grid="~ cols-2 gap-4">
<img src="/before.gif" class="rounded shadow" />
<img src="/before_pool.gif" class="rounded shadow" />
</div>

---
layout: image-right
image: https://images.pexels.com/photos/220996/pexels-photo-220996.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2
---

# Observation
One problem solved, one more to go.

1. Chunk generation no longer impedes rendering.
1. The loading screen is gone.
1. Terrain generation is "parallel" but could be "paralleler"
1. World generation became slower.
1. Workers are not getting utilized enough.

---
layout: image-left
image: https://images.pexels.com/photos/8783845/pexels-photo-8783845.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2
---

# Solution

1. Implement a Worker Manager.
1. Store a list of active / inactive workers.
1. Queue tasks when no worker is available.
1. Figure out how many workers can be utilized at once via navigator.

---

```ts{11-12|4-8,14|16-25|27-47|30,46|32-35|37-44|49-54}{maxHeight:'100%'}
// eslint-disable-next-line @typescript-eslint/no-explicit-any
export type Callback = (args: any) => void

export type WorkerTask = {
  payload: unknown
  transferable?: Transferable[]
  callback: Callback
}

export class WorkerManager {
  public idleWorkers: Worker[] = []
  public activeWorkers: Worker[] = []

  public taskQueue: WorkerTask[] = []

  constructor(public readonly scriptUrl: string, public readonly workerCount: number) {
    this.initializeWebWorkers()
  }

  public initializeWebWorkers() {
    for (let i = 0; i < this.workerCount; i++) {
      const worker = new Worker(this.scriptUrl, { type: 'module' })
      this.idleWorkers.push(worker)
    }
  }

  public enqueueTask(task: WorkerTask) {
    const { payload, callback, transferable } = task

    const idleWorker = this.idleWorkers.pop()

    if (!idleWorker) {
      this.taskQueue.push(task)
      return
    }

    idleWorker.postMessage(payload, transferable || [])
    idleWorker.onmessage = (message: unknown) => {
      callback(message)
      this.idleWorkers.push(idleWorker)
      requestAnimationFrame(() => {
        this.enqueueTaskFromQueue()
      })
    }

    this.activeWorkers.push(idleWorker)
  }

  public enqueueTaskFromQueue() {
    if (this.taskQueue.length === 0) return

    const task = this.taskQueue.shift()
    if (task) this.enqueueTask(task)
  }
}
```

---

```ts{all|6}
// main.ts
export default class Game implements Experience {
  ...
  init() {
    const workerPath = './src/game/world/workers/TerrainGenerationWorker.ts'
    const workerCount = navigator.hardwareConcurrency

    const workerManager = new WorkerManager(workerPath, workerCount)

    chunks.forEach((chunk) => {
      this.engine.scene.add(chunk.mesh)

      const task = chunk.prepareGeneratorWorkerData()

      workerManager.enqueueTask({
        payload: task.payload,
        // eslint-disable-next-line @typescript-eslint/no-explicit-any
        callback: (args: any) => {
          task.callback(args)
          requestAnimationFrame(() => chunk.updateMeshGeometry())
        }
      })
    })
  }
  ...
}
```

---

# Before / After

<div grid="~ cols-2 gap-4">
<img src="/before_pool.gif" class="rounded shadow" />
<img src="/after_pool.gif" class="rounded shadow" />
</div>

---
layout: center
class: text-center
---

# More about Web Workers

[Mozilla Documentation](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) · [TSMC GitHub](https://github.com/CuddlyBunion341/tsmc2) · [Blog by Max Peng](https://medium.com/techtrument/multithreading-javascript-46156179cf9a) · [Blog by Badmus Kola](https://www.honeybadger.io/blog/javascript-web-workers-multithreading/)
