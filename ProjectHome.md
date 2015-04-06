## Implement HAT trie as described by Dr. Nikolas Askitis, www.naskitis.com ##

## Note:  As of January 2014 google code has stopped accepting new download files. ##

I have moved the source code to github in the `malbrain/HatTrie` repository.  The latest download, hattrie64d.c contains several bug fixes reported by Peter Duthie.

A HAT tree consists of a cascaded series of root radix director nodes for leading key bytes, which then point to either another radix node, or a hash bucket, or a string array which contains the rest of the bytes in the keys.  The hash buckets are organized as a hash table for access to one of the array nodes parented by the hash bucket. The root of the HAT trie consists of a variable number (0 - n) of pre-configured cascaded radix director nodes for faster access in a multi-dimensional array that is constructed when the trie is created. Additional radix director nodes are created under the tree root when bucket nodes are burst.  Additional bucket nodes are created as string array nodes under radix nodes are burst.

Key bytes inserted into this implementation of the HAT trie are limited to 7 bit ascii values only. Duplicate keys are not inserted into the trie, although by using the optional auxilliary data (payload) area, a count of multiple occurrences of each key string can be maintained.  Please see the `sorthattrie` function in the source code download for an example of this.

## Radix Director Nodes ##

These nodes each contain 128 pointers to either subsequent Radix nodes, to hash buckets, or to a final string array.  The bottom 3 bits of all pointers indicate the type of node being parented.  One byte of the search key is consumed as each Radix node is traversed down the tree until the key search terminates in a string array node, possibly parented by a containing hash bucket node.

## Hash Bucket Nodes ##

These are hashed containers of pointers to parented string array nodes.  The hash code of the incoming string key suffix selects a pointer to a string array node.  Hash bucket nodes do not consume any key bytes of the search string. Buckets will burst into radix director nodes when either a limit of contained strings is reached, or one of the string array nodes it is parenting overflows the maximum size for a string array node.

## String Array Nodes ##

These nodes contain the remaining bytes of each key string suffix after the leading bytes are consumed by the parent radix nodes.  All key searches terminate in a string array node, even if only zero bytes remain.  To minimize wasted unused space in the string array nodes, there are up to 28 ascending sizes of string array nodes possible, ranging from 16 bytes to a maximum of 65536 bytes each. Each new key string suffix is copied into a string array with a preceeding length encoding byte.  When a string array node overflows, it is copied into the next larger size string array node that fits both the exisiting string array node plus the new key suffix bytes. When the largest size string array overflows, the string array is burst.  String array nodes parented under radix director nodes (or the trie root) are burst first under a hash bucket node.  String arrays overflowing under a hash bucket cause the entire bucket to burst into a new radix director node, with subsequent string array nodes and bucket nodes under it.

At the upper end of each string array node is storage for the optional auxilliary (payload) data associated with each key stored in the string array node.  These storage slots are addressed with negative indexing by the key ordinal down from the end of the array.  The string array overflows when the key storage at the bottom of the string array would extend into the auxilliary storage area at the top of the string array, or vice-versa.  Each string array node contains the size of the string array, a count of contained keys, and the next available offset in the array to use for key storage.

## Memory Allocator ##

In order to ensure that all memory blocks for nodes are located at addresses of a multiple of 8 bytes and in order to save the space required for overhead by traditional memory allocators, a custom allocate and free routine is included which requests memory from the system in relatively large chunks and parcels it out to requests for new nodes. There is no per block overhead imposed by the allocator as the size of each block is already known by its type.  Freed blocks are chained together for later reuse.

## Sorted Access to the Trie ##

Multiple cursors over the tree keys can be opened. The radix nodes are traversed to enumerate in sorted order the bucket and array nodes in the tree.  Since there is a fixed limit to strings contained in a bucket node, the individual key strings are enumerated and sorted one bucket node, or string array node, at a time into a fixed work area contained in the cursor.  The cursor controls the enumeration of keys and access to the associated auxilliary data areas. The cursor can be moved forward or backward, to the first or last key, or to an arbitrary key in the trie greater than or equal to a given key.

## Auxilliary Data Areas ##

Each key in the trie can be optionally assigned a data area, usually the size of a `void *` or `uint`.  A pointer to this area for each key is returned by `hat_cell` and `hat_find` and during sorted cursor access to the trie by `hat_slot`.  The amount of this per key space is passed into the `hat_open` creation call, and can be zero bytes if auxilliary data areas are not needed.  In this case, `hat_cell` and `hat_find` return TRUE or FALSE to indicate whether the search key is already in the trie instead of pointers to the associated payload data.

## Sample Applications ##

The download code contains 2 sample applications: the first is designed for Dr. Askitis' tree benchmark files, distinct\_1 and skew\_1, which reports timing and space requirement data for key insertion and lookup; the second application is a string sorter which illustrates the usage of directional cursors and auxilliary data areas which are used to count duplicate key string occurrences.  The second application is invoked on the command line by supplying an empty string for the search file name, and can be configured at compile time with `-D REVERSE` to traverse the trie in reverse order.  It writes the sorted strings delivered by the cursor to standard output.

## Supported Run Time Environments ##

This implementation will compile and run under either 64 bit or 32 bit environments and has been tested under both Windows and Linux, 32 and 64 bits.

## Command Line Arguments ##

After compiling, run the benchmark with the command line:

`HATtrie64c [load file name] [search file name] [# root levels] [# of slots in a pail node] [hash bucket slots] [maximum strings per hash bucket] [smallest string array size in 16 byte units] ...`

All parameters after the two file names are optional, but positional, and are supplied with the following default values:

  * root levels = 3      -- three radix levels booted into the tree
  * pail slots = 127     -- number of slots for array nodes in a pail, zero to disable pails
  * bucket slots = 2047  -- number of slots for array nodes or pail nodes in bucket node
  * bucket max = 65536   -- number of strings in all contained string arrays
  * smallest array = 1   -- in units of 16 bytes
  * next larger sizes = 2 3 4 6 8 10 12 14 16 24 32

Up to 28 array sizes can be specified.  Two dataset files, distinct\_1 for loading the trie and skew1\_1 for searching it, are available at Dr. Nikolas Askitis' web site: http://www.naskitis.com

Supplying an empty search file name will cause the sorted load file to be written to std-out.  Compiling with -D REVERSE will cause the reverse sorted order to be written.

Sample invocation of loading `distinct_1` and searching `skew1_1`:

`HATtrie64c distinct_1 skew1_1 3 127 2047 65536 1 2 3 4 6 8 12 16 24 32 48 64 96 128 256`

Sample invocation as string sorter:

`HATtrie64 distinct_1 "" 3 > tst.out`

## Application Programming Interface ##

The following API functions are supported by the HAT trie code:

```
hat_open:  open a new HAT trie returning an object needed by the other functions.
hat_close: close an open HAT trie, freeing all memory allocated, except cursors.
hat_data:  allocate data memory within hat trie for external use.
hat_cell:  insert a string into the HAT tree, return associated data addr, or TRUE/FALSE.
hat_find:  find a key in the HAT tree, return associated data addr or NULL, or TRUE/FALSE.

hat_cursor:Create a sort cursor for the HAT tree.  Destroy cursor with free().
hat_key:   return the key from the HAT trie at the current cursor location.
hat_nxt:   move the cursor to the next key in the HAT trie, return TRUE/FALSE.
hat_prv:   move the cursor to the prev key in the HAT trie, return TRUE/FALSE.
hat_slot:  return the pointer to the associated data area for the current cursor key.
hat_start: move the cursor to the first key >= given key, or to the first key, return TRUE/FALSE.
hat_last:  move the cursor to the last key in the trie, return TRUE/FALSE.
```

## PAIL nodes ##

The most recent version of HAT-Trie, `HATtrie64b.c`, uses small hash buckets called PAILs.  The pails are designed to increase the utilization count of bucket nodes by utilizing a small hash table whose slots contain pointers to array hash nodes.  The number of pail slots is configured on the command line, which can be set to zero to disable pail usage.  The maximum array hash node size has also been increased to 65536 bytes.


## Additional Information ##

Please see the original HAT Trie description in the paper by Dr. Askitis and Ranjan Sinha "HAT-trie: A Cache-conscious Trie-based Data Structure for Strings" at: http://crpit.com/confpapers/CRPITV62Askitis.pdf, and "Cache-Conscious Collision Resolution
in String Hash Tables" by Dr. Askitis and Justin Zobel at: http://goanna.cs.rmit.edu.au/~jz/fulltext/spire05.pdf.

Also, see the HAT-trie article on wikipedia http://en.wikipedia.org/wiki/HAT-trie.

## Author Information ##

Karl Malbrain, [mailto:malbrain@cal.berkeley.edu](mailto:malbrain@cal.berkeley.edu), with portions by Dr. Nikolas Askitis, http://www.naskitis.com