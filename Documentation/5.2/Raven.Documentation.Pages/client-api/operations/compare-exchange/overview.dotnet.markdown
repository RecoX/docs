# Compare Exchange Overview 
---

{NOTE: }

* **Compare-exchange** items are cluster-wide key/value pairs where the key is a unique identifier. 

* Each compare-exchange item contains: 
  * A key which is a unique string across the cluster.  
  * A value which can be numbers, strings, arrays, or objects.  
    Any value that can be represented as JSON is valid.
  * Metadata  
  * Raft index - A version number which is modified on each change.  
    Any change to the value or metadata changes the Raft index.  

* Creating and modifying a compare-exchange item is an atomic, thread-safe [compare-and-swap](https://en.wikipedia.org/wiki/Compare-and-swap) interlocked 
  compare-exchange operation.
  * The compare-exchange item is distributed to all nodes in a [cluster-wide transaction](../../../server/clustering/cluster-transactions)
    so that a consistent, unique key is guaranteed cluster-wide.  

* **To Ensure ACID Transactions** RavenDB automatically creates [Atomic Guards](../../../client-api/operations/compare-exchange/atomic-guards) 
  in cluster-wide transactions.  
  * Cluster-wide transactions present a [performance cost](../../../client-api/operations/compare-exchange/overview#performance-cost-of-cluster-wide-sessions) when compared to non-cluster-wide transactions. 
    They prioritize consistency over performance to ensure ACIDity across the cluster.  

*In this page:  
 * [Using Compare-Exchange Items](../../../client-api/operations/compare-exchange/overview#using-compare-exchange-items)  
 * [Transaction Scope for Compare-Exchange Operations](../../../client-api/operations/compare-exchange/overview#transaction-scope-for-compare-exchange-operations)  
 * [Creating a Key](../../../client-api/operations/compare-exchange/overview#creating-a-key)  
 * [Updating a Key](../../../client-api/operations/compare-exchange/overview#updating-a-key)  
 * [Example I - Email Address Reservation](../../../client-api/operations/compare-exchange/overview#example-i---email-address-reservation)  
 * [Example II- Reserve a Shared Resource](../../../client-api/operations/compare-exchange/overview#example-ii---reserve-a-shared-resource)  

{NOTE/}

---

{PANEL: Using Compare-Exchange Items}

#### Why use compare-exchange items

* Compare-exchange items can be used to coordinate work between sessions that are 
  trying to modify a shared resource (such as a document) at the same time.  

  
* You can use compare-exchange items in various situations to protect or reserve a resource.  
  (see [API Compare-exchange examples](../../../client-api/operations/compare-exchange/overview#example-i---email-address-reservation)).
  * If you create a compare-exchange key/value pair, you can decide what actions to implement when the Raft index increments. 
    The Raft index increments as a result of a change in the compare-exchange value or metadata.  

---

#### How compare-exchange items are managed  

* Compare exchange items are created and managed with any of the following approaches:
  * RavenDB creates and maintains [Atomic Guards](../../../client-api/operations/compare-exchange/atomic-guards) automatically in 
    cluster-wide sessions to guarantee ACIDity across the cluster.  (As of RavenDB version 5.2)
  * Via [Client API Operations](../../../client-api/operations/compare-exchange/overview#transaction-scope-for-compare-exchange-operations)  
  * In [Cluster-Wide Sessions](../../../client-api/session/cluster-transaction)
  * Using [Studio](../../../studio/database/documents/compare-exchange-view#the-compare-exchange-view)

---

#### How Atomic Guard items work when protecting shared documents in cluster-wide sessions

* Every time a document in a cluster-wide session is modified, RavenDB automatically creates 
  or uses an associated compare-exchange item called [Atomic Guard](../../../client-api/operations/compare-exchange/atomic-guards).  
  * Whenever one session changes and saves a document, the Atomic Guard version changes. 
    If other sessions loaded the document before the version changed, they will not be able to modify it.  
  * A `ConcurrencyException` will be thrown if the Atomic Guard version was modified by another session.  
* **If a ConcurrencyException is thrown**:  
  To ensure [atomicity](https://en.wikipedia.org/wiki/ACID#Atomicity), if even one session transaction fails, the entire [session will roll back](../../../client-api/session/what-is-a-session-and-how-does-it-work#batching).  
  Be sure that your business logic is written so that **if a concurrency exception is thrown**, your code will re-execute the entire session.  


{INFO: }
####Performance cost of cluster-wide sessions  
Cluster-wide transactions are more expensive than node-local transactions due to Raft concensus checks.  
People prefer a cluster-wide transaction when they prioritize consistency over performance and availability.  
It ensures ACIDity across the cluster.  

* One way to protect ACID transactions **without** using cluster-wide sessions is to ensure that one node is responsible for writing on a specific database.  
    * RavenDB node-local transactions are ACID on the local node, but if two nodes write concurrently on the same document, [conlicts](../../../server/clustering/replication/replication-conflicts)
      can occur. 
    * By default to prevent conflicts, one node is responsible for all reads and writes.  
      You can configure [load balancing](../../../client-api/session/configuration/use-session-context-for-load-balancing)
      to fine-tune the settings to your needs.  
    * To distribute work, you can set different nodes to be responsible for different sets of data.  
      Learn more in the article [Scaling Distributed Work in RavenDB](https://ravendb.net/learn/inside-ravendb-book/reader/4.0/7-scaling-distributed-work-in-ravendb). 
{INFO/}

{PANEL/}

{PANEL: Transaction Scope for Compare-Exchange Operations}

Conventionally, compare-exchange **session operations** are used in [cluster-wide sessions](../../../client-api/session/cluster-transaction).  
Since RavenDB 5.2, we automatically create and maintain [Atomic Guards](../../../client-api/operations/compare-exchange/atomic-guards) 
to guarantee cluster-wide session ACID transactions.

This article is about non-session-specific compare-exchange operations.

* A non-session specific compare-exchange [operation](../../../client-api/operations/what-are-operations) (described below) 
  is performed on the [document store](../../../client-api/what-is-a-document-store) level.  
  It is therefore not part of the session transactions.  

{WARNING: If a session fails}

* Even if written inside the session scope, a non-session compare exchange **operation** will be executed regardless 
  of whether the session `SaveChanges( )` succeeds or fails. 
  * This is not the case when using compare-exchange [session methods](../../../client-api/session/cluster-transaction)

* Thus, upon a [session transaction failure](../../../client-api/session/what-is-a-session-and-how-does-it-work#batching), 
  if you had a successful compare-exchange operation (as described below) inside the failed session block, 
  it will **not** be rolled back automatically with the failed session.  
{WARNING/}

{PANEL/}

{PANEL: Creating a Key}

Provide the following when saving a **key**:

| Parameter | Description |
| ------------- | ---- |
| **Key** | A string under which _Value_ is saved, unique in the database scope across the cluster. This string can be up to 512 bytes. |
| **Value** | The Value that is associated with the _Key_. <br/>Can be a number, string, boolean, array, or any JSON formatted object. |
| **Index** | The _Index_ number is indicating the version of _Value_.<br/>The _Index_ is used for the concurrency control, similar to documents Etags. |

* When creating a _new_ 'Compare Exchange **Key**', the index should be set to `0`.  

* The [Put](../../../client-api/operations/compare-exchange/put-compare-exchange-value) operation will succeed only if this key doesn't exist yet.  

* Note: Two different keys _can_ have the same values as long as the keys are unique.  
{PANEL/}

{PANEL: Updating a Key}

Updating a compare exchange key can be divided into 2 phases:

  1. [Get](../../../client-api/operations/compare-exchange/get-compare-exchange-value) the existing _Key_. The associated _Value_ and _Index_ are received.  

  2. The _Index_ obtained from the read operation is provided to the [Put](../../../client-api/operations/compare-exchange/put-compare-exchange-value) operation along with the new _Value_ to be saved.  
     This save will succeed only if the index that is provided to the 'Put' operation is the **same** as the index that was received from the server in the previous 'Get', 
     which means that the _Value_ was not modified by someone else between the read and write operations.
{PANEL/}

{PANEL: Example I - Email Address Reservation}  

* Compare Exchange can be used to maintain uniqueness across users' email accounts.  

* First try to reserve a new user email.  
  If the email is successfully reserved then save the user account document.  

{CODE email@Server\CompareExchange.cs /}  

**Implications**:

* The `User` object is saved as a document, hence it can be indexed, queried, etc.  

* If `session.SaveChanges` fails, the email reservation is _not_ rolled back automatically. It is your responsibility to do so.  

* The compare exchange value that was saved can be accessed from `RQL` in a query:  

{CODE-TABS}
{CODE-TAB:csharp:Sync query_cmpxchg@Server\CompareExchange.cs /}  
{CODE-TAB-BLOCK:sql:RQL}
from Users as s where id() == cmpxchg("emails/ayende@ayende.com")  
{CODE-TAB-BLOCK/}
{CODE-TABS/}
{PANEL/}

{PANEL: Example II - Reserve a Shared Resource}  

* Use compare exchange for a shared resource reservation.  

* The code also checks for clients who never release resources (i.e. due to failure) by using timeout.  

{CODE shared_resource@Server\CompareExchange.cs /}
{PANEL/}

## Related Articles

### Client API

- [Get a Compare-Exchange Value](../../../client-api/operations/compare-exchange/get-compare-exchange-value)
- [Get Compare-Exchange Values](../../../client-api/operations/compare-exchange/get-compare-exchange-values)
- [Put a Compare-Exchange Value](../../../client-api/operations/compare-exchange/delete-compare-exchange-value)
- [Delete a Compare-Exchange Value](../../../client-api/operations/compare-exchange/delete-compare-exchange-value)
- [Atomic Guards](../../../client-api/operations/compare-exchange/atomic-guards)

### Studio

- [Compare Exchange View](../../../studio/database/documents/compare-exchange-view)  

---

## Code Walkthrough

- [Create CmpXchg Item](https://demo.ravendb.net/demos/csharp/compare-exchange/create-compare-exchange)  
- [Index CmpXchg Values](https://demo.ravendb.net/demos/csharp/compare-exchange/index-compare-exchange)  

