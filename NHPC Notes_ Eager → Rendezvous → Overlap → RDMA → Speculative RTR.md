# NHPC Study Notes: Eager → Rendezvous → Overlap → RDMA Read → Interrupt → Speculative RTR

**Source:** Session 6 deck, "RDMA and MPI Rendezvous Protocol"

**Coverage:** pages 6–12 of the PDF, from "Eager vs Rendezvous (RNDZ) Protocol" to the last slide (Speculative RTR). You said "slides 7–13", but in this PDF the section starts on page 6 and ends on page 12. I started at the Eager vs Rendezvous slide as you originally specified, so nothing is skipped.

---

# 0. Slide Map and Naming Cheat Sheet

| PDF page | Title printed on the slide | What it actually shows |
| --- | --- | --- |
| 6 | Eager vs Rendezvous (RNDZ) Protocol | Eager vs Rendezvous message flows and buffer paths |
| 7 | RDMA write vs RDMA read | RDMA-Write-based vs RDMA-Read-based Rendezvous |
| 8 | Is overlap always possible? | Non-blocking code + timeline where overlap is lost |
| 9 | RDMA write vs RDMA read *(title reused)* | RDMA Read vs RDMA Read **with Interrupt** |
| 10 | Rendezvous Protocol | RDMA Read protocol, **sender arrives first** |
| 11 | Eager vs Rendezvous Protocol *(title reused)* | RDMA Read protocol, **receiver arrives first** (only Rendezvous is drawn) |
| 12 | Eager vs Rendezvous Protocol *(title reused)* | **Speculative RTR + RDMA Write** |

**Warning:** several slides share a title but show different things. Memorize the content, not the titles.

### Same idea, different names across slides

| Pages 6–9 | Pages 10–12 | Meaning |
| --- | --- | --- |
| RNDZ_START | Rendezvous **RTS** (Request To Send) | Sender → Receiver: "I have a big message; here is its envelope." |
| RNDZ_REPLY | (CTS, Clear To Send; not drawn on pages 10–12) | Receiver → Sender: "Matching receive is posted; go ahead." |
| FIN | **Done** / Done packet | "Transfer finished; you may complete." |
| (none) | **RTR** (Ready To Receive) | Receiver → Sender, sent **before** any RTS exists |
| DATA | RDMA Read / RDMA Write | The payload movement itself |

---

# 1. Background Primer (needed to read the slides)

You studied RDMA basics on the earlier slides. This is the minimum needed here.

## 1.1 Vocabulary

| Term | Meaning |
| --- | --- |
| **User buffer** | The array in *your* program (`double a[N]`) passed to `MPI_Send`/`MPI_Recv`. Owned by the application. |
| **Local / internal / library buffer** | Memory owned by the **MPI library** (not your program), used for staging data or control messages. Usually pre-allocated and often pre-registered with the NIC. |
| **Envelope** | The metadata MPI matches on: source rank, tag, communicator (plus message size/datatype). |
| **Matching** | Pairing an incoming message with a posted receive that has the same envelope. |
| **Posted-receive queue** | Receives the application has issued but that have not yet matched a message. |
| **Unexpected queue** | Messages that arrived **before** any matching receive was posted. The library must hold them. |
| **Control message** | Small protocol message (RNDZ_START, RNDZ_REPLY, FIN, RTR). No application payload. |
| **Memory registration (pinning)** | Telling the NIC "this memory range may be DMA'd directly". Pages are pinned (cannot be swapped or moved). Returns keys (`lkey` for local access, `rkey` for remote access). It is expensive. |
| **Initiator / Requester** | The side that *posts* an RDMA operation. |
| **Target / Responder** | The side whose memory is accessed. Its CPU is **not** involved. |
| **One-sided operation** | RDMA Read/Write act on remote memory without the remote CPU taking part, so the remote side is **not notified** when they finish. |
| **Completion Queue (CQ)** | Where the NIC reports "your operation finished". The initiator polls it. |
| **Progress engine** | The MPI library code that polls the NIC, matches messages, replies to handshakes, posts RDMA operations, and sends FIN/Done. In most implementations it runs **only inside MPI calls**. |

## 1.2 RDMA Write vs RDMA Read

The one rule to remember:

> **RDMA Write: the initiator's data goes OUT.** **RDMA Read: data comes IN to the initiator.**

```
RDMA WRITE (initiator pushes)                RDMA READ (initiator pulls)

Initiator                  Target            Initiator                  Target
  |                          |                 |                          |
  |-- data + remote addr --->|                 |-- remote addr + rkey --->|
  |      + rkey              | lands in        |                          | NIC reads its
  |                          | target memory   |<-------- data -----------| own memory
  |<------- ACK -------------|                 |                          |
  | poll CQ -> done          |                 | poll CQ -> done          |
```

Both need the target's memory to be **registered** (with remote-write or remote-read permission) and the initiator to know the target's **address + rkey**. The target CPU does nothing in either case.

## 1.3 MPI direction vs RDMA direction (easy to confuse)

The MPI-level direction never changes: data always goes **Sender's user buffer → Receiver's user buffer**. What changes is **who initiates the RDMA operation**.

|  | MPI-level data flow | RDMA op | Initiator | Target (passive) | Request direction | Data direction |
| --- | --- | --- | --- | --- | --- | --- |
| Write-based RNDZ | Sender → Receiver | **Write** | **Sender** | Receiver | Sender → Receiver | Sender → Receiver |
| Read-based RNDZ | Sender → Receiver | **Read** | **Receiver** | Sender | Receiver → Sender | **Sender → Receiver** |

In the Read case the *request* travels Receiver → Sender, but the *data* still ends up at the Receiver. "RDMA Read" does **not** mean the sender is reading anything. The receiver reads the sender's memory.

---

# 2. Slide 6: Eager vs Rendezvous (RNDZ) Protocol

## 2.1 Main idea

MPI uses **two ways** to move a point-to-point message, chosen by message size:

- **Eager:** just send it immediately (small messages).
- **Rendezvous:** first negotiate, then move data directly buffer-to-buffer (large messages).

## 2.2 Intuition

- **Eager** = dropping a letter in a mailbox without calling ahead. If the recipient isn't home, the post office (MPI library) holds it in a pigeonhole until they show up. This is fast for letters, but you cannot do it with a sofa.
- **Rendezvous** = phoning first: "I have a big sofa. Are you ready? Where do I put it?" Once they say "living room, here's the address", you deliver straight there. There is a delay for the call, but no warehouse needed.

## 2.3 Eager protocol, step by step

```
Sender                                         Receiver
  |                                               |
  |---- Send (envelope + payload together) ------>|
  |                                               |
```

1. Sender's library packages **envelope + data in one message** and sends right away.
2. It does not ask whether the receiver is ready.
3. At the receiver, the message lands in a **library (local) buffer**.
4. If the matching receive is already posted, the library **copies** data into the user buffer. If not, the message waits in the **unexpected queue** until a receive is posted, then it is copied.

**Buffer path (bottom-left diagram on the slide):**

```
Process 0                                  Process 1
[User data] --copy--> [Local buffer] ==network==> [Local buffer] --copy--> [User data]
```

- The green boxes are **library buffers**. The blue boxes are **user buffers**. The red box is the network.
- Data is **copied up to twice** (once per side), which is fine for small messages.
- "Buffered" in the slide means **buffered by the MPI library**. It is *not* `MPI_Bsend`, where the user attaches their own buffer.

## 2.4 Rendezvous protocol, step by step

```
Sender                                         Receiver
  |                                               |
  |------ RNDZ_START (envelope, size) ---------->|  1
  |                                               |  (receiver waits until a matching
  |                                               |   receive is posted)
  |<----- RNDZ_REPLY (go-ahead, +dest info) ------|  2
  |                                               |
  |============== DATA ========================>|  3
  |                                               |
  |------ FIN ---------------------------------->|  4
  |                                               |
```

| Arrow | Meaning | Why it exists |
| --- | --- | --- |
| **RNDZ_START** | Small control message: "I want to send *n* bytes with this envelope." **No payload.** | Announces the message without paying to buffer it. |
| **RNDZ_REPLY** | Receiver has matched a posted receive and says "send now". For RDMA it also carries the destination address + key. | Guarantees the receiver has somewhere to put the data and is ready. |
| **DATA** | The actual payload, moved **directly from sender's user buffer to receiver's user buffer** (zero-copy). | The whole point: no staging copies. |
| **FIN** | "All data has been delivered." | The receiver cannot otherwise know the transfer finished, because RDMA Write is one-sided and the receiver gets no notification. |

**Buffer path (bottom-right diagram):**

```
Process 0                                  Process 1
[User data] =========== network ===========> [User data]
```

There are no library data buffers in the middle. Only the tiny control messages use library buffers.

## 2.5 Why buffering large messages is expensive

1. **Memory:** eager buffers are pre-allocated per peer. If every connection reserved space for large messages, memory would scale as (#peers × big buffer). That is impossible at thousands of ranks.
2. **Unexpected messages:** if the receiver hasn't posted a receive, the *entire* message must be stored somewhere. A large one could exhaust memory.
3. **Copy cost:** two extra `memcpy`s of a huge message waste memory bandwidth and CPU time and pollute caches.
4. **Registered memory is scarce:** NIC-registered (pinned) memory is a limited resource.

**Rendezvous fix:** only small control messages are buffered. Payload goes straight user buffer → user buffer, so the library's memory requirement stays small (this is the slide's last bullet).

**Price of Rendezvous:** an extra handshake (at least one extra round-trip before data moves) plus the cost of registering user buffers. That is why it is *not* used for small messages. Eager's no-handshake speed wins there.

**Threshold:** the switch point is the **eager threshold**, a tunable, implementation- and network-dependent value (typically on the order of KBs to tens of KBs).

## 2.6 "Receiver not ready" vs "receiver never posts a receive"

| Situation | Eager | Rendezvous |
| --- | --- | --- |
| **Receiver not ready *yet*** (will post later) | Data is stored in the unexpected queue; sender's `MPI_Send` can often return early. | Only RNDZ_START is stored. DATA waits. Sender's send **cannot complete** until receiver posts the receive. This is just a *delay*. |
| **Receiver *never* posts a matching receive** | Message sits in the unexpected queue forever. This is a program bug, and it leaks library memory. | Sender **never completes** (`MPI_Send` hangs / `MPI_Wait` never returns). This is a program bug. |

"Not ready" means *not ready yet*. It does **not** mean "refuses to receive". Receivers cannot reject a message.

## 2.7 Blocking, deadlock, and why Eager can hide bugs

Classic unsafe pattern:

```c
// Rank 0                              // Rank 1
MPI_Send(sbuf, N, ..., 1, ...);        MPI_Send(sbuf, N, ..., 0, ...);
MPI_Recv(rbuf, N, ..., 1, ...);        MPI_Recv(rbuf, N, ..., 0, ...);
```

- **Small N (Eager):** each `MPI_Send` copies into a library buffer and returns. Both reach `MPI_Recv`. **It works.**
- **Large N (Rendezvous):** each `MPI_Send` waits for RNDZ_REPLY, which only comes once the *other* rank posts its receive. But the other rank is *also* stuck in `MPI_Send`. **Circular wait = deadlock.**

Key points:

- **The bug is the program's circular ordering**, not Rendezvous. Rendezvous merely *exposes* it.
- Rendezvous itself does **not** automatically cause deadlock. A correctly ordered program (one rank sends while the other receives) works fine with Rendezvous.
- Eager success is **not a guarantee**. The MPI standard calls this pattern "unsafe". Changing the message size, the machine, or the library's threshold can turn a working program into a hanging one.
- Safe fixes: `MPI_Sendrecv`, or `MPI_Irecv` + `MPI_Isend` + `MPI_Waitall`, or reordering (even ranks send first, odd ranks receive first).
- `MPI_Ssend` always has Rendezvous-like semantics (completes only once the receive has started), so it is a handy way to *test* for hidden deadlocks.

## 2.8 Simplifications on this slide (don't memorize them as absolute)

- The slide always shows two copies in Eager. In real implementations, if the receive is already posted, data may be delivered more directly. Small sends may also be "inlined" with no sender-side copy.
- The slide implies that size alone decides the protocol. Really it is a **threshold** that is configurable.
- The "FIN" in this slide is a conceptual completion message. Implementations may use different mechanisms for this notification.
- The slide doesn't show how the receiver's buffer address is communicated, but for RDMA it must be (see §3).

**Exam Point 2A:** *Eager sends envelope and data together without a handshake, buffering the data in MPI library buffers, so it is fast for small messages but costs memory and copies. Rendezvous first exchanges RNDZ_START/RNDZ_REPLY so the data moves directly user-buffer to user-buffer, trading an extra handshake for much lower buffering needs.*

**Exam Point 2B:** *Deadlock in "send-then-receive" code comes from circular blocking waits. Eager can mask it by buffering, and Rendezvous exposes it. The program is unsafe either way.*

---

# 3. Slide 7: RDMA-Write-based vs RDMA-Read-based Rendezvous

## 3.1 Main idea

Rendezvous can move DATA with **RDMA Write** (sender pushes) or **RDMA Read** (receiver pulls). The choice changes the message flow and, as the next slides show, *who needs the CPU* to make progress.

## 3.2 Intuition

- **Write-based:** you phone the recipient, they give you the address, **you** drive over and unload the sofa (you do the work, so you must be available when they reply).
- **Read-based:** you tell them your address in the first call. **They** send movers to pick it up from your house whenever they like. You don't need to be home or even awake. Your door is open (memory registered).

## 3.3 Flows side by side

```
RDMA WRITE-BASED RNDZ                      RDMA READ-BASED RNDZ

Sender                 Receiver            Sender                 Receiver
  |                       |                  |                       |
  |--- RNDZ_START ------->|                  |--- RNDZ_START ------->|
  |                       |                  |  (carries src addr    |
  |<-- RNDZ_REPLY --------|                  |   + rkey)             |
  |  (carries dst addr    |                  |                       |
  |   + rkey)             |                  |<====== RDMA Read =====|
  |==== DATA (RDMA W) ===>|                  |   Receiver's NIC pulls|
  |--- FIN -------------->|                  |   from sender memory  |
                                             |<------ FIN -----------|
```

Slide detail: in the Read diagram, the curved arrow starts at the receiver, goes over to the sender, and the data arrow returns to the receiver. That shape *is* the RDMA Read: a request out, data back.

## 3.4 What each message carries (implicit on the slide)

|  | Write-based | Read-based |
| --- | --- | --- |
| RNDZ_START carries | envelope, size | envelope, size **+ sender buffer address + rkey** |
| RNDZ_REPLY | **Needed**: carries receiver buffer address + rkey | **Not needed** (info already in START) |
| Who initiates DATA | Sender (RDMA Write) | Receiver (RDMA Read) |
| FIN direction | Sender → Receiver ("data is in your memory") | **Receiver → Sender** ("I finished reading; free/reuse your buffer") |
| Registration needed | Receiver's buffer: remote-write; sender's: local | Sender's buffer: **remote-read**; receiver's: local |

## 3.5 Why the FIN direction flips

RDMA is one-sided, so the *passive* side never learns the transfer finished.

- Write-based: the **receiver** is passive, so the sender sends FIN to tell it the data is complete.
- Read-based: the **sender** is passive, so the receiver sends FIN to tell it that its buffer has been fully read and may be reused. Without FIN, the sender's `MPI_Wait` could never complete safely.

## 3.6 Why RDMA Read is used

1. **One fewer control message** (no RNDZ_REPLY), so a simpler handshake.
2. **Sender CPU not needed for data movement:** once RNDZ_START is sent, the sender's NIC serves the read by itself. The sender can be deep in computation (this is the overlap benefit explored on pages 8–10).
3. In the write-based flow, when RNDZ_REPLY arrives the **sender's library must run** to post the RDMA Write. If the sender is computing, nothing happens until it enters MPI.

## 3.7 Limitations / edge cases

- The sender's buffer must be **registered before** RNDZ_START and stay pinned and unmodified until FIN. This exposes sender memory to remote reads.
- The **receiver** now has to *notice* RNDZ_START and issue the read. That creates the **receiver-side progress problem** (§5 and §6).
- RDMA Read support is hardware-dependent, and NICs typically limit the number of outstanding reads per connection.
- If the message is larger than the posted receive buffer, MPI reports a truncation error. This is checked at the receiver when it matches.

**Exam Point 3:** *In the write-based protocol the sender pushes data using the receiver's address from RNDZ_REPLY; in the read-based protocol the sender puts its own address and key in RNDZ_START and the receiver pulls the data with RDMA Read. Read-based removes RNDZ_REPLY and frees the sender CPU, but now the receiver must notice RNDZ_START.*

---

# 4. Slide 8: Is overlap always possible?

## 4.1 Main idea

Even with non-blocking calls and Rendezvous, **overlap of communication and computation is not automatic**. The slide shows a case where **no overlap happens at all**.

## 4.2 Non-blocking communication: concepts first

**Intuition:** `MPI_Isend` is like dropping a parcel at a counter and getting a **tracking slip** (the request). You walk away, but you must check the slip (`MPI_Wait`) before assuming the parcel is gone.

| Call | What it does | What it **guarantees** | What it does **NOT** guarantee |
| --- | --- | --- | --- |
| `MPI_Isend(buf, ..., &req)` | Returns immediately with a request handle; registers intent to send. | The library knows about the send. It will complete *eventually* if a matching receive exists and progress happens. | That data has been sent, copied, received, **or even started**. That any overlap will occur. |
| `MPI_Irecv(buf, ..., &req)` | Posts a receive into the posted-receive queue and returns. | A receive is registered, so the library can match incoming messages to it. | That the buffer contains data. Its contents are **undefined** until completion. |
| `MPI_Wait(&req, &status)` | Blocks until the request completes. | After it: send buffer is **reusable**; receive buffer **holds the full message**. Request is released. | Nothing before it returns. |
| `MPI_Test` | Non-blocking check of completion. | Same completion semantics, and it runs the progress engine. | Doesn't block. |

**Why `MPI_Wait` is required:**

- It is the *only* point where completion is guaranteed.
- Without it you cannot know when the buffer is safe to touch, and the request object leaks.
- It also gives the library CPU time to **make progress**.

**Why the send buffer can't be reused before completion:** in Rendezvous the NIC may still be reading the sender's user buffer directly (zero-copy), possibly long after `MPI_Isend` returned. Modifying it corrupts the message. Even if Eager copied it already, the standard forbids relying on that.

## 4.3 Communication/computation overlap

- **Overlap** = the network moves data **while** the CPU does useful computation.
- Why desirable: communication time is hidden. In the ideal case it is free.

Let `Tc` = compute time and `Tm` = communication time:

```
unit:         0 1 2 3 4 5 6 7 8 9
No overlap    C C C C C C M M M M      total = Tc + Tm = 10
Partial       C C C C C C
                    M M M M            total = 7  (comm starts late)
Perfect       C C C C C C
              M M M M                  total = max(Tc,Tm) = 6
```

| Case | Total time | Meaning |
| --- | --- | --- |
| None | `Tc + Tm` | Communication happens only after (or instead of) computing. |
| Partial | between | Some communication hidden. |
| Perfect | `max(Tc, Tm)` | All of the shorter activity is hidden. |

## 4.4 The code on the slide

```c
// Sender                              // Receiver
/* Compute Large Array */
MPI_Isend(array);                      MPI_Irecv(array);
long_compute();                        long_compute();
MPI_Wait(send_req);                    MPI_Wait(recv_req);
```

The programmer's intent: send/receive while `long_compute()` runs. The protocol is **write-based Rendezvous**.

## 4.5 The timeline, arrow by arrow

```
Sender                                          Receiver
  |                                                |
MPI_Isend --------- RNDZ_START ------------------>| arrives, but receiver is
  |                                           MPI_Irecv  inside long_compute(),
  |                                                |     NOT inside MPI -> nobody
  | long_compute()                  long_compute() |     processes it
  |   (COMPUTE)                       (COMPUTE)    |
  |                                                |
MPI_Wait <------- RNDZ_REPLY ------------------ MPI_Wait   <- receiver finally enters
  |                                                |           MPI, sees START,
  |================= DATA ========================>|           sends REPLY
  |  (COMMUNICATE)                  (COMMUNICATE)  |
  |------------------ FIN ------------------------>|
  done                                             done
```

Step by step:

1. `MPI_Isend` sends RNDZ_START and returns. The sender starts computing.
2. RNDZ_START reaches the receiver while it is in `long_compute()`. The MPI library is **not running**, so the message sits in a queue unprocessed.
3. The receiver finishes computing and calls `MPI_Wait`. Only now does the progress engine notice RNDZ_START, match it with the posted receive, and send RNDZ_REPLY.
4. RNDZ_REPLY arrives at the sender. The sender's library must **also** run to act on it, and it does so at the sender's own `MPI_Wait`, then posts the RDMA Write.
5. DATA flows and FIN follows. Only now is communication happening.

**Result:** the "Compute" and "Communicate" brackets on the slide are **back to back** on both sides, so **zero overlap**. The two sides' progress depends on each other (each needs the other to be inside MPI), so DATA moves only when **both** are in MPI.

## 4.6 MPI progress

- **Progress** = the library actually advancing protocol steps (matching, replying, posting RDMA, sending FIN).
- In most implementations this happens **only inside MPI calls** (the progress engine is driven by the application thread). The MPI standard's progress guarantees are fairly weak and do not require background progress.
- During `long_compute()` the library is "asleep", like a receptionist who only checks the inbox when you ask.
- When `MPI_Wait` runs, it loops the progress engine and performs *all* the pending steps at once, producing the **communication burst after the compute** seen in the slide.

## 4.7 Ways to get real asynchronous progress

1. **Hardware offload:** the NIC does DMA by itself. RDMA already does this for the *data* phase, but the *handshake* (matching, replying) still needs software unless the NIC also offloads matching.
2. **Progress/helper thread:** a library thread polls in the background. It costs a core, adds lock contention, and may disturb the application.
3. **Interrupts:** the NIC interrupts the CPU when a protocol message arrives (page 9).
4. **Protocol redesign:** choose RDMA Read or Speculative RTR + RDMA Write so the *right* side initiates the transfer when it is already in MPI (pages 10–12).
5. **Application help:** call `MPI_Test` / `MPI_Iprobe` periodically inside compute loops, or split the computation into chunks with MPI calls in between.

## 4.8 If overlap fails

- The program is still **correct**. The data arrives and `MPI_Wait` completes. Only **performance** suffers (time ≈ `Tc + Tm`).
- Caveat: a program that *depends* on background progress for correctness (e.g., spinning on a variable that only changes when a message is processed, with no MPI calls) can hang on weak-progress implementations. That is an application bug.

## 4.9 Slide simplifications

- The slide shows a write-based protocol without a progress thread. Other implementations behave differently.
- The slide implies strict "both-must-be-in-MPI". Real systems may have partial background progress.

**Exam Point 4A:** *Non-blocking calls only start communication and return a request. Completion is guaranteed only after MPI_Wait/Test, and overlap happens only if the library or hardware can progress the protocol while the CPU computes.*

**Exam Point 4B:** *In write-based Rendezvous, RNDZ_START sits unprocessed while the receiver computes and RNDZ_REPLY sits unprocessed while the sender computes, so DATA moves only once both are inside MPI_Wait, giving no overlap. This is a performance loss, not a correctness error.*

---

# 5. Slide 9: RDMA Read vs RDMA Read **with Interrupt**

## 5.1 Main idea

Read-based Rendezvous fixes the **sender-side** overlap problem, but the **receiver** still needs to notice RNDZ_START. An **interrupt** fixes that.

## 5.2 WITHOUT interrupt (left diagram)

```
Sender                                          Receiver
MPI_Isend ------- RNDZ_START ------------------> arrives (receiver computing)
  |                                           MPI_Irecv
  | COMPUTE                                        | COMPUTE
  |      <======== RDMA Read (data pulled) ========|   (START sits unnoticed...)
  | (comm overlaps sender's compute:          MPI_Wait <- library finally sees START,
  |  sender NIC serves the read)                   |      posts RDMA Read
  |  <-------------- FIN --------------------------|  COMM (not overlapped)
MPI_Wait (short)                                   done
```

- The **sender is happy**: its CPU isn't needed, so the communication bracket sits *inside* its compute bracket (overlap achieved).
- The **receiver is not**: RNDZ_START arrives during compute and nobody looks at it. The RDMA Read starts only at the receiver's `MPI_Wait`, so its communication happens *after* its compute (no overlap).

**The receiver-side progress problem:** in the read-based protocol the **receiver is the initiator**. The initiator must run MPI code to post the read, but a receiver busy computing is not running MPI code.

## 5.3 WITH interrupt (right diagram)

```
Sender                                          Receiver
MPI_Isend ------- RNDZ_START ------------------> ** INTERRUPT ** (mid-compute)
  |                                           MPI_Irecv     |
  | COMPUTE                                        | library runs briefly:
  |                                                |  match START w/ posted Irecv,
  |      <======== RDMA Read (data pulled) ========|  post RDMA Read, return
  | (sender overlaps)                              | COMPUTE continues; NIC moves data
  |                                                |   (receiver overlaps)
  |  <-------------- FIN --------------------------| (drawn near receiver's MPI_Wait)
MPI_Wait (short)                              MPI_Wait (short)
```

1. RNDZ_START arrives at the receiver NIC.
2. The NIC raises an **interrupt**, and the CPU is forced to run the library's handler.
3. The handler matches the START with the posted `MPI_Irecv` and **posts the RDMA Read**.
4. Control returns to the application, which keeps computing while the NIC pulls the data.
5. At `MPI_Wait`, little work remains. In the right-hand diagram **both** Comm brackets lie inside the Compute brackets.

How the interrupt is typically delivered (implementation detail): the NIC signals an event, and a helper thread or handler that was waiting for it runs the progress engine.

## 5.4 Polling vs interrupt

|  | Polling | Interrupt |
| --- | --- | --- |
| Who checks for messages | CPU repeatedly checks the NIC's CQ | NIC **notifies** the CPU when something arrives |
| When it happens | Only when the application is in an MPI call | **Any time**, even mid-compute |
| Latency to notice a message | Very low *if* actively polling | Higher (interrupt delivery, kernel entry, wake-up) |
| CPU use | Burns cycles while spinning | Idle until needed |
| Effect on compute | None (in compute, it doesn't poll) | **Steals** time from computation, adds jitter |

## 5.5 Advantages and costs of interrupts

**Advantages:**

- Makes progress on the receiver **while it computes**, enabling receiver-side overlap.
- No dedicated polling thread burning a core.

**Costs / limits (interrupts are *not* free):**

- Each interrupt costs on the order of microseconds (kernel entry, context switch, cache disruption).
- It **perturbs the computation** ("OS noise"). On large parallel jobs, noise on any one rank can delay everyone at the next synchronization.
- Many control messages mean many interrupts, so using it for every message is expensive. It pays off only when the overlap saved is larger than the interrupt cost.
- The library must be **thread-safe/re-entrant**, since the handler may run while the main thread is inside the library.
- For short computations or small messages, the interrupt overhead can exceed the overlap gain.
- FIN generation still needs the receiver library to observe read completion. As drawn, FIN leaves near the receiver's `MPI_Wait`.

**Exam Point 5:** *Without an interrupt, the receiver in the RDMA Read protocol only notices RNDZ_START when it enters MPI, so the read begins late and receiver-side overlap is lost. The interrupt forces the library to run on arrival of RNDZ_START, posting the RDMA Read immediately so the NIC transfers data during computation. Interrupts add overhead and jitter.*

---

# 6. Slides 10 and 11: Read-based Rendezvous. Who arrives first matters

## 6.1 Main idea

The effectiveness of the RDMA Read protocol depends on **which side calls MPI first**:

- **Sender first** → good overlap.
- **Receiver first** → the receiver-side progress problem reappears.

(Dashed arrows on these slides = control or RDMA operations. "RTS" = RNDZ_START; "Done" = FIN.)

## 6.2 Slide 10: Sender arrives first

```
Sender                                          Receiver
Non-blocking send call
   |-------- Rendezvous RTS ---------------------> (arrives; stored. It is only a small
   |                                               control message, not the payload)
   | (sender is free to compute)         Non-blocking receive call
   |                                               |  library matches stored RTS NOW
   |<=============== RDMA Read ====================|  and posts the read immediately
   |  (sender NIC serves data)                  MPI wait call
   |                                               |  (read in progress / finishing)
   |<---------------- Done -----------------------|  Rendezvous complete (receiver)
MPI wait call -> Rendezvous complete (sender)
```

Step by step:

1. The sender calls `MPI_Isend`: an RTS goes out, and the sender continues.
2. The RTS arrives at the receiver **before any receive is posted**. It is parked in the **unexpected queue** (cheap: a few bytes of control info).
3. Later the receiver calls `MPI_Irecv`. This call *is itself MPI code*, so the library finds the stored RTS, matches it, and **issues the RDMA Read right then**.
4. The NIC pulls data from the sender's registered memory. Neither CPU is needed.
5. When the read completes, the receiver's library sends **Done**. The sender's `MPI_Wait` completes when Done arrives. The receiver's completes when its read finishes.

**Why it works well:** the party that initiates the RDMA operation (the receiver) arrives **second**, and it does so *inside an MPI call*. So it is guaranteed to be in the library at the moment it has all the info needed.

## 6.3 Slide 11: Receiver arrives first

```
Sender                                          Receiver
                                            Non-blocking receive call (posted EARLY)
Non-blocking send call                              |
   |-------- Rendezvous RTS ---------------------> arrives, but receiver is computing
MPI wait call                                       |  (RTS unnoticed)
   | (sender blocked waiting)                  MPI wait call  <- library finally sees RTS
   |<=============== RDMA Read ====================|  and starts the read
   |                                                |
   |<--------------- Done packet -----------------|  Rendezvous complete (receiver)
Rendezvous complete (sender)
```

Step by step:

1. The receiver posts `MPI_Irecv` first, then goes off to compute.
2. The sender later calls `MPI_Isend`, so an RTS arrives while the receiver is **not inside MPI**.
3. Nothing happens until the receiver reaches `MPI_Wait`. Only then does the library discover the RTS and start the RDMA Read.
4. The read, the Done packet and the completion all occur **after** the receiver's compute, so there is **no receiver-side overlap**, and the sender may also sit waiting for Done.

**Why it fails:** the initiator of the read is the **early** party (the receiver), which has moved on to compute. The info it needs (the sender's address) only arrived after it left MPI.

**Slide note:** page 11 is titled "Eager vs Rendezvous Protocol", but only Rendezvous is drawn.

## 6.4 Side-by-side

|  | Sender first (p.10) | Receiver first (p.11) |
| --- | --- | --- |
| RTS arrives when | Receiver hasn't posted yet | Receiver posted but is computing |
| RDMA Read starts at | Receiver's `MPI_Irecv` call | Receiver's `MPI_Wait` call |
| Receiver-side overlap | Yes | **No** |
| Cause | Initiator arrives last, inside MPI | Initiator arrived first and left MPI |

**Exam Point 6:** *In RDMA Read Rendezvous, if the sender arrives first, the receiver's MPI_Irecv finds the stored RTS and starts the read immediately, giving good overlap. If the receiver arrives first, the RTS arrives while it is computing and the read starts only at MPI_Wait, so overlap is lost.*

---

# 7. Slide 12: Speculative RTR + RDMA Write

## 7.1 Main idea

When the **receiver arrives first**, let the **receiver send a "Ready To Receive" message proactively**, before any sender exists. Then the **sender**, when it arrives, can **RDMA-Write** straight into the receiver's buffer with no handshake.

## 7.2 Intuition

You're expecting a delivery. Instead of waiting for the courier to call you, you **leave a note at the depot in advance**: "If you have my package, here is my address and a key to the gate." When the courier finally arrives, they **deliver immediately** without calling anyone. The risk: the note was a guess. If the package never needs that route (it was a postcard that went through the normal mailbox), the note is stale.

## 7.3 Terminology

- **RTR (Ready To Receive):** a receiver-originated control message: "I'm ready; here is where to write."
- **Speculative:** sent **without knowing** whether the sender will actually use Rendezvous. It is a prediction.
- **Done packet:** sender → receiver notification that the RDMA Write has completed.

## 7.4 Timeline

```
Sender                                          Receiver
                                            Non-blocking receive call
                                                |  receiver predicts "probably Rendezvous"
   <-------- Speculative RTR (dst addr+rkey) ---|
   | (RTR stored at sender's library)           | (receiver can compute; no CPU needed
Non-blocking send call                          |  for the data phase)
   |  library finds the waiting RTR -> NO handshake needed
   |============= RDMA Write (if Rendezvous) ===>| data lands directly in user buffer
   |                                         MPI wait call
MPI wait call                                   |  (waiting for Done)
Rendezvous complete (sender)
   |------------------- Done packet ------------>|  Rendezvous complete (receiver)
```

**Sender timeline:**

1. Receives and **stores** the RTR (it arrives early, before any send).
2. At `MPI_Isend`, the library checks for a matching RTR. It finds one, and since the message is large, it chooses Rendezvous and **posts the RDMA Write immediately** using the address + rkey from the RTR.
3. Returns from `MPI_Isend` and computes. The NIC does the transfer.
4. At `MPI_Wait`, the library observes write completion and sends **Done**. Then the send completes.

**Receiver timeline:**

1. At `MPI_Irecv`, sends the RTR (and may then compute freely).
2. The data arrives silently into its user buffer via RDMA Write. **No receiver CPU is needed.**
3. At `MPI_Wait`, waits for the Done packet. Receive completes when Done arrives, and the buffer is then safe to use.

**As drawn on the slide:** Done leaves the sender around its `MPI_Wait`. So the sender library still has to run to send Done (a leftover dependence on sender progress).

## 7.5 What the RTR conceptually carries

- **Envelope** (source, tag, communicator), so the sender can match it with the right `MPI_Isend`.
- **Receive buffer address** and **length**.
- **rkey**, the permission token for remote write. The buffer must be **registered** with remote-write access.
- Typically an identifier or sequence number (see §7.8).

## 7.6 How the receiver predicts Rendezvous

The receiver doesn't know the actual message size, but it knows the posted **receive buffer size**. The sender picks Eager vs Rendezvous based on size relative to the eager threshold. So the natural heuristic: *if the receive buffer is larger than the eager threshold, the message will probably be Rendezvous, so send an RTR.* (This is the logical mechanism; the slide only says "speculative".)

Prediction can also be impossible: with `MPI_ANY_SOURCE` the receiver doesn't know whom to send an RTR to.

## 7.7 Why RDMA Write here, and how this differs from RDMA Read

**General principle (a way to see pages 10–12 together):** *the first process to arrive advertises its buffer; the second to arrive, who now knows both addresses, initiates the RDMA operation from inside its own MPI call.*

| Who arrives first | Advertises | Message | Late arriver does | RDMA op |
| --- | --- | --- | --- | --- |
| Sender | source buffer | RTS | Receiver reads | **Read** |
| Receiver | destination buffer | RTR | Sender writes | **Write** |

The Read protocol on page 11 breaks this principle (the early party is forced to initiate), so Speculative RTR **swaps the initiator to the late party**, the sender. Because the sender is inside `MPI_Isend` when it learns it can write, no interrupt is needed.

## 7.8 Prediction right vs wrong

**Right (the sender really uses Rendezvous):**

- The sender uses the stored RTR, no RTS/RTR round trips, and data goes directly in.
- Fast: an entire handshake is saved, and both CPUs are free during transfer.

**Wrong (the sender uses Eager, e.g., message is small even though the receive buffer was large):**

- The sender sends the data eagerly through the normal channel.
- The receiver completes the receive via Eager data. The **RTR is now unused and stale**, still sitting at the sender with an address and rkey for a buffer that the receiver may soon reuse or free.
- Wasted control message, plus registered/pinned memory that was prepared for nothing.

**Race conditions:**

1. **Stale RTR confusion:** a later send with the same envelope (same source, tag, communicator) could be wrongly matched with the old RTR. The sender would RDMA-Write into an old address, **corrupting memory** that now belongs to something else.
2. **Crossing messages:** the receiver sends an RTR at the same time as the sender sends an RTS, so both exist and each side must agree on which wins.
3. **Ordering:** MPI's non-overtaking rule means messages must match in order. RTRs must not break that.

**Why mechanisms/fallbacks are needed** (the slide shows only the happy path; details are implementation-specific): unique IDs or sequence numbers on RTRs and sends, a way for the receiver to **cancel/invalidate** an unused RTR (and for the sender to acknowledge before the receiver releases the buffer), and a fallback to normal RTS/CTS (or Eager) when RTR and send don't line up.

## 7.9 Why it improves overlap and when it shines

- Receiver posts `MPI_Irecv` early and goes to compute. The RTR is sent from within `MPI_Irecv`, so no later receiver CPU is needed for the data path.
- The sender writes the moment it arrives, and the NIC then moves data while both sides compute.
- Best for the **common pattern where `MPI_Irecv` is posted before `MPI_Isend`** (the typical advice for performance).
- Saves the handshake latency.

## 7.10 Why `MPI_Wait` is still required

- **Sender:** it must learn the RDMA Write is complete (the send buffer may be reused only then) and the library must send Done.
- **Receiver:** RDMA Write is silent, so only the Done packet tells it that the data is complete. Without `MPI_Wait` the program cannot know the buffer is valid.
- The request must be completed and freed, and `MPI_Wait` drives progress for those last steps.

## 7.11 Costs and limitations

- Wasted RTRs when the guess is wrong.
- Buffers must be registered (pinned) **early**, even if never used.
- Receiver's buffer is exposed for remote writes.
- Extra protocol complexity: ID handling, cancellation, crossing messages.
- Doesn't help (and may hurt) when the sender arrives first: then the normal read-based flow is the better fit.
- Doesn't work for wildcard receives.

**Exam Point 7:** *Speculative RTR: when the receiver posts MPI_Irecv early, it sends a Ready-To-Receive carrying its buffer address and key to the sender before any send exists. If the sender then uses Rendezvous, it RDMA-Writes immediately with no handshake and sends a Done packet. If the guess is wrong, a stale RTR must be cancelled or ignored to avoid corrupting memory.*

---

# 8. BIG PICTURE: The Motivation Chain

```
Eager
  ↓   (buffering cost explodes for big messages)
Rendezvous
  ↓   (blocking calls waste time and risk deadlock)
Non-blocking communication
  ↓   (we want to hide communication time)
Communication/computation overlap
  ↓   (but the library only advances inside MPI calls)
Problem: MPI progress
  ↓   (let the NIC move data without the CPU)
RDMA
  ↓   (write-based still needs sender CPU after REPLY)
RDMA Read
  ↓   (receiver is the initiator, but may be computing)
Receiver-side progress problem
  ↓   (force the library to run on arrival)
RDMA Read + Interrupt
  ↓   (interrupts are costly; for receiver-first, switch the initiator instead)
Speculative RTR
  ↓   (sender is the late party and knows the address)
RDMA Write
  ↓
Improved communication/computation overlap
```

| Step | Problem it solves | New problem it creates |
| --- | --- | --- |
| **Eager** | Low latency for small messages: no handshake. | Needs library buffers and copies; unaffordable for large messages. |
| **Rendezvous** | Large messages: no big staging buffers; zero-copy. | Extra handshake latency; blocking sends can deadlock (when the program is unsafe); both sides must cooperate. |
| **Non-blocking** | Lets the program do other work while the transfer is pending. | Overlap is *possible*, not *guaranteed*. |
| **Overlap goal** | Hide `Tm` behind `Tc`. | Library only progresses inside MPI calls → handshake stalls during compute (slide 8). |
| **RDMA** | The NIC moves data without the CPU or kernel, zero-copy. | The handshake still needs CPU; passive side gets no notification, so FIN/Done is needed; memory must be registered. |
| **RDMA Read** | Removes RNDZ_REPLY; sender CPU not needed after START. | Receiver must initiate the read, so it must notice RNDZ_START. |
| **Receiver-side progress problem** | (the problem) | Read starts late if the receiver is computing (slides 9, 11). |
| **Interrupt** | Wakes the library on arrival of START. | Overhead, jitter, thread-safety. |
| **Speculative RTR** | For receiver-first, advertise the destination early. | Speculation can be wrong; stale RTR races. |
| **RDMA Write** | The sender, who is late and inside `MPI_Isend`, initiates immediately. | Sender must still send Done; needs registered receive buffer. |
| **Result** | Overlap in both arrival orders without interrupts (when the prediction is right). | Complexity and fallback handling. |

---

# 9. Comparison Tables

## A. Eager vs Rendezvous

| Property | Eager | Rendezvous |
| --- | --- | --- |
| Message size | Small | Large |
| Handshake | None (one message) | RNDZ_START / REPLY (or RTS/RTR) before data |
| Payload travels with envelope? | Yes | No (control first, data later) |
| Library buffering | Required (both sides) | Only for control messages |
| Copies | Up to 2 (library buffers) | Zero-copy (user buffer ↔ user buffer) |
| Memory registration | Library's pre-registered buffers | User buffers need registration |
| Latency (small) | Low | Higher (extra round trip) |
| Bandwidth (large) | Hurt by copies | Best |
| Sender completion | Can complete before the receive is posted | Cannot complete until the receive is matched and data is delivered |
| Receiver not ready yet | Data stored in the unexpected queue | Only START stored; data waits |
| Unsafe code (send-send) | Often works (hides the bug) | Exposes the deadlock |
| Overlap potential | Little to hide (small) | Depends on progress |

## B. Blocking vs Non-blocking MPI Communication

| Property | Blocking (`MPI_Send/Recv`) | Non-blocking (`MPI_Isend/Irecv` + `Wait`) |
| --- | --- | --- |
| Returns when | Buffer safe to reuse / data in buffer | Immediately |
| Completion check | Implicit | Explicit (`Wait`/`Test`) |
| Buffer reuse | After the call returns | Only after completion |
| Overlap | Not within the call | Possible, not guaranteed |
| Deadlock risk | Higher with unsafe ordering | Lower (posting order doesn't block) |
| Progress | In the call | Only in MPI calls (usually) |
| Programmer effort | Low | Higher (manage requests/buffers) |

## C. RDMA Read vs RDMA Write

| Property | RDMA Write | RDMA Read |
| --- | --- | --- |
| Initiator's data | Goes **out** (pushed) | Comes **in** (pulled) |
| In Rendezvous, initiator is | Sender | Receiver |
| Needs from the passive side | Destination addr + rkey (remote-write) | Source addr + rkey (remote-read) |
| Control messages (RNDZ) | START, REPLY, FIN | START, FIN |
| Passive side notified? | No | No |
| FIN direction | Sender → Receiver | Receiver → Sender |
| Needs which CPU after START? | Sender (to post write after REPLY) | Receiver (to post read after START) |
| Good when | Receiver arrives first (with RTR) | Sender arrives first |
| Drawback | Extra REPLY round; needs sender CPU | Receiver may not notice START |

## D. RDMA Read vs RDMA Read + Interrupt

| Property | RDMA Read | RDMA Read + Interrupt |
| --- | --- | --- |
| How the receiver notices START | Only on entering MPI | NIC interrupts the CPU on arrival |
| Read starts at | Receiver's `MPI_Wait` (if receiver was computing) | Right after the interrupt, mid-compute |
| Sender-side overlap | Yes | Yes |
| Receiver-side overlap | **No** | **Yes** |
| CPU overhead | Low | Interrupt cost per message |
| Jitter/noise on compute | None | Yes |
| Complexity | Low | Thread-safety, handler design |

## E. Traditional Rendezvous vs Speculative RTR

| Property | Traditional (RTS/CTS or START/REPLY) | Speculative RTR |
| --- | --- | --- |
| Who sends the first control message | Sender (RTS) | **Receiver** (RTR) |
| Sent before the other side posts? | No (reacts to a posted operation) | **Yes**, by design |
| Handshake before data | Yes | None if prediction is right |
| RDMA op | Write (after REPLY) or Read | **Write** |
| Best arrival order | Either, but needs progress | Receiver first |
| Wrong guess consequence | None (nothing is speculated) | Wasted RTR, stale-RTR race |
| Registration timing | At need | Early (before the sender shows up) |
| Wildcard receives | Fine | Problematic |
| Complexity | Moderate | Higher (IDs, cancel, crossing messages) |

---

# 10. Common Misconceptions

1. **"`MPI_Isend` means the message was sent."** It only *starts* the operation. Completion needs `MPI_Wait`/`Test`.
2. **"Non-blocking guarantees overlap."** No. Overlap needs progress while computing (slide 8).
3. **"`MPI_Wait` is optional once the data arrives."** Required for guaranteed completion, buffer reuse and freeing the request.
4. **"Rendezvous causes deadlock."** The *program's circular ordering* does. Rendezvous exposes it.
5. **"Eager working means the code is correct."** It may only be hidden by buffering.
6. **"Receiver not ready = refuses."** It means *not ready yet*. Receivers can't refuse.
7. **"If the receiver never posts a receive, the sender still finishes."** In Rendezvous it can't, and in Eager it's a leaked message and a bug.
8. **"RDMA Read: the sender pulls the data."** The **receiver (initiator)** pulls from the sender's memory.
9. **"RDMA Write: the receiver pulls."** The **initiator pushes** into remote memory.
10. **"Local buffer = my array."** The local buffer belongs to the MPI library. Your array is the user buffer.
11. **"Eager 'buffered' means `MPI_Bsend`."** Different thing: library staging vs user-attached buffer.
12. **"Speculative RTR is certain."** It is a prediction and may be wrong.
13. **"Speculative RTR needs no fallback."** Stale RTRs, crossing messages and cancellation need handling.
14. **"Async-looking API = async network."** Non-blocking is only the *interface*. Hardware and library may not progress in the background.
15. **"RDMA means no CPU is ever needed."** Data movement needs no CPU, but handshakes, matching, FIN/Done and completion still need progress.
16. **"Interrupts are free."** They cost time and add jitter.
17. **"Lack of overlap breaks the program."** It slows it (performance), not correctness.
18. **"RDMA Write notifies the receiver."** One-sided, so the receiver is not told. That is why FIN/Done exists.
19. **"Pages 10–12 are about Eager."** The titles say "Eager vs Rendezvous" but the diagrams show Rendezvous only.
20. **"Rendezvous never copies."** True for payload in the ideal case, but it needs memory registration, which is costly (often mitigated by caching registrations).

---

# 11. Exam Material

## A. Key Definitions (15)

1. **Eager protocol:** sender transmits envelope + data immediately without handshake; data is buffered by the MPI library at the receiver.
2. **Rendezvous protocol:** sender and receiver first handshake, then data is transferred directly between user buffers.
3. **Local (library) buffer:** MPI-library-owned staging memory, distinct from the user buffer.
4. **User buffer:** application memory passed to MPI calls.
5. **Unexpected message:** a message that arrives before a matching receive is posted.
6. **RNDZ_START / RTS:** control message announcing a large message (envelope, size).
7. **RNDZ_REPLY / CTS:** receiver's go-ahead after matching a receive.
8. **FIN / Done:** message signalling that the transfer has completed.
9. **Non-blocking communication:** call returns immediately; completion is checked later with Wait/Test.
10. **Communication/computation overlap:** doing computation while communication progresses, hiding communication time.
11. **MPI progress:** the library advancing pending protocol steps. Usually only inside MPI calls.
12. **RDMA Write / RDMA Read:** one-sided ops where the initiator pushes data to / pulls data from remote registered memory without the remote CPU.
13. **Memory registration:** pinning memory and giving the NIC access, producing lkey/rkey.
14. **RTR (Ready To Receive):** receiver-sent message with buffer address and key telling the sender it can write now.
15. **Speculative RTR:** an RTR sent in advance, guessing Rendezvous will be used.

## B. Ten Conceptual Questions (to practise answering)

1. Why does Eager need library buffering, and why does that not scale to large messages?
2. What does each of RNDZ_START, RNDZ_REPLY, DATA and FIN do?
3. Why can the same program work for small messages and deadlock for large ones?
4. What does `MPI_Isend` guarantee, and what does it not?
5. Why can communication fail to progress while the application computes?
6. Why does the RDMA Read protocol improve sender-side overlap?
7. Why does the receiver-side progress problem arise, and how does an interrupt help?
8. How does the arrival order (sender-first vs receiver-first) change the RDMA Read protocol's behaviour?
9. Why is the RTR called speculative, and what can go wrong?
10. Why is `MPI_Wait` still needed in every protocol?

## C. Ten Likely Exam Questions with Concise Answers

**1. Compare Eager and Rendezvous.** Eager: small messages, one message, library buffers, up to two copies, low latency, can hide unsafe code. Rendezvous: large messages, handshake (START/REPLY/FIN), zero-copy user-to-user, low memory need, extra latency.

**2. Draw and explain the write-based Rendezvous protocol.** Sender→RNDZ_START; Receiver→RNDZ_REPLY (dest addr + key); Sender→DATA (RDMA Write); Sender→FIN. START announces; REPLY says the receiver is ready and where to write; DATA moves the payload; FIN tells the receiver the data is complete.

**3. Explain how blocking sends can deadlock.** If both ranks call `MPI_Send` first, with a large message (Rendezvous) each waits for a REPLY that requires the other to post a receive, a circular wait. Eager hides it by buffering. Fix with `MPI_Sendrecv` or `Isend/Irecv/Waitall`.

**4. Does non-blocking communication guarantee overlap?** No. It only returns control early. Overlap needs the protocol to advance during computation; if progress occurs only inside MPI calls, the transfer happens in `MPI_Wait` and nothing is hidden. This costs performance, not correctness.

**5. Explain why slide 8 shows no overlap.** In write-based Rendezvous, RNDZ_START arrives while the receiver computes (unseen until its Wait), and RNDZ_REPLY arrives while the sender computes (unseen until its Wait). DATA starts only when both are in MPI, so total time ≈ `Tc + Tm`.

**6. How does RDMA Read-based Rendezvous work, and what is its benefit?** START carries the sender's address + key; the receiver RDMA-Reads the data and sends FIN. It removes REPLY and frees the sender CPU, because the sender's NIC serves the read, so the sender overlaps.

**7. What is the receiver-side progress problem and the interrupt solution?** The receiver (initiator) must run MPI code to post the read, but it is computing, so the read is delayed to `MPI_Wait`. An interrupt on RNDZ_START arrival runs the library immediately to post the read, giving overlap, at the cost of interrupt overhead and jitter.

**8. Explain RDMA Read vs RDMA Write in Rendezvous.** MPI data always goes sender→receiver. Write: sender initiates and pushes. Read: receiver initiates and pulls from sender memory. FIN direction flips (the passive side must be told).

**9. Explain Speculative RTR + RDMA Write.** Receiver posts `Irecv`, sends an RTR with buffer address + key. When the sender posts `Isend` (large), it RDMA-Writes immediately, then sends Done. No handshake delay and no receiver CPU needed. If the sender used Eager instead, the RTR is stale and must be cancelled or ignored to avoid corruption.

**10. Why is `MPI_Wait` needed even though RDMA is one-sided?** Wait guarantees completion (buffer reusable / data valid), consumes the FIN/Done notification (RDMA gives no notification to the passive side), frees the request, and drives progress.

## D. One-Page Revision Sheet

**Protocols**

```
EAGER:      S --[env+data]--> R   (copy to lib buf, copy to user buf)
RNDZ WRITE: S -START-> R ; R -REPLY(addr,rkey)-> S ; S =RDMA WRITE=> R ; S -FIN-> R
RNDZ READ:  S -START(addr,rkey)-> R ; R <=RDMA READ= S mem ; R -FIN-> S
SPEC RTR:   R -RTR(addr,rkey)-> S (before Isend) ; S =RDMA WRITE=> R ; S -Done-> R
```

**Key facts**

- Eager: small, buffered, fast, hides unsafe code. Rendezvous: large, zero-copy, handshake, low memory.
- Rendezvous doesn't cause deadlock; circular blocking order does.
- "Not ready" = not ready yet. Never posting a receive = program bug (Rendezvous sender can't complete).
- `Isend`/`Irecv` start only; `Wait`/`Test` completes. Buffer untouchable until completion.
- Overlap: none `Tc+Tm`, perfect `max(Tc,Tm)`. Lack of overlap is a **performance** issue only.
- Progress is usually only inside MPI calls. Fixes: NIC offload, helper thread, interrupts, protocol redesign, `MPI_Test` in compute loops.
- RDMA is one-sided: the passive side isn't notified, so FIN/Done is required. Memory must be registered (addr + rkey).
- **Write = initiator's data goes out. Read = data comes in to the initiator.** MPI direction is always S→R.

**Comparisons**

- Write-based: 3 control msgs, sender CPU needed after REPLY. Read-based: 2 control msgs, sender CPU free, receiver must notice START.
- Sender first + Read → receiver's `Irecv` starts the read (good). Receiver first + Read → read starts only at `Wait` (bad).
- Interrupt: fixes the receiver's progress at the cost of overhead/jitter. Speculative RTR + Write: late sender initiates, no handshake, but the guess may be wrong (stale RTR, races, wasted registration, no wildcards).

**Evolution:** Eager → (buffer cost) → Rendezvous → (waiting) → Non-blocking → (want hidden comm) → Overlap → (library only progresses in MPI) → RDMA → Read → (receiver must notice START) → Interrupt → (costly; receiver-first) → Speculative RTR + Write → good overlap in both arrival orders.

**Principle:** the first to arrive advertises its buffer (RTS or RTR). The late arriver, already inside MPI, initiates the RDMA op (Read or Write).