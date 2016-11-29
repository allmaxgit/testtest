LOVII Onyx follows very closely to the CQRS pattern. Commands are put on the queue. Processors process the commands. These processors create events. The events are put back on the queue. The events then are processed by processors. These processors create commands, so the cycle continues until there is no work left to do.
Commands are generally (99.9%) transactions to bring about a new state in Datomic. The corresponding events are Notifications of entites being created, changed or retracted. This means that Datomic is the state management for distributed event processors, holding the state for where each workflow is up to.
```
                  Queue (SQS)
                       |
                       |
                       v
                Dispatcher (Onyx)
                       |
                       |
         ----------------------------
         |                          |
         |                          |
         v                          v
 Command Processors          Event Processors
         |                          |
         |                          |
         v                          v
   Events (Recur To Queue)    Commands (Recur To Queue)
```

##Example Workflow - sending a message via SMS/MMS/Email
1) :template-usages transacted.
2) :template-usages/uuid entity created event triggers a function that creates many :template-activities with pending status.
3) :template-activities/uuid entity created event triggers a function to side effect the transport of the SMS/MMS/Email then creates commands to update the status with the status of the transport.

##Features and Benefits of this approach:
- Each time we progress around through the queue we have an opportunity to break the work we have to do into smaller pieces. This allows us to queue work to be done and distribute our problems.
- As each step becomes an almost pure function (pure except the functions that must side-effect with the outside world like SMS), testing our workflow functions becomes much easier.
- As our workflows are defined in terms of events (usually data) and transaction commands; changes to data over time; the interfaces to our workflow functions are defined in terms of our data schema.

##Schema
Data schema plays a central role in the architecture, as such it is important to have a clearly defined schema. We are writing a multi tenanted system where some tenants share data with other tenants, these sharing rules can be complex and often predicated on the data, as such we have built the security and multi-tenancy rules to be defined on the structure of the data. This separates the data from it's security requirements as much as possible. More on this will be covered in the Security Abstraction section.
The schema is defined in an EDN file. This means that all the security rules, data structure rules, and validation is defined in data allowing us to serialize the schema file and transport it over the wire. This in turn allows api clients to consume the schema and build data models dynamically with all the validation logic delivered up front. We intend in the future to develop a clojurescript library with some js externs to allow client apps to easily validate their data structures easily.
Please read the lovii-schema readme.md on github for more detail.

##Security Model
Our security abstraction is defined using the extensibility of lovii-schema by defining the security rules as properties of attributes and entities (abstracts and variants).
Conceptually the security model is based on data relationship to a target security entity (generally the user). Dependent on the paths from the entity in question to the security entity, different security groups may be applied. The resulting security groups will then have permissions associated with them for that entity. (At the moment the naming is confusing and needs a bit of thought to get right.)
Specifically, an entity can assign a security group to it's identity if a predicate defining graph relationship to the security entity is true. Note, the predicate syntax may change dependent on the underlying storage system. The security groups propagate down graph edges with predicate guards to stop propagation if the predicate doesn't match. Again the predicate syntax changes dependent on the underlying storage.
As the security groups only propagate down the edges and all edges are directional, we can resolve the security groups by taking the union of an identities self assigned security groups and the security groups propagated to the entity by all it's parents. As each parent's security groups are resolved the same way, this is a recursive operation until the top of the graph is reached. As this is a pure operation, memoization can be used to improve performance. In the case of mutual security group propagation; where entity A propagates to entity B and entity B propagates to entity A where they may also be more edges in between A and B; we can detect mutual propagation by tracking the propagation path to the entity when resolving what security groups propagate to another entity, if the same entity is already present in the propagation path, then we can safely return an empty set of security groups as this indicates that the operation won't yield any more security groups.
This model doesn't prescribe what the security groups are or what permissions they resolve to on an entity.
This security model is implemented via the query abstraction.

##Query Abstraction
Query is built on the Om.Next query parser. This gives us the ability to implement a thin wrapper over the datomic pull api. However given the security constraints the query abstraction is implemented by walking the datomic entity tree.
When a security entity and a set of required permissions set is presented. The query engine will filter out entities that don't have the required permissions.
The Om.Next query parser also allow for easy extensibility to define derived data; such as security groups and permissions; we use multi methods dispatched on the attribute to provide a nice api. The default behaviour is driven by information detailed in the lovii-schema document. This allows us to define in one place all the required information for different data types.
As this is an abstraction layer, we can then provide query interface into other data sources. Giving us the ability to expose third party rest api's data as extensions of our internal data sets. This gives consumers of our api a unified way of accessing information.
The query abstraction also provides a thin layer over the top of the datomic datalog logic engine. It is implemented so we can provide a mechanism for filtering collections of data by use of the datalog DSL. At the moment only a subset of the datomic datalog query language is supported. This abstraction takes a datalog query and transforms the syntax for datom relationship binding into a graph walking implementation, hooking into the existing query implementation. This means that the datomic query engine now is aware of our security model, at the expense of performance.

##Transact Abstraction
NOTE: This is an early stage abstraction and may be subject to change as more elements of transact are built out.
This is a thin wrapper over the datomic transaction DSL, this is done precisely as we are using datomic under the hood.
The job of this wrapper is to provide security when transacting. Security is implemented in terms of Query; querying for entities with a writing context and ensuring that every entity touched passes all validation and structure checks before transacting. 
Transact is also responsible for moving files when uploading to our permanent storage when transacting file data.

##QUESTIONS
Why did we choose the CQRS pattern?
- Many aspect of our business logic are distributed / scaling background processes. Eg SMS/MMS/Email/Letters/Portal Upload.
- These distributed background processes all require queue and scaling architecture.

Why did we choose Datomic?
- Datomic has the fantastic ability to allow us to co-ordinate processes for read. By passing around the basis-t we can ensure all events are looking at the same point in time for query.
- Datomic also gives us the ability to look back in time and see how information changes over time. This allows us to say yes to our customers changing business requirements.

Why did we choose to write our own schema?
- We wanted a schema DSL that was data driven and extensible with arbitrary data.
- Of all the schema libraries available at the time of development. Only the datomic schema DSL was data driven. However the datomic schema DSL is very verbose and not very extensible.
- We also wanted to define stricter constraints on our data structure then what the datomic schema DSL provided.
- We almost went with a wrapped version of Prismatic Schema, however it isn't easily serialized as its API is function composition.