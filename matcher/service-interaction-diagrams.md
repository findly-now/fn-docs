# Matching Flow Diagrams

**Visual representation of the intelligent matching flows and processes in the fn-matcher service.**

## 1. Overall Matching Flow

```mermaid
graph TD
    A[Post Created] --> B{Post Type?}
    B -->|Lost| C[fn-posts: PostCreated Event]
    B -->|Found| C
    C --> D[fn-matcher: Event Consumer]
    D --> E[Initial Matching Algorithm]

    E --> F[Location Proximity Scoring]
    E --> G[Text Similarity Scoring]
    E --> H[Temporal Proximity Scoring]

    F --> I[Calculate Weighted Score]
    G --> I
    H --> I

    I --> J{Confidence >= 0.50?}
    J -->|No| K[Discard Low Confidence]
    J -->|Yes| L[Store Match Record]

    L --> M{Confidence >= 0.85?}
    M -->|Yes| N[Auto-Notify Users]
    M -->|No| O{Confidence >= 0.70?}
    O -->|Yes| P[Surface in UI]
    O -->|No| Q[Store for Review]

    N --> R[fn-notifications: PostMatched Event]
    P --> S[Available for User Action]

    S --> T[User Confirms/Rejects]
    T --> U[Update Match Status]

    U --> V{Match Confirmed?}
    V -->|Yes| W[Enable Claiming]
    V -->|No| X[Mark as Rejected]

    W --> Y[Item Claiming Flow]
    Y --> Z[fn-notifications: PostClaimed Event]
```

## 2. Enhanced Matching Flow with AI

```mermaid
graph TD
    A[Post Created] --> B[fn-media-ai: Photo Analysis]
    B --> C[AI Enhancement Complete]
    C --> D[fn-matcher: PostEnhanced Event]
    D --> E[Enhanced Matching Algorithm]

    E --> F[Location Proximity Scoring]
    E --> G[Visual Similarity Scoring]
    E --> H[Text Similarity Scoring]
    E --> I[Temporal Proximity Scoring]

    G --> G1[AI Object Detection]
    G --> G2[Color Analysis]
    G --> G3[Brand Recognition]
    G1 --> J[Calculate Weighted Score]
    G2 --> J
    G3 --> J

    F --> J
    H --> J
    I --> J

    J --> K{Higher Confidence Than Initial?}
    K -->|Yes| L[Update Existing Match]
    K -->|No| M[Create New Match]

    L --> N[Re-evaluate Thresholds]
    M --> N

    N --> O{Auto-Notify Threshold?}
    O -->|Yes| P[Send Enhanced Match Alert]
    O -->|No| Q[Update UI Display]

    P --> R[fn-notifications: Enhanced Match Event]
```

## 3. Multi-Factor Scoring Algorithm

```mermaid
graph LR
    A[Lost Post] --> B[Matching Engine]
    C[Found Post] --> B

    B --> D[Location Scorer]
    B --> E[Visual Scorer]
    B --> F[Text Scorer]
    B --> G[Temporal Scorer]

    D --> D1[PostGIS Distance Query]
    D1 --> D2[Exponential Decay Function]
    D2 --> D3[Location Score: 0.92]

    E --> E1[AI Tags Comparison]
    E1 --> E2[Jaccard Similarity]
    E2 --> E3[Visual Score: 0.85]

    F --> F1[TF-IDF Vectorization]
    F1 --> F2[Cosine Similarity]
    F2 --> F3[Text Score: 0.78]

    G --> G1[Time Difference Calculation]
    G1 --> G2[Exponential Decay]
    G2 --> G3[Temporal Score: 0.95]

    D3 --> H[Weighted Combination]
    E3 --> H
    F3 --> H
    G3 --> H

    H --> I[Final Score: 0.87]

    H --> J[Weight: Location 40%]
    H --> K[Weight: Visual 35%]
    H --> L[Weight: Text 15%]
    H --> M[Weight: Temporal 10%]
```

## 4. Match Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> Pending : Match Created

    Pending --> Confirmed : User Confirms
    Pending --> Rejected : User Rejects
    Pending --> Expired : 7 Days Elapsed

    Confirmed --> Claimed : Item Claimed
    Confirmed --> Expired : 30 Days No Claim

    Rejected --> [*] : End State
    Expired --> [*] : End State
    Claimed --> [*] : End State

    note right of Pending
        Initial state after match detection
        Confidence score determines visibility
    end note

    note right of Confirmed
        Both parties acknowledged match
        Claiming workflow enabled
    end note

    note right of Claimed
        Item successfully claimed
        Recovery process complete
    end note
```

## 5. Item Claiming Workflow

```mermaid
sequenceDiagram
    participant U as User (Lost Item)
    participant M as fn-matcher
    participant N as fn-notifications
    participant F as Finder

    U->>M: POST /matches/{id}/claim
    M->>M: Generate Verification Code
    M->>M: Store Claim Record
    M->>N: Publish PostClaimed Event
    M->>U: Return Claim Details + Code

    N->>F: Send Urgent SMS to Finder
    N->>F: "Item claimed! Verification: ABC123"

    F->>F: Contact Claimer
    F->>F: Verify Code + Identity
    F->>F: Arrange Meetup

    Note over U,F: Offline Exchange

    F->>M: Mark Item as Transferred (future)
    M->>N: Recovery Success Event
    N->>U: Success Confirmation
    N->>F: Thank You Message
```

## 6. Event-Driven Architecture Flow

```mermaid
graph TD
    subgraph "fn-posts"
        A[User Creates Post] --> B[PostCreated Event]
    end

    subgraph "fn-media-ai"
        C[Photo Analysis] --> D[PostEnhanced Event]
    end

    subgraph "fn-matcher"
        E[Event Consumer] --> F[Matching Engine]
        F --> G[Match Detection]
        G --> H[Confidence Evaluation]
        H --> I[Store Match]
        I --> J[PostMatched Event]

        K[User Interaction] --> L[Match Confirmation]
        L --> M[PostClaimed Event]
    end

    subgraph "fn-notifications"
        N[Notification Engine] --> O[Multi-Channel Delivery]
    end

    B --> E
    D --> E
    J --> N
    M --> N

    style E fill:#f9f,stroke:#333,stroke-width:2px
    style F fill:#bbf,stroke:#333,stroke-width:2px
    style N fill:#bfb,stroke:#333,stroke-width:2px
```

## 7. Location-Based Matching Algorithm

```mermaid
graph TD
    A[Lost Item Location] --> B[PostGIS Spatial Query]
    C[Found Items in Area] --> B

    B --> D{Within Max Radius?}
    D -->|No| E[Skip Item]
    D -->|Yes| F[Calculate Haversine Distance]

    F --> G[Apply Distance Decay Function]
    G --> H[Location Proximity Score]

    H --> I{Score > Threshold?}
    I -->|No| J[Low Priority Candidate]
    I -->|Yes| K[High Priority Candidate]

    K --> L[Proceed to Multi-Factor Analysis]
    J --> M[Store for Potential Re-evaluation]

    style B fill:#e1f5fe
    style G fill:#f3e5f5
    style L fill:#e8f5e8
```

## 8. Visual Similarity Matching Process

```mermaid
graph TD
    A[Lost Post Photos] --> B[AI Tags from fn-media-ai]
    C[Found Post Photos] --> D[AI Tags from fn-media-ai]

    B --> E[Filter High-Confidence Tags]
    D --> F[Filter High-Confidence Tags]

    E --> G[Object Set: phone, black_case, apple]
    F --> H[Object Set: smartphone, case, iphone]

    G --> I[Calculate Jaccard Similarity]
    H --> I

    I --> J[Intersection / Union]
    J --> K[Visual Similarity Score]

    K --> L{Score > 0.70?}
    L -->|Yes| M[Strong Visual Match]
    L -->|No| N[Weak Visual Match]

    M --> O[High Weight in Final Score]
    N --> P[Low Weight in Final Score]

    style B fill:#ffe0b2
    style D fill:#ffe0b2
    style I fill:#f1c2f0
    style M fill:#c8e6c9
```

## 9. Confidence-Based Action Flow

```mermaid
graph TD
    A[Match Score Calculated] --> B{Confidence Level?}

    B -->|0.85+ High| C[Auto-Notify Users]
    B -->|0.70+ Medium| D[Surface in UI]
    B -->|0.50+ Low| E[Store for Review]
    B -->|< 0.50 Very Low| F[Discard]

    C --> G[Immediate Push Notification]
    C --> H[Email Alert]
    C --> I[SMS Notification]

    D --> J[Show in Match Dashboard]
    D --> K[Available for User Action]

    E --> L[Admin Review Queue]
    E --> M[Potential Manual Verification]

    F --> N[Not Stored in Database]

    G --> O[User Engagement]
    H --> O
    I --> O
    J --> O
    K --> O

    O --> P[Confirm/Reject Decision]
    P --> Q[Update Match Status]

    style C fill:#4caf50
    style D fill:#ff9800
    style E fill:#f44336
    style F fill:#9e9e9e
```

## 10. Temporal Proximity Analysis

```mermaid
graph TD
    A[Lost Item Time] --> B[Calculate Time Difference]
    C[Found Item Time] --> B

    B --> D[Hours Difference: 2.5h]
    D --> E[Apply Exponential Decay]
    E --> F[Temporal Score Calculation]

    F --> G{Time Score Categories}
    G -->|0-6 hours| H[Score: 0.90-1.00]
    G -->|6-24 hours| I[Score: 0.70-0.90]
    G -->|1-3 days| J[Score: 0.40-0.70]
    G -->|3-7 days| K[Score: 0.10-0.40]
    G -->|> 7 days| L[Score: 0.00-0.10]

    H --> M[Recent Loss - High Priority]
    I --> N[Same Day - Good Match]
    J --> O[Recent - Medium Priority]
    K --> P[Older - Low Priority]
    L --> Q[Very Old - Minimal Weight]

    style H fill:#4caf50
    style I fill:#8bc34a
    style J fill:#ff9800
    style K fill:#f44336
    style L fill:#9e9e9e
```

---

*These diagrams illustrate the complex matching flows and algorithms within the fn-matcher service. For detailed technical implementation, see [domain-architecture.md](domain-architecture.md). For API integration examples, see [api-documentation.md](api-documentation.md).*