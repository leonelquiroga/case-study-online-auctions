# Key flows

[Back to README](../README.md)

Sequence diagrams for the platform's most important flows. Steps marked **(simplified)** are inferred to explain the flow and may differ from the original implementation in details.

- [1. Sign-in: email and password with 2FA, and Google](#1-sign-in-email-and-password-with-2fa-and-google)
- [2. Registration and guarantee deposit](#2-registration-and-guarantee-deposit)
- [3. Live bidding, closing and winner payment](#3-live-bidding-closing-and-winner-payment)

---

## 1. Sign-in: email and password with 2FA, and Google

Users can sign in with email and password, protected by **TOTP two-factor authentication** (Laravel Jetstream), or with their **Google account**.

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant B as Browser
    participant L as Laravel app
    participant DB as MySQL
    participant G as Google

    alt Email and password
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
    else Google sign-in
        U->>B: Click "Sign in with Google"
        B->>G: Request Google sign-in and consent
        G-->>B: Return to the app with Google identity
        B->>L: Google identity (simplified)
        L->>G: Verify identity (simplified)
        L->>DB: Find or create user by email (simplified)
        L->>DB: Create session
        L-->>B: Session cookie, redirect to dashboard
    end
```

<!-- TODO: Describe the Google sign-in implementation (Socialite isn't a dependency), so the Google branch can be made precise. -->

**Why both?** Google sign-in removes friction for most users. Email and password with TOTP 2FA gives users without a Google account (or who prefer not to link one) an equally strong option, which matters on a platform that handles money.

---

## 2. Registration and guarantee deposit

Before bidding, a user has to (1) finish sign-up through an email link, (2) pay a guarantee deposit for a specific auction, and (3) get that deposit enabled by an admin.

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
    L->>DB: Create user and one-time token
    L->>E: Send email with tokenized link
    E-->>U: Sign-up link
    U->>B: Open link
    B->>L: GET link with token
    L->>DB: Validate token
    U->>B: Complete profile (location, personal data)
    B->>L: Save profile
    L->>DB: Store user profile, invalidate token (simplified)

    U->>B: Choose an auction
    B->>L: Request guarantee deposit
    L->>MP: Create payment preference (PHP SDK)
    MP-->>L: Preference ID
    L-->>B: Checkout with preference ID
    B->>MP: Pay with Mercado Pago (JS SDK checkout)

    par User returns to the site
        MP-->>B: Redirect back with payment status
        B->>L: Return URL with payment status
    and Asynchronous notification
        MP->>L: Webhook notification (simplified)
        L->>MP: Fetch payment by ID to confirm status (simplified)
    end
    L->>DB: Record payment, deposit pending approval

    A->>L: Open deposits in admin panel
    L->>DB: List pending deposits
    A->>L: Enable deposit for this user and auction
    L->>DB: Mark deposit as enabled
    Note over U,L: The user can now bid in this auction's live room
```

**Why manual approval?** Payment confirmation proves the money arrived. Admin approval lets the auctioneer also check who the bidder is before they can bid, so a human stays in control of who joins a live auction.

**Why handle both the redirect and the notification?** The redirect gives users instant feedback, but it fails if they close the tab before returning. The server-to-server notification makes sure the payment is recorded anyway.

---

## 3. Live bidding, closing and winner payment

The live room runs over WebSockets. The Ratchet server forwards each message (bid or chat) to every other connected client, including the admin's live console.

```mermaid
sequenceDiagram
    autonumber
    actor B1 as Bidder A
    actor B2 as Bidder B
    participant WS as WebSocket server (Ratchet)
    participant L as Laravel app
    participant DB as MySQL
    actor A as Admin console
    participant MP as Mercado Pago

    B1->>WS: Connect (wss)
    B2->>WS: Connect (wss)
    A->>WS: Connect (wss)

    B1->>WS: Bid on lot (amount)
    WS-->>B2: Broadcast bid
    WS-->>A: Broadcast bid
    B1->>L: Record bid (simplified)
    L->>DB: Store bid in lot_bids (simplified)

    B2->>WS: Chat message
    WS-->>B1: Broadcast message
    WS-->>A: Broadcast message

    B2->>WS: Higher bid
    WS-->>B1: Broadcast bid
    WS-->>A: Broadcast bid

    Note over A,L: The lot closes (closing rule pending confirmation)
    A->>L: Close lot (simplified)
    L->>DB: Determine highest bid, mark winner (simplified)
    L-->>B2: Notify winner (channel pending confirmation)

    B2->>L: Open payment
    L->>MP: Create payment preference
    MP-->>L: Preference ID
    B2->>MP: Pay with Mercado Pago
    MP-->>L: Payment status (redirect and notification)
    L->>DB: Record winner payment
    A->>L: Auction report
```

<!-- TODO: How did a lot close: a timer, or the admin from the live console? -->
<!-- TODO: Was the winner notified by email, on screen, or both? -->
<!-- TODO: Did the winner pay the full amount or the balance after the guarantee deposit? -->
<!-- TODO: Where were bids persisted: did the client post them to Laravel, or did the admin console record the result? -->

**Why a pure relay?** A WebSocket server that only forwards messages is small, fast and easy to reason about, which is a good fit for one room with up to 50 people. The downside is that it doesn't check what it forwards. Today I'd validate and persist each bid on the server *before* broadcasting it, so the server is the single source of truth for the current highest bid.
