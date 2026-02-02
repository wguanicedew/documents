
## 1. current architecture

```mermaid
graph LR
    DAQ[E1/E2 Fast Monitor]
    PanDA[PanDA]
    Rucio[Rucio]
    ActiveMQ[ActiveMQ]
    PostgreSQL[PostgreSQL]
    DAQSim[swf-daqsim-agent]
    DataAgent[swf-data-agent]
    ProcAgent[swf-processing-agent]
    FastMon[swf-fastmon-agent]
    Monitor[swf-monitor]
    WebUI[Web Dashboard]
    RestAPI[REST API]
    MCP[MCP Server]
    
    DAQSim -->|1| ActiveMQ
    ActiveMQ -->|1| DataAgent
    ActiveMQ -->|1| ProcAgent
    DataAgent -->|2| Rucio
    ProcAgent -->|3| PanDA
    
    DAQSim -->|4| ActiveMQ
    ActiveMQ -->|4| DataAgent
    ActiveMQ -->|4| ProcAgent
    ActiveMQ -->|4| FastMon
    
    DataAgent -->|5| Rucio
    ProcAgent -->|6| PanDA
    FastMon -.->|7| DAQ
    
    ActiveMQ -.-> Monitor
    Monitor --> PostgreSQL
    Monitor --> WebUI
    Monitor --> RestAPI
    Monitor --> MCP
```

**Workflow Steps:**
1. **Run Start** - daqsim-agent generates a run start broadcast message indicating a new datataking run is beginning
2. **Dataset Creation** - data-agent sees the run start message and has Rucio create a dataset for the run
3. **Processing Task** - processing-agent sees the run start message and establishes a PanDA processing task for the run
4. **STF Available** - daqsim-agent generates a broadcast message that a new STF data file is available
5. **STF Transfer** - data-agent sees the message and initiates Rucio registration and transfer of the STF file to E1 facilities
6. **STF Processing** - processing-agent sees the new STF file in the dataset and transferred to the E1 by Rucio, and initiates a PanDA job to process the STF
7. **Fast Monitoring** - fastmon-agent sees the broadcast message that a new STF data file is available and performs a partial read to inject a data sample into E1/E2 fast monitoring

*Figure: Testbed agent architecture and data flow diagram*

## 2. With PanDA/iDDS

```mermaid
graph LR
    DAQ[E1/E2 Fast Monitor]
    PanDA[PanDA]
    PanDA1[PanDA]
    iDDS[iDDS]
    Pilot[PanDA-Pilot-Transformer]
    Rucio[Rucio]
    ActiveMQ[ActiveMQ]
    PostgreSQL[PostgreSQL]
    DAQSim[swf-daqsim-agent]
    DataAgent[swf-data-agent]
    ProcAgent[swf-processing-agent]
    FastMon[swf-fastmon-agent]
    FastProc[swf-fastprocessing-agent]
    Monitor[swf-monitor]
    WebUI[Web Dashboard]
    RestAPI[REST API]
    MCP[MCP Server]

    DAQSim -->|1| ActiveMQ
    ActiveMQ -->|1| DataAgent
    ActiveMQ -->|1| ProcAgent
    ActiveMQ -->|1| FastProc
    FastProc -->|1| iDDS
    DataAgent -->|2| Rucio
    ProcAgent -->|3| PanDA

    DAQSim -->|4| ActiveMQ
    ActiveMQ -->|4| DataAgent
    ActiveMQ -->|4| ProcAgent
    ActiveMQ -->|4| FastMon

    DataAgent -->|5| Rucio
    ProcAgent -->|6| PanDA
    FastMon -.->|7| DAQ

    ActiveMQ -.-> Monitor
    Monitor --> PostgreSQL
    Monitor --> WebUI
    Monitor --> RestAPI
    Monitor --> MCP

    iDDS -->|8| PanDA1
    PanDA1 -->|8| Pilot
    FastProc -.->|9| ActiveMQ
    ActiveMQ -.->|9| Pilot
```

**Workflow Steps:**
1. **Run Start** - daqsim-agent generates a run start broadcast message indicating a new datataking run is beginning
2. **Dataset Creation** - data-agent sees the run start message and has Rucio create a dataset for the run
3. **Processing Task** - processing-agent sees the run start message and establishes a PanDA processing task for the run
4. **STF Available** - daqsim-agent generates a broadcast message that a new STF data file is available
5. **STF Transfer** - data-agent sees the message and initiates Rucio registration and transfer of the STF file to E1 facilities
6. **STF Processing** - processing-agent sees the new STF file in the dataset and transferred to the E1 by Rucio, and initiates a PanDA job to process the STF
7. **Fast Monitoring** - fastmon-agent sees the broadcast message that a new STF data file is available and performs a partial read to inject a data sample into E1/E2 fast monitoring
8. **PanDA worker** - iDDS sees the run start message and creates PanDA transformer workers (A transformer in running PanDA Pilots)
9. **TF slice** - fast-processing-agent generates TF slices and distributes them to ActiveMQ. The transformer in PanDA Pilot consumes TF slice messages to process the payloads.

## 3. flow

```mermaid
flowchart TB
%% ---------- Styles ----------
classDef blue fill:#e8f0fe,stroke:#3b82f6,stroke-width:2px,color:#111;
classDef green fill:#eaf7ea,stroke:#22c55e,stroke-width:2px,color:#111;
classDef orange fill:#fff3e6,stroke:#f97316,stroke-width:2px,color:#111;
classDef purple fill:#f3e8ff,stroke:#a855f7,stroke-width:2px,color:#111;
classDef gray fill:#f3f4f6,stroke:#9ca3af,stroke-width:2px,color:#111;

%% ---------- Main pipeline ----------
A[DAQ Simulator]:::blue --> B[STF]:::orange --> C[Data Agent]:::green --> D[FastMon Agent]:::green --> E[STF Sample]:::orange --> F[Fast Processing Agent]:::green

%% ---------- DB ----------
DB((Testbed\nDB)):::blue
C -. "STF record" .-> DB
D -. "sample records" .-> DB
F -. "slice bookkeeping" .-> DB

%% ---------- TF slices / Workers ----------
subgraph SLICES["TF Slices"]
direction LR
S1[slice 1]:::gray
S2[slice 2]:::gray
S3[slice 3]:::gray
Sx[...]:::gray
end
style SLICES fill:#fff7ed,stroke:#f97316,stroke-width:2px,rx:10,ry:10

F --> SLICES
subgraph W["PanDA Workers"]
direction LR
W1[Worker 1\nEICrecon]:::purple
W2[Worker 2\nEICrecon]:::purple
W3[Worker 3\nEICrecon]:::purple
Wx[...]:::purple
end
style W fill:#f5f3ff,stroke:#7c3aed,stroke-width:2px,rx:10,ry:10

SLICES --> W --> OUT[Reconstruction Output]:::gray --> ANA[Analytics]:::orange

%% ---------- IDDS/PanDA ----------
IDDS[IDDS\nWorkflow Mgmt]:::green --> PANDA[PanDA\nWorkload Mgmt]:::green
F --> IDDS
PANDA --> W

%% ---------- Legend (manual) ----------
subgraph LEG["Legend"]
direction TB
L1["→ control flow"]:::gray
L2["- - → records"]:::gray
end
style LEG fill:#ffffff,stroke:#d1d5db,stroke-width:1px,rx:10,ry:10
```
