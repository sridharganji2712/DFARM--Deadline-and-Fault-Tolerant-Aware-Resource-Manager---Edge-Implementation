# DFARM Edge Scheduling — CloudSim Implementation

## Introduction

This project implements DFARM (Deadline and Fault-aware Resource Manager), a scheduling algorithm originally designed for cloud environments, adapted  for  Edge Computing using the CloudSim framework.

### What DFARM Does

DFARM schedules incoming tasks (cloudlets) across a multi-node edge topology while satisfying Deadline compliance — Every task carries an absolute deadline. DFARM prioritises tasks with tighter deadlines using the DPL (Deadline-Per-Length) metric and attempts to assign each task to a VM whose predicted completion time fits within the deadline.


### Edge Adaptations

The original cloud paper uses a flat pool of homogeneous VMs and a 30% replication ratio. This implementation makes the following edge-specific changes:


| Topology |
cloud: Single datacenter 
edge: 3 nodes (A, B, C) with wireless + backhaul links 


| CT formula | 
cloud; boot + acq + MI/MIPS 
edge: uplink + boot + acq + MI/MIPS + downlink 


| Deadline Per Length formula | 
cloud; (deadline - arrivalTime) / MI  
edge; (deadline - currentTime - uplink) / MI 


| Replication ratio 
cloud: 30% 
edge:  10% (configurable, edge-safe) 


| VM acquisition |
cloud: 50-VM pool
edge: 4–6 VMs per node


### Topology

```
Node A  —  4 VMs, up to 1000 MIPS, wireless 50 Mbps,  boot 5s,  acqDelay 10s
Node B  —  6 VMs, up to 2000 MIPS, wireless 35 Mbps,  boot 3s,  acqDelay 8s
Node C  —  4 VMs, up to  750 MIPS, wireless 10 Mbps,  boot 8s,  acqDelay 15s
Inter-node backhaul: 500 Mbps fibre
```

All three nodes are fully connected (each is a neighbour of the other two), enabling cross-node task offload when local VMs are saturated.

### VM Acquisition Cascade (Phase 3)

When direct assignment and queue adjustment both fail, DFARM evaluates all acquisition sources simultaneously:

1. **Released-idle VM** on origin node — no acquisition delay, just boot time
2. **Fresh VM** from origin node's cold pool — full boot + acquisition delay
3. **Cross-node VM** from any neighbour — adds inter-node transfer delay

The candidate with the lowest predicted CT that still meets the deadline is chosen.

### Baseline Comparisons

Four baseline algorithms run analytically on the same workload after the DFARM simulation, using the identical edge CT model, so results are directly comparable:

- **RS** — Random Selection
- **RR** — Round Robin
- **MCT** — Minimum Completion Time
- **RALBA** — Resource-Aware Load Balancing Algorithm

### Output

Results are stored into `C:\MINIPROJECT\DFARM_EDGE_RESULTS\`:


---

## File Index


| `EdgeDFARMExample.java` | `modules/cloudsim-examples/src/main/java/org/cloudbus/cloudsim/examples/` |

| `DFARMScheduler.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `EdgeDFARMBroker.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `EdgeNode.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `EdgeVm.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `EdgeCloudlet.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `EdgeSchedulerBase.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `RSScheduler.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `RRScheduler.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `MCTScheduler.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `RALBAScheduler.java` | `modules/cloudsim/src/main/java/org/cloudbus/cloudsim/` |

| `DFARMResultsExporter.java` | `modules/cloudsim-examples/src/main/java/org/cloudbus/cloudsim/examples/` |

| `AlgorithmComparisonExporter.java` | `modules/cloudsim-examples/src/main/java/org/cloudbus/cloudsim/examples/` |

---


### `EdgeDFARMExample.java`

**Overview:**  
The simulation entry point. Builds the 3-node edge topology, creates all VMs and cloudlets, runs the CloudSim simulation, prints results, runs the four baseline schedulers analytically on the same workload, and triggers XLSX export.


| `main()` |
 Orchestrates the full simulation: initialises CloudSim, creates nodes and broker, submits VMs and cloudlets, starts and stops simulation, prints results, exports all XLSX reports, runs baselines. |

| `createNodeVms()` |
 Creates one `EdgeVm` per MIPS value in the array, registers each VM with the given edge node's cold pool, and returns the list. Used once per node during setup. |

| `createCloudlets()` | 
Generates 400(configurable) `EdgeCloudlet` tasks with randomised arrival times (burst pattern: early wave 0–10 s, heavy burst 10–20 s, tail 20–60 s), task sizes (small/medium/large/huge), and deadlines scaled to node bias and task size. Sets origin node for uplink delay computation. |

| `printResults(EdgeDFARMBroker broker)` |
 Prints the cloudlet execution table (ID, status, VM, node, start/finish/CPU times) and the DFARM summary block (total tasks, accepted, rejected, TRR, makespan, throughput, ARUR). |

---

### `DFARMScheduler.java`

**Overview:**  
The core DFARM scheduling engine. A pure logic container  not a CloudSim `SimEntity`. Implements all 8 algorithms from the paper (adapted for edge) using arrival-driven scheduling. `EdgeDFARMBroker` calls its three public methods as simulation events start.

**Key data structures:**
- `availableVms` — VMs currently in the active pool (acquired from node pools)
- `vmCtMap` — predicted completion time per VM (boot+acq overhead + queued task costs)
- `vmQueueMap` — ordered task queue per VM
- `taskVmMap` — task-to-VM assignment (includes replicas)
- `dplHistory` — sorted DPL values for fault-tolerance threshold decisions


| `initScheduler(List<EdgeVm> vms, double currentTime)` |
 Algorithm 3. Resets all internal state and starts with an empty active VM pool so the acquisition cascade is triggered on demand. Called once when VMs are ready. |

| `scheduleOneTask(EdgeCloudlet task, double currentTime)` | 
Algorithm 1. Decides fault-tolerance (replicate or not), inserts DPL into history, calls `doSchedule` for the task, and if replication was decided, schedules a replica on a second VM. Returns a list: empty = rejected, one item = assigned, two items = assigned with replica. |

| `notifyTaskCompleted(EdgeCloudlet task)` | 
Algorithm 6. Removes a finished task from its VM's queue. If the queue empties, moves the VM to the released-idle pool so future tasks skip the acquisition delay. |

| `assignAcquire(EdgeCloudlet task, List<EdgeVm> availableVms, List<Map.Entry<EdgeVm,Double>> sortedVmCts, double currentTime)` | 
Algorithm 4. Phase 1: iterates active VMs sorted by CT — returns the first whose estimated finish meets the deadline. Phase 2: if Phase 1 fails, tries queue adjustment via `adjustPossible`. Phase 3: if both fail, calls `pickBestAcquisition` to acquire a new VM. |

| `pickBestAcquisition(EdgeCloudlet task, List<EdgeVm> availableVms, double bestLocalCt)` |
 Unified acquisition. Builds candidates from all three sources (released-idle, fresh, cross-node), classifies each as feasible or infeasible against the deadline, and selects the best feasible candidate. Falls back to the best infeasible candidate only if it beats an existing overloaded VM's CT. |

| `buildAcqCandidatesWithLogging(EdgeCloudlet task)` | 
Builds the full candidate list with per-source diagnostic log lines. Used in production for transparency during development and testing. |

| `buildAcqCandidates(EdgeCloudlet task)` | 
Silent version of the above — builds candidates from the same three sources without logging. Available for use when logging is not needed. |

| `activateCandidate(AcqCandidate winner, EdgeCloudlet task, List<EdgeVm> availableVms)` |
 Promotes the winning VM from its node pool (released-idle or fresh) to active, seeds its `vmCtMap` entry with boot+acq overhead, and adds it to `availableVms`. Also records cross-node assignments and reused VM IDs. |

| `adjustPossible(EdgeCloudlet newTask, EdgeVm vm)` |
 Algorithm 7. Traverses the VM's queue in reverse, checking whether swapping `newTask` earlier would let both tasks meet their deadlines. Returns the candidate cloudlet to swap before, or null if no valid reordering exists. Respects `deepSearch` mode for aggressive vs simple search. |

| `adjustToVm(EdgeCloudlet newTask, EdgeVm vm, EdgeCloudlet candidate)` | 
Inserts `newTask` before `candidate` in the VM's ordered queue. |

| `determineFaultTolerance(EdgeCloudlet task, List<Double> history, double ratio, double currentTime)` |
 Algorithm 8. Computes the network-aware DPL for the incoming task and compares it against the `ratio`-th percentile of the DPL history. Returns true (replicate) if the task's DPL is at or below that threshold, meaning its deadline is relatively tight. |

| `doSchedule(EdgeCloudlet task, double currentTime)` | 
Helper that calls `assignAcquire` and immediately calls `commit` if a VM was found. Returns the assigned VM or null. |

| `commit(EdgeCloudlet task, EdgeVm vm)` | 
Records task→VM in `taskVmMap`, adds the task's cost to `vmCtMap`, and appends the task to the VM's queue in `vmQueueMap`. |

| `releaseIdleVm(EdgeVm vm)` |
 Removes a VM from `availableVms`, locates it in its node's active list, and moves it to `releasedIdleVms`. Future tasks on this VM skip the acquisition delay. |

| `taskCost(EdgeVm vm, EdgeCloudlet task)` | 
Computes `MI/MIPS + downlinkDelay`. Uplink is excluded because it is per-task network overhead and must not be stored in the VM's CT backlog. |

| `taskCostWithUplink(EdgeCloudlet task, EdgeVm vm)` |
 Returns `taskCost + uplinkDelay`. Used where feasibility must include the full end-to-end delay (e.g., queue adjustment checks). |

| `networkAwareDPL(EdgeCloudlet task, double currentTime)` |
 Computes `(deadline - currentTime - uplinkDelay) / MI`. Subtracting uplink gives the true remaining execution budget rather than an optimistic value. |

| `sortedVmCtEntries()` |
 Returns active VMs (those in `availableVms`) sorted by ascending CT from `vmCtMap`. Historical VMs not in `availableVms` are excluded to prevent stale data driving Phase 1/2 routing. |

| `insertSortedDPL(EdgeCloudlet task, double currentTime)` |
 Computes network-aware DPL and inserts it into `dplHistory` using binary search to maintain ascending order. |

| `makeCopy(EdgeCloudlet original)` |
 Creates a replica cloudlet with a negative ID (`-(|id|+1)`), same parameters as the original, and `isDuplicate = true`. |

| `computeTRR(int totalOriginalTasks)` |
 Task Rejection Rate: rejected originals / total originals. |

| `computeMakespan()` |
 Maximum CT across all VMs in `vmCtMap`. |

| `computeThroughput(int totalOriginalTasks)` | 
Accepted originals / makespan (tasks per second). |

| `computeARUR()` | 
Average Resource Utilisation Ratio: avg(VM CT) / makespan. Values near 1.0 indicate balanced load. |


---

### `EdgeDFARMBroker.java`

**Overview:**  
Extends CloudSim's `DatacenterBroker` to wire `DFARMScheduler` into the event loop using arrival-driven scheduling. Manages two custom event tags (`TASK_ARRIVED`, `DOWNLINK_COMPLETE`) and models the full end-to-end delay: uplink → boot/acq → execution → downlink.


| `submitCloudlets()` |
 Step 1. Called by CloudSim when VMs are ready. Initialises the DFARM scheduler with the active VM list, then schedules one `TASK_ARRIVED` event per task at its `arrivalTime`. Clears `cloudletList` to prevent the base class from interfering. |

| `processEvent(SimEvent ev)` | 
Event dispatcher. Routes `TASK_ARRIVED` to `handleTaskArrival`, `DOWNLINK_COMPLETE` to `handleDownlinkComplete`, and all other events to the parent class. |

| `handleTaskArrival(SimEvent ev)` |
 Step 2. Called when a `TASK_ARRIVED` event fires. Decrements `pendingArrivals`, calls `dfarm.scheduleOneTask()` at the exact simulation clock, and submits all scheduled cloudlets (original + optional replica) to their datacenters via `submitWithDelay`. Logs hard-line rejections. |

| `submitWithDelay(EdgeCloudlet cloudlet, EdgeVm vm, int datacenterId)` |
 Computes the pre-execution delay (uplink + boot+acq for the first task on a VM + inter-node delay for cross-node tasks) and sends the cloudlet to the datacenter after that delay using `CloudActionTags.CLOUDLET_SUBMIT`. |

| `processCloudletReturn(SimEvent ev)` |
 Step 3a. Called by CloudSim when CPU execution finishes. Adds the cloudlet to the received list, then schedules a `DOWNLINK_COMPLETE` event `downlinkDelay` seconds later to model result delivery to the device. |

| `handleDownlinkComplete(SimEvent ev)` |
 Step 3b. Called when result delivery is complete. Notifies DFARM that the task finished (triggering VM idle release if applicable), logs predicted-vs-actual CT delta, decrements `cloudletsSubmitted`, and checks for shutdown. |


| `checkShutdown()` |
 Triggers `finishExecution()` only when both `cloudletsSubmitted == 0` and `pendingArrivals == 0`. This prevents premature shutdown between task arrival waves. |


| `logActualVsPredicted(EdgeCloudlet cloudlet)` |
 Computes and logs the delta between DFARM's predicted CT and the actual end-to-end simulation time (`execFinishTime + downlinkDelay`). Useful for validating the CT model accuracy. |


| `shutdownEntity()` |
 Called by CloudSim at simulation end. Invokes parent shutdown then calls `printDFARMMetrics()`. |

| `printDFARMMetrics()` |
 Prints the DFARM performance summary: total/accepted/rejected tasks, TRR, makespan, throughput, ARUR, and per-VM CT breakdown with node labels and reuse flags. |

---

### `EdgeNode.java`

**Overview:**  
A plain state container representing a physical edge node (cell tower / MEC server). Not a CloudSim `SimEntity`. Maintains three VM lifecycle pools and provides bandwidth-based delay calculations. `DFARMScheduler` queries this object to make VM assignment decisions.

**VM lifecycle pools:**
- `availableVmPool` — cold VMs not yet acquired (full boot + acq delay on first use)
- `activeVms` — currently scheduled VMs
- `releasedIdleVms` — previously active, now idle (boot delay only on reuse)


| `addVm(EdgeVm vm)` |
 Registers a VM into the node's cold pool at setup time. |

| `addNeighbor(EdgeNode node)` |
 Adds a neighbour node for cross-node offload (Phase 3 of acquisition). |

| `findReleasedVm()` |
 Scans `releasedIdleVms` and returns the highest-MIPS VM. Returns null if the pool is empty. Preferred over fresh VMs because it skips the acquisition delay. |

| `findFreshVm()` |
 Scans `availableVmPool` and returns the highest-MIPS VM. Returns null if the pool is exhausted. |

| `computeUplinkDelay(double inputSizeKB)` |
 Returns `inputSizeKB / wirelessBandwidthKBps`. Always called on the origin node — the device sends to its nearest node regardless of where the assigned VM runs. |

| `activateReleasedVm(EdgeVm vm)` |
 Moves a VM from `releasedIdleVms` to `activeVms`. Called by `DFARMScheduler` when a released-idle VM is selected as the winner. |

| `activateFreshVm(EdgeVm vm)` |
 Moves a VM from `availableVmPool` to `activeVms`. Called by `DFARMScheduler` for cold-start acquisitions. |

| `releaseIdleVms(Map<EdgeVm, List<?>> vmQueueMap)` |
 Algorithm 6 (node-side). Scans active VMs, finds those with empty queues, and moves them to `releasedIdleVms`. Freed VMs skip the acquisition delay when reused for future tasks. |

| `findNeighborWithFreeVm()` | 
Iterates `neighborNodes` and returns the first neighbour that has either a released-idle or fresh VM available. Returns null if all neighbours are exhausted. |

| `computeInterNodeDelay(double inputSizeKB)` | 
Returns `inputSizeKB / interNodeBandwidthKBps`. Used when a task is offloaded across the backhaul link to a neighbour node. |

---

### `EdgeVm.java`

**Overview:**  
Extends CloudSim's `Vm` with edge-specific fields required by the DFARM CT formula. Each VM carries its node's wireless bandwidth (uplink/downlink to devices) and inter-node backhaul bandwidth (cross-node offload), along with cold-start timing parameters.



| `computeUplinkDelay(double inputSizeKB)` |
 Returns `inputSizeKB / wirelessBandwidthKBps`. Represents the time to transmit the task's input payload from the device to the edge node. |

| `computeDownlinkDelay(double outputSizeKB)` |
 Returns `outputSizeKB / wirelessBandwidthKBps`. Represents the time to transmit the result from the edge node back to the device. |

| `computeInterNodeDelay(double inputSizeKB)` |
 Returns `inputSizeKB / interNodeBandwidthKBps`. Represents the backhaul transfer time when a task is routed to a neighbouring node. |

| `computeEdgeCT(EdgeCloudlet cloudlet, double currentVmCt)` |
 Computes the full CT for a fresh VM: `currentVmCt + uplinkDelay + bootTime + acquisitionDelay + MI/MIPS + downlinkDelay`. Used for analytical CT comparisons. |

| `computeEdgeCTReused(EdgeCloudlet cloudlet, double currentVmCt)` |
 Computes CT for a released-idle VM (no acquisition delay): `currentVmCt + uplinkDelay + bootTime + MI/MIPS + downlinkDelay`. |




---

### `EdgeCloudlet.java`

**Overview:**  
Extends CloudSim's `Cloudlet` with edge-specific fields used by DFARM for scheduling decisions: deadline, arrival time, input/output sizes (driving network delay calculations), source device ID, duplicate flag, and origin node reference.


| `EdgeCloudlet(int cloudletId, long cloudletLength, double inputSizeKB, double outputSizeKB, double deadline, double arrivalTime, String sourceDeviceId, double deviceLatencyMs)` |
 Full constructor. Converts KB sizes to bytes for CloudSim's base class, sets all edge fields, and marks `isDuplicate = false`. |


| `EdgeCloudlet(long cloudletLength, double inputSizeKB, double outputSizeKB, double deadline, double arrivalTime, String sourceDeviceId, double deviceLatencyMs)` |
 Auto-ID constructor. Calls the full constructor with an auto-incremented ID from CloudSim's `AtomicInteger`. |


| `computeDPL(double currentTime)` |
 Returns `(deadline - currentTime) / cloudletLength`. Lower DPL means tighter deadline relative to computation size — higher priority for replication and queue adjustment. |


| Getters/Setters |
 `getDeadline`, `getArrivalTime`, `getInputSizeKB`, `getOutputSizeKB`, `getDeviceLatencyMs`, `getSourceDeviceId`, `isDuplicate`, `getOriginNode` and their setters. |



---

### `EdgeSchedulerBase.java`

**Overview:**  
Abstract base class for all four baseline schedulers. Provides the shared state (VM CT map, task-to-VM map, deadline tracking) and the identical edge CT model used by DFARM, ensuring fair metric comparison. Baselines run analytically — they do not interact with CloudSim events.



| `schedule(List<EdgeCloudlet> tasks, List<EdgeVm> vms, double currentTime)` |
 Abstract. Subclasses implement their task-to-VM assignment policy here and return the completed `taskVmMap`. |


| `getAlgorithmName()` |
 Abstract. Returns a short identifier string shown in reports ("RS", "RR", "MCT", "RALBA"). |


| `initVmCtMap(List<EdgeVm> vms)` |

 Seeds `vmCtMap` with each VM's cold-start overhead (`bootTime + acquisitionDelay`). Called at the start of `schedule()` in every subclass. |


| `taskCost(EdgeVm vm, EdgeCloudlet task)` |
 Returns `uplinkDelay + MI/MIPS + downlinkDelay`. Boot/acq overhead is already in the `vmCtMap` base so it is not added here. Identical formula to `DFARMScheduler.taskCost`. |


| `assign(EdgeCloudlet task, EdgeVm vm)` |
 Records the assignment: updates `vmCtMap`, stores `[start, finish]` timestamps in `taskTimestamps`, and appends the task to `deadlineMissedTasks` if `endCt > deadline`. |


| `computeMakespan()` | 
Maximum CT across all assigned VMs. |


| `computeMissRate(int totalOriginalTasks)` |
 Deadline-missed originals / total originals. Equivalent to DFARM's TRR for baseline comparison. |


| `computeThroughput(int totalOriginalTasks)` |
 Accepted originals / makespan (tasks per second). |


| `computeARUR()` |
 Average VM CT / makespan. Measures load balance across VMs. |

---

### `RSScheduler.java`

**Overview:**  
Random Selection baseline. Assigns each task to a randomly chosen VM from the pool using a fixed seed (42) for reproducibility. No deadline awareness; all tasks are always assigned.



| `schedule(List<EdgeCloudlet> tasks, List<EdgeVm> vms, double currentTime)` |
 Initialises VM CTs, then for each task picks `vms.get(rng.nextInt(vms.size()))` and calls `assign`. Returns the completed task map. |

| `getAlgorithmName()` |
 Returns `"RS"`. |

---

### `RRScheduler.java`

**Overview:**  
Round Robin baseline. Assigns tasks to VMs in strict cyclic order (VM 0, 1, 2, …, n−1, 0, 1, …). No deadline awareness; all tasks are always assigned.


| `schedule(List<EdgeCloudlet> tasks, List<EdgeVm> vms, double currentTime)` | Initialises VM CTs, then iterates tasks with a counter `index % vms.size()` to pick the next VM in rotation. Returns the completed task map. |


| `getAlgorithmName()` | Returns `"RR"`. |

---

### `MCTScheduler.java`

**Overview:**  
Minimum Completion Time baseline. For each task, evaluates every VM and assigns to the one that minimises `vmCt[vm] + taskCost(vm, task)`. Classic greedy algorithm; no deadline awareness; all tasks are always assigned.

| Function | Responsibility |
|---|---|
| `schedule(List<EdgeCloudlet> tasks, List<EdgeVm> vms, double currentTime)` |
 Initialises VM CTs, then for each task scans all VMs to find `argmin(vmCt + taskCost)` and calls `assign`. Returns the completed task map. |


| `getAlgorithmName()` | Returns `"MCT"`. |

---

### `RALBAScheduler.java`

**Overview:**  
Resource-Aware Load Balancing Algorithm baseline. Two-phase scheduling designed to evenly distribute load across heterogeneous VMs before falling back to MCT for overflow tasks.


| `schedule(List<EdgeCloudlet> tasks, List<EdgeVm> vms, double currentTime)` |
 Runs the full two-phase algorithm: (1) **Fill phase** — computes a per-VM time budget (`totalMI / totalMIPS` seconds), sorts tasks descending by MI, and assigns each task to the VM with the most remaining capacity that can still fit it; (2) **Spill phase** — handles tasks that exceeded the fill budget using MCT. Returns the completed task map. |


| `getAlgorithmName()` | Returns `"RALBA"`. |

---
