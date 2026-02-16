# Order-Management

## Order Creation

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef logic fill:#f5f5f5,stroke:#333,stroke-width:1px;
    classDef order fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Order_Creation [Order Creation]
        A[Customer Places Order] --> B{Reserve Inventory?}
        B -->|Out of Stock| C[Return Reject Order]
        B --> |Reserved| D[Save Order in database]
        D --> D2{Payment type?}
        D2--> |"Cash on Delivery/<br/>In Person"| D1[Order confirmed]       
        D2 --> |"Credit/Debit/Tokenized"| D4[Order created]
        D1--> D3[Return order confirmed]        
        D4 --> E1[Return payment page]
    end

    subgraph Events [Events / Message Bus]
        H([Send event:<br/>Order confirmed])
        H1([Send event:<br/>Order created])
    end

    subgraph Email [Email]
        I[Send e-mail: <br/>Order Confirmed]
        I1[Send e-mail: <br/>Order Created and<br/>Awaiting Payment]
    end

    %% Logic to Events
    H --> I
    H1 --> I1
    D1 --> H
    D4 --> H1

    %% Class Assignments
    class H,H1 event;
    class B,D2 logic;
    class A,C,D,D1,D3,E1,D4 order;
    class I,I1 note;
```

### Notes:

1 - Reserve/Lock Inventory:

- It should be an atomic operation at database level to avoid race conditions between multiple users in the same or on different application instances on the server/cloud.

2 - Transactional outbox pattern

- Save Order in databas and Order created (outbox message) to the same database in a single transaction. A separate, decoupled process reads from the outbox table and publishes to a message broker and then it deletes the current outbox message. 

![Transactional outbox pattern](imgs/outboxpattern.png)

## Order Payment

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef logic fill:#f5f5f5,stroke:#333,stroke-width:1px;
    classDef order fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Order_Payment [Order Payment]
        A1[Pay Order] --> |"Credit/Debit/Tokenized"| E{Authorize Payment}
        E -->|Failed| G1[Return Payment Failed/<br/>Try again]
    end

    subgraph Order_Creation [Order Creation]
        G[Update Order Status]
    end

    subgraph Order_Payment_Background_Job [Background Job]
        A2[Order Expired] --> |Expired| F[Release Inventory<br/>Update Order Status]
    end

    subgraph Events [Events / Message Bus]
        H([Send event:<br/>Payment authorized])
        H1([Send event:<br/>Payment failed])
        H2([Send event:<br/>Payment expired])
        H3([Send event:<br/>Order confirmed])
    end

    subgraph Email [Email]
        I[Send e-mail: <br/>Payment Authorized]
        I1[Send e-mail: <br/>Payment failed]
        I2[Send e-mail: <br/>Payment expired]
    end

    %% Logic to Events
    F --> H2
    G1 --> H1
    
    %% Event Distribution
    E -->|Authorized| H
    H --> I
    H --> G
    G --> H3
    H1 --> I1
    H2 --> I2

    %% Class Assignments
    class H,H1,H2,H3 event;
    class E logic;
    class A1,G,G1,A2,F order;
    class I,I1,I2 note;
```

### Notes:

1 - Order Status:

- It should have 2 status columns: Order Status and Payment Status.

1.1 - Digital Payment Flow

| Order Status       | Payment Status     | Trigger / Event                                                                 |
|--------------------|--------------------|----------------------------------------------------------------------------------|
| Pending            | Pending            | Order created; inventory reserved; redirecting to payment gateway.             |
| Awaiting Payment   | Authorized         | Card verified, but funds not yet captured (common in e-commerce).              |
| Confirmed          | Paid               | Payment confirmed by the bank or gateway.                                       |
| Canceled           | Failed             | Card declined or user abandoned the session.                                    |
| Canceled           | Expired            | Payment window timed out.                              |
| Canceled           | Refunded           | Order canceled by user/admin after payment was successful.                      |

---

1.2 - Order Lifecycle – Cash / Pay-on-Delivery Flow

| Order Status       | Payment Status | Trigger / Event                                                      |
|--------------------|---------------|----------------------------------------------------------------------|
| Confirmed          | Pending       | Order confirmed; no digital payment needed to start fulfillment.      |
| Out for Delivery   | Pending       | Logistics has the package; payment expected at delivery.           |
| Completed          | Paid          | Delivery person confirms receipt of cash/card in person.           |
| Canceled           | Unpaid        | Customer refused delivery or was not found at the address.         |

## Fulfillment

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef fulfill fill:#f3e5f5,stroke:#7b1fa2,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Fulfillment [FULFILLMENT]
        K[Store order in<br/>Fulfillment Center]
        L[Pick Items, Quality Check,<br/>Pack Order and Create<br/>Shipping Label]
        V[Capture Payment]
    end

    subgraph Events [Events / Message Bus]
        H([Event:<br/>Order Confirmed])
        M([Send event:<br/>Order Ready])
        S([Send event:<br/>Invoice created])
    end

    subgraph Invoice [Invoice]
        J[Generate Invoice]
    end

    %% Fulfillment Flow
    H --> K
    K --> L
    L --> |"Credit Card/<br/>Tokenized"| V
    L --> |"Debit/Cash/<br/>In Person"| M
    V --> M
    
    %% Invoice Flow
    M --> J
    J --> S

    %% Class Assignments
    class H,M,S event;
    class K,L,V fulfill;
    class J note;
```

## Order Deliver

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef order fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;
    classDef fulfill fill:#f3e5f5,stroke:#7b1fa2,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Fulfillment [FULFILLMENT]
        U[Ship order]
        Q[Confirm Payment]
    end

    subgraph Events [Events / Message Bus]
        S([Event: Invoice created])
        P([Send event:<br/>Order shipped])
        W([Send event:<br/>Order Completed])
    end

    subgraph Order_Management [Order Management]
        N[Update Order Status]
        R[Update Order Status]
    end

    subgraph Email [Email]
        O[Send e-mail:<br/> Order shipped]
    end

    %% Shipping Flow
    S --> U
    U --> P
    
    %% Final Updates
    P --> O
    P --> N
    P --> |"Cash on Delivery/<br/>In Person"| Q
    P --> |"Credit/Debit/<br/>Tokenized"| W
    Q --> W
    W --> R

    %% Class Assignments
    class S,P,W event;
    class U,Q fulfill
    class N,R order;
    class O note;
```