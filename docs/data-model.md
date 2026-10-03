# Data model

[Back to README](../README.md)

The schema is based on the original project's migrations. Table names are translated to English here.

> **Note:** table names and their purpose come from the real migrations. Columns and relationships labeled **"simplified"** are inferred to explain the model and may not match the original columns exactly.

## Entity-relationship diagram

```mermaid
erDiagram
    users ||--o| user_profiles : "has"
    users ||--o{ sessions : "has"
    users ||--o{ guarantee_deposits : "pays"
    users ||--o{ payments : "makes"
    users ||--o{ lot_bids : "places"
    users ||--o{ chat_messages : "writes"

    countries ||--o{ provinces : "contains"
    provinces ||--o{ cities : "contains"
    cities ||--o{ user_profiles : "located in"

    auctions ||--o{ lots : "contains"
    auctions ||--o{ guarantee_deposits : "requires"
    auctions ||--o{ chat_messages : "hosts"

    lots ||--o{ lot_attachments : "has images"
    lots ||--o| vehicle_details : "extends"
    lots ||--o{ lot_bids : "receives"

    guarantee_deposits ||--o| payments : "paid by"

    users {
        bigint id PK
        string name
        string email UK
        string password
        text two_factor_secret
        text two_factor_recovery_codes
        string role "simplified: user or admin"
        timestamp email_verified_at
    }

    user_profiles {
        bigint id PK
        bigint user_id FK
        bigint city_id FK "simplified"
        string document_number "simplified"
        string phone "simplified"
        string address "simplified"
    }

    countries {
        bigint id PK
        string name
    }

    provinces {
        bigint id PK
        bigint country_id FK
        string name
    }

    cities {
        bigint id PK
        bigint province_id FK
        string name
    }

    auctions {
        bigint id PK
        string title "simplified"
        datetime starts_at "simplified"
        string status "simplified"
        decimal deposit_amount "simplified"
    }

    lots {
        bigint id PK
        bigint auction_id FK
        string category "vehicles, trucks, machinery, materials, pools"
        string title "simplified"
        decimal base_price "simplified"
        bigint winner_user_id FK "simplified"
    }

    lot_attachments {
        bigint id PK
        bigint lot_id FK
        string file_path "simplified"
    }

    vehicle_details {
        bigint id PK
        bigint lot_id FK
        string make "simplified"
        string model "simplified"
        int year "simplified"
        string plate "simplified"
    }

    guarantee_deposits {
        bigint id PK
        bigint user_id FK
        bigint auction_id FK
        decimal amount "simplified"
        boolean enabled "simplified: set by admin"
    }

    payments {
        bigint id PK
        bigint user_id FK
        bigint guarantee_deposit_id FK "simplified"
        string provider_payment_id "simplified: Mercado Pago ID"
        string status "simplified"
        decimal amount "simplified"
    }

    lot_bids {
        bigint id PK
        bigint lot_id FK
        bigint user_id FK
        decimal amount "simplified"
        datetime created_at
    }

    chat_messages {
        bigint id PK
        bigint auction_id FK "simplified"
        bigint user_id FK
        text message "simplified"
        datetime created_at
    }

    sessions {
        string id PK
        bigint user_id FK
        string ip_address
        text user_agent
        text payload
        int last_activity
    }
```

## Tables

| Table (English name) | Original purpose | Notes |
|---|---|---|
| `users` | Accounts and credentials | Includes Jetstream's TOTP 2FA columns (secret and recovery codes) |
| `user_profiles` | Personal data needed to bid | One profile per user, linked to a city |
| `countries` / `provinces` / `cities` | Location catalog | Three-level hierarchy for addresses |
| `auctions` | An auction event | Groups lots; deposits are paid per auction |
| `lots` | An item being auctioned | Categories: vehicles, trucks, machinery, materials, swimming pools |
| `lot_attachments` | Lot images | Uploaded from the admin panel |
| `vehicle_details` | Extra data for vehicle lots | Only vehicle lots have a row here |
| `guarantee_deposits` | Deposit a user pays to bid in an auction | An admin enables each deposit per user |
| `payments` | Payment records from Mercado Pago | Covers deposits and, likely, winner payments (simplified) |
| `lot_bids` | Link between users and lots | Original table: `usuarios_lote`. Its exact role (bids, participants or winners) is simplified here |
| `chat_messages` | Live room chat history | |
| `sessions` | Laravel database sessions | Standard Laravel table |

<!-- TODO: Confirm what the original `usuarios_lote` table stored: every bid, only the highest bid, or the winner per lot. -->
<!-- TODO: Confirm whether winner payments were stored in `payments` too, and how they linked to a lot. -->

## Design notes

- **Category-specific data in its own table.** Instead of adding nullable vehicle columns to every lot, vehicle data lives in `vehicle_details` (one-to-one with `lots`). Other categories don't pay for columns they never use.
- **Deposits are tied to an auction, not only to a user.** A user picks an auction, pays the deposit for it, and an admin enables that specific deposit. Bidding rights come from an approved deposit, not from the account itself.
- **Normalized location catalog.** Countries, provinces and cities live in separate tables, so profile addresses use consistent, selectable values instead of free text.
