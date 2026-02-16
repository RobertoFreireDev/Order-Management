# Order-Management

## Order Creation

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef logic fill:#f5f5f5,stroke:#333,stroke-width:1px;
    classDef fulfill fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Order_Management [Order Management]
        A[Customer Places Order] --> B{Reserve/Lock Inventory}
        B -->|Out of Stock| C([Reject Order])
        B --> D[Create Order Record]
        
        D --> |"Credit/Debit/Tokenized"| E{Authorize/Confirm<br/>Payment}
        E -->|Failed| F([Release Inventory<br/>Update Status: <br/>Payment Failed])
        E -->|Authorized| G[Update Order Status]
    end

    subgraph Events [Events / Message Bus]
        H([Send event:<br/>Order placed])
    end

    subgraph Email [Email]
        I[Send order placed<br/>e-mail]
    end

    %% Logic to Events
    G --> H
    D --> |"Cash on Delivery/<br/>In Person"| H
    
    %% Event Distribution
    H --> I

    %% Class Assignments
    class H event;
```

### Notes:

1 - Reserve/Lock Inventory:

- It should be an atomic operation at database level to avoid race conditions

2 - Order Status:

- It should have 2 status columns: Order Status and Payment Status.

2.1 - Digital Payment Flow

| Order Status       | Payment Status     | Trigger / Event                                                                 |
|--------------------|--------------------|----------------------------------------------------------------------------------|
| Pending            | Pending            | Order created; inventory reserved; redirecting to payment gateway.             |
| Awaiting Payment   | Authorized         | Card verified, but funds not yet captured (common in e-commerce).              |
| Confirmed          | Paid               | Payment confirmed by the bank or gateway.                                       |
| Canceled           | Failed             | Card declined or user abandoned the session.                                    |
| Canceled           | Expired            | Payment window timed out.                              |
| Canceled           | Refunded           | Order canceled by user/admin after payment was successful.                      |

---

2.2 - Order Lifecycle – Cash / Pay-on-Delivery Flow

| Order Status       | Payment Status | Trigger / Event                                                      |
|--------------------|---------------|----------------------------------------------------------------------|
| Confirmed          | Pending       | Order placed; no digital payment needed to start fulfillment.      |
| Out for Delivery   | Pending       | Logistics has the package; payment expected at delivery.           |
| Completed          | Paid          | Delivery person confirms receipt of cash/card in person.           |
| Canceled           | Unpaid        | Customer refused delivery or was not found at the address.         |


## Fulfillment

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef fulfill fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;
    classDef note fill:#f9fbe7,stroke:#827717,stroke-width:1px;

    subgraph Fulfillment [FULFILLMENT]
        K[Store order in<br/>Fulfillment Center]
        L[Pick Items, Quality Check,<br/>Pack Order and Create<br/>Shipping Label]
        V[Capture Payment]
    end

    subgraph Events [Events / Message Bus]
        H([Event:<br/>Order placed])
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
    class M,S event;
    class K,L,V fulfill;
    class J note;
```

## Order Deliver

```mermaid
flowchart TD
    %% Subgraph Styling
    classDef event fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef fulfill fill:#fff3e0,stroke:#ef6c00,stroke-width:1px;

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
        O[Send order shipped<br/>e-mail]
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
    class P,W event;
    class U,Q fulfill;
```