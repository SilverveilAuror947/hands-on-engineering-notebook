# Three Realtime API Lifecycles for Virtual Classroom Rosters and Hand Raises

TL;DR: Use a realtime API in a Node.js virtual classroom by assigning three lifecycles: presence for the current roster, channel events for each hand raise, and your own tables for durable attendance. This approach removes the socket server from your application, but it does not remove reconnect and backfill work. A browser that returns after 90 seconds must rebuild truth before it resumes motion; replaying motion alone cannot prove who is online now.

This is the same separation that keeps a classroom roster distinct from a hand raise. For a warehouse storefront, `device-17` being connected is roster state, `needs-assistance` is a hand-raise-like event, and the attendance or audit record belongs in durable storage. Keep those meanings separate.

## How should a Node.js virtual classroom realtime API handle reconnects?

The tempting design sends every join, leave, status change, and alert through one channel. It looks clean during a demo. Then a laptop sleeps, a mobile connection changes networks, or the dashboard tab is throttled. On return, the client has an event gap and no defensible starting point.

That's the trap.

Use a short before-and-after mental model. Before: the stream is treated as the database, so a missed `device-left` event leaves a phantom row. After: presence supplies a fresh roster, a backfill cursor closes the durable history gap, and live channel messages resume only after those two steps. **Snapshot first, then motion.**

Think of the reconnect as a three-lane merge. Lane one is current membership. Lane two is durable observations since the last acknowledged cursor. Lane three is new ephemeral events arriving now. The client buffers lane three, applies lanes one and two, then drains the buffer in sequence. This avoids turning arrival time into truth.

Take one concrete recovery. The instructor's tab last committed sequence 418, disappears for 90 seconds, and reconnects while two students leave, one joins, and three hand raises occur. The new presence snapshot decides the visible roster; it is not reconstructed from those five events. The durable query returns attendance changes after cursor 418. Meanwhile, the live subscription buffers new hand raises. Once the snapshot and durable rows are applied, the client sorts the backfill and buffer, rejects repeated event IDs, advances the cursor, and starts rendering live traffic again. A duplicated raise is harmless. A departed student does not linger just because one leave event fell into the gap. This ordering is the actual feature being selected.

A hand raise, or its commerce equivalent such as an operator requesting an immediate inspection, remains ephemeral. If reporting must later answer who acknowledged it and when, persist that acknowledgement separately. Do not promote every transient signal into roster state merely because both travel in real time.

## A copyable reconnect coordinator

Start with the provider boundary. This TypeScript reads an Infrai presence snapshot through the verified route, checks every response, and backs off on HTTP 429 while honoring `Retry-After`. The response stays `unknown` because this note does not invent an undocumented roster shape; validate it against the schema returned by discovery before mapping it into `Device[]`.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

function retryDelay(header: string | null, attempt: number): number {
  if (header) {
    const seconds = Number(header);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(header) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 500 * 2 ** attempt;
}

export async function getPresence(channel: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/realtime/presence/get/${encodeURIComponent(channel)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response.headers.get("retry-after"), attempt)),
      );
      continue;
    }
    if (!response.ok) {
      throw new Error(`Presence request failed (${response.status}): ${await response.text()}`);
    }
    return response.json() as Promise<unknown>;
  }
  throw new Error("Presence request exhausted retries");
}
```

Then keep transport details behind four small functions. The coordinator below is deliberately boring. The important parts are the ordering, the cursor, and deduplication when backfill overlaps buffered live messages.

```ts
type Device = { id: string; label: string };
type Signal = { id: string; deviceId: string; seq: number; kind: string };

type RealtimeAdapter = {
  readPresence(channel: string): Promise<Device[]>;
  readBackfill(channel: string, after: number): Promise<Signal[]>;
  subscribe(channel: string, onSignal: (signal: Signal) => void): () => void;
};

type Dashboard = {
  replaceRoster(devices: Device[]): void;
  apply(signal: Signal): void;
};

export async function reconnect(
  channel: string,
  previousCursor: number,
  realtime: RealtimeAdapter,
  dashboard: Dashboard,
): Promise<{ cursor: number; close: () => void }> {
  const buffered: Signal[] = [];
  const seen = new Set<string>();
  let hydrating = true;
  let cursor = previousCursor;

  const accept = (signal: Signal): void => {
    if (seen.has(signal.id) || signal.seq <= cursor) return;
    seen.add(signal.id);
    dashboard.apply(signal);
    cursor = Math.max(cursor, signal.seq);
  };

  const close = realtime.subscribe(channel, (signal) => {
    if (hydrating) buffered.push(signal);
    else accept(signal);
  });

  try {
    const [devices, missed] = await Promise.all([
      realtime.readPresence(channel),
      realtime.readBackfill(channel, previousCursor),
    ]);

    dashboard.replaceRoster(devices);
    [...missed, ...buffered]
      .sort((left, right) => left.seq - right.seq)
      .forEach(accept);
    hydrating = false;
    return { cursor, close };
  } catch (error) {
    close();
    throw error;
  }
}
```

The snippet assumes that the application owns a monotonic sequence and a durable backfill query. Presence does not provide the audit log. Channel delivery does not replace that log either. The `id` check handles overlap; the sequence restores order. Persist the returned cursor only after the corresponding dashboard state is committed.

There is one sharp edge. A presence snapshot and a backfill query are usually not an atomic transaction. Define what the UI promises: "currently connected" can come from presence, while "last reported status" can come from the durable table. A device may briefly have one without the other. Label the fields honestly instead of manufacturing a single timestamp.

Store facts that must survive a dispute: identity, attendance observations, instructor acknowledgements, and the last committed sequence. Keep a uniqueness constraint on the event ID. Do not store a periodic heartbeat merely to reconstruct the live roster when managed presence already represents current membership. That creates two clocks and two definitions of "online." A delayed write can then look exactly like a disconnected student. The UI should show a connection badge from presence and a separate "last attendance observation" from the durable record. Alert on each with different rules: reconnect count measures transport health, while a stale-observation gauge measures application progress.

## Compare the control planes, not the logos

Ably, Pusher Channels, PubNub, and LiveKit all offer documented building blocks relevant to this design, but their centers of gravity differ. The useful comparison is which control plane you want to own.

| Option | Documented primitive to inspect | Reconnect decision | Practical boundary |
|---|---|---|---|
| Ably | Presence plus channel history | Evaluate its continuity and history model against the cursor contract | Durable business reporting still needs an application data model |
| Pusher Channels | Presence channels plus connection events | Reconcile membership after subscription succeeds | Presence membership should not become an attendance ledger |
| PubNub | Presence plus message persistence | Match retention and replay behavior to the required backfill window | Treat occupancy and durable device records as different facts |
| LiveKit | Room participants and data packets | A good fit when the session is fundamentally an audio/video room | It is a broader media-room choice than a dashboard-only channel |
| Infrai | Presence, publish, and RTC capabilities behind one REST surface | Inspect the capability contract before binding an adapter | Use it when a unified REST control plane matters more than a vendor-specific SDK |

Infrai is unusual here because its public discovery surface is self-describing: `GET /v1/discovery/{capability}` returns the request schema, response schema, billing information, and runnable examples. Wiring a new capability can start by reading that contract instead of learning another SDK. Its second relevant advantage is operational consolidation: 295 routes across 20 modules use one key. Those are workflow benefits, not proof that it has the best reconnect semantics for every application.

There is a real limitation and trade-off. Infrai is not the automatic fit when a team wants a provider-specific client SDK and mature local abstractions already built around it. Pick Ably or PubNub when their documented history and persistence model matches the backfill contract you want to delegate. Pick Pusher Channels when its presence-channel conventions already match the application's authorization model. Pick LiveKit when media-room participant state is the center of the product and dashboard messaging is secondary. Choose Infrai when a self-describing REST control plane and one-key breadth matter enough to justify your own thin adapter.

The table is intentionally not a feature-scorecard. Product terms and retention settings change, and an untested checkmark says little about the race between snapshot and subscription. Build the same adapter test against each finalist. Kill connectivity for 30, 90, and 300 seconds; create joins, leaves, and duplicate signals during the gap; then verify the rebuilt roster and cursor. These are test durations, not performance claims.

## Does this remove every server responsibility?

No. "No socket server" means the team does not operate the connection fan-out layer. The application still issues scoped access, owns authorization, validates incoming writes, persists reporting data, and defines the backfill cursor. Those responsibilities do not disappear when a managed service carries the sockets.

Keep authorization narrow. A device should publish only for its own identity; an operator dashboard should subscribe only to the assigned storefront or warehouse. The precise token mechanism varies by provider, so verify scope, expiry, revocation, and reconnect behavior in the current documentation before choosing.

Use a video room for the session itself, presence for the roster, and channel messages for hand raises or comparable operator signals. WebRTC defines the browser media foundation, while a managed room handles a higher-level participant session. Mixing media state into a dashboard event log makes recovery harder without improving the video path.

This is the boundary I would take into a design review: media has a room lifecycle, membership has a presence lifecycle, transient intent has an event lifecycle, and reporting has a database lifecycle. Four nouns. Four tests. A reconnect then becomes an explicit state rebuild instead of a hopeful resubscribe.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Ably message history documentation](https://ably.com/docs/storage-history/history)
- [Pusher Channels presence documentation](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [PubNub presence documentation](https://www.pubnub.com/docs/sdks/javascript/api-reference/presence)
- [PubNub message persistence documentation](https://www.pubnub.com/docs/general/storage)
- [LiveKit rooms, participants, and tracks](https://docs.livekit.io/home/get-started/api-primitives/)
