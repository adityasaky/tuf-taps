* TAP: 19
* Title: Content Addressable Systems and TUF
* Version: 1
* Last-Modified:
* Author: Aditya Sirish A Yelgundhalli, Renata Vaderna
* Type: Standardization
* Status: Draft
* Content-Type: markdown
* Created: 09/05/2022
* Requires: TAP-11
* +TUF-Version:
* +Post-History:

# Abstract

This TAP proposes extending TUF to content addressed applications and
ecosystems. These systems typically have non-file representations of objects
with specific hashing routines. This document describes how TUF implementations
can adopt these hashing routines and the properties these content addressed
systems must have to ensure their hash values are robust. Some popular
content addressed ecosystems or applications are Git, IPFS, and OSTree, while
these semantics are also visible elsewhere such as in containers.

# Motivation

The current TUF specification requires `targets` to be files--TUF is designed
around the assumption that the artifacts being distributed are "regular"
files. However, emerging applications and ecosystems that defy this assumption
can still greatly benefit from TUF's security properties.

Content addressed systems are those in which objects are addressed and accessed
by hashes of their content rather than by reference, such as their location. A
common paradigm used to achieve content addressability is to represent objects
via a Merkle Tree or Directed Acyclic Graph (DAG). A Merkle Tree is a hash-based
data structure where the leaf nodes are identified by the hash of their data and
non-leaf nodes are identified by the hash of their child nodes. A Merkle DAG is
similar, except non-leaf nodes can be associated with some data instead of just
the leaf nodes. Also, an instance of a Merkle DAG does not need to be balanced
and each node can have several parents. Essentially, Merkle DAGs offer more
freedom than typical Merkle Trees.

Both data structures are used in various applications today. Perhaps the most
ubiquitous is Git. They are also used in other ecosystems like the
Interplanetary File System (IPFS). In IPFS, files are addressed and accessed by
the hashes of their content rather than by reference, using their location. Any
file can exist at some specified location, but much stronger claims can be made
about a file identified by its hash.

These two systems are the driving use cases of this TAP and therefore the
specification uses Merkle DAG semantics to describe its changes. Further, from
the perspective of this TAP, there are no differences to how objects of these
data structures are handled. As a result, the terms "tree" and "DAG" are used
interchangeably throughout this document to refer to an instance of a Merkle
Tree or DAG. Additionally, "Merkle object" refers to a node in a Merkle Tree or
DAG.

## Use Case 1: Open Law Library's The Archive Framework

The Open Law Library is an open access publisher that makes laws freely
accessible to governments and their citizens. They build tools that help
governments with the drafting, codifying, and publishing aspects of the
legislative process.

The organization have developed a variant of TUF called The Archive Framework
(TAF), designed to support Git repositories as targets rather than regular
files. TAF uses a stand-in file representing each repository which records the
specific commit ID. This file is then used as a target in TUF metadata.

This particular use case can be generalized to supporting any Git repositories
as Targets.

## Use Case 2: IPFS as a Backend for Targets

IPFS builds on a variety of other technologies to provide a peer-to-peer
protocol that can store and transfer files. Files are broken up into multiple
blocks that are part of a Merkle DAG. Each file is identified by either the
root node when there are multiple blocks, or a single node that also contains
the data of the file.

Adding support for IPFS to TUF allows developers to distribute files stored on
IPFS as opposed to traditional servers and distributed via HTTP. This can be
achieved in two manners: by abstracting the delivery protocol, and by treating
IPFS nodes as targets rather than the files they represent.

TUF is already protocol agnostic, so merely having IPFS as an alternative
protocol for repository backends requires few or no changes to TUF metadata.
On the other hand, directly recording IPFS nodes brings it in line with other
attempts to record non-traditional, Merkle DAG targets such as TAF.

## Use Case 3: Distributing Artifacts Using OSTree

OSTree, or libostree, is a library and tool that applies a Git-like model for
entire, bootable filesystem trees. The project includes utilities for deploying
these images. It follows a similar principle as Git, using hash values to build
content addressability. OSTree is used by a variety of projects such as Flatpak
and Fedora CoreOS.

One key use of OSTree is for packaging and distribution operations. With
OSTree, a package manager can distribute an entire filesystem tree. In some
cases, such a tree can be the artifact itself, say an operating system image,
but OSTree also makes it possible to distribute multiple artifacts using a
single identifier--that of the root of the tree. The entire directory structure
uses a Merkle tree under the hood.

# Specification

The key differences between regular file targets and content addressable
objects such as Merkle DAG nodes are in how their hashes are computed and how
the TUF verification workflow applies to them. As such, the key focus of this
document is to articulate what is required to design a TUF implementation
capable of recording Merkle objects. This TAP considers two content addressed
systems that both use Merkle DAGs--Git and the Interplanetary Filesystem (IPFS).
These systems differ significantly in the type of data each Merkle node
represents. In Git, each node in the DAG represents a _commit_, or a record of
changes made, while in IPFS, each DAG node represents a file, or the root of a
tree of nodes that collectively represent a file. These systems are different
enough to ensure the contents of this TAP can apply to multiple types of Merkle
Tree or DAG systems not explicitly considered here.

Presently, each entry in TUF's targets metadata has two key parts--the
identification of the target, and the characteristics of the target.
Incorporating Merkle objects will require consideration to both of these
aspects, as well as to how they are handled during verification.

## Identifying the Target

Currently, file targets are identified by a path that is relative to the
repository's base URL. As discussed before, a Merkle DAG is a hash-based data
structure, so every node is associated with a hash value. Therefore, as the
identifier of each node is ecosystem specific, the strategy used to identify a
target node will vary accordingly.

In order to support different Merkle DAG ecosystems, this TAP proposes using
RFC 3986's URI structure for the target identifier. This has the following
structure.

```
<scheme>:<hier-part>
```

The `scheme` contains a token that uniquely identifies the Merkle DAG ecosystem
while `hier-part` contains the location or identifier of the specific target.

For example, every Git repository contains a Merkle DAG, in which every node is
a commit object, and each commit has a unique identifier generated using SHA-1.
So, when the Merkle DAG in question is that of a Git repository, the target
identifier may point to the repository as a whole or perhaps a specific branch
or tag within it. The details of a Git-specific implementation of this TAP
must be communicated using a POUF.

```
git:<repo identifier>
git:<repo identifier>?branch=<branch name>
git:<repo identifier>?tag=<tag name>
```

On the other hand, IPFS introduces the concept of locating arbitrary artifacts
by their content, rather than by a particular location. When a file is added to
IPFS, it is then available at an endpoint that uses the cryptographic hash of
its contents. In this instance, it makes sense to use this identifier in TUF
metadata.

```
ipfs:<node identifier>
```

It is important to note that a file can encompass multiple nodes in the IPFS
Merkle DAG, and in such situations, the identifier should be the root node
which points to the other nodes that make up the file.

As noted above, this TAP considers these ecosystems at a high level to
demonstrate the proposed changes. More detailed descriptions of how to record
Git or IPFS artifacts considering various use and edge cases must be published
as a POUF dedicated to each ecosystem.

## Recording the Characteristics of the Target

In the current TUF specification, each target entry has the following format:

```
{
   "length" : LENGTH,
   "hashes" : {ALG: HASH, ...},
   ("custom" : CUSTOM)
}
```

The opaque `custom` field requires no change to make this TAP possible.

The `length` field is an integer that captures the length in bytes of the
target. While this is straightforward for files, it can be more complicated to
define what the length of Merkle DAG objects are. This field may also be
entirely dropped if there is no clear value for a particular ecosystem. The
corresponding POUF must provide a clear direction for populating (or not) this
field.

The `hashes` field points to a dictionary object that captures the
cryptographic hashes of the target in one or more algorithms. While vital for
targets that are regular files, in the case of Merkle DAG objects, the
identifier self certifies the contents associated with the node. Thus, this
self certifying value can be used as the hash for the object.

In the case of Git, this field can contain the identifier of the commit at the
tip of the branch in question. When this repository is fetched by the client
during verification, receiving the commit with the specific identifier is akin
to receiving a file with a particular recorded hash. However, this requires a
degree of trust in the hash computation mechanism built into the Git
implementations used while recording the hash and verifying on the client.

For recording artifacts stored in IPFS, a similar approach, in which the
content identifier is computed by the system, can be used. This identifier is
not the same as the hash of the artifact itself, but rather identifies the root
node of the subgraph used to represent the artifact. When using this
identifier, it is also important to be aware of, and to account for, the
multibase representation used. Multibase, a protocol that can disambiguate the
encoding used for base-encoded text, is used by IPFS for its hash values. The
same hash value can have multiple distinct representations, depending on the
base. If an implementation of this TAP is directly using the values provided by
IPFS , it is important to note or choose a specific base or encoding to avoid
confusion in the future. The `custom` field can be used to communicate these
configuration choices for each object.

Also, it is important to remember that the "multihash" system used by IPFS is
“crypto-agile,” meaning its  content identifying system is not locked into one
cryptographic hash algorithm. As TUF is similarly designed, and does not
mandate a particular hash algorithm, its metadata structure allows for any
number of hashes to be recorded for every target. This can be leveraged when
multiple hash values exist for a particular target.

As before, the specific details associated with recording characteristics of an
artifact are left to the POUF detailing the implementation of the corresponding
ecosystem or application.

## Verifying the Target

The verification workflow also depends on the application or ecosystem to
validate hash values. In a well designed application or ecosystem, nodes with
invalid identifiers should not be allowed to exist without causing errors. Git
is an example of such an ecosystem. If a commit object no longer matches the
claimed hash value, the main Git implementations immediately flag the issue,
essentially halting all operations that can apply to the corrupted object.

In such an ecosystem, the ability for a node to legally exist in the system with
its identifier being recomputable for its data is equivalent to verifying a
given file has a particular hash value, as in the current TUF verification
workflow. An example of a robust application is in the
[appendix](#appendix-ideal-application-behaviour).

This delegates some trust to the implementation. In situations where this is
not ideal, the node hash can be calculated manually as part of the verification
process as well. Continuing with the example of Git, the hashes of all nodes
can be verified as part of the verification process. This is also demonstrated
in the [appendix](#appendix-ideal-application-behaviour).

# Rationale

This TAP proposes several changes to the general artifact recording process
currently employed in TUF.

## Use of URIs for Target Identification

The TAP updates the definition of a target identifier from being only a path
relative to a repository. It also allows URIs, which are a broadly understood
and widely accepted method, to point to different resources. Thus, URIs are an
ideal choice when an identifier must clearly specify the specific system of a
particular target, while also locating the object in question.

## Use of Self Certified Hash Values

A key change proposed in this TAP is the use of hash values calculated by
individual content addressed systems such Merkle DAG applications or ecosystems
rather than those generated by the developers using TUF. Yet, as discussed in
the security analysis, as long as the application is careful with its selection
of hash algorithms, the only critical element in this change is how the hash
values are used in the verification workflow. It is vital to always remember
that the hash calculation is no longer directly controlled by the developer. As
a consequence, audits of the mechanism must be regularly performed rather than
blindly trusting the application in question.

This may not always be possible--the application in question may not be open
source or not auditable for other reasons. In these situations, it is highly
recommended that the developers not take self certified hashes at face value.

It is also not necessary to take an application or ecosystem at face value.
Instead, the TUF implementation in use can be extended to interface with the
application, re-implementing the hashing mechanism used to record the target's
hashes. What this means is that the TUF implementation uses the application's
hashing mechanisms rather than reinventing the wheel. This ensures that for a
given cryptographic hash algorithm, there are not multiple values for a given
object.

This is not a concern for regular files because when computing their hashes,
the inputs can only be structured in one way--the files themselves. This is not
the case for more abstract object representations that exist in content
addressed systems such as those of Merkle DAG applications. It is possible to
use the characteristics of a Merkle DAG node in multiple ways when computing its
hash. In order to avoid confusion, this TAP specifies using the existing hashing
routine as long as it is robust and secure.

# Security Analysis

There are several considerations to be made when this TAP is applied in
practice.

## Auditing Hash Computation

Hash calculation is the biggest change proposed in this TAP. In TUF, the
developers distributing the targets control the hash algorithms used and the
actual computation. On the other hand, this TAP recommends using the node's
identifier, which is itself a hash, instead of recording a new hash. This
transfers the control to the _application_. Note that if the distributor also
controls the application, this is not a concern.

Yet, this conditional solution will not always be true. So, it is important to
consider several factors when choosing to record content addressed objects as
targets. As always, the algorithm or hashing routine used should result in
**unique** hashes for distinct objects. Two distinct objects should under no
circumstances share a hash value. Further, the hash value should be
**repeatable**. During verification, the hash values of the nodes are not
necessarily _explicitly_ calculated by the TUF client. Instead, the client
checks that a node exists in the respective system with the hash and expects the
system to detect if some other node is masquerading using the provided hash.
Therefore, the client should not recognize the same node with a different hash
when all other parameters such as the algorithm used are the same. A poorly
written implementation may compute one hash at the repository and expect a
different hash on the client, considering the original hash invalid on the
client. The systems considered in this TAP, Git and IPFS, do not have this
problem, but this is one hypothetical issue to be considered when evaluating
other ecosystems.

## Unavailable Resources

The availability of targets in a content addressed context is no different from
that of regular files. For the metadata to be signed, the specific object must
be available from the corresponding source. Similarly, verification of the
metadata is contingent on actually receiving the resource in question--here,
that takes the form of the specified nodes being present on the client
post-fetch.

# Adoption Considerations

While this TAP goes into detail about handling Git and IPFS, it should be
possible to apply the same techniques to other robust content addressable
systems. There are several factors to consider as an adopter looking to
implement this TAP.

## Applicability of the TAP

An important aspect of applying the ideas in this TAP is ensuring the target
system is indeed content addressable.  This TAP is **not**, for example,
generalizable to all version control systems (VCSs). Consider Subversion (SVN),
an alternative to Git. Like Git, SVN has a concept of recording changes which it
calls _revisions_. However, SVN **does not** use a Merkle DAG to store these
revisions. Instead, each revision is identified by an auto-incrementing integer,
one more than the previous revision. This identifier does not make any claims
about the specific changes in the revision. Indeed, the identifier is entirely
disconnected from the contents of the changes contained in the corresponding
revision, and using it as a self certified value of a revision as prescribed in
this document for Git is **dangerous**, entirely undermining the security
properties offered by TUF.

Implementers must be therefore very careful with adopting this TAP for a new
system. They must be familiar with the characteristic properties of content
addressable systems. If they are implementing this TAP for an existing system
they do directly control, they must thoroughly and regularly
[audit the hash computation](#auditing-hash-computation) mechanisms used by the
system.

## Registering a Scheme for New Applications

As noted [previously](#identifying-the-target), non file objects are identified
using URIs, where the `scheme` describes the specific ecosystem the target
belongs to. In order to avoid collisions for these values, adopters should
communicate any new applications they implement this TAP for to the broader TUF
community. Adopters should announce the new application and the identifier they
have selected for it via the forums used by the community such as the mailing
list, Slack channels, and the monthly community meetings. They can also seek the
community's feedback in assessing the ecosystem for the applicability of this
TAP.

Further, the adopters must communicate the details of the ecosystem via the
[POUF](https://github.com/theupdateframework/taps/blob/master/tap11.md) of
their TUF implementation. This way, not only is the scheme recorded, other
implementations can also support the ecosystem in an interoperable manner.

# Backwards Compatibility

This TAP has no direct impact on the TUF specification with respect to
backwards compatibility. However, existing implementations of TUF that choose
to use this TAP for one or more content addressed systems may end up with
metadata that is not compatible with other implementations that do not support
the selected ecosystems. In these situations, the POUFs for all the involved
implementations must be used to establish compatibility.

# Augmented Reference Implementation

None at the moment.

# Copyright

This document has been placed in the public domain.

# References

* [Merkle Trees](https://xlinux.nist.gov/dads/HTML/MerkleTree.html)
* [RFC 3986 - Uniform Resource Identifier (URI): Generic Syntax](https://tools.ietf.org/html/rfc3986)
* [Interplanetary Filesystem](https://ipfs.io/)
* [IPFS Merkle DAGs](https://docs.ipfs.io/concepts/merkle-dag/)
* [IPFS Content Addressing](https://docs.ipfs.io/concepts/content-addressing/)
* [IPFS Hashing](https://docs.ipfs.io/concepts/hashing/)
* [Multibase](https://github.com/multiformats/multibase)
* [Multihash](https://github.com/multiformats/multihash)

# Appendix: Ideal Application Behaviour

For example, consider what happens when a Git commit is manually overwritten
with different information.

```bash
$ cat .git/object/65/774be295aaf5ac9412ebe81584138643ebded2 | zlib-flate -uncompress
commit 727tree b4d01e9b0c4a9356736dfddf8830ba9a54f5271c
author Aditya Sirish <aditya@saky.in> 1654557334 -0400
committer Aditya Sirish <aditya@saky.in> 1654557334 -0400
gpgsig -----BEGIN PGP SIGNATURE-----

 iQEzBAABCAAdFiEE4ylBKZy4wNk9zyesuDEQ0BJUVgQFAmKeipYACgkQuDEQ0BJU
 VgSmUgf9FSwk2VVPn0vWmFzx6x5JdT9CQ3Tl9cqxug0/Zu8xfesQlMgpcpDDMHSf
 ZdmGfYaLb7aqSL0jE+pwytAfhGN4xwegqS4/YrzqnPZPjxtj5JlwBVtdMtsYRVHN
 QvsDBZEYYd/MFGqSyVkJwFAH9idRwdki8wQ/JwtbAf0QIkqWdIORckh75V7VxX1r
 Rv5jU9luU60NbEzAHa/W3xvfKVgaA4a1VjmS7ATOrAS4maNi+VzXjBnvhmR4z7zS
 FF4N3QkZ8XwHMu/uuldTq2mB4/uJ/BXP5TNZULn7sbYHKMXrH4ZscqDFplRMeah/
 XxcVTwUVn2zHdmOMf7xw6goFszPaDg==
 =lcDr
 -----END PGP SIGNATURE-----

Initial commit

Signed-off-by: Aditya Sirish <aditya@saky.in>
$ cat .git/object/65/774be295aaf5ac9412ebe81584138643ebded2 | zlib-flate -uncompress | sha1sum
65774be295aaf5ac9412ebe81584138643ebded2  -  # this matches the commit ID
$ cp .git/object/65/774be295aaf5ac9412ebe81584138643ebded2{,.valid}  # copy of the original commit object
```

As seen above, the commit IDs are merely the SHA-1 hash of the contents of the
commit object. Now, if we replace the original commit object with a new one
that is not exactly the same, Git shows an error.

```bash
$ ls -l .git/object/65
-rw-r--r-- 1 saky users 143 Jun  6 19:20 774be295aaf5ac9412ebe81584138643ebded2
-r--r--r-- 1 saky users 532 Jun  6 19:15 774be295aaf5ac9412ebe81584138643ebded2.valid
$ cat .git/object/65/774be295aaf5ac9412ebe81584138643ebded2 | zlib-flate -uncompress
commit 222tree b4d01e9b0c4a9356736dfddf8830ba9a54f5271c
author Aditya Sirish <aditya@saky.in> 1654557334 -0400
committer Aditya Sirish <aditya@saky.in> 1654557334 -0400

Initial commit

Signed-off-by: Aditya Sirish <aditya@saky.in>
```

In this situation, we have replaced the commit object with one without a GPG
signature. This commit object should in fact have a different ID.

```bash
$ cat .git/object/65/774be295aaf5ac9412ebe81584138643ebded2 | zlib-flate -uncompress | sha1sum
2c65b30ada8c10d9ec028359ca82d7a410067ed4  -  # this does not match the commit ID
$ git show
error: hash mismatch 65774be295aaf5ac9412ebe81584138643ebded2
fatal: bad object HEAD
```

Note that these checks are not performed every time. As Git repositories grow,
these checks become expensive. In this case, Git detected the issue because the
affected object was the HEAD to be used as the parent for the new commit. If the
commit was earlier in the graph, the replacement would not have been detected.
However, checks can be explicitly invoked using the `git fsck` tool.

While this may seem like less than ideal, Git ensures that any time an object is
used in a significant operation, its hash is checked against the contents. As
such, attempting to `git checkout` a commit that has been tampered with, for
example, will result in an error. On the other hand, viewing it via `git log`,
`git show`, and `git cat-file` will not result in an error.
