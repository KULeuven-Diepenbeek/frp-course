---
title: "Concurrency and Queues"
weight: 3
draft: false
---

## Concurrency in Haskell

Haskell's concurrency model is built on **lightweight threads** managed by the GHC runtime. These are not OS threads but user-space green threads that are multiplexed across a pool of OS threads (the number of OS threads is controlled by the `+RTS -N` runtime flag).

### Spawning threads

```haskell
import Control.Concurrent

forkIO :: IO () -> IO ThreadId
```

`forkIO` spawns a new lightweight thread that runs the given `IO` action concurrently. The parent thread continues immediately after the call.

```haskell
import Control.Concurrent

main :: IO ()
main = do
  forkIO $ do
    threadDelay 500000   -- wait 0.5 s
    putStrLn "Thread done"
  putStrLn "Main continues"
  threadDelay 1000000    -- keep alive long enough
```

### `threadDelay`

```haskell
threadDelay :: Int -> IO ()
```

Pauses the current thread for (at least) the given number of **microseconds**:

```haskell
threadDelay 1000000   -- 1 second
threadDelay 16667     -- ~60 FPS frame time
```

---

## Channels and queues

Passing data between threads requires a shared communication channel. Haskell provides several options.

### `Chan` — unbounded FIFO channel

`Chan a` is an unbounded, thread-safe FIFO channel.

```haskell
import Control.Concurrent.Chan

newChan   :: IO (Chan a)
writeChan :: Chan a -> a -> IO ()
readChan  :: Chan a -> IO a          -- blocks if empty
dupChan   :: Chan a -> IO (Chan a)   -- create a duplicate reader
```

**Example — producer/consumer**

```haskell
import Control.Concurrent
import Control.Concurrent.Chan

producer :: Chan Int -> IO ()
producer chan = mapM_ (\i -> do
    putStrLn ("Producing: " ++ show i)
    writeChan chan i
    threadDelay 200000
  ) [1..5]

consumer :: Chan Int -> IO ()
consumer chan = do
  val <- readChan chan
  putStrLn ("Consuming: " ++ show val)
  consumer chan

main :: IO ()
main = do
  chan <- newChan
  forkIO (consumer chan)
  producer chan
  threadDelay 2000000
```

### `TChan` — STM-based channel

`TChan` is the transactional version of `Chan`. It can be used inside `atomically` blocks, allowing you to compose channel reads and writes with other STM operations.

```haskell
import Control.Concurrent.STM
import Control.Concurrent.STM.TChan

newTChanIO  :: IO (TChan a)
writeTChan  :: TChan a -> a -> STM ()
readTChan   :: TChan a -> STM a      -- retries if empty
tryReadTChan :: TChan a -> STM (Maybe a)  -- non-blocking
```

**Example — non-blocking read**

```haskell
import Control.Concurrent.STM
import Control.Concurrent.STM.TChan

drainQueue :: TChan String -> IO [String]
drainQueue chan = atomically $ do
  first <- tryReadTChan chan
  case first of
    Nothing  -> return []
    Just msg -> do
      rest <- drainAll chan
      return (msg : rest)
  where
    drainAll c = do
      m <- tryReadTChan c
      case m of
        Nothing -> return []
        Just x  -> fmap (x:) (drainAll c)
```

### `TQueue` — STM-based queue (preferred over `TChan`)

`TQueue` is a simpler, more efficient FIFO queue built on STM. It does not support `dupChan`-style broadcasting but is faster for straightforward producer/consumer patterns.

```haskell
import Control.Concurrent.STM.TQueue

newTQueue    :: STM (TQueue a)
newTQueueIO  :: IO (TQueue a)
writeTQueue  :: TQueue a -> a -> STM ()
readTQueue   :: TQueue a -> STM a         -- blocks (retries) if empty
tryReadTQueue :: TQueue a -> STM (Maybe a) -- non-blocking
isEmptyTQueue :: TQueue a -> STM Bool
```

**Example — event queue for a game loop**

```haskell
import Control.Concurrent
import Control.Concurrent.STM
import Control.Concurrent.STM.TQueue

data InputEvent = KeyPress Char | MouseClick Int Int deriving (Show)

-- Separate thread that generates input events
inputThread :: TQueue InputEvent -> IO ()
inputThread queue = do
  atomically $ writeTQueue queue (KeyPress 'w')
  threadDelay 100000
  atomically $ writeTQueue queue (KeyPress 's')
  threadDelay 100000
  atomically $ writeTQueue queue (MouseClick 320 240)

-- Main loop that processes events
processEvents :: TQueue InputEvent -> IO ()
processEvents queue = do
  mEvent <- atomically $ tryReadTQueue queue
  case mEvent of
    Nothing  -> putStrLn "(no event)"
    Just evt -> putStrLn ("Event: " ++ show evt)
  threadDelay 50000
  processEvents queue

main :: IO ()
main = do
  queue <- newTQueueIO
  forkIO (inputThread queue)
  processEvents queue
```

This pattern — **input thread writes to queue, main loop drains queue** — is exactly what you will use when integrating Yampa's `reactimate` function with real input sources (see the Yampa section).

---

## `MVar` as a one-element queue

An `MVar` can act as a **synchronous one-shot channel**: the sender blocks if the receiver has not yet consumed the previous message. This is useful for handshakes.

```haskell
import Control.Concurrent
import Control.Concurrent.MVar

main :: IO ()
main = do
  mvar <- newEmptyMVar

  forkIO $ do
    threadDelay 500000
    putMVar mvar "Hello from thread"    -- send

  msg <- takeMVar mvar                  -- receive (blocks until sent)
  putStrLn msg
```

---

## Putting it all together — summary

| Tool | Characteristic | Best for |
|---|---|---|
| `IORef` | No thread safety | Single-threaded state |
| `MVar` | Mutex / blocking 1-slot | Simple synchronisation, locks |
| `Chan` | Unbounded FIFO, thread-safe | Simple producer/consumer |
| `TChan` | STM channel, composable | Complex transactional reads |
| `TQueue` | STM FIFO, efficient | Game loops, event queues |
| `TVar` | Atomic transactional state | Shared state with composable transactions |

For the FRP use-case covered in this course, `TQueue` is the most practical choice: the input thread pushes sensor/event values into a queue, and the Yampa game loop drains the queue each tick.
