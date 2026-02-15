# Order-Management

```mermaid
flowchart TD

    subgraph Order_Management [Order Management]
        A[Customer Places Order] --> B[Check Inventory Availability]
        B -->|Out of Stock| C[Reject Order]
        B --> D[Reserve/Lock Inventory and Create Order Record with Customer, Items, Payment type, Order Status: Pending]
        D --> |Credit Card, Debit Card, Card on File Tokenized| E[Authorize/Confirm Payment]
        E -->|Failed| F[Release Inventory and Update Status: Payment Failed]
        E -->|Authorized| G[Update Order Status: Confirmed]
        G --> H(Send event: Order placed)
        D --> |Cash on Delivery or Payment in Person| H
        N[Update Order Status: Shipped]
        R[Update Order Status: Delivered]
    end

    subgraph Email [Email]
        I[Send order placed e-mail]
        O[Send order shipped e-mail]
    end

    subgraph Fulfillment [FULFILLMENT]
        K[Store order in Fulfillment Center]
        L[Pick Items, Quality Check, Pack Order and Create Shipping Label]
        V[Capture Payment]
        M(Send event: Order Ready)
        U[Ship order]
        P(Send event: Order shipped)
        Q[Confirm Payment]
        W(Send event: Order Completed) 
    end

    subgraph Invoice [Invoice]
        J[Generate Invoice]
        S(Send event: Invoice created)
    end

    %% Connections between groups
    H --> I
    H --> K
    K --> L
    L --> |Credit Card, Card on File Tokenized| V
    V --> M
    L --> |Debit Card, Cash on Delivery or Payment in Person| M
    M --> J
    J --> S
    S --> U
    U --> P
    P --> O
    P --> N
    P --> |Cash on Delivery or Payment in Person| Q
    P --> |Credit Card, Debit Card, Card on File Tokenized| W
    Q --> W
    W --> R
```