# Linux High Availability (HA) PCS Cluster Comprehensive Guide

## Section 1: Deep Architectural Insights

A PCS cluster operates as a multi-layered stack designed to eliminate single points of failure (SPOF) within enterprise environments.

```text
+---------------------------------------------+

|            Application / Service            |
+---------------------------------------------+

| Pacemaker (CRM - Cluster Resource Manager)  |
+---------------------------------------------+

|   Corosync (Messaging & Membership Layer)   |
+---------------------------------------------+

|     Linux OS (RHEL, Rocky, SLES, etc.)      |
+---------------------------------------------+
```


### 1. Corosync (The Communication Layer)
* **Purpose:** Acts as the low-level heartbeat and membership mechanism. It uses UDP/IP multicast or unicast to constantly track which nodes are alive.
* **Quorum Engine:** Corosync calculates whether the cluster has a "mathematical majority" (N/2 + 1) to safely make decisions. If a cluster drops below quorum, it stops services by default to prevent data corruption.
* **Configuration:** Primary settings reside in `/etc/corosync/corosync.conf`.

### 2. Pacemaker (The Brains)
* **Purpose:** Monitors and manages the actual applications and resources (e.g., Virtual IPs, Web Servers, Database engines).
* **Cluster Information Base (CIB):** An XML file replicated across all nodes containing the cluster's current live state and layout. Administrators modify this safely using `pcs` commands.
* **Designated Coordinator (DC):** The cluster automatically elects one node as the leader (DC). The DC's `crmd` daemon initiates resource placement decisions across the cluster.

### 3. STONITH / Fencing (The Enforcer)
* **The Problem:** If Node A freezes or drops off the network, Node B might think Node A is dead and try to mount the shared storage. If Node A wakes up later, both write to the disk simultaneously, causing catastrophic data corruption (**Split-Brain**).
* **The Solution:** **S**hoot **T**he **O**ther **N**ode **I**n **T**he **H**ead (STONITH). Before Node B takes over any resources, it uses a fencing agent (via hardware interfaces like IPMI, iLO, or cloud APIs) to forcefully cut power to or reboot Node A.

---

## Section 2: Linux Administrator Interview Prep (Top 10 Questions)

### Architectural & Concept Questions

#### Q1: What is the distinct operational difference between Corosync and Pacemaker?
* **Answer:** Corosync handles node infrastructure membership, heartbeat messaging, and quorum verification. Pacemaker operates directly on top of Corosync to monitor, start, stop, and migrate actual services and resources.

#### Q2: What is Split-Brain, and how does STONITH resolve it?
* **Answer:** Split-brain occurs when cluster nodes lose their communication link but remain independently alive. Both sides assume the other is dead and attempt to seize active resources. STONITH protects data integrity by physically powering off the uncommunicative node before any failover resources are allowed to start.

#### Q3: Can you run a production PCS cluster safely without STONITH enabled?
* **Answer:** No. While you can run `pcs property set stonith-enabled=false` for sandbox environments, enterprise vendors (like Red Hat) will refuse to support a production cluster without a fencing mechanism because the risk of data loss is too high.

#### Q4: What is Quorum, and how do you handle it in a 2-node cluster?
* **Answer:** Quorum is a strict majority vote (N/2 + 1). In a 2-node cluster, if 1 node drops, your vote ratio hits 50%, losing quorum. To prevent the cluster from freezing up, administrators must either set `no-quorum-policy=ignore` or deploy a **QDevice** (a lightweight third-party tie-breaker server).

### Operational & Command-Line Questions

#### Q5: How do you gracefully put a node into maintenance mode to perform OS updates?
* **Answer:** Run `pcs node standby <node_name>`. This safely migrates all running resources off that node to other cluster members without triggering a false failover alarm. Run `pcs node unstandby <node_name>` when the work is complete.

#### Q6: What is a "Resource Constraint" in Pacemaker? Name the three types.
* **Answer:** Constraints control resource behavior rules. The types are:
  * **Location:** Dictates *which* node a resource prefers to run on based on a score.
  * **Colocation:** Dictates that Resource A *must* run on the same node as Resource B.
  * **Order:** Dictates the sequence of events (e.g., Database *must* start before Web App).

#### Q7: How do you clear a failed resource error after fixing an underlying configuration bug?
* **Answer:** Run `pcs resource cleanup <resource_id>`. This clears the failure counter and forces the cluster to re-verify the active state of the application.

### Troubleshooting Scenarios

#### Q8: A resource failed over to Node 2. You fixed Node 1, but the resource won't move back automatically. Why?
* **Answer:** This is caused by **Resource Stickiness**. If the stickiness value is set high, the resource will stay on its current node to prevent unnecessary downtime. To force it back automatically, stickiness must be set to `0`, or you must run `pcs resource move`.

#### Q9: The command `pcs status` shows "Resource is unmanaged". What does this mean and how do you fix it?
* **Answer:** It means Pacemaker has stopped monitoring or controlling that resource entirely (often triggered manually during maintenance). You can re-enable active monitoring by running `pcs resource manage <resource_id>`.

#### Q10: Where do you check logs when a cluster resource fails to start?
* **Answer:** Look at system logging using `journalctl -u pcsd` or `journalctl -u pacemaker`. You should also verify `/var/log/messages`, `/var/log/cluster/corosync.log`, and the specific application log (e.g., Apache or PostgreSQL error logs).


# 🛠️ Pacemaker & PCS Command Cheatsheet

### 1. Cluster Status & Verification

| Command Syntax | Usage / Description |
| :--- | :--- |
| `pcs status` | Displays full cluster health, active nodes, and resource states. |
| `pcs status resources` | Filters the status to show only resources and their current locations. |
| `pcs status nodes` | Filters the status to show only node states (Online, Standby, Offline). |
| `pcs cluster verify -v` | Validates the current cluster configuration syntax and checks for errors. |
| `corosync-cmapctl \| grep members` | Verifies the low-level Corosync node membership status directly. |

---

### 2. Node Management

| Command Syntax | Usage / Description |
| :--- | :--- |
| `pcs cluster start --all` | Starts the cluster services (`corosync` and `pacemaker`) on all nodes. |
| `pcs cluster stop --all` | Gracefully stops cluster services on all nodes. |
| `pcs node standby <node_name>` | Moves a node into standby mode; safely migrates all resources off it. |
| `pcs node unstandby <node_name>` | Re-enables a standby node, making it available to host resources again. |
| `pcs cluster enable --all` | Configures systemd to automatically start cluster services on system boot. |
| `pcs cluster disable --all` | Prevents cluster services from starting automatically on system boot. |

---

### 3. Resource Basic CRUD (Create, Read, Update, Delete)

| Command Syntax | Usage / Description |
| :--- | :--- |
| `pcs resource standards` | Lists all supported resource standards (ocf, lsb, service, systemd). |
| `pcs resource agents <standard>:<provider>` | Lists available resource scripts (e.g., `pcs resource agents ocf:heartbeat`). |
| `pcs resource create <resource_id> <agent_name> [options]` | Creates a new cluster resource (e.g., `pcs resource create VirtualIP ocf:heartbeat:IPaddr2 ip=192.168.1.50`). |
| `pcs resource delete <resource_id>` | Permanently removes a resource from the cluster configuration. |
| `pcs resource update <resource_id> [options]` | Modifies parameters of an existing resource without deleting it. |
| `pcs resource config <resource_id>` | Shows the full configuration details and parameters of a specific resource. |

---

### 4. Resource Control & Troubleshooting

| Command Syntax | Usage / Description |
| :--- | :--- |
| `pcs resource disable <resource_id>` | Forces a resource to stop and prevents it from starting anywhere. |
| `pcs resource enable <resource_id>` | Allows a disabled resource to start up again. |
| `pcs resource manage <resource_id>` | Tells Pacemaker to take control and monitor the resource (Default). |
| `pcs resource unmanaged <resource_id>` | Decouples the resource from Pacemaker so you can manually debug the app. |
| `pcs resource cleanup <resource_id>` | Clears resource failure counts and forces the cluster to re-detect its state. |
| `pcs resource move <resource_id> <node_name>` | Forces a resource to move to a specific node (Creates a temporary location constraint). |
| `pcs resource clear <resource_id>` | Removes the temporary constraint created by `pcs resource move`. |

---

### 5. Constraints (Location, Colocation, Order)

| Command Syntax | Usage / Description |
| :--- | :--- |
| `pcs constraint list` | Displays all active constraints configured in the cluster. |
| `pcs constraint location <resource_id> prefers <node_name>=<score>` | Configures a resource preference for a node (e.g., Score `INFINITY` or `100`). |
| `pcs constraint colocation add <resource_id1> with <resource_id2> [score]` | Forces Resource 1 to run on the exact same node as Resource 2. |
| `pcs constraint order <resource_id1> then <resource_id2>` | Directs the cluster to fully start Resource 1 before starting Resource 2. |
| `pcs constraint delete <constraint_id>` | Deletes a specific constraint using its ID (found via `pcs constraint list --ids`). |

---

### 6. Advanced Groups & Clones

| Command Syntax | Usage / Description |
| :--- | :--- |
| `pcs resource group add <group_name> <resource_id1> <resource_id2>` | Groups resources. They will start in order (1 then 2) and stay on the same node. |
| `pcs resource clone <resource_id>` | Configures a resource to run simultaneously across multiple nodes (e.g., GFS2/DLM). |
| `pcs resource promotable <resource_id>` | Creates a multi-state resource (formerly Master/Slave, e.g., replicated databases). |

---

### 7. Global Properties & STONITH

| Command Syntax | Usage / Description |
| :--- | :--- |
| `pcs property config` | Lists all global cluster properties and their current configurations. |
| `pcs property set stonith-enabled=false` | Disables STONITH (Use **only** for non-production testing environments). |
| `pcs property set no-quorum-policy=ignore` | Tells a 2-node cluster to keep running resources even if it loses quorum. |
| `pcs resource defaults resource-stickiness=100` | Sets a global rule preventing resources from moving back if a failed node recovers. |
| `pcs stonith create <stonith_id> <agent_name> [options]` | Configures a hardware fencing agent (e.g., `fence_ipmilan` or `fence_vmware_soap`). |
