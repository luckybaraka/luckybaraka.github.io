





### I. Offline-first Architecture
Offline-first architecture is an approach to application development where an application is designed to remain functional even when the device has no or unreliable internet connnectivity. Data is stored locally on the device, allowing the user to contniue working without depending on a constant connection to a remote server. When connectivity is unavailable, the application stores new or modified data locally. Once the connection is restored, the application synchronizes the local changes with the server in the background and the user does not feel any of the effect. The synchronization process ensures that the local and remote data eventually become consistent while allowing the user to continue using the application regardless of network availability.  


### II. Offline-first architecture assumptions
Offline-first architecture is based on the idea that the application should not depend on a constant connection to the server to provide a usable experience. We can think about this using two main assumptions:

- The device's local data store acts as the primary source of data for the - application while the user is working.
- The server acts as the remote source with which the device synchronizes when connectivity is available. (Basically the server acts as a resource for the device to connect to when it is necessary)

When a user reads data, the application can serve that data from the local store instead of waiting for a response from the server. When the user creates or modifies data, the changes are first persisted locally. The application can then synchronize those changes with the server in the background when a network connection becomes available.

This changes the way we think about the network. Instead of making the network a requirement for every user interaction, the application is designed to continue functioning without it. The network becomes something that the application uses for synchronization rather than something that every operation depends on.

From here, we can break offline-first architecture into several areas:

 1. Local data storage - How is data stored on the device? What technologies or databases are used, such as IndexedDB, SQLite, or other local stores, and why?
 2. Change tracking - How does the application know which data has been created, updated, or deleted while offline?
 3. Operation queuing - How are changes stored and queued until they can be sent to the server?
 4. Synchronization - How are local changes synchronized with the remote server once connectivity is restored?
 5. Conflict resolution - What happens when two devices modify the same piece of data while disconnected?
 6. Consistency - How does the system ensure that the local and remote copies of the data eventually become consistent?
 7. Failure handling - What happens when synchronization itself fails, a request is duplicated, or the connection is lost during synchronization?



## III. System Architecture
An offline-first system can be thought of as having four main layers. Each layer solves a different part of the problem. The four layers are:
1. **Local Storage Layer**
2. **Change Tracking and Operation Queue**
3. **Synchronization Engine**
4. **Conflict Resolution Layer**

Together, these layers allow an application to continue working when the network is unavailable and synchronize its data when connectivity returns.

### 1. Local Storage Layer
Everything starts with the local storage layer. The application maintains a local database on the device, either containing all the data the application needs or only the portion that is likely to be needed while the user is working. On the web, this is commonly done using **IndexedDB**, which is built into modern browsers. On mobile devices, **SQLite** is a common choice, although other technologies such as **Realm** and **WatermelonDB** can also be used. The important idea here is that **reads and writes happen locally first**. The application does not need to wait for the network every time the user wants to read or modify data.
For example: If the device is offline, the user can still continue working because the application is communicating with its local database rather than depending on the server.

### 2. Change Tracking and Operation Queue
If changes are being made locally, the application needs a way to remember those changes so that they can eventually be sent to the server.
This is where change tracking and the operation queue come in. Whenever the user creates, updates, or deletes something, the application records the operation in a queue. The operation might contain information such as: 
 - The type of the operation that it was
 - Which record changed, or what was created
 - The date and time of this event happening
 - The device that performed the operation

When the device is offline, these operations simply remain in the queue. When connectivity becomes available again, the synchronization engine can process the pending operations and send them to the server. The queue should also be persistent. This means that if the application crashes, the device restarts, or the application is closed, the pending operations should not disappear. They should still be available when the application starts again.

### 3. Synchronization Engine
The synchronization engine is responsible for moving data between the local device and the server. It generally works in the background rather than requiring the user to manually synchronize the application. When a network connection becomes available, the synchronization engine can:

1. Find changes that are waiting in the local operation queue.
2. Send those changes to the server.
3. Fetch changes that were made on the server or by other devices.
4. Apply those changes to the local database.
5. Detect situations where local and remote changes conflict and pass them to the conflict resolution mechanism.

The important thing is that synchronization happens in the background. The user should not have to wait for the network before continuing to use the application.


### 4. Conflict Resolution Layer
This is where things become more interesting. Because each device can continue working independently while offline, two devices can modify the same piece of data without knowing about each other's changes. When both devices reconnect, the system now has two different changes to the same piece of data. The conflict resolution layer determines how these changes should be handled and how the system eventually gets all replicas back to a consistent state. There are different approaches to conflict resolution. Three commonly discussed approaches are 
 - **Last-Write-Wins (LWW)**
 - **Operational Transformation (OT)**
 - **Conflict-Free Replicated Data Types (CRDTs).**

#### Last-Write-Wins (LWW)
Last-Write-Wins is the simplest approach. Each change has a timestamp, and when two changes conflict, the system accepts the change that happened last. This approach is simple, but the earlier change is discarded. This can result in data being lost when the earlier change was also important.

#### Operational Transformation (OT)
Operational Transformation takes a different approach. Instead of simply choosing one change as the winner, it attempts to transform concurrent operations so that they can be applied together while maintaining a consistent result. This approach is particularly associated with collaborative editing, where multiple users can modify the same document at the same time. Rather than simply asking which operation should win, OT attempts to adjust the operations so that the changes can coexist.

#### Conflict-Free Replicated Data Types (CRDTs)
CRDTs are specially designed data structures that allow different replicas to be modified independently and then merged in a way that allows them to eventually converge to the same state. The key idea is **convergence**.Different devices may temporarily have different versions of the data, but after their changes are exchanged and merged, the replicas should eventually arrive at the same state. CRDTs are particularly useful for applications that need offline operation and collaborative editing. Technologies such as Automerge and Yjs are examples of systems based on CRDT concepts.

Together, these four layers form the basic architecture of an offline-first system:
![Offline-first layers](/assets/img/offline-architecture/fig1-offline.png)
_Fig 1 - Offline-first architecture layers: This shows all the 4 offline-first layers._

The four layers can be viewed from the bottom up, starting with the **Local Storage Layer**, where data is persisted on the device. Above it is the **Change Tracking and Operation Queue**, which records changes made locally and keeps them pending until they can be synchronized. The third layer is the **Synchronization Engine**, which is responsible for exchanging changes between the device and the server. At the top is the **Conflict Resolution Layer**, which handles situations where changes from different devices or the server conflict with one another.

- **Layer 1** handles the application's read and write operations locally, without requiring the network for every interaction.
- **Layer 2** keeps track of changes made to the local data and maintains pending operations that need to be synchronized with the server.
- **Layer 3** manages synchronization between the local device and the server. It detects when synchronization can take place, sends pending local changes, and retrieves changes made remotely.
- **Layer 4** comes into play when independently made changes conflict with each other. It applies the application's chosen conflict-resolution strategy to determine how those changes should be reconciled.

The interaction between these layers allows data to move through the system in both directions - that means it is bi-directional. Local changes originate in the storage layer and move through change tracking and synchronization toward the server. Changes received from the server can move back through the synchronization process and eventually be applied to the local store, while conflicting changes are handled by the conflict resolution layer before the final state is persisted.(The bi-directional arrows between layers indicate that data can be passed either up or down; change of state is passed upwards from storage, while resolved states are passed downwards to storage.)


## IV The System Working (How an offline-First System Works)
The four layers describe the structure of an offline-first system, but it is useful to see what actually happens when a user interacts with the application. The easiest way to understand this is to follow a single user action. The user performs an action, the application responds immediately, the change is recorded locally, and the synchronization process takes care of communicating with the server when a connection is available.

### 1. Local Write and Immediate UI Response
When a user performs an action, the application does not first ask the server for permission to update the screen. Instead, the change is written to the local database first, and the user interface is updated from that local state. For example, imagine a user changing the status of a record from `ACTIVE` to `INACTIVE`. The application saves that change locally and immediately shows `INACTIVE` on the screen. This happens whether the device is connected to a fast Wi-Fi network or has no connection at all. From the user's perspective, the application behaves the same way because the network is not required for the immediate interaction. The important idea here is that the user is interacting with the **local copy of the data**, not waiting for the remote server to respond.

### 2. Operation Logging
Writing the change to the local database is only part of the process. The application also needs to remember that this change still needs to reach the server. For this reason, the operation is recorded in a persistent queue. The queue can contain information such as the operation's unique ID, the type of operation, the record that was changed, the new value, the timestamp, and the device that made the change. The important part is that the queue is persisted rather than kept only in memory. If the application crashes, the device restarts, or the user closes the application while offline, the pending operation should still be there when the application starts again. The application therefore does not have to remember only the current state, it also remembers **what needs to be synchronized**.

### 3. Connectivity Monitoring
The synchronization engine needs to know when it can communicate with the server. While the device is offline, the application does not stop working. New changes continue to be written locally and added to the pending operation queue. When connectivity becomes available again, the synchronization engine can begin processing those pending operations. The important point is that connectivity is treated as something that can change at any time. The application therefore needs to be able to move between an offline state and an online state without interrupting the user's work. The exact mechanism used to detect connectivity depends on the platform. Mobile applications can use platform-specific network APIs(IoS has its own ways and Android also have its own way of detecting that the network is restored), while web applications have browser APIs that can provide information about network connectivity.

### 4. Synchronization
Once connectivity is available, the synchronization engine begins exchanging changes with the server. Rather than unnecessarily sending the entire local database every time synchronization occurs, an offline-first system can send only the changes that have occurred since the last successful synchronization. These are often referred to as **deltas** or changes. For example, instead of sending an entire patient record containing dozens of fields, the application may only need to communicate that the `status` field changed from `ACTIVE` to `INACTIVE`. The server receives these changes, processes them, and returns any relevant changes that the device does not yet have. Those remote changes can then be applied to the local database. This creates a two-way synchronization process:

* The device sends its local changes to the server.
* The device receives changes made elsewhere.
* The local database is updated with the resulting state.

The same process can happen on other devices, allowing the different copies of the data to gradually converge.

### 5. Conflict Detection and Resolution
Synchronization becomes more complicated when two devices have changed the same data while they were offline.
Imagine that both devices started with:


`Status = ACTIVE`

Device A changes it to:

`INACTIVE`

while Device B changes it to:


`SUSPENDED`

Neither device knew about the other's change because they were offline. When they eventually synchronize, the server or synchronization system discovers that the changes cannot simply be applied independently without deciding how the conflict should be handled. This is where the conflict-resolution strategy comes in. With **Last-Write-Wins (LWW)**, the system can compare the changes and accept the one considered to be the latest. With **Operational Transformation (OT)**, concurrent operations are transformed so that they can be applied together in a consistent way. With **Conflict-Free Replicated Data Types (CRDTs)**, the data structures and their merge rules are designed so that independently made changes can be combined and the replicas can eventually converge to the same state. The important thing is that conflict resolution is not simply about choosing a winner in every situation. The appropriate strategy depends on the type of data and what the application considers to be a correct result. After the conflict has been resolved, the resulting state can be synchronized back to the other devices so that they eventually converge on the same data.

### 6. Service Workers on the Web
For web applications, there is another important mechanism that can participate in offline-first behavior: **Service Workers**.
A Service Worker is a script that runs separately from the main web page and can intercept network requests made by the application. This gives the application a place to implement offline behavior, such as serving cached resources or handling requests when the network is unavailable. For example, if a web application needs to communicate with a server but the browser currently has no connection, the Service Worker can participate in the offline strategy instead of allowing the request to simply fail. The browser also provides mechanisms such as Background Sync that can allow work to be retried when connectivity becomes available again, although support and behavior depend on the browser and platform. It is therefore useful to think of the Service Worker as part of the infrastructure that helps a web application operate offline. It is **not itself the entire offline-first architecture or automatically a complete operation queue**. The application still needs to decide how data is stored, how changes are recorded, how synchronization works, and how conflicts are resolved.

### Putting It All Together

The complete process can therefore be understood as a continuous cycle:
**User action → local write → operation recorded → application continues working → connectivity returns → synchronization → conflict resolution when necessary → local state updated.**
The important idea behind all of this is that **the network is no longer in the critical path of every user interaction**. The application can continue working locally and treat synchronization with the server as an ongoing background process. That is what makes an offline-first application different from an application that simply happens to have some offline functionality.

