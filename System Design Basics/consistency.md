# Consistency patterns
With multiple copies of the same data, we are faced with options on how to synchronize them so clients have a consistent view of the data.

## Weak consistency
After a write, reads may or may not see it. A best effort approach is taken.
### Use cases
This approach is seen in systems such as memcached. 
Weak consistency works well in real time use cases such as VoIP, video chat, and realtime multiplayer games.

## Eventual consistency
After a write, reads will eventually see it (typically within milliseconds). Data is replicated asynchronously.
### Use cases
This approach is seen in systems such as DNS and email. Eventual consistency works well in highly available systems.

## Strong consistency
After a write, reads will see it. Data is replicated synchronously.
### Use cases
This approach is seen in file systems and RDBMSes. Strong consistency works well in systems that need transactions.

