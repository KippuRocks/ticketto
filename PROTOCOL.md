---
authors: ["Pablo Dorado <hola@pablodorado.com>"]
start_date: 2024/01/14
published_date: N/A
version: 1.0.0-draft.0
summary: The Ticketto Protocol is a decentralised protocol to easily and securely issue, hold, and transfer tokens to grant access to events.
---

# The Ticketto Protocol

**Version**: 1.0.0-draft.0 (2024/01/14)

**Authors**:

- [P. Dorado](https://github.com/pandres95)

## Table of Contents

1. [Summary](#summary)
2. [Motivation](#motivation)
3. [Proposed Implementation](#proposed-implementation)
    1. [Definitions](#definitions)
    2. [Participants](#participants)
    3. [Modules](#modules)
    4. [Queries](#queries)
    5. [Commands](#commands)
4. [Conclusion](#conclusion)

## Summary

The Ticketto Protocol is a decentralised protocol to easily and securely issue, hold, and transfer tokens to grant access to events.

## Motivation

When organising an event, one of the crucial steps (aside from picking a venue and deciding the content that will be shown) is determining which people will attend to it, and how to verify people are granted to do so. This is an already solved problem: you either decide a closed list of attendees, or issue tickets and distribute among attendees through various means (like selling, giving away, or a combination of both).

Centralised ticketing systems overall tend to lack have of some (if not all) issues related to credibility, protection against fraud, falsification, and control over secondary markets (when intended).

NFTs provide the necessary tools to circumvent those issues. Also, providing a publicly auditable solution that helps other stakeholders (like venue owners and event promotors) gain credibility by giving information of the status of an event success in terms of ticket issuance/selling.

Finally, some tickets might grant multiple instances of acccess to an event, such a club membership or a ticket for an online conference.

This protocol aims to define a ticketing system based on NFTs, helping to solve the aforementioned issues.

## Proposed Implementation

### Definitions

- **Event**: A gathering, both physical and virtual. On-chain, events are usually defined as nonfungible collections.
- **Ticket**: A token used to get access to the event.  On-chain, tickets are usually defined as nonfungible items (a.k.a. NFTs).
- **Attendance**: The act of marking a timestamp on a ticket, indicating it has been used for having gotten access to an event. Tickets might support single or multiple attendances, depending on the rules applied to it.
- **Deferred transfer**: The act of transferring a ticket to an unknown account, via a commit-reveal scheme.

### Participants

- **Event organisers**: An account that creates an handles the state of an _event_.  
- **Event promoters**: An entity (person, company, etc.) enabled to sell _event tickets_ and receive a referral fee in exchange.
- **Ticket holders**: An account that owns a _ticket_.
- **Event attendee**: A **ticket holder**, that intend to get access to an _event_.
- **Entrance operators**: An entity (application, person) in charge of granting or denying access to an _event_.
<!-- - **Ticket issuer**: An account that can intermediate in the process of issuing the tickets and covering deposit costs to accounts, usually charging amounts via stablecoin assets. -->
- **Ticket claimer**: An account that owns a commit message enabling them to receive a _ticket_ that is pending to be transfered via _deferred transfer_.
- **Ticket resellers**: A **ticket holder** that is interested in reselling their own ticket on a secondary market.
- **Venue owners**: An entity that owns the venue where an _event_ will be held.

### Modules

- **Events Module**: Allows handling operations of an _event_.
- **Tickets Module**: Allows handling actions related to owning a _ticket_, such as _registering an attendance_, _deferred transfering_ and safely _selling a ticket_ using a fungible asset.
- **Attendances Module**: Allows handling actions related to attending an _event_ with a _ticket_, such as _registering an attendance_.

### Types

#### Ticket Restrictions (part of _Tickets Module_)

Enables some restrictions for the ticket at issuance time

```rs
#[derive(Default)]
struct TicketRestrictions {
    cannot_resale: bool;
    cannot_transfer: bool;
}
```

#### Attendance Type (part of _Tickets Module_)

Sets the behaviour for attending with a ticket. May be `Single` (i.e. entering a concert), `Multiple` (i.e. a fast pass with a limit of `n` usages), or `Unlimited` (a membership on a night club, or a day pass at a hotel) with an optional expiration date.

```rs
enum AttendancePolicy {
    Single,
    Multiple { max: u32, maybe_until: Option<Timestamp> },
    Unlimited { maybe_until: Option<Timestamp> },
}
```

### Queries

#### Attendance Validation (part of _Tickets Module_)

Returns whether an **event attendee** is able to get access to an _event_, depending on the attributes (_attendance_type_, _attendances_, _expiration_date_, etc.) marked on their _ticket_.

```rs
fn can_attend(
    event: EventId,
    ticket: TicketId
) -> bool
```

```mermaid
sequenceDiagram
    actor Attendee
    participant Tickets
    participant Nfts
    participant Env

    Attendee ->> Tickets: can_attend
    
    Tickets ->> Nfts: do_get_attribute "attendance_type"
    activate Nfts
        Nfts ->> Tickets: attendance_type
    deactivate Nfts
    
    Tickets ->> Env: block_timestamp
    activate Env
        Env ->> Tickets: now
    deactivate Env

    break when there's no attendance_type
        Tickets ->> Attendee: TicketFormatError
    end
    
    alt attendance_type is Unlimited
        Tickets ->> Attendee: now <= until
    else
        Tickets ->> Nfts: do_get_attribute "attentances"
        activate Nfts
            Nfts --) Tickets: maybe_attendances
        deactivate Nfts

        alt attendance_type is Single
            Nfts ->> Attendee: true if no attendances
        else attendance_type is Multiple
            Nfts ->> Attendee: true if max < attendances and now <= until
        end
    end

    Note over Attendee,Nfts: Do the same for other properties, depending on the rules defined for the ticket
```

### Commands

#### Creating an event (part of _Events Module_)

An **organiser** calls up a method called `create_event`, passing a definition of the event (`max_capacity` and (optional) `metadata`).

Then, an account on behalf of the protocol (a.k.a. the **issuer**) is assigned as admin, and the **organiser** is assigned as owner. This is done, so it's easier to the protocol can execute permissioned actions over collections and items.

```rs
fn create_event(
    origin: AccountId,
    capacity: MaxCapacity,
    maybe_metadata: Option<Vec<u8>>,
) -> Result<EventId>;
```

```mermaid
sequenceDiagram
    actor Organiser
    participant Events
    participant Nfts
    actor Issuer

    Organiser ->> Events: create_event
    Events ->> Nfts: do_create_collection
    activate Nfts
    Nfts --) Events: Created
    Nfts --) Issuer: assign as issuer / admin / freezer
    Nfts --) Organiser: assign as owner
    Nfts ->> Events: CollectionId
    deactivate Nfts
    par 
        opt request includes metadata
            Events ->> Nfts: do_set_metadata
            activate Nfts
            Nfts --) Events: CollectionMetadataSet
            deactivate Nfts
        end
    and
        Events ->> Nfts: do_set_max_supply
        activate Nfts
        Nfts --) Events: CollectionMaxSupplyset
        deactivate Nfts
    end
    Events ->> Organiser: CollectionId
```

#### Issuing a ticket (part of _Tickets Module_)

An **organiser** issues a _ticket_, including the `attendance_type`, and optionally stating some `attributes` to it and/or `metadata`.

If desired by the **event organiser**, a _ticket_ can be immediately minted on behalf of a **holder**.

It's possible for a _ticket_ to be restricted (for _selling_, such as always free tickets, or _transferring_, like scolarship-class tickets).

Finally, **organiser** can also set a `price`, and mark it as `for_sale` (if this option is true, and the price is set as `None`, a default price ).

```rs
fn issue_ticket(
    origin: AccountId,
    event: EventId,
    attendance_type: AttendancePolicy,
    maybe_holder: Option<AccountId>,
    maybe_attributes: Option<Vec<ItemAttribute>>,
    maybe_metadata: Option<Vec<u8>>,
    maybe_restrictions: Option<TicketRestrictions>,
    maybe_price: Option<{ asset: AssetId, amount: Balance }>,
    for_sale: bool,
) -> Result<TicketId>;
```

```mermaid
sequenceDiagram
    actor Organiser
    participant Tickets
    participant Nfts
    actor Holder
    actor Issuer

    Organiser ->> Tickets: issue_ticket
    
    Tickets ->> Nfts: do_mint_item
    activate Nfts
        Nfts --) Tickets: Issued
        alt there is a holder
            Nfts --) Holder: assign as owner
        else 
            Nfts --) Organiser: assign as owner
        end
        Nfts ->> Tickets: TicketId
    deactivate Nfts
    
    opt request includes metadata
        Tickets ->> Nfts: do_set_metadata
        activate Nfts
            Nfts --) Tickets: ItemMetadataSet
        deactivate Nfts
    end
    
    opt request includes restrictions
        note over Tickets,Nfts: Consider restrictions might also make items be locked for transfer
        loop every restriction
            Tickets ->> Nfts: do_set_attribute
            activate Nfts
                Nfts --) Tickets: AttributeSet
            deactivate Nfts
        end 
    end
    
    opt request includes price
        Tickets ->> Nfts: do_set_attribute
        activate Nfts
            Nfts --) Tickets: AttributeSet
        deactivate Nfts
    end

    alt item is for_sale
        Tickets ->> Nfts: do_item_lock_transfer
        Tickets ->> Nfts: set_transfer_delegate
        activate Nfts
            Nfts --) Tickets: ItemTransferLocked
            Nfts --) Issuer: assign as transfer delegate
        deactivate Nfts
    end

    Tickets ->> Organiser: TicketId
```

#### Selling a ticket (part of _Tickets Module_)

A **ticket holder** can mark a ticket as `for_sale` and set a `price` at any moment, unless the ticket is marked as `cannot_resale`.

```rs
fn sell_ticket(
    origin: AccountId,
    event: EventId,
    ticket: TicketId,
    price: { asset: AssetId, amount: Balance }
) -> Result<()>;
```

At any point, the **holder** may choose to stop selling the _ticket_.

```rs
fn withdraw_sell(
    origin: AccountId,
    event: EventId,
    ticket: TicketId,
) -> Result<()>;
```

```mermaid
sequenceDiagram
    actor Seller
    participant Tickets
    participant Nfts
    actor Issuer

    Seller ->> Tickets: sell_ticket

    Tickets ->> Nfts: do_get_attribute "restrict_resell"
    activate Nfts
        Nfts ->> Tickets: restrict_resell
    deactivate Nfts

    break restrict_resell exists
        Tickets ->> Seller: CannotSell
    end

    Tickets ->> Nfts: do_set_attribute "price"
    activate Nfts
        Nfts -->> Tickets: AttributeSet
    deactivate Nfts

    Tickets ->> Nfts: do_set_attribute "for_sale"
    activate Nfts
        Nfts -->> Tickets: AttributeSet
    deactivate Nfts

    Tickets ->> Nfts: do_item_lock_transfer
    Tickets ->> Nfts: do_item_set_transfer_delegate
    activate Nfts
        Nfts -->> Tickets: ItemTransferLocked
        Nfts --) Issuer: assign as transfer delegate
    deactivate Nfts

    Tickets -->> Seller: SetForSale
    
    Seller ->> Tickets: withdraw_sell
    
    Tickets ->> Nfts: do_get_attribute "for_sale"
    activate Nfts
        Nfts -->> Tickets: for_sale
    deactivate Nfts
    
    break if not for sale
        Tickets ->> Seller: NotForSale
    end
    
    Tickets ->> Nfts: do_item_unlock_transfer
    Tickets ->> Nfts: do_item_clear_transfer_delegate
    activate Nfts
        Nfts -->> Tickets: ItemTransferUnlocked
        Nfts --) Issuer: cleared as transfer delegate
    deactivate Nfts
    
    Tickets ->> Nfts: do_clear_attribute "for_sale"
    activate Nfts
        Nfts -->> Tickets: AttributeSet
    deactivate Nfts
```

#### Buying a ticket (part of _Tickets Module_)

When a ticket is marked as `for_sale`, an account holding the amount requested in the price would be able to execute this call and get the _ticket_ transferrd in exchange to the specified funds. Once sold, the protocol will modify the _ticket_ to remove the `price` and the `for_sale` flag.

```rs
fn buy_ticket(
    origin: AccountId,
    event: EventId,
    ticket: TicketId,
) -> Result<()>;
```

```mermaid
sequenceDiagram
    actor Buyer
    participant Tickets
    participant Nfts
    actor Seller

    Buyer ->> Tickets: buy_ticket

    Tickets ->> Tickets: do_get_attribute "for_sale"
    activate Tickets
        Tickets ->> Tickets: for_sale
    deactivate Tickets

    break ticket is not for sale
        Tickets ->> Buyer: NotForSale
    end


    Tickets ->> Assets: do_transfer_asset
    activate Tickets
        break if Buyer does not have enough funds
            Assets ->> Tickets: BalanceLow
            Tickets ->> Buyer: BalanceLow
        end
        Buyer --> Assets: [withdraws]
        Assets -->> Seller: [deposits]
        Assets -->> Seller: Transferred
        Assets -->> Buyer: Transferred
    deactivate Tickets

    Tickets ->> Nfts: do_transfer_item
    activate Nfts
        Nfts -->> Buyer: assigned as owner
        Nfts -->> Seller: unset as owner
        Nfts -->> Tickets: ItemTransferred
    deactivate Nfts

    Tickets -->> Buyer: TicketSold
    Tickets -->> Seller: TicketSold
```

#### Transferring a ticket (part of _Tickets Module_)

When a _ticket_ is not marked as `restrict_transfer`, its **ticket holder** may transfer it to another account, without the receiver needing to accept the transfer.

```rs
fn transfer_ticket(
    origin: AccountId,
    event: EventId,
    ticket: TicketId,
    receiver: AccountId,
) -> Result<()>;
```

```mermaid
sequenceDiagram
    actor Sender
    participant Tickets
    participant Nfts
    actor Receiver

    Sender ->> Tickets: transfer_ticket

    break if Ticket is resitrcted for transfer
        Tickets ->> Sender: CannotTransfer
    end

    Tickets ->> Nfts: do_transfer_item
    activate Nfts
        Nfts -->> Receiver: assigned as owner
        Nfts -->> Sender: unset as owner
        Nfts -->> Tickets: ItemTransferred
    deactivate Nfts

    Tickets -->> Sender: TicketTransferred
    Tickets -->> Receiver: TicketTransferredsequenceDiagram
```

#### Deferred transferring a ticket (part of _Tickets Module_)

When a _ticket_ is not marked as `restrict_transfer`, its **ticket holder** may commit for transfer using a [commit-reveal scheme](https://en.wikipedia.org/wiki/Commitment_scheme). This is so a receiver that initially doesn't own an account is able to claim it later.

The scheme starts by calling `defer_transfer`, with the `commit` message, and an optional `expiration` date.

```rs
fn defer_transfer(
    origin: AccountId,
    event: EventId,
    ticket: TicketId,
    commit_message: Vec<u8>,
    maybe_expiration: Optional<Timestamp>,
) -> Result<()>;
```

Once the receiver has created an account, they can call `claim_deferred_transfer` to reveal the commitment, using a `claim` message, receiving the transfered _ticket_ as a result.

```rs
fn claim_deferred_transfer(
    origin: AccountId,
    event: EventId,
    ticket: TicketId,
    claim_message: Vec<u8>,
) -> Result<()>;
```

It's possible, at any moment, that the **ticket holder** cancels the deferred transferring, thus unlocking the item for transfer to another party (or even, [selling](#selling-a-ticket-part-of-tickets-module) it).

```rs
fn cancel_deferred_transfer(
    origin: AccountId,
    event: EventId,
    ticket: TicketId,
) -> Result<()>;
```

```mermaid
sequenceDiagram
    actor Sender
    participant Tickets
    participant Nfts
    actor Issuer
    actor Claimer

    Sender ->> Tickets: defer_transfer

    break if Ticket is resitrcted for transfer
        Tickets ->> Sender: CannotTransfer
    end

    Tickets ->> Nfts: do_item_lock_transfer
    Tickets ->> Nfts: do_set_transfer_delegate
    activate Nfts
        Nfts --) Tickets: ItemTransferLocked
        Nfts --) Issuer: assign as transfer delegate
    deactivate Nfts

    Tickets ->> Tickets: set_transfer_claim
    
    alt
        Claimer ->> Tickets: claim_deferred_transfer

        activate Tickets

            Tickets ->> Nfts: do_transfer_item
            activate Nfts
                Nfts -->> Issuer: unset as transfer delegate
                Nfts -->> Sender: unset as owner
                Nfts -->> Claimer: assigned as owner
                Nfts --) Tickets: ItemTransferred
            deactivate Nfts

            Tickets -->> Sender: TicketTransferred
            Tickets -->> Claimer: TicketTransferred
        deactivate Tickets
        
    else Sender withdraws the transfer
        Sender ->> Tickets: withrdraw_deferred_transfer
        
        Tickets ->> Nfts: do_item_unlock_transfer
        Tickets ->> Nfts: do_clear_transfer_delegate
        activate Nfts
            Nfts --) Tickets: ItemTransferUnlocked
            Nfts --) Issuer: unassign as transfer delegate
        deactivate Nfts
        
        Tickets ->> Tickets: clear_transfer_claim

        Claimer ->> Tickets: claim_deferred_transfer
        Tickets ->> Claimer: NoTransferInPlace
    end
```

<!-- #### Future outcomes

It is possible that part of this module (the selling use cases) can be moved towards a new system pallet (a.k.a. `nfts-marketplace`, where also NFTs auctions can be included).  -->

#### Registering an attendance (part of _Attendances_ Module)

The main use case on this protocol. grants **ticket holder** to get access to an _event_ using a _ticket_.

```rs
fn mark_attendance(
    origin: AccountId,
    event: EventId,
    ticket: TicketId,
) -> Result<()>;
```

```mermaid
sequenceDiagram
    actor Attendee
    participant Attendances
    participant Tickets
    participant Nfts

    Attendee ->> Attendances: mark_attendance    

    Attendances ->> Tickets: can_attend "event, ticket"
    activate Tickets
        Tickets ->> Attendances: can_attend
    deactivate Tickets

    break cannot attend
        Attendances ->> Attendee: CannotAttend
    end

    Attendances ->> Nfts: do_set_attribute "attendances"
    activate Nfts
        Nfts -->> Attendances: AttributeSet
    deactivate Nfts

    Attendances -->> Attendee: AttendanceMarked
```

## FAQ

<details>
    <strong><summary>On the commit-reveal scheme that's involved in the <code>deferred_transfer</code> flow, how can we solve up for frontrunning on the <i>reveal</i> phase?</strong></summary>
    <p></p>
</details>
