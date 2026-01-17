
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
