# Enhanced Mermaid Diagram Templates

## Flow Chart Template 1 (Simple)

```mermaid
---
config:
    theme: 'base'
    curve: 'straight'
    themeVariables:
        darkMode: true
        clusterBkg: '#22272f62'
        clusterBorder: '#6a6f77ff'
        clusterTextColor: '#6a6f77ff'
        lineColor: '#C1C4CAAA'
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#C1C4CAff'
        primaryBorderColor: '#6a6f77ff'
        primaryLabelBkg: '#262B33'
        secondaryColor: '#425f5fff'
        secondaryBorderColor: '#8c9c81ff'
        secondaryTextColor: '#C1C4CAff'
        tertiaryColor: '#4d4962ff'
        tertiaryBorderColor: '#8983a5ff'
        tertiaryTextColor: '#eeeeee55'
        nodeTextColor: '#C1C4CA'
        defaultLinkColor: '#C1C4CA'
        edgeLabelBackground: '#262B33'
        edgeLabelBorderColor: '#C1C4CA'
        labelTextColor: '#C1C4CA'
        errorBkgColor: '#724848ff'
        errorTextColor: '#C1C4CA'
        flowchart:
            curve: 'basis'
            nodeSpacing: 50
            rankSpacing: 50
            subGraphTitleMargin:
                top: 15
                bottom: 15
                left: 15
                right: 15
---
flowchart TD
    A[Component A] --> B[Component B]
    B --> C[Component C]
    C --> D[Component D]

    D --> E[Module E]
    D --> F[Module F]
    D --> G[Module G]

    E --> H[Service H]
    F --> H
    G --> H

    H --> I[Output I]

    linkStyle default stroke:#C1C4CAaa,stroke-width:2px,color:#C1C4CAaa

    style A fill:#2b4268ff,stroke:#779DC9ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style B fill:#425f5fff,stroke:#8c9c81ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style C fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style D fill:#7a6253ff,stroke:#c7ac9bff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style E fill:#724848ff,stroke:#ac9696ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style F fill:#7a7253ff,stroke:#c7c19bff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style G fill:#2b5f5fff,stroke:#6d9c9cff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style H fill:#3a3f47ff,stroke:#6a6f77ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style I fill:#425f5fff,stroke:#8c9c81ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
```

## Flow Chart Template 2 (Complex Flow with Subgraphs)

```mermaid
---
config:
    theme: 'base'
    curve: 'straight'
    themeVariables:
        darkMode: true
        clusterBkg: '#22272f62'
        clusterBorder: '#6a6f77ff'
        clusterTextColor: '#6a6f77ff'
        lineColor: '#C1C4CAAA'
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#C1C4CAff'
        primaryBorderColor: '#6a6f77ff'
        primaryLabelBkg: '#262B33'
        secondaryColor: '#425f5fff'
        secondaryBorderColor: '#8c9c81ff'
        secondaryTextColor: '#C1C4CAff'
        tertiaryColor: '#4d4962ff'
        tertiaryBorderColor: '#8983a5ff'
        tertiaryTextColor: '#eeeeee55'
        nodeTextColor: '#C1C4CA'
        defaultLinkColor: '#C1C4CA'
        edgeLabelBackground: '#262B33'
        edgeLabelBorderColor: '#C1C4CAff'
        labelTextColor: '#ffffff'
        errorBkgColor: '#724848ff'
        errorTextColor: '#C1C4CA'
        flowchart:
            curve: 'basis'
            nodeSpacing: 50
            rankSpacing: 50
            subGraphTitleMargin:
                top: 15
                bottom: 15
                left: 15
                right: 15
---
flowchart TB
    subgraph INPUT["Input & Validation"]
        direction TB
        A[Start: fathomgrids.exe] --> B{Validate Arguments}
        B -->|Invalid| C[Print Usage & Exit]
        B -->|Valid| D[Parse Cutoff Depth]
        D --> E{Grid Level?}
        E -->|Yes| F[Map Grid Level]
        E -->|No| G[Default Level 0]
        F --> H[Build Path]
        G --> H
    end

    %% Initialization Phase
    subgraph INIT["Initialization"]
        direction TB
        I[Open Output Files]
        I -->|Failed| J[Report Error & Exit]
        I -->|Success| K[Init Memory]
        K --> L[Allocate Arrays]
        L --> M[Calculate Buckets]
        M --> N[Open Quad File]
    end

    %% Processing Phase
    subgraph PROCESS["Quad Tree Processing"]
        direction LR
        subgraph TRAVERSE["Traverse"]
            P[Start Traversal] --> Q[Read Node]
            Q --> R{Has Children?}
        end

        subgraph HANDLE["Handle Nodes"]
            R -->|No| S[Process Grid]
            R -->|Yes| T[Process Children]
            S --> U[Generate ID]
            U --> V{Depth Check}
            V -->|Above| W[Individual]
            V -->|Below| X[Composite]
        end

        subgraph CHILDREN["Child Processing"]
            T  --> Y[Q0] --> Z[Q1]
            Z  --> AA[Q2] --> BB[Q3]
            BB --> CC[Combine]
        end

        DD[Update Arrays]
        W  --> DD
        X  --> DD
        CC --> DD
        DD --> EE{More?}
        EE -->|Yes| Q
    end

    %% Enumeration Phase
    subgraph ENUM["Enumeration"]
        direction TB
        FF[Build Tree] --> GG[Traverse]
        GG --> HH{Type?}
        HH -->|Single| II[Write Single]
        HH -->|Composite| JJ{Subdivide?}
        JJ -->|Yes| KK[Subdivide]
        JJ -->|No| LL[Write Composite]
        KK --> GG
        II --> MM[Update Stats]
        LL --> MM
        MM --> NN{More?}
        NN -->|Yes| GG
    end

    %% Output Phase
    subgraph OUTPUT["Output & Cleanup"]
        direction TB
        OO[Generate Reports] --> PP[Depth Dist]
        PP --> QQ[Grid Dist]
        QQ --> RR[Cleanup]
        RR --> SS[Close Files]
        SS --> TT[Success Exit]
    end

    %% Main Flow Connections
    H --> I
    N -->|Found| P
    N -->|Not Found| O[No Data]
    O --> OO
    EE -->|No| FF
    NN -->|No| OO

    linkStyle default stroke:#C1C4CAaa,stroke-width:2px,color:#C1C4CA

    style A fill:#425f5fff,stroke:#8c9c81ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style C fill:#724848ff,stroke:#ac9696ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style J fill:#724848ff,stroke:#ac9696ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style TT fill:#425f5fff,stroke:#8c9c81ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style P fill:#7a6253ff,stroke:#c7ac9bff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style FF fill:#2b4268ff,stroke:#779DC9ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style B fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style E fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style R fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style V fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style HH fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style JJ fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style EE fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style NN fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
```

## Sequence Diagram Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#FFFFFF'
        primaryBorderColor: '#779DC9'
        lineColor: '#C1C4CA'
        actorBkg: '#2b4268ff'
        actorBorder: '#779DC9'
        actorTextColor: '#C1C4CA'
        actorLineColor: '#779DC9'
        activationBorderColor: '#c7ac9bff'
        activationBkgColor: '#7a6253ff'
        sequenceNumberColor: '#FFFFFF'
        noteBkgColor: '#3a3f47ff'
        noteTextColor: '#C1C4CA'
        noteBorderColor: '#6a6f77ff'
        labelBoxBkgColor: '#425f5fff'
        labelBoxBorderColor: '#8c9c81ff'
        labelTextColor: '#C1C4CA'
        loopTextColor: '#82867eff'
        altSectionBkgColor: '#4d4962ff'
        altSectionBorderColor: '#8983a5ff'
        signalColor: '#C1C4CA'
        signalTextColor: '#C1C4CA'
        messageTextColor: '#C1C4CA'
---
sequenceDiagram
    participant A as Component A
    participant B as Component B
    participant C as Component C
    participant D as Component D

    A->>+B: Initialize Process
    B->>B: Setup Configuration
    B-->>-A: Ready

    A->>+C: Start Processing

    loop For Each Item
        C->>C: Process Item
        C->>+D: Validate
        D-->>-C: Result
    end

    C-->>-A: Processing Complete

    alt Success Path
        A->>+D: Generate Output
        D-->>-A: Output Ready
    else Error Path
        A->>A: Handle Error
        A->>A: Log Error
    end

    note over A,D: Process Complete
```

## Class Diagram Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#C1C4CA'
        primaryBorderColor: '#3a3f47ff'
        mainBkg: '#262B33'
        secondBkg: '#425f5fff'
        textColor: '#C1C4CA'
        tertiaryBkg: '#4d4962ff'
        classText: '#C1C4CA'
        lineColor: '#978c72ff'
        labelBoxBkgColor: '#425f5fff'
        labelBoxBorderColor: '#8c9c81ff'
        labelTextColor: '#C1C4CA'
        mainContrastColor: '#FFFFFF'
        noteColor: '#7a7253ff'
        noteBkgColor: '#3a3f47ff'
        noteTextColor: '#C1C4CA'
        noteBorderColor: '#6a6f77ff'
---
classDiagram
    class ClassA {
        <<interface>>
        +String attribute
        -int privateField
        #protected value
        +method() String
        +calculate(int param) void
        -privateMethod() bool
    }

    class ClassB {
        <<abstract>>
        +String attribute
        +List~String~ items
        -Connection connection
        +method() String
        +process(data) Result
        #validate() bool
    }

    class ClassC {
        +id: UUID
        +name: String
        +created: DateTime
        +initialize() void
        +execute() Result
    }

    ClassA <|-- ClassB : inherits
    ClassB *-- ClassC : composition
    ClassA ..> ClassC : dependency

    note for ClassA "Primary interface
                     Defines contract"

    style ClassA fill:#2b4268ff,stroke:#779DC9ff,stroke-width:2px,color:#FFFFFF
    style ClassB fill:#425f5fff,stroke:#8c9c81ff,stroke-width:2px,color:#FFFFFF
    style ClassC fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#FFFFFF
```

## Entity Relationship Diagram Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#FFFFFF'
        primaryBorderColor: '#779DC9'
        lineColor: '#779DC9'
        mainBkg: '#2b4268ff'
        secondBkg: '#425f5fff'
        tertiaryBkg: '#4d4962ff'
        textColor: '#C1C4CA'
        mainContrastColor: '#C1C4CA'
        secondaryTextColor: '#C1C4CA'
        tertiaryTextColor: '#C1C4CA'
        relationColor: '#779DC9'
        relationLabelBackground: '#262B33'
        relationLabelColor: '#C1C4CA'
        attributeBackgroundColorOdd: '#3a3f47ff'
        attributeBackgroundColorEven: '#2f343cff'
---
erDiagram
    CAR ||--o{ NAMED-DRIVER : allows
    CAR {
        string registrationNumber PK
        string make
        string model
        string[] parts
    }
    PERSON ||--o{ NAMED-DRIVER : is
    PERSON {
        string driversLicense PK "The license #"
        string(99) firstName "Only 99 characters are allowed"
        string lastName
        string phone UK
        int age
    }
    NAMED-DRIVER {
        string carRegistrationNumber PK,FK
        string driverLicence PK,FK
    }
    MANUFACTURER ||--o{ CAR : makes
```

## Gantt Chart Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryTextColor: '#C1C4CA'
        primaryColor: '#2b4268ff'
        primaryBorderColor: '#779DC9'

        # Grid and structure
        gridColor: '#3a3f4755'
        excludeColor: '#3a3f47ff'

        # Default tasks (future)
        taskBkgColor: '#2b4268ff'
        taskBorderColor: '#779DC9'
        taskTextColor: '#FFFFFF'
        taskTextOutsideColor: '#C1C4CA'
        taskTextLightColor: '#FFFFFF'
        taskTextDarkColor: '#C1C4CA'

        # Completed tasks
        doneTaskBkgColor: '#425f5fff'
        doneTaskBorderColor: '#8c9c81ff'

        # Active tasks
        activeTaskBkgColor: '#7a6253ff'
        activeTaskBorderColor: '#c7ac9bff'

        # Critical tasks
        critTaskBkgColor: '#724848ff'
        critTaskBorderColor: '#ac9696ff'

        # Late tasks
        lateTaskBkgColor: '#855A5Aff'
        lateTaskBorderColor: '#ac9696ff'

        # Milestones
        milestoneBackgroundColor: '#7a7253ff'
        milestoneBorderColor: '#c7c19bff'

        # Sections
        sectionBkgColor: '#22272f62'
        sectionBorderColor: '#6a6f77ff'
        altSectionBkgColor: '#2a2f3862'
        altSectionBorderColor: '#6a6f77ff'
        sectionLabelColor: '#C1C4CA'

        # Today marker
        todayLineColor: '#c7c19bff'

        # General
        numberColor: '#C1C4CA'
        axisTextColor: '#C1C4CA'
        tickLabelColor: '#C1C4CA'
---
gantt
    dateFormat  YYYY-MM-DD
    title       Project Timeline

    section Phase A
    Completed task            :done,    des1, 2024-01-06,2024-01-08
    Active task               :active,  des2, 2024-01-09, 3d
    Future task               :         des3, after des2, 5d
    Future task2              :         des4, after des3, 5d

    section Critical Path
    Completed critical task   :crit, done, 2024-01-06,24h
    Active critical task      :crit, active, after des1, 2d
    Future critical task      :crit, 3d
    Milestone                 :milestone, 2024-01-25, 0d

    section Documentation
    Document task 1           :active, a1, after des1, 3d
    Document task 2           :after a1, 20h
    Document task 3           :doc1, after a1, 48h

    section Testing
    Test suite 1              :after doc1, 3d
    Test suite 2              :20h
    Final testing             :48h
```

## Git Graph Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        primaryColor: '#2b4268ff'
        primaryTextColor: '#C1C4CA'
        primaryBorderColor: '#779DC9'
        lineColor: '#C1C4CA'
        git0: '#2b4268ff'
        git1: '#425f5fff'
        git2: '#4d4962ff'
        git3: '#7a6253ff'
        git4: '#724848ff'
        git5: '#2b5f5fff'
        git6: '#7a7253ff'
        git7: '#3a3f47ff'
        gitBranchLabel0: '#FFFFFF'
        gitBranchLabel1: '#FFFFFF'
        gitBranchLabel2: '#FFFFFF'
        gitBranchLabel3: '#FFFFFF'
        gitBranchLabel4: '#FFFFFF'
        gitBranchLabel5: '#FFFFFF'
        gitBranchLabel6: '#FFFFFF'
        gitBranchLabel7: '#C1C4CA'
        commitLabelColor: '#C1C4CA'
        commitLabelBackground: '#262B33'
        tagLabelColor: '#FFFFFF'
        tagLabelBackground: '#7a6253ff'
        tagLabelBorder: '#c7ac9bff'
---
gitGraph
    commit
    branch hotfix
    checkout hotfix
    commit
    branch develop
    checkout develop
    commit id:"ash" tag:"v1.0"
    branch featureB
    checkout featureB
    commit type:HIGHLIGHT
    checkout main
    checkout hotfix
    commit type:NORMAL
    checkout develop
    commit type:REVERSE
    checkout featureB
    commit
    checkout main
    merge hotfix
    checkout featureB
    commit
    checkout develop
    branch featureA
    commit
    checkout develop
    merge hotfix
    checkout featureA
    commit
    checkout featureB
    commit
    checkout develop
    merge featureA
    branch release
    checkout release
    commit tag:"v2.0-rc"
    checkout main
    commit
    checkout release
    merge main
    checkout develop
    merge release
```

## User Journey Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        primaryColor: '#2b4268ff'
        primaryTextColor: '#C1C4CAff'
        primaryBorderColor: '#779DC9'
        clusterBkg: '#22272f62'
        clusterBorder: '#6a6f77ff'
        lineColor: '#C1C4CAff'
        secondaryColor: '#425f5fff'
        secondaryBorderColor: '#8c9c81ff'
        tertiaryColor: '#4d4962ff'
        tertiaryBorderColor: '#8983a5ff'
        nodeTextColor: '#C1C4CAff'
        journeyTitleColor: '#FFFFFF'
        journeySectionFill: '#2b4268ff'
        journeyActorBorderColor: '#779DC9'
        journeyActorBkgColor: '#425f5fff'
        journeyActorTextColor: '#C1C4CAff'
        journeyTaskBackgroundColor: '#4d4962ff'
        journeyTaskBorderColor: '#8983a5ff'
        journeyTaskTextColor: '#C1C4CAff'
        journeyLabelColor: '#C1C4CAff'
---
journey
    title User Experience Journey
    section Discovery
      Landing Page: 5: User
      Search: 3: User
      Browse: 4: System
    section Engagement
      Sign Up: 5: System
      Tutorial: 2: User
      First Action: 4: User
    section Retention
      Daily Use: 5: User
      Share: 3: User
      Premium: 4: System
```

## Pie Chart Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        primaryColor: '#C1C4CA'
        primaryTextColor: '#C1C4CA'
        primaryBorderColor: '#C1C4CA'
        lineColor: '#C1C4CA'
        pie1: '#2b4268ff'
        pie2: '#425f5fff'
        pie3: '#4d4962ff'
        pie4: '#7a6253ff'
        pie5: '#724848ff'
        pie6: '#2b5f5fff'
        pie7: '#7a7253ff'
        pie8: '#3a3f47ff'
        pie9: '#855A5A'
        pie10: '#6a6f77ff'
        pie11: '#779DC9'
        pie12: '#8c9c81ff'
        pieTitleTextColor: '#C1C4CA'
        pieSectionTextColor: '#e4e4e4ff'
        pieLegendTextColor: '#C1C4CA'
        pieStrokeColor: '#C1C4CA'
        pieStrokeWidth: '1px'
        pieOuterStrokeWidth: '2px'
        pieOuterStrokeColor: '#6a6f77ff'
        pieOpacity: '1'
---
pie title Distribution Analysis
    "Category A" : 35
    "Category B" : 25
    "Category C" : 20
    "Category D" : 15
    "Category E" : 5
```

## Graph Template

```mermaid
graph LR
    A[Start Node] --> B[Process Node]
    B --> C[Decision Node]
    B --> D[Alternative Node]
    C --> E[End Node]
    D --> E

    style A fill:#425f5fff,stroke:#8c9c81ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style B fill:#2b4268ff,stroke:#779DC9ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style C fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style D fill:#7a6253ff,stroke:#c7ac9bff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
    style E fill:#425f5fff,stroke:#8c9c81ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
```

## State Diagram Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryColor: '#262B33'
        primaryTextColor: '#C1C4CA'
        primaryBorderColor: '#779DC9'
        edgeLabelBackground: '#262B33'
        edgeLabelBorderColor: '#779DC9'
        lineColor: '#C1C4CA'
        stateBkg: '#2b4268ff'
        stateLabelColor: '#C1C4CA'
        startStateColor: '#425f5fff'
        endStateColor: '#724848ff'
        labelBackgroundColor: '#262B33'
        labelBorderColor: '#779DC9'
        transitionColor: '#C1C4CA'
        transitionLabelColor: '#C1C4CA'
        transitionBorderColor: '#779DC9'
        stateColor: '#2b4268ff'
        noteColor: '#7a7253ff'
        noteBkgColor: '#3a3f47ff'
        noteTextColor: '#C1C4CA'
        noteBorderColor: '#6a6f77ff'
---
stateDiagram-v2
    [*] --> Idle
    Idle --> Processing : Start
    Processing --> Success : Complete
    Processing --> Error : Fail
    Success --> [*]
    Error --> Idle : Retry

    note right of Processing : Active State
```

## Mind Map Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#FFFFFF'
        primaryBorderColor: '#779DC9'
        edgeLabelBackground: '#262B33'
        cScale0: '#2b4268ff'
        cScale1: '#425f5fff'
        cScale2: '#4d4962ff'
        cScale3: '#7a6253ff'
        cScale4: '#724848ff'
        cScale5: '#2b5f5fff'
        cScale6: '#7a7253ff'
        cScale7: '#3a3f47ff'
        scaleLabelColor: '#C1C4CA'
        sectionBkgColor: '#2b4268ff'
        altSectionBkgColor: '#425f5fff'
        gridColor: '#4a4f57ff'
        gridLineColor: '#4a4f57ff'
        titleColor: '#FFFFFF'
---
mindmap
  root))Central Topic((
    Branch A
      Sub A1
      Sub A2
        Detail A2a
    Branch B
      Sub B1
      Sub B2
        Detail B2a
    Branch C
      Sub C1
      Sub C2
    Branch D
      Sub D1
```

## Timeline Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#C1C4CA'
        primaryBorderColor: '#779DC9'
        cScale0: '#2b4268ff'
        cScale1: '#425f5fff'
        cScale2: '#4d4962ff'
        cScale3: '#7a6253ff'
        cScale4: '#724848ff'
        cScale5: '#2b5f5fff'
        cScale6: '#7a7253ff'
        cScale7: '#3a3f47ff'
        scaleLabelColor: '#C1C4CA'
        sectionBkgColor: '#2b4268ff'
        altSectionBkgColor: '#425f5fff'
        gridColor: '#4a4f57ff'
        gridLineColor: '#4a4f57ff'
        titleColor: '#FFFFFF'
---
timeline
    title Project Timeline
    2023 : Planning Phase
         : Initial Research
    2024 : Development Start
         : Alpha Release
         : Beta Testing
    2025 : Production Release
         : First Update
```

## Quadrant Chart Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#C1C4CA'
        primaryBorderColor: '#C1C4CA'
        quadrant1Fill: '#425f5fff'
        quadrant2Fill: '#2b4268ff'
        quadrant3Fill: '#724848ff'
        quadrant4Fill: '#7a6253ff'
        quadrant1TextFill: '#FFFFFF'
        quadrant2TextFill: '#FFFFFF'
        quadrant3TextFill: '#FFFFFF'
        quadrant4TextFill: '#FFFFFF'
        quadrantPointFill: '#FFFFFF'
        quadrantPointTextFill: '#C1C4CA'
        quadrantXAxisTextFill: '#C1C4CA'
        quadrantYAxisTextFill: '#C1C4CA'
        quadrantInternalBorderStrokeFill: '#C1C4CA'
        quadrantExternalBorderStrokeFill: '#C1C4CA'
        quadrantTitleFill: '#FFFFFF'
---
quadrantChart
    title Priority Matrix
    x-axis Low Effort --> High Effort
    y-axis Low Impact --> High Impact
    quadrant-1 Quick Wins
    quadrant-2 Major Projects
    quadrant-3 Fill Ins
    quadrant-4 Thankless Tasks

    Campaign A: [0.3, 0.7]
    Campaign B: [0.8, 0.4]
    Campaign C: [0.2, 0.3]
    Campaign D: [0.7, 0.8]
```

## Block Diagram Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#C1C4CA'
        primaryBorderColor: '#779DC9'
        lineColor: '#C1C4CA'
        fillType0: '#2b4268ff'
        fillType1: '#425f5fff'
        fillType2: '#4d4962ff'
        fillType3: '#7a6253ff'
        fillType4: '#724848ff'
        fillType5: '#2b5f5fff'
        fillType6: '#7a7253ff'
        fillType7: '#3a3f47ff'
        clusterBkg: '#22272f62'
        clusterBorder: '#6a6f77ff'
        edgeLabelBackground: '#262B33'
        nodeTextColor: '#C1C4CA'
---
block-beta
    columns 3

    block:GroupA
        A["Component A"]
        B["Component B"]
    end

    space

    block:GroupB
        C["Component C"]
        D["Component D"]
    end

    GroupA --> GroupB
```

## Requirement Diagram Template

```mermaid
---
config:
    theme: 'base'
    themeVariables:
        darkMode: true
        background: '#262B33'
        primaryColor: '#2b4268ff'
        primaryTextColor: '#C1C4CA'
        primaryBorderColor: '#779DC9'
        lineColor: '#C1C4CA'
        requirementBackground: '#2b4268ff'
        requirementBorderColor: '#779DC9'
        requirementTextColor: '#C1C4CA'
        relationColor: '#C1C4CA'
        relationLabelBackground: '#262B33'
        relationLabelColor: '#C1C4CA'
        fillType0: '#2b4268ff'
        fillType1: '#425f5fff'
        fillType2: '#4d4962ff'
        fillType3: '#7a6253ff'
---
requirementDiagram
    requirement ReqA {
        id: 1
        text: System shall provide user authentication
        risk: High
        verifymethod: Test
    }

    requirement ReqB {
        id: 2
        text: System shall encrypt sensitive data
        risk: High
        verifymethod: Inspection
    }

    element ComponentA {
        type: Component
        docref: design.md
    }

    element ComponentB {
        type: Module
        docref: security.md
    }

    ReqA - satisfies -> ComponentA
    ReqB - satisfies -> ComponentB
```

## Enhanced Style Reference

### Core Color Palette
- **Primary Blue**: Fill `#2b4268ff`, Stroke `#779DC9` - Main components, actors, primary nodes
- **Secondary Green**: Fill `#425f5fff`, Stroke `#8c9c81ff` - Secondary elements, success states, start nodes
- **Tertiary Purple**: Fill `#4d4962ff`, Stroke `#8983a5ff` - Decision points, conditions, alt blocks
- **Accent Brown**: Fill `#7a6253ff`, Stroke `#c7ac9bff` - Special processes, activations
- **Accent Red**: Fill `#724848ff`, Stroke `#ac9696ff` - Errors, critical items, end states
- **Accent Yellow**: Fill `#7a7253ff`, Stroke `#c7c19bff` - Warnings, milestones, highlights
- **Accent Teal**: Fill `#2b5f5fff`, Stroke `#6d9c9cff` - Alternative states, experimental
- **Neutral Gray**: Fill `#3a3f47ff`, Stroke `#6a6f77ff` - Notes, disabled states, neutral elements

### Text Colors
- **Primary Text**: `#C1C4CA` - Standard text color
- **Bright Text**: `#FFFFFF` - Titles, emphasis, critical labels
- **Muted Text**: `#95aea9ff` - Secondary information
- **Grid/Guide Lines**: `#4a4f57ff` - Subtle structural elements

### Semantic Color Mapping

#### Node Type Styling Rules
**IMPORTANT**: Apply these styles consistently across all diagrams:
- **Decision Nodes** (diamond shapes with `{}` in flowcharts): ALWAYS use Purple (`#4d4962ff`)
- **Process/Action Nodes** (rectangles with `[]`): Use Blue (`#2b4268ff`) for primary actions
- **Start/Entry Points**: Use Green (`#425f5fff`)
- **End/Exit Points**: Use Green (`#425f5fff`) for success, Red (`#724848ff`) for errors
- **Special/Important Processes**: Use Brown (`#7a6253ff`)
- **Data/Storage Nodes**: Use Gray (`#3a3f47ff`)

#### Process Flow
- **Start/Entry**: Green (`#425f5fff`)
- **Process/Action**: Blue (`#2b4268ff`)
- **Decision/Logic**: Purple (`#4d4962ff`)
- **Special/Key**: Brown (`#7a6253ff`)
- **Error/Critical**: Red (`#724848ff`)
- **Warning/Alert**: Yellow (`#7a7253ff`)
- **Info/Note**: Gray (`#3a3f47ff`)
- **Success/Complete**: Green (`#425f5fff`)

#### Git Branches
- **main/master**: Blue (`#2b4268ff`)
- **develop**: Green (`#425f5fff`)
- **feature**: Purple (`#4d4962ff`)
- **release**: Brown (`#7a6253ff`)
- **hotfix**: Red (`#724848ff`)
- **experimental**: Teal (`#2b5f5fff`)
- **docs**: Yellow (`#7a7253ff`)
- **archived**: Gray (`#3a3f47ff`)

#### Gantt Tasks
- **Default**: Blue (`#2b4268ff`)
- **Completed**: Green (`#425f5fff`)
- **Active**: Brown (`#7a6253ff`)
- **Critical**: Red (`#724848ff`)
- **Future**: Purple (`#4d4962ff`)
- **Milestone**: Yellow (`#7a7253ff`)

### Visual Hierarchy Guidelines

1. **Size & Position**
   - Larger elements = higher importance
   - Central/top position = primary focus
   - Peripheral = supporting information

2. **Color Intensity**
   - Bright text (#FFFFFF) = critical information
   - Standard text (#C1C4CA) = normal content
   - Muted text (#95aea9ff) = secondary details

3. **Border Styling**
   - 2px = standard emphasis
   - 3px = high emphasis
   - Dashed = optional/alternative paths

4. **Corner Radius**
   - 8px = standard rounded corners
   - 0px = sharp for technical/data elements

### Style Declaration Format

**CRITICAL**: All style declarations must follow this exact format:
```mermaid
style NODE_ID fill:#HEXCOLORff,stroke:#STROKECOLORff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
```

**Important Notes**:
- NO spaces after `fill:` or any other property
- Always include `ff` suffix for colors (full opacity)
- Standard stroke-width is `2px` (not 1px)
- Text color should be `#C1C4CA` for consistency
- Corner radius `rx:8,ry:8` for rounded rectangles

**Example Style Declarations**:
```mermaid
style A fill:#425f5fff,stroke:#8c9c81ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
style B fill:#2b4268ff,stroke:#779DC9ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
style C fill:#4d4962ff,stroke:#8983a5ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
style D fill:#7a6253ff,stroke:#c7ac9bff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
style E fill:#724848ff,stroke:#ac9696ff,stroke-width:2px,color:#C1C4CA,rx:8,ry:8
```

### Best Practices

#### Complex Diagrams
- Use subgraphs to organize >20 nodes
- Apply consistent color coding by phase
- Minimize line crossings with proper layout
- Break very complex flows into linked diagrams

#### Accessibility
- Maintain 4.5:1 contrast ratio minimum
- Don't rely solely on color (add labels/patterns)
- Test with color blindness simulators
- Provide descriptive text for all elements

#### Performance
- Limit to ~50 nodes per diagram
- Use appropriate diagram type for data
- Cache rendered diagrams when possible
- Optimize complex calculations

### Quick Reference

```javascript
// Color definitions for copy-paste
const darkTheme = {
  // Backgrounds
  background: '#262B33',
  clusterBkg: '#22272f62',

  // Primary Palette
  blue: { fill: '#2b4268ff', stroke: '#779DC9ff' },
  green: { fill: '#425f5fff', stroke: '#8c9c81ff' },
  purple: { fill: '#4d4962ff', stroke: '#8983a5ff' },
  brown: { fill: '#7a6253ff', stroke: '#c7ac9bff' },
  red: { fill: '#724848ff', stroke: '#ac9696ff' },

  // Extended Palette
  yellow: { fill: '#7a7253ff', stroke: '#c7c19bff' },
  teal: { fill: '#2b5f5fff', stroke: '#6d9c9cff' },
  gray: { fill: '#3a3f47ff', stroke: '#6a6f77ff' },
  darkRed: { fill: '#855A5A', stroke: '#ac9696ff' },

  // Text
  text: {
    primary: '#C1C4CA',
    bright: '#FFFFFF',
    muted: '#95aea9ff'
  },

  // Lines
  lines: {
    default: '#C1C4CA',
    grid: '#4a4f57ff'
  }
};
```
