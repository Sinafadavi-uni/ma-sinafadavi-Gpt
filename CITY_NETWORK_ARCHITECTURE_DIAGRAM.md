# Possible City Network - Logical Architecture Diagram

```
										POSSIBLE CITY NETWORK
				Distributed City-Wide Computing Platform Architecture

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                                                                     │
│  ┌──────────────┐                                          ┌─────────────────┐    │
│  │ DATA STORE   │                                          │ CLIENT 1        │    │
│  │ ┌──────────┐ │                                          │     📱          │    │
│  │ │ Storage  │ │                                          │                 │    │
│  │ │ Servers  │ │                                          │                 │    │
│  │ └──────────┘ │                                          └─────────────────┘    │
│  │              │                                                    │             │
│  │              │                                                    │ (dashed)    │
│  └──────────────┘                                                    │             │
│         │                                                            │             │
│         │ Input Data (dashed)                                        ▼             │
│         │                                                                          │
│         ▼                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                     CENTRAL BROKER                                          │  │
│  │                   📶 Wireless Router                                        │  │
│  │                                                                             │  │
│  │  • Receives jobs from clients                                               │  │
│  │  • Retrieves input data from data store                                    │  │
│  │  • Dispatches jobs to executors                                            │  │
│  │  • Collects result data                                                    │  │
│  │  • Returns results to clients                                              │  │
│  │                                                                             │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│         │                           │                            ▲               │
│         │ Result Data               │                            │               │
│         │ (solid)                   │                            │ Job (solid)   │
│         ▼                           │                            │               │
│  ┌──────────────┐                   │                   ┌─────────────────┐     │
│  │ DATA STORE   │                   │                   │ CLIENT 2        │     │
│  │ (storage)    │                   │                   │     📱          │     │
│  └──────────────┘                   │                   │                 │     │
│                                     │                   │                 │     │
│                                     │                   └─────────────────┘     │
│                                     │                            │               │
│                                     │                            │ Result Data   │
│                                     │                            │ (solid)       │
│                                     │                            ▼               │
│                                     │                                            │
│         ┌───────────────────────────┼───────────────────────────────────────┐    │
│         │                           │                                       │    │
│         │ Job (solid)               │ Input Data (solid)                    │    │
│         │                           │                                       │    │
│         ▼                           ▼                                       │    │
│  ┌─────────────────┐                                       ┌─────────────────┐  │
│  │ SERVER-CLASS    │ ◀──────── dashed ──────────┐          │ LAPTOP          │  │
│  │ EXECUTOR        │                            │          │ EXECUTOR        │  │
│  │ ┌─────────────┐ │                            │          │     💻         │  │
│  │ │ Rack of     │ │                            │          │                 │  │
│  │ │ Servers     │ │                            │          │                 │  │
│  │ └─────────────┘ │                            │          └─────────────────┘  │
│  │                 │                            │                    │           │
│  └─────────────────┘                            │                    │           │
│         │                                       │                    │ Result    │
│         │ Result Data (solid)                   │                    │ Data      │
│         │                                       │                    │ (solid)   │
│         ▼                                       │                    ▼           │
│                                                 │                                 │
│                    ┌────────────────────────────┼──────────────────────────────┐ │
│                    │           SECOND BROKER    │                              │ │
│                    │         📶 Wireless Router │                              │ │
│                    │                            │                              │ │
│                    │  • Local connectivity      │                              │ │
│                    │  • Remote executor access  │                              │ │
│                    │  • Client connection       │                              │ │
│                    │                            │                              │ │
│                    └────────────────────────────┼──────────────────────────────┘ │
│                                                 │                                 │
│                                                 │ (dashed backhaul)               │
│                                                 │                                 │
│                                     ┌───────────┼───────────────┐                │
│                                     │           │               │                │
│                                     │ (dotted)  │ (dashed)      │ (dashed)       │
│                                     │           │               │                │
│                                     ▼           ▼               ▼                │
│                            ┌─────────────────┐         ┌─────────────────┐       │
│                            │ DESKTOP         │         │ CLIENT 2        │       │
│                            │ EXECUTOR        │         │ (alternative    │       │
│                            │     🖥️         │         │  route)         │       │
│                            │                 │         │     📱          │       │
│                            │                 │         │                 │       │
│                            └─────────────────┘         └─────────────────┘       │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘

LEGEND:
────────  Solid arrows: Explicit data flows (Job, Input Data, Result Data)
- - - -   Dashed lines: Network connectivity 
· · · ·   Dotted lines: Network connectivity (alternative)
📶        Wireless Router (Broker)
💻        Laptop Executor
🖥️        Desktop Executor
📱        Client (Person with smartphone)
```

## Component Details

### 1. **Data Store (Top-Left)**
```
┌──────────────┐
│ DATA STORE   │
│ ┌──────────┐ │
│ │ Storage  │ │  ← Stack of storage servers
│ │ Servers  │ │  
│ └──────────┘ │
│              │
└──────────────┘
```
- **Function**: Provides input datasets, stores results
- **Connections**: 
	- Dashed network link to server-class executor
	- Input Data (dashed) → Central Broker
	- Result Data (solid) ← Central Broker

### 2. **Central Broker (Centre)**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CENTRAL BROKER                                          │
│                   📶 Wireless Router                                        │
│                                                                             │
│  • Receives jobs from clients                                               │
│  • Retrieves input data from data store                                    │
│  • Dispatches jobs to executors                                            │
│  • Collects result data                                                    │
│  • Returns results to clients                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```
- **Function**: Computation orchestration hub
- **Data Flows**:
	- ← Job (solid) from Client 2
	- → Result Data (solid) to Client 2
	- ← Input Data (dashed) from Data Store
	- → Result Data (solid) to Data Store
	- → Job + Input Data (solid) to Server-class Executor
	- ← Result Data (solid) from Server-class Executor
	- → Job + Input Data (solid) to Laptop Executor
	- ← Result Data (solid) from Laptop Executor

### 3. **Second Broker (Bottom-Centre)**
```
┌────────────────────────────────────────────────────────────────────────────┐
│           SECOND BROKER                                                    │
│         📶 Wireless Router                                                 │
│                                                                            │
│  • Local connectivity                                                      │
│  • Remote executor access                                                  │
│  • Client connection                                                       │
└────────────────────────────────────────────────────────────────────────────┘
```
- **Function**: Network extension, local connectivity
- **Connections**:
	- Dashed backhaul to Central Broker
	- Dotted line to Desktop Executor
	- Dashed lines to Server-class Executor, Client 2

### 4. **Executors** (Computing Devices)

#### **Server-Class Executor (Top-Right)**
```
┌─────────────────┐
│ SERVER-CLASS    │
│ EXECUTOR        │
│ ┌─────────────┐ │
│ │ Rack of     │ │
│ │ Servers     │ │
│ └─────────────┘ │
└─────────────────┘
```
- **Connections**: 
	- Dashed to Data Store
	- Dashed to Second Broker
	- Job + Input Data (solid) ← Central Broker
	- Result Data (solid) → Central Broker

#### **Laptop Executor (Bottom-Left)**
```
┌─────────────────┐
│ LAPTOP          │
│ EXECUTOR        │
│     💻         │
└─────────────────┘
```
- **Connections**:
	- Job + Input Data (solid) ← Central Broker
	- Result Data (solid) → Central Broker
	- Dashed to Second Broker (alternative path)

#### **Desktop Executor (Bottom-Right)**
```
┌─────────────────┐
│ DESKTOP         │
│ EXECUTOR        │
│     🖥️         │
└─────────────────┘
```
- **Connections**: Dotted line to Second Broker (available for jobs)

### 5. **Clients** (End-Users)

#### **Client 1 (Top-Right)**
```
┌─────────────────┐
│ CLIENT 1        │
│     📱          │
└─────────────────┘
```
- **Connection**: Dashed vertical line to Central Broker (connected, no explicit flows)

#### **Client 2 (Right-Centre)**
```
┌─────────────────┐
│ CLIENT 2        │
│     📱          │
└─────────────────┘
```
- **Active Data Flows**:
	- Job (solid) → Central Broker
	- Result Data (solid) ← Central Broker
- **Alternative Route**: Dashed to Second Broker

## Data Flow Summary

### **Primary Operation Flow:**
1. **Client 2** submits **Job** → **Central Broker**
2. **Central Broker** retrieves **Input Data** ← **Data Store**
3. **Central Broker** dispatches **Job + Input Data** → **Executor** (Server/Laptop)
4. **Executor** processes and returns **Result Data** → **Central Broker**
5. **Central Broker** returns **Result Data** → **Client 2**
6. **Central Broker** stores **Result Data** → **Data Store**

### **Network Connectivity:**
- **Mesh Network**: All components interconnected via dashed/dotted lines
- **Redundant Paths**: Second Broker provides alternative routing
- **Scalable Design**: Additional executors and clients can connect via either broker

This architecture enables distributed city-wide computation with fault tolerance, load distribution, and scalable connectivity for emergency response scenarios.




