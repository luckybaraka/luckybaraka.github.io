





### Offline-first Architecture

Offline-first architecture is an approach to application development where an application is designed to remain functional even when the device has no or unreliable internet connnectivity. Data is stored locally on the device, allowing the user to contniue working without depending on a constant connection to a remote server. When connectivity is unavailable, the application stores new or modified data locally. Once the connection is restored, the application synchronizes the local changes with the server in the background and the user does not feel any of the effect. The synchronization process ensures that the local and remote data eventually become consistent while allowing the user to continue using the application regardless of network availability.  


### Offline-first architecture assumptions
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




