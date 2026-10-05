# Key flows

[Back to README](../README.md)

Sequence diagrams for the platform's most important flows.

- [1. Sign-in with 2FA](#1-sign-in-with-2fa)
- [2. Registration and guarantee deposit](#2-registration-and-guarantee-deposit)
- [3. Live bidding, closing and winner payment](#3-live-bidding-closing-and-winner-payment)

---

## 1. Sign-in with 2FA

Users sign in with email and password, protected by **TOTP two-factor authentication** (Laravel Jetstream).

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant B as Browser
    participant L as Laravel app
    participant DB as MySQL

    U->>B: Enter email and password
    B->>L: POST /login
    L->>DB: Find user, verify password hash
    DB-->>L: User found, 2FA enabled
    L-->>B: Redirect to 2FA challenge
    U->>B: Enter 6-digit code from authenticator app
    B->>L: POST /two-factor-challenge
    L->>L: Verify TOTP code against stored secret
    Note over L: A recovery code is accepted instead if the device is lost
    L->>DB: Create session
    L-->>B: Session cookie, redirect to dashboard
```

**Why TOTP?** The platform handles deposits and payments, so I added a second factor beyond the password. Jetstream's built-in support meant this didn't need a custom implementation.

---

## 2. Registration and guarantee deposit

Before bidding, a user finishes sign-up through an email link, then pays the guarantee deposit required for the active auction.

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant B as Browser
    participant L as Laravel app
    participant DB as MySQL
    participant E as Email
    participant MP as Mercado Pago
    actor A as Admin

    U->>B: Fill in sign-up form
    B->>L: POST /register
    L->>DB: Create user with a registration token
    L->>E: Send email with tokenized link
    E-->>U: Sign-up link
    U->>B: Open link
    B->>L: GET link with token
    L->>DB: Validate token
    U->>B: Complete profile (location, national tax ID)
    B->>L: Save profile
    L->>DB: Store user profile

    U->>B: Pay the guarantee deposit
    B->>MP: Pay with Mercado Pago (JS SDK checkout)
    MP-->>B: Redirect back with a payment ID
    B->>L: Return URL with payment ID
    L->>MP: Fetch payment by ID (server-to-server)
    MP-->>L: Payment status: approved
    L->>DB: Record payment, mark it active
    L-->>B: Deposit active — ready to bid
    Note over B,L: No webhook backs this up — a user who never returns after paying has no deposit recorded

    opt Exception handling
        A->>L: Open the admin panel
        A->>L: Activate a deposit for this user manually
        L->>DB: Record payment, mark it active
        Note over A,L: Used for cases outside the normal flow, such as a payment arranged by phone
    end
```

**Why re-fetch the payment instead of trusting the redirect?** A redirect's query string can be tampered with or arrive incomplete. Asking Mercado Pago directly for that payment's status means what gets saved reflects what the provider actually recorded.

**Why no webhook?** That's a gap, not a design choice — see "What I'd do differently today" in the [README](../README.md).

---

## 3. Live bidding, closing and winner payment

The live room runs over WebSockets. The Ratchet server forwards each message (bid or chat) to every other connected client, including the admin's live console. Bids are also posted to a small Laravel endpoint so they're persisted, not just broadcast.

```mermaid
sequenceDiagram
    autonumber
    actor B1 as Bidder A
    actor B2 as Bidder B
    participant WS as WebSocket server (Ratchet)
    participant L as Laravel app
    participant DB as MySQL
    actor A as Admin console

    B1->>WS: Connect (wss)
    B2->>WS: Connect (wss)
    A->>WS: Connect (wss)

    B1->>WS: Bid on lot (amount)
    WS-->>B2: Broadcast bid
    WS-->>A: Broadcast bid
    B1->>L: PUT offer for this lot and user
    L->>DB: Store the offer (overwrites this user's previous one)
    Note over L,DB: The amount isn't checked against the current highest offer before it's stored

    B2->>WS: Chat message
    WS-->>B1: Broadcast message
    WS-->>A: Broadcast message

    B2->>WS: Higher bid
    WS-->>B1: Broadcast bid
    WS-->>A: Broadcast bid
    B2->>L: PUT offer for this lot and user
    L->>DB: Store the offer

    A->>L: Close the lot from the live console
    L->>DB: Mark the lot as closed

    Note over A,DB: From here it's a manual process — the admin reads the stored offers,<br/>identifies the highest one as the winner, and contacts that bidder directly<br/>to arrange payment (the full lot price) outside the live room
```

**Why a pure relay?** A WebSocket server that only forwards messages is small, fast and easy to reason about, which is a good fit for one room with up to 50 people. The downside is that it doesn't check what it forwards, and the endpoint behind it doesn't either — it accepts whatever offer a client sends. Today I'd validate each bid on the server before accepting and broadcasting it, so the server is the single source of truth for the current highest offer.

**Why no automated winner flow?** The application doesn't determine or notify a winner on its own; closing a lot just marks it closed. Identifying the winner and collecting payment happened as a manual step outside the system.
