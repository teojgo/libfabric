---
layout: page
title: fi_peer(3)
tagline: Libfabric Programmer's Manual
---
{% include JB/setup %}

# NAME

fi_export_fid / fi_import_fid
: Share a fabric object between different providers or resources

struct fid_peer_cq
: A completion queue that may be shared between independent providers

# SYNOPSIS

```c
#include <rdma/fabric.h>
#include <rdma/fi_ext.h>

int fi_export_fid(struct fid *fid, uint64_t flags,
    struct fid **expfid, void *context);

int fi_import_fid(struct fid *fid, struct fid *expfid, uint64_t flags);

struct fid_peer_cq {
    struct fid fid;
    struct fi_ops_cq_write *export_ops;
    struct fi_ops_cq_progress *import_ops;
};
```

# ARGUMENTS

*fid*
: Returned fabric identifier for opened object.

*expfid*
: Exported fabric object that may be shared with another provider.

*flags*
: Control flags for the operation.

*context:
: User defined context that will be associated with a fabric object.

# DESCRIPTION

Functions defined in this man page are typically used by providers to
communicate with other providers, known as peer providers, or by other
libraries to communicate with the libfabric core, known as peer libraries.
Most middleware and applications should not need to access this functionality,
as the documentation mainly targets provider developers.

Peer providers are a way for independently developed providers to be used
together in a tight fashion, such that layering overhead and duplicate
provider functionality can be avoided.  Peer providers are linked by having
one provider export specific functionality to another.  This is done by
having one provider export a sharable fabric object (fid), which is imported
by one or more peer providers.

As an example, a provider which uses TCP to communicate with remote peers
may wish to use the shared memory provider to communicate with local peers.
To remove layering overhead, the TCP based provider may export its completion
queue and shared receive context and import those into the shared memory
provider.

The general mechanisms used to share fabric objects between peer providers are
similar, independent from the object being shared.  However, because the goal
of using peer providers is to avoid overhead, providers must be explicitly
written to support the peer provider mechanisms.

# PEER CQ

The peer CQ defines a mechanism by which a peer provider may insert completions
into the CQ owned by another provider.  This avoids the overhead of the
libfabric user from needing to access multiple CQs.

To setup a peer CQ, a provider creates a fid_peer_cq object, which links
back to the provider's actual fid_cq.  The fid_peer_cq object is then
imported by a peer provider.  The fid_peer_cq defines callbacks that the
providers use to communicate with each other.  The provider that allocates
the fid_peer_cq is known as the owner, with the other provider referred to
as the peer.  An owner may setup peer relationships with multiple providers.

Peer CQs are configured by the owner calling the peer's fi_cq_open() call.
The owner passes in the FI_PEER_IMPORT flag to fi_cq_open().  When
FI_PEER_IMPORT is specified, the context parameter passed
into fi_cq_open() must reference a struct fi_peer_cq_context.  Providers that
do not support peer CQs must fail the fi_cq_open() call with -FI_EINVAL
(indicating an invalid flag).  The fid_peer_cq referenced by struct
fi_peer_cq_context must remain valid until the peer's CQ is closed.

The data structures to support peer CQs are defined as follows:

```c
struct fi_ops_cq_owner {
    size_t  size;
    ssize_t (*write)(struct fid_peer_cq *cq, void *context, uint64_t flags,
        size_t len, void *buf, uint64_t data, uint64_t tag, fi_addr_t src);
    ssize_t (*writeerr)(struct fid_peer_cq *cq,
        const struct fi_cq_err_entry *err_entry);
};

struct fi_ops_cq_peer {
    size_t size;
    void (*progress)(struct fid_peer_cq *cq);
};

struct fid_peer_cq {
    struct fid fid;
    struct fi_ops_cq_owner *owner_ops;
    struct fi_ops_cq_peer *peer_ops;
};

struct fi_peer_cq_context {
    size_t size;
    struct fid_peer_cq *cq;
};
```
For struct fid_peer_cq, the owner initializes the fid and owner_ops
fields.  struct fi_ops_cq_owner is used by the peer to communicate
with the owning provider.

The size field in struct fi_ops_cq_peer is filled by the owner
and used as a versioning check by the peer.  However, the function
handlers are filled out by the peer and must be initialized before
returning from fi_cq_open().  These calls are used by the owner to
communicate with the peer.

Because the peer's CQ should not be directly accessed, functions to
read the peer's CQ (i.e. read, readerr, sread, etc.) should be set
to return -FI_ENOSYS.

## fi_ops_cq_owner::write()

This call directs the owner to insert new completions into the CQ.
The fi_cq_attr::format field, along with other related attributes,
determines which input parameters are valid.  Parameters that are not
reported as part of a completion are ignored by the owner, and should
be set to 0, NULL, or other appropriate value by the user.  For example,
if source addressing is not returned with a completion, then the src
parameter should be set to FI_ADDR_NOTAVAIL and ignored on input.

The owner is responsible for locking, event signaling, and handling CQ
overflow.  Data passed through the write callback is relative to the
user.  For example, the fi_addr_t is relative to the peer's AV.
The owner is responsible for converting the address if source
addressing is needed.

(TBD: should CQ overflow push back to the user for flow control?
Do we need backoff / resume callbacks in ops_cq_user?)

## fi_ops_cq_owner::writeerr()

The behavior of this call is similar to the write() ops.  It inserts
a completion indicating that a data transfer has failed into the CQ.

## fi_ops_cq_peer::progress()

This callback is invoked by the owner to drive progress in the peer
provider on endpoints owned by the peer.  The owner must support the
thread calling the progress() function invoking one of the owner
ops from within the progress() function.  For example, the peer calling
write() to insert a new completion.

## EXAMPLE PEER CQ SETUP

The above description defines the generic mechanism for sharing CQs
between providers.  This section outlines one possible implementation
to demonstrate the use of the APIs.  In the example, provider A
uses provider B as a peer for data transfers targeting endpoints
on the local node.

```
1. Provider A is configured to use provider B as a peer.  This may be coded
   into provider A or set through an environment variable.
2. The application calls:
   fi_cq_open(domain_a, attr, &cq_a, app_context)
3. Provider A allocates cq_a and automatically configures it to be used
   as a peer cq.
4. Provider A takes these steps:
   allocate peer_cq and reference cq_a
   set peer_cq_context->cq = peer_cq
   set attr_b.flags |= FI_PEER_IMPORT
   fi_cq_open(domain_b, attr_b, &cq_b, peer_cq_context)
5. Provider B allocates a cq, but configures it such that all completions
   are written to the peer_cq.  The cq ops to read from the cq are
   set to enosys calls.
8. Provider B inserts its own callbacks into the peer_cq object.  It
   creates a reference between the peer_cq object and its own cq.
```

# PEER SRX

The peer SRX defines a mechanism by which peer providers may share a common
shared receive context.  This avoids the overhead of having separate receive
queues, can eliminate memory copies, and ensures correct application level
message ordering.

The setup of a peer SRX is similar to the setup for a peer CQ outlined above.
A fid_peer_srx object links the owner of the SRX with the peer provider.
Peer SRXs are configured by the owner calling the peer's fi_srx_context()
call with the FI_IMPORT_PEER flag set.  The context parameter passed to
fi_srx_context() must be a struct fi_peer_srx_context.

The data structures to support peer SRXs are defined as follows:

```
struct fid_peer_srx;

/* Castable to dlist_entry */
struct fi_peer_rx_entry {
    struct fi_peer_rx_entry *next;
    struct fi_peer_rx_entry *prev;
    fi_addr_t addr;
    size_t size;
    uint64_t tag;
    void *context;
    size_t count;
    void **desc;
    struct iovec iov[];
};

struct fi_ops_srx_owner {
    size_t size;
    int (*get_msg_entry)(struct fid_peer_srx *srx,
        struct fi_peer_rx_entry *entry);
    int (*get_tag_entry)(struct fid_peer_srx *srx,
        struct fi_peer_rx_entry *entry);
};

struct fi_ops_srx_peer {
    size_t size;
    int (*start_msg)(struct fid_peer_srx *srx,
        struct fi_peer_rx_entry *entry);
    int (*start_tag)(struct fid_peer_srx *srx,
        struct fi_peer_rx_entry *entry);
    int (*discard_msg)(struct fid_peer_srx *srx,
        struct fi_peer_rx_entry *entry);
    int (*discard_tag)(struct fid_peer_srx *srx,
        struct fi_peer_rx_entry *entry);
};

struct fid_peer_srx {
    struct fid fid;
    struct fi_ops_srx_owner *owner_ops;
    struct fi_ops_srx_peer *peer_ops;
};

struct fi_peer_srx_context {
    size_t size;
    struct fid_peer_srx *srx;
};
```
The ownership of structure field values and callback functions is similar
to those defined for peer CQs, relative to owner versus peer ops.

## fi_ops_srx_owner::get_msg_entry() / get_tag_entry()

These calls are invoked by the peer provider to obtain the receive buffer(s)
where an incoming message should be placed.  The fi_peer_rx_entry is allocated
by the peer provider and initialized with information about the received
message.  If source addressing is required, the addr field will specify the
source address for the incoming message, otherwise, the source address will
be set to FI_ADDR_NOTAVAIL.  The size field indicates the received message
size.  This field is used by the owner when handling multi-receive data
buffers, but may be ignored otherwise.  The peer provider is responsible
for checking that an incoming message fits within the provided buffer space.
The tag field is set for tagged messages.
(TBD: The peer may need to update the src addr if the remote endpoint is
inserted into the AV after the message has been received.)

The next, prev, and context fields may be used by the owner while the
fi_peer_rx_entry is under its control.  On input to the get entry calls,
the count should be set by the peer to the number of entries in the iov
array, and desc array if applicable.  When returning the fi_peer_rx_entry
to the peer, the owner will update the count to the number of iov entries
containing valid data.  The iov entries will reference the receive
buffer(s) where the message should be placed.

The get_msg_entry() and get_tag_entry() calls may complete synchronously
or asynchronously, depending on whether an application buffer is ready to
receive the incoming data.  A return value of FI_SUCCESS indicates that the
call has completed synchronously.  In this case, the fi_peer_rx_entry is
returned to the peer with the iov referencing the receive buffer(s) and
the context field set to the application's context (to pass to any
completion).  A return value of -FI_EINPROGRESS indicates that a matching
receive entry was not found and the request has been queued.  The owner
will notify the peer when a match has been made or if the message should
be discarded.

## fi_ops_srx_peer::start_msg() / start_tag()

These calls indicate that an asynchronous get_msg_entry() or get_tag_entry()
has completed and a buffer is now available to receive the message.  Control
of the fi_peer_rx_entry is returned to the peer provider and has been
initialized for receiving the incoming message.

## fi_ops_srx_peer::discard_msg() / discard_tag()

Indicates that the message and data associated with the specified
fi_peer_rx_entry should be discarded.  This often indicates that
the application has canceled or discarded the receive operation.
No completion should be generated by the peer provider for a
discarded message.  Control of the fi_peer_rx_entry is returned to
the peer provider.

## EXAMPLE PEER SRX SETUP

The above description defines the generic mechanism for sharing SRXs
between providers.  This section outlines one possible implementation
to demonstrate the use of the APIs.  In the example, provider A
uses provider B as a peer for data transfers targeting endpoints
on the local node.

```
1. Provider A is configured to use provider B as a peer.  This may be coded
   into provider A or set through an environment variable.
2. The application calls:
   fi_srx_context(domain_a, attr, &srx_a, app_context)
3. Provider A allocates srx_a and automatically configures it to be used
   as a peer srx.
4. Provider A takes these steps:
   allocate peer_srx and reference srx_a
   set peer_srx_context->srx = peer_srx
   set attr_b.flags |= FI_PEER_IMPORT
   fi_srx_context(domain_b, attr_b, &srx_b, peer_srx_context)
5. Provider B allocates an srx, but configures it such that all receive
   buffers are obtained from the peer_srx.  The srx ops to post receives are
   set to enosys calls.
8. Provider B inserts its own callbacks into the peer_srx object.  It
   creates a reference between the peer_srx object and its own srx.
```

## EXAMPLE PEER SRX RECEIVE FLOW

The following outlines shows simplified, example software flows for receive
message handling using a peer SRX.  The first flow demonstrates the case
where a receive buffer is waiting when the message arrives.

```
1. Application calls fi_recv() / fi_trecv() on owner.
2. Owner queues the receive buffer.
3. A message is received by the peer provider.
4. The peer calls owner->get_msg_entry() / get_tag_entry().
5. The owner removes the queued receive buffer and returns it to
   the peer.  The get entry call will complete with FI_SUCCESS.
```

The second case below shows the flow when a message arrives before the
application has posted the matching receive buffer.

```
1. A message is received by the peer provider.
2. The peer calls owner->get_msg_entry() / get_tag_entry().
3. The owner fails to find a matching receive buffer.
4. The owner queue the peer request on an unexpected/pending list
   and returns -FI_EINPROGRESS.
5. The application calls fi_recv() / fi_trecv() on owner, posting the
   matching receive buffer.
6. The owner matches the receive with the queued message on the peer.
7. The owner removes the queued request and calls the peer->start_msg() /
   start_tag() function.
```

# fi_export_fid / fi_import_fid

The fi_export_fid function is reserved for future use.

The fi_import_fid call may be used to import a fabric object created
and owned by the libfabric user.  This allows upper level libraries or
the application to override or define low-level libfabric behavior.
Details on specific uses of fi_import_fid are outside the scope of
this documentation.

# RETURN VALUE

Returns FI_SUCCESS on success. On error, a negative value corresponding to
fabric errno is returned. Fabric errno values are defined in
`rdma/fi_errno.h`.

# SEE ALSO

[`fi_provider`(7)](fi_provider.7.html),
[`fi_provider`(3)](fi_provider.3.html),
[`fi_cq`(3)](fi_cq.3.html),

