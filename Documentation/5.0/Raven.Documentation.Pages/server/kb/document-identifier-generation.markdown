# Knowledge Base: Document Identifier Generation
---

{NOTE: }

* A document identifier (**document ID**) is a string that is associated with the document.  
  It is globally unique in the scope of the database. No two documents in the same database will have the same ID.  

* Usually, the document ID will be composed of the collection name as the prefix, a slash and then the actual unique portion of the ID. 
  But this is not a requirement, RavenDB doesn't require that the collection prefix will be included within the document ID string.  

* RavenDB server supports six document ID generation strategies:  

  * Generated by the _user_:
      * **1 - Semantic ID**  

  * Generated by the _server_:  
      * **2 - Guid**  
      * **3 - Server-Side ID**  
      * **4 - Identity**  

  * Generated by cooperation of the _server_ & the _client_:  
      * **5 -Hilo Algorithm**  

  * Generated from a _Map-Reduce index output_:
      * **6 - Artificial Document ID**  

* The description below refers to the _server-side ID generation behaviour_ when saving a document by REST API.  
  For creating a document from the **Client API** see [Working with Document Identifiers](../../client-api/document-identifiers/working-with-document-identifiers)  
  For creating a document from the **Studio** see [Create New Document](../../studio/database/documents/create-new-document#create-new-document)  

{INFO: Identity Parts Separator}
By default, document IDs created by the server use the character / to separate their components. 
This separator can be changed to any other character except | in the 
[Document Store Conventions](../../client-api/configuration/conventions#changing-the-identity-separator).  
{INFO/}

* In this page:  
  * [Semantic ID](../../server/kb/document-identifier-generation#semantic-id)  
  * [GUID](../../server/kb/document-identifier-generation#guid)  
  * [Server-Side ID](../../server/kb/document-identifier-generation#server-side-id)  
  * [Identity](../../server/kb/document-identifier-generation#identity)  
  * [HiLo Algorithm](../../server/kb/document-identifier-generation#hilo-algorithm)  
  * [Artificial Document ID](../../server/kb/document-identifier-generation#artificial-document-id)  
  * [Document IDs - Limitations](../../server/kb/document-identifier-generation#document-ids---limitations)  
{NOTE/}

---

{PANEL: Semantic ID}

* **Generated by**:  
  The user  

* **Description**:  
  * The **semantic ID** is generated by _the user_ (using the Client API or from the Studio) and not by RavenDB server.  
    As such, it is the user responsibility to generate unique ID's.  
  * Creating a new document with an existing semantic ID will _overwrite_ the existing document.  

* **When to use**:  
  Use a semantic ID when you want the document to have an identifier that has some meaningful value.  

* **Example**:  
  * Documents with a unique semantic ID containing a user's _email_ can be generated under the 'Users' collection:  
      * users/ayende@ayende.com  
      * users/john@john.doe  
  * For clarity, the document content can be indicated within the Semantic ID string:  
      * accounts/591-192/txs/2017-05-17  
        Implying that the document holds all the transactions from May 17th, 2017 for account 591-192
{PANEL/}

{PANEL: Guid}

* **Generated by**:  
  The server  

* **Description**:  
  * When a document ID is _not_ specified, RavenDB server will generate a **globally unique identifier** (GUID) for the stored new document.
  * Although this is the simplest way to generate a document ID, Guids are _not_ human-friendly when it comes to debugging or troubleshooting 
    and are less recommended.  

* **When to use**:  
  Only When you don't care about the exact ID generated and about the ease of troubleshooting your app...
{PANEL/}

{PANEL: Server-Side ID}

* **Generated by**:  
  The server  

* **Description**:  
  * Upon document creation, providing a document ID string that ends with a _slash_ ( / ) will cause the server to generate a **server-side ID**.  
  * The RavenDB server that is handling the request will increment the value of its [Last Document Etag](../../glossary/etag).  
    This _Etag_ and the _Server Node Tag_ are appended by the server to the end of the ID string provided.  
  * Since the etag on which the ID is based changes upon any adding, deleting, or updating a document,  
    the only guarantee about the Server-Side ID is that it is always increasing, but not always sequential.  

* **When to use**:  
  * Use the server-side ID when you don't care about the exact ID that is given to a newly created document.  
  * Recommended when a large number of documents are needed to be created, such as in bulk insert scenarios,  
    as this method requires the least amount of work from RavenDB.  

* **Example**:  
  * From a server running on node 'A':  
      * Creating the first document with 'users/' => will result with document ID: _'users/0000000000000000001-A'_  
      * Creating a second document with 'users/' => will result with document ID: _'users/0000000000000000002-A'_  
  * From a server running on node 'B':  
      * Creating a third document with 'users/' => can result for example with document ID: _'users/0000000000000000034-B'_  
      * Note: node tag 'B' was appended to the ID generated, as the server handling the request is on node 'B'.  
        But, since each server has its own local Etag, the numeric part in the ID will _not_ necessarily be sequential (or unique) across the nodes  
        within the database group in the cluster, as can happen when creating documents at partition time.  

* **Note**:  
  If you _manually_ generate a document ID with a pattern that matches the server-side generated IDs,  
  RavenDB will _not_ check for that and will overwrite the existing document.  
  The leading zeros help avoid such conflicts with any existing document by accident.
{PANEL/}

{PANEL: Identity}

* **Generated by**:  
  The server  

* **Description**:  
  *  Upon document creation, providing a document ID string that ends with a _pipe symbol_ ( | ) will cause the server to generate an **identity**.  
  *  RavenDb will create a simple, always-incrementing value and append it to the ID string provided (replacing the pipe with a slash).  
  *  As opposed to the Server-Side ID, This value _will be unique_ across all the nodes within the Database Group in the cluster.  

* **When to use**:  
  Use an identity only if you really need documents with incremental IDs,  
  i.e. when generating invoices, or upon legal obligation.  
    {NOTE: }
     Using an identity guarantees that IDs will be incremental, but does **not** guarantee 
     that there wouldn't be gaps in the sequence.  
     The IDs sequence can therefore be, for example, `companies/1`, `companies/2`, `companies/4`..  
     This is because -  

      *  Documents could have been deleted.  
      *  A failed transaction still increments the identity value, thus causing a gap in the sequence.  

    {NOTE/}

* **Example**:  
  * From a server running on node 'A':  
      * Creating the first document with 'users|' => will result with document ID: _'users/1'_  
      * Creating a second document with 'users|' => will result with document ID: _'users/2'_  
  * From a server running on node 'B':  
      * Creating a third document with 'users|' => will result with document ID: _'users/3'_  

* **Note**:
  * Identity has a real cost associated with it.  
    Generating identities in a cluster where the database is replicated across more than one node requires a lot of work.  
  * Network round trips are required as the nodes must coordinate with one another so that the same identity is _not_ generated on 2 different nodes in the cluster.  
  * Moreover, upon a failure scenario, if the node cannot communicate with the other cluster members, or the majority of nodes cannot be communicated with,
    saving the document will fail as the requested identity cannot be generated.  
  * All the other ID generation methods can work without any issue when the server is disconnected from the cluster,  
    so unless you truly need incremental IDs, use one of the other options.
{PANEL/}

{PANEL: HiLo Algorithm}

* **Generated by**:  
  Both the server and the client  

* **Description**:  
  * The Hilo algorithm enables generating document IDs on the client.  
  * The client reserves a range of identifiers from the server and the server ensures that this range will be provided only to this client.  
  * Different clients will receive different ranges.  
  * Each client can then safely generate identifiers within the range it was given, no further coordination with the server is required.  
  * For a more detailed explanation see [HiLo Algorithm](../../client-api/document-identifiers/hilo-algorithm)  

* **When to use**:  
  When you want to write code on the client that will create a new document and use its ID immediately in the same transaction, 
  without making another call to the server to get the ID and use it in a separate transaction.  

* **Example**:  
  people/128-A, people/129-B
{PANEL/}

{PANEL: Artificial Document ID}

* **Generated by**:  
  The server  

* **Description**:  
  * The output from a Map-Reduce index can be saved as Artificial Documents in a new collection.  
  * You have no control over these artificial documetns IDs, they are generated by RavenDB based on the hash of the reduce key.  
  * For a more detailed explanation see [Artificial Documents](../../indexes/map-reduce-indexes#reduce-results-as-artificial-documents).

* **When to use**:  
  When you you need to further process the Map-Reduce index results by:
    * Having a recursive Map-Reduce index operation, setting up indexes on top of the Artificial Documents.  
    * Setting up a RavenDB ETL Task on the Artificial Documents collection to a dedicated database on 
      a separate cluster for further processing, as well as other ongoing tasks such as: SQL ETL and Subscriptions.

* **Example**:  
  MonthlyProductSales/13770576973199715021
{PANEL/}

{PANEL: Document IDs - Limitations}

There are following limitations for document IDs:  

* The identifier length limit is 512 bytes (in UTF8)  
* The identifier cannot contain `\` character
* You can't store identifier ending with `/` or `|` characters (reserved for Server-Side ID and Identity generation strategies)

{PANEL/}

## Related Articles

### Document Identifiers

- [Working with Document Identifiers](../../client-api/document-identifiers/working-with-document-identifiers)
- [HiLo Algorithm](../../client-api/document-identifiers/hilo-algorithm)

### Studio

- [Create New Document From Studio](../../studio/database/documents/create-new-document)
- [Artificial Documents](../../studio/database/indexes/create-map-reduce-index#saving-map-reduce-results-in-a-collection-(artificial-documents))
