





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


## IV. The System Working (How an offline-First System Works)
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

## V. System Screens
So far, we have looked at what happens behind the scenes when an offline-first application is being used: data is written locally, changes are placed in an operation queue, synchronization happens when connectivity is available, and conflicts may need to be resolved.
But there is another important part of the design that is easy to overlook: **the user interface needs to communicate what is happening.** In a traditional application, a user can often assume that clicking a button means the server has received and saved the change. In an offline-first application, that assumption is no longer always true. A change may have been saved successfully on the device while still waiting to be synchronized with the server. The UI therefore needs to give the user enough information to understand the current state of their data without exposing all of the complexity of the synchronization system.

### 1. Online and Offline Status
The first thing an offline-first application should communicate is whether the device currently has a connection that can be used for synchronization. This does not necessarily need to be a large message saying *"You are offline."* A small indicator in the application header can be enough. For example, when the device loses connectivity, the application could indicate that changes are being stored locally. When connectivity returns, the indicator can change to show that synchronization is taking place. The important thing is that the user should not be left wondering, `"Did my change actually save?"`
If the user edits a record while offline, the application should make it clear that the change has been saved locally even though it has not yet reached the server. This gives the user confidence that being offline does not mean their work has been lost.

![Offline-first layers](/assets/img/offline-architecture/offline-online.png)
_Fig 1 - Offline-first architecture: This shows if the status is offline or online._

### 2. Local-First Data View
The second important part of the interface is how data is loaded. In a traditional application, navigating to a screen might trigger a request to the server (the remote server). An offline-first application takes a different approach. The application can read the data from its local database first. This means that when the user opens a screen, the application does not necessarily need to wait for a network request before displaying the data. It can immediately show the latest version that exists locally.

That local version may contain two things:
* data that was previously synchronized from the server, and
* changes that the user has made locally but have not yet been synchronized.

For example, suppose the server says that a patient's status is `ACTIVE`. The user changes it to `INACTIVE` while offline. The local database now contains `INACTIVE`, even though the server still contains `ACTIVE`. If the user navigates away and comes back to that record, the application should show `INACTIVE` because that is the latest state known to the device. This is one of the major advantages of local-first design. The application does not have to make a network request every time the user navigates between screens. The network becomes important for synchronization rather than for every interaction.

![Offline-first layers](/assets/img/offline-architecture/local-data-view.png)
_Fig 1 - Offline-first architecture: This shows the local data view._

### 3. Pending Changes Queue
Behind the interface, the application may have an operation queue containing changes that have not yet reached the server. These operations can remain in the local queue until synchronization becomes possible. For the normal user, the application does not necessarily need to expose all of these operations. However, having visibility into the queue is extremely useful for administrators, developers, and support teams.

For example, the system could show information such as:
* how many operations are currently pending,
* what types of operations are waiting,
* how long the oldest operation has been waiting, and
* whether the queue is successfully being processed.

This becomes particularly useful when monitoring an offline-first system in production. Imagine that the queue normally contains a few pending operations and then suddenly grows to thousands. At the same time, no operations are successfully leaving the queue. That could indicate that something is wrong with the synchronization engine, the network connection, or the server receiving the changes. The queue therefore becomes more than just a mechanism for synchronization. It also becomes an important source of operational visibility.

![Offline-first layers](/assets/img/offline-architecture/pending-changes.png)
_Fig 1 - Offline-first architecture: This shows pending changes._

### 4. Conflict Resolution Notification
Conflicts are another situation that the interface needs to communicate carefully. Suppose two devices were working with the same record while offline. Both devices make different changes, and later they reconnect. The synchronization system may determine that the changes conflict and apply a conflict-resolution strategy such as Last-Write-Wins. From the system's perspective, the conflict may already be resolved. But from the user's perspective, something important may have happened to their data. For example, the application might display a small notification:

> "A change from another device was merged."

This does not interrupt the user's workflow, but it tells them that synchronization did something they should be aware of. The situation becomes even more important when a conflict-resolution strategy causes a user's change to be discarded. If Last-Write-Wins is being used and another update has a later timestamp, the user's change may lose the conflict. In that case, silently replacing the user's work can be confusing and potentially dangerous. The application could instead notify the user that their change was not retained and, where appropriate, give them an opportunity to review or enter the information again. The goal is not to expose the entire conflict-resolution algorithm to the user. The goal is to make important changes to their data visible.

![Offline-first layers](/assets/img/offline-architecture/conflict-resolution.png)
_Fig 1 - Offline-first architecture: This shows conflict resolution._

### 5. Sync History and Audit Log
The final element is a synchronization history or audit log. This becomes particularly important in systems where data is sensitive, regulated, or operationally important. A synchronization history can record events such as:

* when synchronization occurred,
* what changes were synchronized,
* whether synchronization succeeded or failed,
* whether conflicts were detected, and
* how those conflicts were resolved.

This creates a history of what happened to the data as it moved between the device and the server. For example, if a user notices that a record contains unexpected information, an administrator or support engineer can investigate the synchronization history instead of simply asking,*"What happened?"*. They can trace the sequence of events and determine whether the record was changed locally, synchronized to the server, modified by another device, or affected by a conflict-resolution rule. This is especially valuable in enterprise systems because synchronization is no longer just a background technical process. It becomes part of the system's data history.

<div style="text-align: center;">
  <img src="/assets/img/offline-architecture/resolution-sync.png"
       alt="Offline-first architecture showing synchronization"
       style="width: 60%; height: 60%;">
</div>
_Fig 1 - Offline-first architecture: This shows sync data._





