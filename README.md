## Concept ##

A "registration" is a "frozen" version of a project. The state of a project will typically consist both entries in a database and files.

The service that will eventually handle registrations for the OSF:

* Clone metadata
* Archive files from third party services
* Create append-only chain for verification

Not handled here:

* Curating the resulting file snapshots to maintain redundancy.
* Migrating the resulting block chain to new hash algorithms over time

This library is an abstraction layer that orchestrates the workflow necessary to create registrations.

## Assumptions ##

1. All storage is ultimately mutable.
2. All storage eventually fails.
3. Information technology will continue to make exponential improvements in effective speed, storage space, and algorithmic prowess.

## Problem ##

How to create a "frozen" version a piece of information that resists:

* the deterioration of storage
* attempts by bad actors (attackers) to present an altered version of frozen information so that it would appear authentic

## Strategy ##

* Creation
    1. Give every piece of information a unique identifier.
    2. Create secure hashes of every piece of information.
    3. Copy the information to redundant repositories.
    4. Add the association between the unique identifiers and the secure hashes to a publicly visible redundant journal.
* Curation
    1. Update the secure hashes over time as technology evolves, retaining both the old and new hashes.
    2. Create additional copies of the information over time as existing storage systems deteriorate.

Currently, this library orchestrates the creation workflow, not the curation workflow.

## Details ##

The registration system uses UUIDs to reference objects and cryptographic hashes verify (authenticate) them. This system is blind to the existence of any other identifiers, such as short URL codes. Any other identifiers that must be persisted should be stored within the pieces of information presented to this system.

When a registration occurs, it is necessary to create create a serialized object that represents the registration along with other serialized objects for any wiki pages or other nested metadata. Typically the objects are serialized to JSON, but any format will work.

For textual representations, client code may pass in format strings. In such cases the number of slots in the format string should equal the number of direct children. This system will insert the UUIDs in order of the children. Currently the `format` method is used.

Whether formatted or not, any text data must be encoded to raw bytes. The default encoding is UTF-8.

In API documentation, we avoid the term "str" due to its ambiguous meaning between Python 2 and Python 3. Instead we use the terms:

* for textual information: "unicode" and "text"
* for non-textual information: "bytes", "data", and "binary"

A compatibility module will define the names `unicode` and `bytes` within the package.

Object Model:

* `Persistable`, abstract
    * `shallow_hash()` - `int`, abstract, for detecting reference cycles
    * `payload_hash()` - `bytes`, abstract, cryptographic, delegated to config option
    * `children` - sequence of Persistable, default=`tuple()`
    * `uuid` - `None` or a `uuid.UUID`
* `PayloadPersistable`, for things like JSON, where payload is small
    * `payload` - bytes or unicode
    * `interpolate` - `False` or `True`
    * `encoding` - default = `'utf_8'`
    * `payload_hash()` - hash of payload
* `URLPersistable`, for things with large payload that must be fetched
    * `fetch()` - blocking, populates `temp_file`
    * `payload_hash()` - hash of `temp_file` contents
    * `temp_file`

TODO: When filling UUIDs into templates, should we use the term "format", "interpolate", or "render"?

### Primary entry point ###

TODO:

* immediate return
* async recursive fetching
* block chain decoupling question

`create_registration(root_object)` - non-blocking

Input:

* the `Persistable` representing the registration, which recursively contains all internal objects for the project.

Output:

* the UUID generated for this registration.

Raises various exceptions of an error occurs during UUID generation, text interpolation, text encoding, or queuing.

Steps:

1. Generate the UUIDs.
2. Perform any text interpolations.
3. Encode any text.
4. Queue the registration object.
5. Return the UUID of the registration object.

Steps for a registration worker:

1. Recursively fetch all internal objects for the project.
2. Create UUID links between everything.
3. Bundle the git repo.
4. Begin a block chain transaction.
5. Queue all external objects for fetch and persist.
6. Persist the bundle of the git repo.
7. Persist the internal objects.
8. Persist the registration object.
9. Commit the block chain transaction.

Steps for a persistence worker:

1. Securely hash the object.
2. Check the hash against an index mapping hash -> UUID.
    1. If found, return the UUID.
    2. Else, continue...
3. Generate a UUID.
4. Store object in "permanent storage" based on size and type.
5. Add the UUID, hash, type, size, and name to the current block chain transaction.

TODO: Storing the metadata about the objects such as the type, size, and name in the block chain is probably a bad idea. There should probably be a table of contents object. That would clean up the block chain to just be a mapping between UUIDs and secure hashes. On the other hand, Removing the size and type from the block chain gives an attacker more freedom when attempting to create a forgery.

## Dummy Block Chain Implementation ##

A transaction is just the next CSV file in a numerical sequence. There is no protection against forgery. This should be replaced with a proper cryptographic system using strong hashes and peer-to-peer.

Format:

* UUID
* hash
* type, examples
    * binary file
    * registration/json/v1 - schema version 1
* size
* name

transaction-1.csv:

    ABCD,1234,registration/utf-8/json/v1,1300,Project X
    BCDE,2345,file,2000,my_file.csv
    ...

transaction-2.csv:

    CDEF,3456,registration/json/v1,1122,Project Y
    DEF0,2345,wiki,Hypothesis
    ...

The current head of history is just the most recent transaction.

TODO: We **MUST** replace this with a secure block chain that will survive the evolution of crypto through hash migrations.

---

TODO: There should be a way for a project to reference a large immutable file by UUID. The most common use case is for a scientist to replace a mutable reference to Dropbox or S3 with a corresponding immutable reference that was just generated during registration. This use case should have an optimized workflow allowing the user to do the replacement by selecting an option and clicking checkboxes.

TODO: There should be a way for a project to reference a large immutable file that is never actually fetched by some external ID or URI.

TODO: Think about what happens when Big Data becomes small. Over time, people may wish to fetch some files that were previously described as un-fetched links.

Started by @walkerh at PyCon2014
