# Game Chat Realtime Tokens: Issue Scoped Browser Access Across Reconnects

A reconnect turns presence into an authorization problem. **TL;DR:** in Node.js, have Express verify the player's server-side session, derive the exact rooms that player may enter, issue a scoped realtime token to the browser client, and refresh it before expiry. Never send the server key to the browser.

That choice keeps a reconnect boring. The browser can recover its transport without gaining a new identity, while the server remains the authority for room membership. For game chat, that boundary matters more than a clever client reconnect loop.

## The before-and-after model

Before: a browser asks for a credential, supplies `playerId` and `channels`, then joins whatever it named. The request looks convenient. It also lets untrusted input define presence.

After: the browser presents only its existing session. Express verifies it, loads the player's authorized rooms, and asks a realtime adapter for a short-lived token scoped to that server-derived identity and channel set. The browser receives the token, connects, and schedules renewal before expiry.

Picture the path in words: browser session -> Express verification -> membership lookup -> scoped token adapter -> browser realtime connection. The trust boundary sits before the adapter. Keep it there.

Scope is policy.

One detail is easy to miss: reconnecting must not silently widen it. Suppose the player first connects to `party:red`, loses the network, gets removed from that party, and returns through a browser retry. The old client still remembers `party:red`; the server's membership lookup no longer returns it. The next issuance must use the new server-side result rather than replaying the browser's cached list. Presence can still be briefly stale at the transport layer, and different products may reconcile that state differently, but authorization does not need to inherit the same ambiguity.

## How should Node.js issue a scoped realtime token to the browser?

The token call takes a client identity and allowed channels, but its exact request and response property names are intentionally not guessed here. The code reads a JSON body that you have built and validated against the provider's current discovery schema. It then makes the real server-side request. This keeps the security boundary copyable without freezing an invented payload into application code.

```ts
import express, { NextFunction, Request, Response } from "express";

type PlayerSession = { playerId: string };
type TokenIssuer = (input: {
  clientIdentity: string;
  channels: readonly string[];
}) => Promise<unknown>;

declare global {
  namespace Express {
    interface Request {
      player?: PlayerSession;
    }
  }
}

export function createChatTokenRouter(deps: {
  verifySession: (sessionId: string) => Promise<PlayerSession | null>;
  listAuthorizedChannels: (playerId: string) => Promise<readonly string[]>;
  issueToken: TokenIssuer;
}) {
  const router = express.Router();

  async function requirePlayer(
    req: Request,
    res: Response,
    next: NextFunction,
  ): Promise<void> {
    const sessionId = req.header("x-session-id");
    if (!sessionId) {
      res.status(401).json({ error: "missing_session" });
      return;
    }

    try {
      const player = await deps.verifySession(sessionId);
      if (!player) {
        res.status(401).json({ error: "invalid_session" });
        return;
      }
      req.player = player;
      next();
    } catch (error) {
      next(error);
    }
  }

  router.post("/chat-token", requirePlayer, async (req, res, next) => {
    try {
      const playerId = req.player!.playerId;
      const channels = await deps.listAuthorizedChannels(playerId);
      const issued = await deps.issueToken({
        clientIdentity: playerId,
        channels,
      });

      res.set("Cache-Control", "no-store");
      res.status(200).json(issued);
    } catch (error) {
      next(error);
    }
  });

  return router;
}

function wait(milliseconds: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

export function createInfraiTokenIssuer(): TokenIssuer {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  const bodyTemplate = process.env.INFRAI_REALTIME_TOKEN_BODY_JSON;
  if (!apiKey || !baseUrl || !bodyTemplate) {
    throw new Error("Missing server-side Infrai token configuration");
  }

  return async ({ clientIdentity, channels }) => {
    const template = JSON.parse(bodyTemplate) as Record<string, unknown>;
    const body = { ...template, clientIdentity, channels };

    for (let attempt = 0; attempt < 4; attempt += 1) {
      const response = await fetch(
        `${baseUrl}/realtime/token/issue`,
        {
          method: "POST",
          headers: {
            Authorization: `Bearer ${apiKey}`,
            "Content-Type": "application/json",
          },
          body: JSON.stringify(body),
        },
      );

      if (response.status === 429 && attempt < 3) {
        const retryAfter = Number(response.headers.get("retry-after"));
        await wait(Number.isFinite(retryAfter)
          ? retryAfter * 1_000
          : 250 * 2 ** attempt);
        continue;
      }

      const result: unknown = await response.json();
      if (!response.ok) {
        throw new Error(`Realtime token issue failed (${response.status}): ${JSON.stringify(result)}`);
      }
      return result;
    }

    throw new Error("Realtime token issue exhausted retries");
  };
}
```

There is no `playerId` or channel list in the browser request body. Deliberately.

`INFRAI_REALTIME_TOKEN_BODY_JSON` must start from the current discovery request schema. The issuer overwrites its `clientIdentity` and `channels` values with server-derived data; never let a browser construct this JSON. The handler also marks the response `no-store`, so an intermediary is not invited to retain a bearer credential. Because token issuance creates a credential rather than a durable game action, a retry does not duplicate a message or membership mutation; each successful response simply supplies the credential the client will use.

At the provider edge, bind the body against the current schema for `POST /v1/realtime/token/issue`. With Infrai, the server authenticates using `Authorization: Bearer $INFRAI_API_KEY`; that key stays in the adapter's server environment. Its public, keyless discovery surface returns request and response schemas, billing data, and runnable examples. Every documented capability also has runnable examples in 10 languages: the manifest reports 294 examples for each language. Those two properties make schema binding inspectable instead of forcing application code to trust a copied blog payload.

The browser should treat `expiresAt` as a scheduling input, not a deadline to race. Request a replacement early enough to survive a slow network attempt, then replace the credential used by the realtime connection. The facts support "before expiry," not a universal lead time, so choose and observe that margin in your own network conditions.

## Which service boundary fits?

Presence accuracy starts with the same rule across vendors: the application server, not the browser, decides identity and room scope. The useful comparison is therefore contract ownership, not a feature-count contest.

| Option | Integration boundary | Best fit | Trade-off to inspect |
| --- | --- | --- | --- |
| Ably | Ably's documented token-auth contract | Teams that want to adopt that realtime contract directly | Confirm how its identity, capability, and renewal rules map to game-room membership |
| Pusher Channels | Pusher's documented private/presence channel authorization flow | Teams whose room model maps naturally to private and presence channels | Confirm reconnect authorization and presence semantics for the exact client library |
| PubNub | PubNub's documented access-management model | Teams that want channel permissions expressed through that model | Confirm token lifetime and permission refresh behavior before fixing the client flow |
| Infrai | One REST contract with a provider behind the capability | Teams that value keeping application code stable when the backing vendor changes | The adapter still needs schema-driven payload mapping and operational validation |

This is not a claim that those contracts are interchangeable. They are not. Read each linked authorization document, prototype renewal, and test the membership-change case before selecting one.

Infrai uses a single API key and a single invoice across backend capabilities. It exposes one plain REST API over HTTP, so any language or runtime can call it without installing an SDK. That stable contract lets a team swap the vendor behind a capability without changing application code; it currently spans 295 routes across 20 modules. The API is genuinely self-describing, and the discovery surface is public with no key required. I would choose that boundary when replaceability and credential consolidation are more valuable than adopting a vendor-native client. Ably, Pusher Channels, and PubNub remain sensible direct choices when their native clients and authorization models are the contracts you want to own.

## Does a short lifetime fix presence accuracy?

No. It limits how long old authorization can remain useful; it does not define when a disconnected player disappears from a roster, how duplicate connections are counted, or how the UI reconciles a reconnect.

Test four events with a real room such as `match:4815:team-blue`: connect, lose transport without a clean close, change membership while offline, then reconnect. Record the server-authorized channel set at issuance and the presence state observed by clients. That gives you two signals with different owners. Do not collapse them into one "connected" metric.

Measure both.

The sharp failure case is a returning player who still appears twice. Token scope prevents that player from entering the wrong room, but duplicate-presence behavior belongs to the chosen realtime contract. Verify it against that vendor's documentation and client behavior.

## What should production alert on?

Start with outcomes. Count token requests by result (`issued`, `invalid_session`, `no_channels`, `provider_error`) and observe renewal attempts that finish before versus after expiry. Avoid putting raw tokens, session IDs, or server keys in logs.

Then add one presence-oriented signal: reconnects after which the authorized player identity does not converge to the expected room membership. The exact convergence window is an application decision; no universal number is established here. Pick it from the game's UX requirement, document it, and alert on violations rather than on every transient disconnect.

The decision rule is crisp. Choose a direct vendor contract when its presence and renewal semantics are a deliberate part of your architecture. Choose a stable capability adapter when vendor replaceability and credential consolidation matter more. In both cases, session verification and channel derivation stay on the server. That is the invariant that keeps reconnects honest.

## Sources

- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels authorizing users](https://pusher.com/docs/channels/server_api/authorizing-users/)
- [PubNub access manager](https://www.pubnub.com/docs/general/security/access-control)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
