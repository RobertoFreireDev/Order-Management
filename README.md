# Order-Management

## Order Creation

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef logic fill:#f5f5f5,stroke:#333,stroke-width:1px;
    classDef order fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Order_Payment [Order Payment]
        D5[Authorize payment]
    end

    subgraph Order_Creation [Order Management]
        A[Customer Places Order] --> A1[Start create order transaction]
        A1 --> B{Reserve Inventory?}
        B -->|Out of Stock| C[Return one or more items missing]
        B --> B2[Commit create order in database]
        B2 --> D2{Payment type?}
        D2--> |"Cash on Delivery/<br/>In Person"| D1[Order confirmed]       
        D5 --> |"Authorized"| D1
        D5 --> |"Not authorized"| D6[Return payment failed]
        D2 --> |"Asynchronous payment"| E1[Return payment page]
        D1--> D3[Return order confirmed]
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
    D2 --> |"Credit/Debit/Tokenized"| D5
    D1 --> H
    B2 --> H1

    %% Class Assignments
    class H,H1 event;
    class B,D2 logic;
    class A,A1,B1,B2,C,D,D1,D3,E1,D4,D5,D6 order;
    class I,I1 note;
```

### Notes:

1 - Reserve/Lock Inventory:

- It should be an atomic operation at database level to avoid race conditions between multiple users in the same or on different application instances on the server/cloud.

2 - Transactional outbox pattern

- Save both order details and order created (outbox message) to the same database in a single transaction. A separate and decoupled process reads from the outbox table and publishes to a message broker and then it deletes the current outbox message. Still has a very small chance to not be able to delete outbox message and send more than 1 order created event to message bus. Need to implement Idempotency on the consumer side.

![Transactional outbox pattern](imgs/outboxpattern.png)

3 - Asynchronous vs Synchronous Payments

- Direct Authorization Http request to authorized during the Create Order request.
- Deferred Confirmation: Authorized after the Create Order request via Webhook.

![Webhook vs polling](imgs/webhookpolling.png)

## Order Payment

- For Asynchronous or Synchronous Payment

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef logic fill:#f5f5f5,stroke:#333,stroke-width:1px;
    classDef order fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Order_Payment [Order Payment]
        A1[Authorize payment] --> B{Valid and Credit/Debit/Tokenized?}
        B --> |"Yes"| E
        B --> |"Expired"| C[Return payment expired]
        B --> |"Invalid payment"| C1[Return invalid payment]
        E{Authorize Payment}
        E -->|Failed| G1[Return Payment Failed/<br/>Try again]
    end

    subgraph Order_Creation [Order Management]
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
    class E,B logic;
    class A1,C,C1,G,G1,A2,F order;
    class I,I1,I2 note;
```


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