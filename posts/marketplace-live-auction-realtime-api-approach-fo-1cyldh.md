# Marketplace Live Auction: Realtime API Approach for Bidding Sequence Integrity

Choose a server-authoritative bid ledger, then treat every realtime channel as a delivery path rather than the judge. For a small SaaS team running marketplace auctions, this keeps bid sequence explainable while allowing presence to be approximate: accepted bids receive server sequence numbers, clients reconcile from that sequence, and notification delivery never decides who won.

**TL;DR:** start with WebSocket-style bid delivery plus an HTTP recovery path. Use Server-Sent Events when browsers only need downstream notifications. Reserve WebRTC data channels for peer-to-peer media or data cases where their extra signaling and topology are justified. In every option, presence must expire, reconnects must resume from a cursor, and the database transaction must settle ordering.

| Pick | Pick this when | Presence implication | Main operational burden |
| --- | --- | --- | --- |
| Bidirectional socket | A bidder sends bids and receives auction events on one long-lived channel | Heartbeats can refresh a short lease | Reconnect, backpressure, and per-connection state |
| Server-Sent Events plus HTTP | Bids can use ordinary HTTP while updates only flow server to browser | The stream proves recent connectivity, not continued attention | Cursor replay and browser connection lifecycle |
| WebRTC data channel | Peers already need direct realtime data alongside media | Peer connectivity describes a session, not marketplace authority | Signaling, connectivity negotiation, and topology |

The default is deliberately boring. It gives the team one authoritative write path and one observable event stream without confusing transport arrival time with auction order.

## How should a realtime API order live auction bidding?

No single signal does. An open connection says the transport was reachable recently. A heartbeat says a client ran recently. Page visibility says whether a document is visible. None proves that a person is watching, ready to bid, or receiving pixels at this instant. Model presence as a lease with an expiry, not a permanent Boolean, and keep three states that are useful to operations: `connected`, `stale`, and `offline`. A heartbeat extends the lease. Silence moves it to stale before offline. A clean disconnect can shorten the transition, but correctness cannot depend on receiving one. This distinction matters in a marketplace: “12 bidders online” is a display hint, while “Bid 842 was accepted before bid 843” is durable state. The first may lag. The second cannot.

The diagram in words is short: browser sends command, auction service validates it, database commits it, outbox publishes it, connected browsers receive it, and lagging browsers replay it. Presence sits beside that line. It does not enter the ordering decision.

## Pick this when the channel matches the job

A bidirectional socket is the practical fit when the same browser both submits commands and consumes rapid updates. Keep its contract narrow. A client sends a bid intent with an idempotency key; the server answers with accepted or rejected state and a server cursor. The connection does not grant authority.

Server-Sent Events are a strong shape when notification traffic is one-way. Submit a bid over HTTP, return the committed result, and stream subsequent auction changes downstream. This split often makes retries easier to inspect because command handling retains ordinary request semantics. The trade-off is two lifecycles instead of one.

WebRTC can carry arbitrary data between peers through `RTCDataChannel`, and the W3C specification defines its connection and channel behavior. That capability does not make peers suitable auction sequencers. A marketplace still needs a trusted service to validate funds, increments, auction state, and closing rules. Direct peer paths also make a single, auditable ordering point harder to explain. Pick this route only when the application already has a genuine peer-to-peer requirement.

One rule survives all three choices: **acknowledge a bid only after its authoritative state is committed.** A fast animation may be optimistic, but it must remain visually provisional until the acceptance event arrives.

## Build the authoritative path once

The implementation below sketches the boundary that matters. It intentionally leaves storage behind interfaces so the example does not pretend an in-memory counter is a fair auction database. Every accepted event gets an auction-local sequence inside the same transaction as the bid, while repeated idempotency keys return the original result.

```ts
type BidCommand = {
  auctionId: string;
  bidderId: string;
  amountCents: number;
  idempotencyKey: string;
};

type BidEvent = {
  auctionId: string;
  bidderId: string;
  amountCents: number;
  sequence: number;
  acceptedAt: string;
};

interface AuctionTransaction {
  findAccepted(key: string): Promise<BidEvent | undefined>;
  readAuctionForUpdate(auctionId: string): Promise<{
    open: boolean;
    currentAmountCents: number;
    nextSequence: number;
  }>;
  insertBid(command: BidCommand, event: BidEvent): Promise<void>;
  appendOutbox(event: BidEvent): Promise<void>;
}

interface AuctionStore {
  transaction<T>(work: (tx: AuctionTransaction) => Promise<T>): Promise<T>;
}

async function acceptBid(
  store: AuctionStore,
  command: BidCommand,
  now: () => Date,
): Promise<BidEvent> {
  return store.transaction(async (tx) => {
    const prior = await tx.findAccepted(command.idempotencyKey);
    if (prior) return prior;

    const auction = await tx.readAuctionForUpdate(command.auctionId);
    if (!auction.open) throw new Error("auction_closed");
    if (command.amountCents <= auction.currentAmountCents) {
      throw new Error("bid_too_low");
    }

    const event: BidEvent = {
      auctionId: command.auctionId,
      bidderId: command.bidderId,
      amountCents: command.amountCents,
      sequence: auction.nextSequence,
      acceptedAt: now().toISOString(),
    };

    await tx.insertBid(command, event);
    await tx.appendOutbox(event);
    return event;
  });
}
```

There are two sharp edges here. First, `acceptedAt` is useful evidence but should not sort competing client requests; server receipt and transaction serialization determine the committed sequence. Client clocks are outside that boundary. Second, publishing directly to connected clients before commit creates a phantom bid if the transaction later fails. The transactional outbox closes that gap by recording delivery work beside the accepted bid.

On reconnect, the browser sends its last applied sequence. The service replays later events, then resumes live delivery. The reducer ignores duplicates and refuses to jump over a gap. Tiny rule, huge payoff.

```ts
type AuctionView = {
  lastSequence: number;
  leadingAmountCents: number;
};

function applyBid(view: AuctionView, event: BidEvent): AuctionView {
  if (event.sequence <= view.lastSequence) return view;
  if (event.sequence !== view.lastSequence + 1) {
    throw new Error("sequence_gap");
  }
  return {
    lastSequence: event.sequence,
    leadingAmountCents: event.amountCents,
  };
}
```

A sequence gap should trigger replay, not guesswork.

## Observe fairness, presence, and delivery separately

One dashboard should not flatten three different questions. Record acceptance latency around the authoritative transaction, delivery lag from the committed event to each connection, and presence freshness from the last valid heartbeat. Then alert on symptoms tied to those boundaries.

Useful counters include rejected stale bids, duplicate idempotency keys, replay requests, sequence gaps, and outbox backlog. Useful distributions include commit latency and event delivery lag. Add auction ID and connection ID for correlation, but keep bidder identity out of broad metric labels; unrestricted identity values create an unbounded label set and make the operational view harder to use.

Test the ugly transitions. Send the same bid twice. Drop the connection after commit but before the acknowledgment. Deliver events twice. Pause one subscriber while later bids commit. Expire presence without a disconnect frame. Race a bid against auction closure. The invariant is crisp: each idempotency key maps to at most one accepted event, and each auction’s accepted events form one increasing sequence with no client-created entries.

For deployment, drain connections while leaving replay available. A new process can accept reconnects from a cursor even when an old process disappears. That makes connection churn visible without turning it into lost auction state.

## Limits and decision rule

This pattern does not define marketplace policy. Tie handling, minimum increments, reserve prices, payment authorization, and the exact closing boundary still need explicit business rules enforced in the authoritative transaction. Geographic distribution also complicates the location of that authority; do not imply global fairness from low delivery latency alone.

Pick the least complex channel that supports the traffic direction. Keep ordering in durable server state. Treat presence as expiring evidence. **If a transport can disconnect, duplicate, delay, or reorder a message, recovery by cursor is part of the protocol, not an optional enhancement.**

## References

- W3C, “WebRTC 1.0: Real-Time Communication Between Browsers”: https://www.w3.org/TR/webrtc/
