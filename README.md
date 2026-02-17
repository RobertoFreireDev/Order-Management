# Order-Management

## Order Creation

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef logic fill:#f5f5f5,stroke:#333,stroke-width:1px;
    classDef order fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Order_Creation [Order Management]
        A[Customer Places Order] --> A1[Start create order transaction]
        A1 --> B{Reserve Inventory? Call to Inventory api}
        B -->|Out of Stock| C[Return one or more items missing]
        B --> B2[Commit create order in database]
        B2 --> D2{Payment type?}
        D2--> |"Cash on Delivery/<br/>In Person"| D1[Order confirmed]       
        D5 --> |"Authorized"| D1
        D5 --> |"Not authorized"| D6[Return payment failed]
        D5 --> |"Expired"| D7[Return payment expired]
        D2 --> |"Asynchronous payment"| E1[Return payment page]
        D1--> D3[Return order confirmed]
        D2 --> |"Credit/Debit/Tokenized"| D5{Payment authorized? Call to Order Payment api}
    end

    subgraph Events [Events / Message Bus]
        H([Send event:<br/>Order confirmed])
        H1([Send event:<br/>Order created])
    end

    subgraph Email [Email]
        I[Send e-mail: <br/>Order confirmed]
        I1[Send e-mail: <br/>Order created and awaiting payment]
    end

    %% Logic to Events
    H --> I
    H1 --> I1
    D1 --> H
    B2 --> H1

    %% Class Assignments
    class H,H1 event;
    class B,D2,D5 logic;
    class A,A1,B1,B2,C,D,D1,D3,E1,D4,D6,D7 order;
    class I,I1 note;
```

### Notes:

1 - Reserve/Lock Inventory:

- It should be an atomic operation at database level to avoid race conditions between multiple users in the same or on different application instances on the server/cloud.

2 - Transactional outbox pattern

- All the events sent should follow this pattern.
- Example: Save both order details and order created (outbox message) to the same database in a single transaction. A separate and decoupled process reads from the outbox table and publishes to a message broker and then it deletes the current outbox message. There is a very small chance that after a event being publish, for some reason (application is down, for example), the outbox message was not deleted. Thus, a duplicated order created event will be sent to message bus. To avoid this, we must implement Idempotency on the consumer side.

![Transactional outbox pattern](imgs/outboxpattern.png)

3 - Idempotency on the consumer side

- One solution is to store the fact that event has been processed. So, next time, if the same event is triggered, it will be acknowledge but no processed (Silently ignore it). You can hash the event payload and store in the database or use orderid as an unique idempotency key.
- Another solution, if possible, is to make the operation on the consumer side, idempotent. This way, the following duplicated events will result the same output. Example: UPDATE Orders SET Status = 'Processed' WHERE OrderId = 123 AND Status = 'Pending'.

![idempotency](imgs/idempotency.png)

3 - Synchronous/Async payment methods:

- Direct authorization via http request
- Authorize payment request via webhook or polling.

![Webhook vs polling](imgs/webhookpolling.png)

## Order Payment

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef logic fill:#f5f5f5,stroke:#333,stroke-width:1px;
    classDef order fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Order_Payment [Order Payment]
        A1[Authorize payment] --> E{Payment authorized? Call to payment gateway}
        E -->|Failed| G1[Return payment failed]
        E -->|Authorized| E1[Authorized]
        E1 --> E4[Save payment authorized for OrderId]
        E -->|External timeout| E3[Return payment expired]
        E1 --> E2[Return payment authorized]
    end

    subgraph Order_Creation [Order Management]
        G[Update Order Status]
    end

    subgraph Events [Events / Message Bus]
        H([Send event:<br/>Payment authorized])
        H3([Send event:<br/>Order confirmed])
    end
    
    %% Event Distribution
    E1 --> H
    H --> G
    G --> H3

    %% Class Assignments
    class H,H1,H2,H3 event;
    class E,B logic;
    class A1,C,C1,E1,E2,E3,E4,G,G1,A2,F order;
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
        H([Event:<br/>Order confirmed])
        M([Send event:<br/>Order ready])
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
        S1[Retrieve invoice]
        U[Ship order]
        U1[Order delivered]
        Q[Confirm Payment]
    end

    subgraph Events [Events / Message Bus]
        S([Event: Invoice created])
        P([Send event:<br/>Order shipped])
        W([Send event:<br/>Order completed])
    end

    subgraph Order_Management [Order Management]
        N[Update order status]
    end

    subgraph Email [Email]
        O[Send e-mail:<br/> Order shipped]
    end

    %% Shipping Flow
    S --> S1
    S1 --> U
    U --> P
    U --> U1
    
    %% Final Updates
    P --> O
    P --> N
    U1 --> |"Cash on Delivery/<br/>In Person"| Q
    U1 --> |"Credit/Debit/<br/>Tokenized"| W
    Q --> W
    W --> N

    %% Class Assignments
    class S,P,W event;
    class U,U1,S1,Q fulfill
    class N,R order;
    class O note;
```