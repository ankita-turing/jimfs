# Debugging Exercise Proposal (Jimfs) — 100 New Bug Candidates

## Repo map
- Entry points / provider integration
  - jimfs/src/main/java/com/google/common/jimfs/Jimfs.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileSystems.java
  - jimfs/src/main/java/com/google/common/jimfs/SystemJimfsFileSystemProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileSystemProvider.java
  - Responsibilities: create file systems, expose provider to java.nio APIs, URI scheme.
- Core file system object model
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileSystem.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileStore.java
  - jimfs/src/main/java/com/google/common/jimfs/FileSystemView.java
  - jimfs/src/main/java/com/google/common/jimfs/FileSystemState.java
  - Responsibilities: roots, working directory, file store/locks, open/close lifecycle.
- Path creation, parsing, normalization, equality
  - jimfs/src/main/java/com/google/common/jimfs/PathService.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsPath.java
  - jimfs/src/main/java/com/google/common/jimfs/PathType.java
  - jimfs/src/main/java/com/google/common/jimfs/UnixPathType.java
  - jimfs/src/main/java/com/google/common/jimfs/WindowsPathType.java
  - jimfs/src/main/java/com/google/common/jimfs/PathNormalization.java
  - Responsibilities: parse/normalize paths, compare paths, create URIs.
- File tree and directory structure
  - jimfs/src/main/java/com/google/common/jimfs/FileTree.java
  - jimfs/src/main/java/com/google/common/jimfs/Directory.java
  - jimfs/src/main/java/com/google/common/jimfs/DirectoryEntry.java
  - jimfs/src/main/java/com/google/common/jimfs/Name.java
  - Responsibilities: path lookup, symlink resolution, directory entry storage.
- File types and content storage
  - jimfs/src/main/java/com/google/common/jimfs/File.java
  - jimfs/src/main/java/com/google/common/jimfs/RegularFile.java
  - jimfs/src/main/java/com/google/common/jimfs/SymbolicLink.java
  - jimfs/src/main/java/com/google/common/jimfs/HeapDisk.java
  - jimfs/src/main/java/com/google/common/jimfs/Util.java
  - Responsibilities: file metadata, block allocation, content IO and locks.
- Channels and streams
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileChannel.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsAsynchronousFileChannel.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsInputStream.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsOutputStream.java
  - Responsibilities: FileChannel/AsyncFileChannel/InputStream/OutputStream behavior and locking.
- Attributes and attribute views
  - jimfs/src/main/java/com/google/common/jimfs/AttributeService.java
  - jimfs/src/main/java/com/google/common/jimfs/BasicAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/OwnerAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/PosixAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/UnixAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/DosAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/AclAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/UserDefinedAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/StandardAttributeProviders.java
  - Responsibilities: attribute storage, view composition, defaults, copy behavior.
- Watch service
  - jimfs/src/main/java/com/google/common/jimfs/AbstractWatchService.java
  - jimfs/src/main/java/com/google/common/jimfs/PollingWatchService.java
  - jimfs/src/main/java/com/google/common/jimfs/WatchServiceConfiguration.java
  - Responsibilities: directory change polling and WatchKey event queues.
- Path matching / globbing
  - jimfs/src/main/java/com/google/common/jimfs/PathMatchers.java
  - jimfs/src/main/java/com/google/common/jimfs/GlobToRegex.java
  - Responsibilities: PathMatcher implementation for glob/regex patterns.
- Configuration and options
  - jimfs/src/main/java/com/google/common/jimfs/Configuration.java
  - jimfs/src/main/java/com/google/common/jimfs/Options.java
  - jimfs/src/main/java/com/google/common/jimfs/Feature.java
  - jimfs/src/main/java/com/google/common/jimfs/FileTimeSource.java
  - jimfs/src/main/java/com/google/common/jimfs/SystemFileTimeSource.java
  - Responsibilities: configuration for path types, block sizes, supported features, option parsing.
- URL/URI integration
  - jimfs/src/main/java/com/google/common/jimfs/PathURLConnection.java
  - jimfs/src/main/java/com/google/common/jimfs/Handler.java
  - Responsibilities: URL stream handler and URLConnection for jimfs URLs.

## Bug candidates (new 100, distinct from prior 30)

### N01
- Location: FileSystemView#getOrCreateRegularFile (approx lines 293-309)
- Core relevance: file open/create hot path.
- Bug type: correctness / option handling
- Proposed change: Move CREATE_NEW short-circuit after lookUpRegularFile, allowing existing file to open.
- Trigger conditions: open with CREATE_NEW on existing path.
- Expected symptom: file opens instead of FileAlreadyExistsException.
- Why it’s hard: only manifests for CREATE_NEW; most callers use CREATE.
- Static-analysis discoverability: medium.
- Suggested detection: unit test open with CREATE_NEW on existing file.
- Rank (value/stealth/scorability): 4/4/5

### N02
- Location: FileSystemView#lookUpRegularFile (approx lines 316-333)
- Core relevance: regular-file lookup before create/truncate.
- Bug type: correctness / type confusion
- Proposed change: Return null instead of throwing when entry exists but is not regular.
- Trigger conditions: open path that is a directory/symlink with WRITE.
- Expected symptom: create path overlays non-regular file or wrong error.
- Why it’s hard: only triggered with non-regular paths.
- Static-analysis discoverability: medium.
- Suggested detection: open directory path with WRITE should fail with FileSystemException.
- Rank: 4/4/4

### N03
- Location: FileSystemView#getOrCreateRegularFileWithWriteLock (approx lines 336-346)
- Core relevance: file creation for channels/streams.
- Bug type: correctness / CREATE_NEW
- Proposed change: Pass failIfExists=false to createFile even when CREATE_NEW set.
- Trigger conditions: CREATE_NEW on existing file.
- Expected symptom: existing file reused silently.
- Why it’s hard: only for CREATE_NEW.
- Static-analysis discoverability: medium.
- Suggested detection: unit test for CREATE_NEW.
- Rank: 4/4/5

### N04
- Location: FileSystemView#newDirectoryStream (approx lines 125-137)
- Core relevance: directory listing and secure streams.
- Bug type: correctness / link handling
- Proposed change: Always use NOFOLLOW_LINKS when creating directory streams.
- Trigger conditions: list through symlink to directory.
- Expected symptom: NotDirectoryException or missing entries.
- Why it’s hard: only when listing via symlink.
- Static-analysis discoverability: low.
- Suggested detection: list entries through symlinked directory.
- Rank: 3/4/4

### N05
- Location: FileSystemView#link (approx lines 399-436)
- Core relevance: hard links used in core I/O.
- Bug type: correctness / cross-FS integrity
- Proposed change: Remove isSameFileSystem check.
- Trigger conditions: attempt link across file systems.
- Expected symptom: inconsistent link count or corrupted tree.
- Why it’s hard: requires multi-FS setups.
- Static-analysis discoverability: low.
- Suggested detection: integration test linking across FS should fail.
- Rank: 4/5/4

### N06
- Location: FileSystemView#isSameFile (approx lines 181-191)
- Core relevance: Files.isSameFile semantics.
- Bug type: correctness / hard link semantics
- Proposed change: Compare DirectoryEntry equality (directory+name) instead of file identity.
- Trigger conditions: two hard links to same file.
- Expected symptom: isSameFile returns false for hard links.
- Why it’s hard: only manifests with hard links.
- Static-analysis discoverability: low.
- Suggested detection: create hard link and assert isSameFile true.
- Rank: 5/5/5

### N07
- Location: FileSystemView#toRealPath (approx lines 201-220)
- Core relevance: real-path resolution.
- Bug type: correctness / ordering
- Proposed change: Don’t reverse the collected name list before creating path.
- Trigger conditions: any toRealPath call.
- Expected symptom: reversed paths like /c/b/a.
- Why it’s hard: only used by callers of toRealPath.
- Static-analysis discoverability: medium.
- Suggested detection: test toRealPath on nested path.
- Rank: 4/3/5

### N08
- Location: FileSystemView#deleteFile (approx lines 440-447)
- Core relevance: delete path in core I/O.
- Bug type: concurrency / integrity
- Proposed change: Use readLock instead of writeLock in deleteFile.
- Trigger conditions: concurrent delete with create/link.
- Expected symptom: inconsistent directory state or missed deletes.
- Why it’s hard: race-dependent.
- Static-analysis discoverability: low.
- Suggested detection: concurrent create/delete stress test.
- Rank: 5/5/4

### N09
- Location: FileSystemView#copy (approx lines 646-666)
- Core relevance: move/copy semantics.
- Bug type: correctness / cycles
- Proposed change: Remove checkNotAncestor for moves in same FS.
- Trigger conditions: move directory into its own subtree.
- Expected symptom: cycles or errors later during traversal.
- Why it’s hard: only for specific move patterns.
- Static-analysis discoverability: medium.
- Suggested detection: move dir into child should throw.
- Rank: 5/4/4

### N10
- Location: FileSystemView#copy (approx lines 542-548)
- Core relevance: replace-existing semantics.
- Bug type: data loss
- Proposed change: When REPLACE_EXISTING, delete source entry instead of dest entry.
- Trigger conditions: Files.copy with REPLACE_EXISTING.
- Expected symptom: source disappears, dest left unchanged.
- Why it’s hard: only with REPLACE_EXISTING.
- Static-analysis discoverability: low.
- Suggested detection: copy with REPLACE_EXISTING preserves source.
- Rank: 5/4/4

### N11
- Location: FileSystemView#readAttributes(Path,String,LinkOption...) (approx lines 339-345)
- Core relevance: metadata access path.
- Bug type: correctness / link handling
- Proposed change: Always pass FOLLOW_LINKS regardless of options.
- Trigger conditions: NOFOLLOW_LINKS reads on symlinks.
- Expected symptom: attributes of target not link.
- Why it’s hard: only for NOFOLLOW_LINKS.
- Static-analysis discoverability: low.
- Suggested detection: readAttributes with NOFOLLOW_LINKS on symlink.
- Rank: 4/4/4

### N12
- Location: FileSystemView#setAttribute (approx lines 737-742)
- Core relevance: metadata updates.
- Bug type: correctness / link handling
- Proposed change: Always follow links when setting attributes.
- Trigger conditions: setAttribute on symlink with NOFOLLOW_LINKS.
- Expected symptom: target updated instead of link.
- Why it’s hard: requires NOFOLLOW_LINKS path.
- Static-analysis discoverability: low.
- Suggested detection: setAttribute on symlink should affect link.
- Rank: 4/4/4

### N13
- Location: FileSystemView#checkAccess (approx lines 389-392)
- Core relevance: access checks called by NIO APIs.
- Bug type: correctness
- Proposed change: Use NOFOLLOW_LINKS for lookup even when caller expects follow.
- Trigger conditions: checkAccess on symlink to existing target.
- Expected symptom: NoSuchFileException.
- Why it’s hard: only with symlinks.
- Static-analysis discoverability: low.
- Suggested detection: checkAccess on symlink should succeed.
- Rank: 3/4/4

### N14
- Location: FileSystemView#createDirectory (approx lines 230-233)
- Core relevance: directory creation in core APIs.
- Bug type: correctness / idempotency
- Proposed change: Allow existing directory by passing failIfExists=false.
- Trigger conditions: createDirectory on existing directory.
- Expected symptom: silently succeeds instead of FileAlreadyExistsException.
- Why it’s hard: only when directory exists.
- Static-analysis discoverability: medium.
- Suggested detection: createDirectory twice should fail.
- Rank: 3/3/4

### N15
- Location: FileSystemView#readSymbolicLink (approx lines 372-382)
- Core relevance: symlink read path.
- Bug type: correctness / link target
- Proposed change: Resolve link target with FOLLOW_LINKS before returning.
- Trigger conditions: reading symlink pointing to symlink.
- Expected symptom: returns real path instead of link target.
- Why it’s hard: multi-symlink chains.
- Static-analysis discoverability: low.
- Suggested detection: readSymbolicLink should return immediate target.
- Rank: 3/4/4

### N16
- Location: FileTree#lookUp (approx lines 88-103)
- Core relevance: core path resolution.
- Bug type: correctness / root handling
- Proposed change: When absolute root missing, fall back to workingDirectory instead of null.
- Trigger conditions: lookups with unknown root in multi-root FS.
- Expected symptom: paths resolve under wrong root.
- Why it’s hard: only on multi-root configs.
- Static-analysis discoverability: low.
- Suggested detection: access to unknown root should fail.
- Rank: 4/5/4

### N17
- Location: FileTree#lookUp (approx lines 103-106)
- Core relevance: path resolution for empty paths.
- Bug type: correctness / empty path
- Proposed change: Skip EMPTY_PATH_NAMES substitution for empty relative path.
- Trigger conditions: path == "" or "." relative.
- Expected symptom: NoSuchFileException for empty path.
- Why it’s hard: empty path usage is rare.
- Static-analysis discoverability: low.
- Suggested detection: getPath("") should resolve to working directory.
- Rank: 3/4/4

### N18
- Location: FileTree#lookUp (approx lines 131-144)
- Core relevance: path resolution for intermediate segments.
- Bug type: correctness / symlink traversal
- Proposed change: Respect NOFOLLOW_LINKS for intermediate symlinks (treat as file).
- Trigger conditions: NOFOLLOW_LINKS lookup with symlink in middle.
- Expected symptom: path lookup fails even though target exists.
- Why it’s hard: specific to NOFOLLOW_LINKS + intermediate symlink.
- Static-analysis discoverability: low.
- Suggested detection: access "link/child" with NOFOLLOW_LINKS should still follow for intermediate.
- Rank: 4/5/4

### N19
- Location: FileTree#followSymbolicLink (approx lines 176-182)
- Core relevance: symlink resolution.
- Bug type: correctness
- Proposed change: Use Options.NOFOLLOW_LINKS when resolving the target path.
- Trigger conditions: symlink pointing to another symlink.
- Expected symptom: resolution stops after one hop.
- Why it’s hard: chained symlinks only.
- Static-analysis discoverability: low.
- Suggested detection: chain symlink resolution should follow fully.
- Rank: 3/4/4

### N20
- Location: FileTree#getRealEntry (approx lines 194-203)
- Core relevance: correct parent resolution for "." and "..".
- Bug type: correctness
- Proposed change: Return entry as-is even for "."/"..".
- Trigger conditions: toRealPath and ".." resolution.
- Expected symptom: incorrect parent relationships.
- Why it’s hard: only on "."/".." cases.
- Static-analysis discoverability: low.
- Suggested detection: toRealPath on "foo/.." resolves to parent.
- Rank: 4/4/4

### N21
- Location: FileTree#isEmpty (approx lines 210-212)
- Core relevance: empty-path handling.
- Bug type: correctness / boundary
- Proposed change: Treat single empty-string name as non-empty.
- Trigger conditions: getPath("") or parse empty path.
- Expected symptom: empty path resolves incorrectly.
- Why it’s hard: empty paths are edge cases.
- Static-analysis discoverability: low.
- Suggested detection: empty path should behave like ".".
- Rank: 3/4/4

### N22
- Location: Directory#remove (approx lines 309-325)
- Core relevance: delete/unlink semantics.
- Bug type: data integrity
- Proposed change: Remove entry but skip decrementLinkCount.
- Trigger conditions: delete/unlink files.
- Expected symptom: links() stays high, preventing deletion.
- Why it’s hard: only apparent when link counts inspected.
- Static-analysis discoverability: low.
- Suggested detection: create/delete file and check link count returns to 0.
- Rank: 4/4/4

### N23
- Location: Directory#expandIfNeeded (approx lines 262-265)
- Core relevance: directory storage performance.
- Bug type: performance / correctness
- Proposed change: Use `entryCount < resizeThreshold` (no resize at threshold).
- Trigger conditions: directories at load factor ~0.75.
- Expected symptom: excessive collisions, slow lookups.
- Why it’s hard: performance only under load.
- Static-analysis discoverability: low.
- Suggested detection: microbenchmark directory insert/lookup.
- Rank: 3/5/3

### N24
- Location: Directory#bucketIndex (approx lines 188-190)
- Core relevance: directory lookup correctness.
- Bug type: correctness
- Proposed change: Use `name.hashCode() % tableLength` (negative index possible).
- Trigger conditions: negative hash codes.
- Expected symptom: ArrayIndexOutOfBounds or missing entries.
- Why it’s hard: only for certain names.
- Static-analysis discoverability: medium.
- Suggested detection: property test with random names.
- Rank: 4/4/4

### N25
- Location: Directory#iterator (approx lines 347-349)
- Core relevance: directory iteration and snapshot.
- Bug type: correctness / iteration
- Proposed change: Pre-increment index before reading bucket.
- Trigger conditions: iterating directories.
- Expected symptom: missing entries from first bucket.
- Why it’s hard: depends on hash distribution.
- Static-analysis discoverability: low.
- Suggested detection: property test comparing snapshot size to entryCount.
- Rank: 3/4/4

### N26
- Location: Directory#link (approx lines 128-131)
- Core relevance: directory table integrity.
- Bug type: correctness / reserved names
- Proposed change: Skip checkNotReserved, allowing "." or ".." to be linked.
- Trigger conditions: create directory entries named "." or "..".
- Expected symptom: cycles, corrupted traversal.
- Why it’s hard: only appears with hostile inputs.
- Static-analysis discoverability: low.
- Suggested detection: attempt to create file named "." should fail.
- Rank: 5/5/4

### N27
- Location: Directory#forcePut (approx lines 258-259)
- Core relevance: internal directory invariants.
- Bug type: correctness
- Proposed change: Call put(entry, false) so overwrite fails.
- Trigger conditions: root/self/parent relinks during moves.
- Expected symptom: IllegalArgumentException for internal operations.
- Why it’s hard: appears in move/link edge cases.
- Static-analysis discoverability: low.
- Suggested detection: move directory with self/parent links.
- Rank: 4/4/3

### N28
- Location: Directory#isEmpty (approx lines 102-104)
- Core relevance: delete-directory semantics.
- Bug type: correctness / boundary
- Proposed change: Return `entryCount < 2`.
- Trigger conditions: directory with only "."/"..".
- Expected symptom: non-empty directory treated as empty.
- Why it’s hard: only when entryCount is off.
- Static-analysis discoverability: low.
- Suggested detection: delete non-empty dir should throw.
- Rank: 3/4/3

### N29
- Location: RegularFile#truncate (approx lines 236-242)
- Core relevance: truncation semantics used by channels.
- Bug type: correctness / off-by-one
- Proposed change: Set lastPosition = size (not size-1) before blockIndex.
- Trigger conditions: truncate to block boundary.
- Expected symptom: extra block retained, size/space mismatch.
- Why it’s hard: only when size aligns with block size.
- Static-analysis discoverability: low.
- Suggested detection: truncate to block boundary and check disk usage.
- Rank: 4/4/4

### N30
- Location: RegularFile#truncate (approx lines 239-243)
- Core relevance: disk free correctness.
- Bug type: data integrity
- Proposed change: Free `blocksToRemove + 1` blocks.
- Trigger conditions: truncate small file.
- Expected symptom: frees too many blocks; later reads may crash.
- Why it’s hard: appears after truncation and reuse.
- Static-analysis discoverability: low.
- Suggested detection: truncate then read file contents.
- Rank: 5/5/4

### N31
- Location: RegularFile#prepareForWrite (approx lines 253-258)
- Core relevance: writes allocate blocks.
- Bug type: correctness / allocation
- Proposed change: `additionalBlocksNeeded = endBlockIndex - lastBlockIndex - 1`.
- Trigger conditions: write spanning into new block.
- Expected symptom: block missing, write throws or corrupts.
- Why it’s hard: only when crossing block boundary.
- Static-analysis discoverability: low.
- Suggested detection: write across block boundary.
- Rank: 5/5/4

### N32
- Location: RegularFile#prepareForWrite (approx lines 277-278)
- Core relevance: file size semantics.
- Bug type: correctness
- Proposed change: set `size = end` instead of `pos`.
- Trigger conditions: sparse writes.
- Expected symptom: file size grows to include unwritten zero range.
- Why it’s hard: only when sparse writes used.
- Static-analysis discoverability: low.
- Suggested detection: write single byte at offset, check size.
- Rank: 4/4/4

### N33
- Location: RegularFile#write(byte[],off,len) (approx lines 337-340)
- Core relevance: core write path.
- Bug type: correctness / off-by-one
- Proposed change: update size to `endPos - 1`.
- Trigger conditions: write exactly at EOF.
- Expected symptom: last byte invisible to readers.
- Why it’s hard: boundary condition.
- Static-analysis discoverability: low.
- Suggested detection: write then read last byte.
- Rank: 4/4/4

### N34
- Location: RegularFile#write(ByteBuffer) (approx lines 355-358)
- Core relevance: channel writes.
- Bug type: correctness
- Proposed change: use `buf.capacity()` instead of `buf.remaining()` for len.
- Trigger conditions: buffer with non-zero position.
- Expected symptom: writes extra bytes from buffer, corrupting data.
- Why it’s hard: only with sliced/positioned buffers.
- Static-analysis discoverability: medium.
- Suggested detection: write from ByteBuffer with position>0.
- Rank: 5/4/4

### N35
- Location: RegularFile#read(long) (approx lines 469-476)
- Core relevance: byte reads in streams/channels.
- Bug type: correctness / sign handling
- Proposed change: return `block[off]` (signed) instead of unsigned int.
- Trigger conditions: bytes >= 0x80.
- Expected symptom: negative values when reading bytes.
- Why it’s hard: only on high-bit data.
- Static-analysis discoverability: low.
- Suggested detection: write 0xFF then read.
- Rank: 4/4/4

### N36
- Location: RegularFile#read(byte[],off,len) (approx lines 491-505)
- Core relevance: core read path.
- Bug type: correctness / bounds
- Proposed change: use `length(remaining)` ignoring `offsetInBlock`.
- Trigger conditions: read spanning block boundary with non-zero offset.
- Expected symptom: misaligned read results.
- Why it’s hard: only when read not aligned to block.
- Static-analysis discoverability: low.
- Suggested detection: read spanning boundary with offset.
- Rank: 4/4/4

### N37
- Location: RegularFile#read(ByteBuffer) (approx lines 517-534)
- Core relevance: channel reads.
- Bug type: correctness / EOF
- Proposed change: treat bytesToRead==0 as valid read of 0 (return 0).
- Trigger conditions: read exactly at EOF.
- Expected symptom: read loops fail to terminate.
- Why it’s hard: only at EOF.
- Static-analysis discoverability: low.
- Suggested detection: read until EOF should return -1.
- Rank: 4/4/4

### N38
- Location: RegularFile#read(Iterable<ByteBuffer>) (approx lines 545-548)
- Core relevance: scatter reads.
- Bug type: correctness / EOF
- Proposed change: return 0 when pos>=size instead of -1.
- Trigger conditions: scatter read at EOF.
- Expected symptom: loops treat 0 as progress.
- Why it’s hard: scatter read not common.
- Static-analysis discoverability: low.
- Suggested detection: scatter read at EOF should return -1.
- Rank: 3/4/3

### N39
- Location: RegularFile#transferFrom (approx lines 408-413)
- Core relevance: FileChannel.transferFrom.
- Bug type: correctness / boundary
- Proposed change: change guard to `startPos >= size`.
- Trigger conditions: transferFrom at EOF.
- Expected symptom: no bytes transferred even though should append.
- Why it’s hard: only at EOF.
- Static-analysis discoverability: medium.
- Suggested detection: transferFrom with position == size should write.
- Rank: 4/4/4

### N40
- Location: RegularFile#transferFrom (approx lines 458-459)
- Core relevance: transferFrom size updates.
- Bug type: data integrity
- Proposed change: remove `size = currentPos` update.
- Trigger conditions: transferFrom extends file.
- Expected symptom: size stays old, data invisible to readers.
- Why it’s hard: appears only after transferFrom.
- Static-analysis discoverability: low.
- Suggested detection: transferFrom then read size.
- Rank: 4/4/4

### N41
- Location: RegularFile#transferTo (approx lines 590-593)
- Core relevance: transferTo path.
- Bug type: correctness
- Proposed change: use `ByteBuffer.wrap(block, off, ...)` for subsequent blocks too.
- Trigger conditions: transferTo across multiple blocks.
- Expected symptom: repeated offset, missing bytes.
- Why it’s hard: only on multi-block files.
- Static-analysis discoverability: low.
- Suggested detection: transferTo large file, compare content.
- Rank: 4/4/4

### N42
- Location: RegularFile#copyBlocksTo (approx lines 107-113)
- Core relevance: block movement for caching.
- Bug type: correctness / off-by-one
- Proposed change: start = blockCount - count - 1.
- Trigger conditions: copy last N blocks.
- Expected symptom: wrong blocks copied; data corruption.
- Why it’s hard: only when caching is used.
- Static-analysis discoverability: low.
- Suggested detection: block-level copy tests.
- Rank: 5/5/4

### N43
- Location: RegularFile#transferBlocksTo (approx lines 117-120)
- Core relevance: cache transfer.
- Bug type: correctness / off-by-one
- Proposed change: truncateBlocks(blockCount - count - 1).
- Trigger conditions: transfer blocks after delete.
- Expected symptom: extra block retained or freed.
- Why it’s hard: cache-dependent.
- Static-analysis discoverability: low.
- Suggested detection: cache reuse with content checks.
- Rank: 4/5/3

### N44
- Location: RegularFile#truncateBlocks (approx lines 123-125)
- Core relevance: block lifecycle.
- Bug type: data leak
- Proposed change: clear with len = blockCount - count - 1.
- Trigger conditions: truncate blocks.
- Expected symptom: stale blocks referenced later.
- Why it’s hard: only after reuse.
- Static-analysis discoverability: low.
- Suggested detection: delete then recreate file and read data.
- Rank: 5/5/4

### N45
- Location: RegularFile#deleteContents (approx lines 218-221)
- Core relevance: file deletion semantics.
- Bug type: data integrity
- Proposed change: remove `size = 0`.
- Trigger conditions: delete and recreate file.
- Expected symptom: size reported non-zero for empty file.
- Why it’s hard: only after delete.
- Static-analysis discoverability: low.
- Suggested detection: delete file then check size on new file.
- Rank: 4/4/3

### N46
- Location: RegularFile#deleted (approx lines 205-209)
- Core relevance: hard links and delete semantics.
- Bug type: data loss
- Proposed change: drop links()==0 guard before freeing content.
- Trigger conditions: delete one hard link while another exists.
- Expected symptom: remaining link sees data loss.
- Why it’s hard: only with hard links.
- Static-analysis discoverability: low.
- Suggested detection: create hard link, delete one, read from other.
- Rank: 5/5/4

### N47
- Location: RegularFile#closed (approx lines 194-197)
- Core relevance: file lifecycle.
- Bug type: correctness / counter underflow
- Proposed change: use `if (openCount-- == 0)` instead of `if (--openCount == 0)`.
- Trigger conditions: close last open stream.
- Expected symptom: contents deleted one close too early.
- Why it’s hard: race with open/close.
- Static-analysis discoverability: low.
- Suggested detection: open two streams, close one, ensure data remains.
- Rank: 4/4/4

### N48
- Location: RegularFile#bytesToRead (approx lines 631-636)
- Core relevance: read semantics.
- Bug type: correctness / EOF
- Proposed change: return 0 when available==0 instead of -1.
- Trigger conditions: read at EOF.
- Expected symptom: infinite loops in readers.
- Why it’s hard: EOF edge case.
- Static-analysis discoverability: low.
- Suggested detection: read loop should terminate.
- Rank: 4/4/4

### N49
- Location: RegularFile#length(int off,long max) (approx lines 623-625)
- Core relevance: block boundary math.
- Bug type: correctness
- Proposed change: use min(disk.blockSize(), max) ignoring off.
- Trigger conditions: read/write from non-zero offset.
- Expected symptom: overrun into next block.
- Why it’s hard: only with non-zero offsets.
- Static-analysis discoverability: low.
- Suggested detection: read/write across block boundary.
- Rank: 4/4/4

### N50
- Location: RegularFile#blockIndex(long position) (approx lines 611-613)
- Core relevance: block addressing.
- Bug type: correctness / overflow
- Proposed change: cast to int before division: `(int) position / blockSize`.
- Trigger conditions: very large positions (>2^31).
- Expected symptom: negative indices and corruption.
- Why it’s hard: only on huge files.
- Static-analysis discoverability: low.
- Suggested detection: large file seek/write test.
- Rank: 4/5/3

### N51
- Location: HeapDisk#allocate (approx lines 121-125)
- Core relevance: disk space enforcement.
- Bug type: correctness / off-by-one
- Proposed change: use `>= maxBlockCount` check.
- Trigger conditions: allocate exactly remaining space.
- Expected symptom: out-of-space one block early.
- Why it’s hard: only at capacity.
- Static-analysis discoverability: medium.
- Suggested detection: allocate up to max size.
- Rank: 3/4/4

### N52
- Location: HeapDisk#allocate (approx lines 127-128)
- Core relevance: block allocation.
- Bug type: correctness
- Proposed change: `newBlocksNeeded = min(count - blockCache.blockCount(), 0)`.
- Trigger conditions: cache has fewer blocks than needed.
- Expected symptom: no new blocks allocated, later NPE.
- Why it’s hard: depends on cache state.
- Static-analysis discoverability: low.
- Suggested detection: allocate when cache empty.
- Rank: 4/4/4

### N53
- Location: HeapDisk#allocate (approx lines 137-138)
- Core relevance: capacity accounting.
- Bug type: data integrity
- Proposed change: skip `allocatedBlockCount = newAllocatedBlockCount`.
- Trigger conditions: repeated allocations.
- Expected symptom: over-allocation beyond max size.
- Why it’s hard: only under heavy use.
- Static-analysis discoverability: low.
- Suggested detection: allocate many files until expected limit.
- Rank: 5/5/4

### N54
- Location: HeapDisk#free (approx lines 153-154)
- Core relevance: space reclamation.
- Bug type: resource leak
- Proposed change: do not decrement allocatedBlockCount.
- Trigger conditions: delete/truncate files.
- Expected symptom: usable space never recovers.
- Why it’s hard: gradual degradation.
- Static-analysis discoverability: low.
- Suggested detection: create/delete loop and check space.
- Rank: 4/5/4

### N55
- Location: HeapDisk#free (approx lines 147-150)
- Core relevance: cache sizing.
- Bug type: memory leak
- Proposed change: always cache all freed blocks (ignore maxCachedBlockCount).
- Trigger conditions: heavy create/delete workload.
- Expected symptom: memory bloat from cached blocks.
- Why it’s hard: only under load.
- Static-analysis discoverability: low.
- Suggested detection: stress test with GC/memory monitoring.
- Rank: 4/5/3

### N56
- Location: HeapDisk#getUnallocatedSpace (approx lines 116-118)
- Core relevance: disk reporting.
- Bug type: correctness / accounting
- Proposed change: subtract blockCache.blockCount() from unallocated space.
- Trigger conditions: cached blocks >0.
- Expected symptom: free space appears smaller than actual.
- Why it’s hard: depends on cache usage.
- Static-analysis discoverability: low.
- Suggested detection: compare space before/after delete with cache.
- Rank: 3/4/3

### N57
- Location: HeapDisk#toBlockCount (approx lines 83-85)
- Core relevance: capacity sizing.
- Bug type: correctness / rounding
- Proposed change: use RoundingMode.CEILING.
- Trigger conditions: maxSize not multiple of blockSize.
- Expected symptom: over-allocations beyond configured size.
- Why it’s hard: only with non-aligned sizes.
- Static-analysis discoverability: medium.
- Suggested detection: set maxSize to non-multiple and allocate.
- Rank: 4/4/4

### N58
- Location: HeapDisk#createBlockCache (approx lines 87-95)
- Core relevance: memory footprint.
- Bug type: performance / memory
- Proposed change: allocate block cache array size = maxCachedBlockCount (not min with 8192).
- Trigger conditions: large maxCacheSize.
- Expected symptom: huge immediate memory allocation.
- Why it’s hard: only with large config.
- Static-analysis discoverability: low.
- Suggested detection: config with large cache, check memory.
- Rank: 4/5/3

### N59
- Location: FileSystemState#register (approx lines 74-86)
- Core relevance: resource lifecycle safety.
- Bug type: concurrency / race
- Proposed change: increment `registering` after resources.add().
- Trigger conditions: close during register.
- Expected symptom: resource can be registered after close without being closed.
- Why it’s hard: race-dependent.
- Static-analysis discoverability: low.
- Suggested detection: concurrent open/close stress test.
- Rank: 5/5/4

### N60
- Location: FileSystemState#unregister (approx lines 95-97)
- Core relevance: resource cleanup.
- Bug type: resource leak
- Proposed change: skip removal when open==false.
- Trigger conditions: close after resource closed.
- Expected symptom: resources set grows; close loop longer.
- Why it’s hard: only on close paths.
- Static-analysis discoverability: low.
- Suggested detection: close FS after many resources and check set empty.
- Rank: 3/4/3

### N61
- Location: FileSystemState#close (approx lines 109-111)
- Core relevance: close sequencing.
- Bug type: correctness / lifecycle
- Proposed change: run onClose after closing resources.
- Trigger conditions: onClose expects resources to be open.
- Expected symptom: onClose fails or misses cleanup.
- Why it’s hard: depends on onClose semantics.
- Static-analysis discoverability: low.
- Suggested detection: custom onClose that inspects resources.
- Rank: 3/4/3

### N62
- Location: FileSystemState#close (approx lines 124-126)
- Core relevance: close idempotency.
- Bug type: reliability
- Proposed change: remove resources.remove(resource) in finally.
- Trigger conditions: resource close throws or re-registers.
- Expected symptom: repeated close attempts or lingering entries.
- Why it’s hard: depends on exception paths.
- Static-analysis discoverability: low.
- Suggested detection: resource throwing on close.
- Rank: 3/4/3

### N63
- Location: JimfsFileSystem#getDefaultThreadPool (approx lines 295-313)
- Core relevance: async IO thread lifecycle.
- Bug type: resource leak
- Proposed change: don’t register Closeable to shutdown thread pool.
- Trigger conditions: create FS, open async channel, close FS.
- Expected symptom: thread leak after close.
- Why it’s hard: only visible via thread dump.
- Static-analysis discoverability: low.
- Suggested detection: close FS and assert no pool threads alive.
- Rank: 4/5/4

### N64
- Location: JimfsFileSystem#toUri (approx lines 260-263)
- Core relevance: URI correctness.
- Bug type: correctness
- Proposed change: pass path without toAbsolutePath.
- Trigger conditions: relative path toUri.
- Expected symptom: malformed relative URIs.
- Why it’s hard: only when path relative.
- Static-analysis discoverability: medium.
- Suggested detection: toUri on relative path should be absolute.
- Rank: 3/4/4

### N65
- Location: JimfsFileSystem#getPath (approx lines 253-257)
- Core relevance: path creation on open/closed FS.
- Bug type: correctness / lifecycle
- Proposed change: remove state.checkOpen.
- Trigger conditions: create path after close.
- Expected symptom: path operations succeed on closed FS.
- Why it’s hard: only after close.
- Static-analysis discoverability: low.
- Suggested detection: close FS then getPath should throw.
- Rank: 3/4/4

### N66
- Location: JimfsFileSystem#getFileStores (approx lines 243-246)
- Core relevance: file store enumeration.
- Bug type: correctness
- Proposed change: return empty set when open.
- Trigger conditions: callers enumerating file stores.
- Expected symptom: file store info missing.
- Why it’s hard: only used by some tools.
- Static-analysis discoverability: low.
- Suggested detection: getFileStores should include fileStore.
- Rank: 2/3/3

### N67
- Location: JimfsFileSystem#newWatchService (approx lines 284-286)
- Core relevance: watch service correctness.
- Bug type: correctness / view mismatch
- Proposed change: construct WatchService with a new FileSystemView rooted at / instead of default view.
- Trigger conditions: WatchService on non-default working directory.
- Expected symptom: events relate to wrong directory.
- Why it’s hard: only on custom working directories.
- Static-analysis discoverability: low.
- Suggested detection: watch dir with custom working dir.
- Rank: 3/4/3

### N68
- Location: JimfsFileStore#getRootDirectoryNames (approx lines 99-102)
- Core relevance: root directory enumeration.
- Bug type: lifecycle
- Proposed change: remove state.checkOpen.
- Trigger conditions: access after close.
- Expected symptom: operations on closed FS succeed.
- Why it’s hard: only after close.
- Static-analysis discoverability: low.
- Suggested detection: getRootDirectories after close should throw.
- Rank: 2/3/3

### N69
- Location: JimfsFileStore#supportsFileAttributeView(String) (approx lines 249-252)
- Core relevance: attribute view capability checks.
- Bug type: correctness
- Proposed change: compare using `contains(name.toLowerCase())`.
- Trigger conditions: view name with mixed case (e.g., "Posix").
- Expected symptom: false negatives for supported views.
- Why it’s hard: only for mixed-case inputs.
- Static-analysis discoverability: low.
- Suggested detection: supportsFileAttributeView("BASIC") should be true.
- Rank: 2/3/3

### N70
- Location: JimfsFileStore#setAttribute (approx lines 196-199)
- Core relevance: attribute updates.
- Bug type: correctness
- Proposed change: call attributes.setAttribute(..., create=true) always.
- Trigger conditions: setting unknown attribute.
- Expected symptom: invalid attributes silently created.
- Why it’s hard: only on invalid attributes.
- Static-analysis discoverability: low.
- Suggested detection: set unsupported attribute should throw.
- Rank: 3/4/3

### N71
- Location: JimfsFileSystemProvider#newFileChannel (approx lines 141-147)
- Core relevance: core channel creation.
- Bug type: correctness / feature gating
- Proposed change: invert FILE_CHANNEL support check.
- Trigger conditions: FILE_CHANNEL supported.
- Expected symptom: UnsupportedOperationException on valid configs.
- Why it’s hard: only on certain configs.
- Static-analysis discoverability: medium.
- Suggested detection: open FileChannel on default config.
- Rank: 3/4/4

### N72
- Location: JimfsFileSystemProvider#newByteChannel (approx lines 160-166)
- Core relevance: channel creation fallback.
- Bug type: correctness
- Proposed change: return JimfsFileChannel even when FILE_CHANNEL unsupported (skip downgrade).
- Trigger conditions: config without FILE_CHANNEL.
- Expected symptom: UnsupportedOperationException later or wrong behavior.
- Why it’s hard: only on non-default configs.
- Static-analysis discoverability: low.
- Suggested detection: open SeekableByteChannel with FILE_CHANNEL disabled.
- Rank: 3/4/4

### N73
- Location: JimfsFileSystemProvider#newInputStream (approx lines 186-190)
- Core relevance: stream creation.
- Bug type: correctness / option validation
- Proposed change: use Options.getOptionsForOutputStream for input.
- Trigger conditions: input stream with no options.
- Expected symptom: write-related options accepted or wrong link handling.
- Why it’s hard: only on odd options.
- Static-analysis discoverability: low.
- Suggested detection: open input stream with WRITE should fail.
- Rank: 3/4/3

### N74
- Location: JimfsFileSystemProvider#newOutputStream (approx lines 197-203)
- Core relevance: output stream semantics.
- Bug type: correctness
- Proposed change: always pass append=true to JimfsOutputStream.
- Trigger conditions: write without APPEND.
- Expected symptom: data always appended; overwrite impossible.
- Why it’s hard: only visible with overwrite patterns.
- Static-analysis discoverability: medium.
- Suggested detection: write to existing file should truncate/overwrite.
- Rank: 4/4/4

### N75
- Location: JimfsFileSystemProvider#copy (approx lines 257-259)
- Core relevance: copy semantics.
- Bug type: correctness / option handling
- Proposed change: use Options.getMoveOptions for copy.
- Trigger conditions: COPY_ATTRIBUTES or ATOMIC_MOVE in copy options.
- Expected symptom: unexpected option validation errors.
- Why it’s hard: only with certain options.
- Static-analysis discoverability: low.
- Suggested detection: copy with COPY_ATTRIBUTES should succeed.
- Rank: 3/4/3

### N76
- Location: FileFactory#nextFileId (approx lines 46-47)
- Core relevance: file identity uniqueness.
- Bug type: correctness
- Proposed change: use incrementAndGet (first id becomes 1).
- Trigger conditions: code relying on id starting at 0.
- Expected symptom: id-based ordering changes.
- Why it’s hard: only for tests relying on IDs.
- Static-analysis discoverability: low.
- Suggested detection: fileKey uniqueness tests.
- Rank: 2/4/3

### N77
- Location: FileFactory#createRootDirectory (approx lines 55-58)
- Core relevance: root directory invariants.
- Bug type: correctness
- Proposed change: use Directory.create instead of createRoot.
- Trigger conditions: initialize file system.
- Expected symptom: root parent/self links incorrect.
- Why it’s hard: only in root semantics and link counts.
- Static-analysis discoverability: low.
- Suggested detection: root.isRootDirectory should be true.
- Rank: 4/4/4

### N78
- Location: File#copyAttributes (approx lines 247-258)
- Core relevance: attribute copy semantics.
- Bug type: correctness / metadata loss
- Proposed change: skip copyBasicAttributes when copying attributes.
- Trigger conditions: copy with ALL attributes.
- Expected symptom: times not preserved.
- Why it’s hard: only when checking timestamps.
- Static-analysis discoverability: low.
- Suggested detection: copy file and compare times.
- Rank: 3/4/3

### N79
- Location: File#setAttribute (approx lines 220-224)
- Core relevance: attribute writes.
- Bug type: correctness
- Proposed change: return early if attributes == null (no initialization).
- Trigger conditions: first setAttribute call on file.
- Expected symptom: attribute silently not stored.
- Why it’s hard: only on first attribute set.
- Static-analysis discoverability: low.
- Suggested detection: set then get attribute.
- Rank: 3/4/4

### N80
- Location: File#getAttributeNames (approx lines 190-195)
- Core relevance: user-defined attributes enumeration.
- Bug type: correctness
- Proposed change: remove null guard; directly access attributes.row(view).
- Trigger conditions: file with no attributes.
- Expected symptom: NullPointerException on getAttributeNames.
- Why it’s hard: only when attribute table empty.
- Static-analysis discoverability: medium.
- Suggested detection: list attribute names on new file.
- Rank: 3/3/4

### N81
- Location: DirectoryEntry#requireDoesNotExist (approx lines 79-85)
- Core relevance: create/link correctness.
- Bug type: correctness
- Proposed change: return this without throwing when entry exists.
- Trigger conditions: create/link on existing name.
- Expected symptom: overwrites existing entries without error.
- Why it’s hard: only on collisions.
- Static-analysis discoverability: low.
- Suggested detection: create file twice should fail.
- Rank: 4/4/4

### N82
- Location: DirectoryEntry#requireDirectory (approx lines 95-101)
- Core relevance: path traversal.
- Bug type: correctness / error type
- Proposed change: throw NoSuchFileException instead of NotDirectoryException.
- Trigger conditions: path component is regular file.
- Expected symptom: wrong exception type.
- Why it’s hard: only in error handling.
- Static-analysis discoverability: low.
- Suggested detection: expect NotDirectoryException.
- Rank: 2/4/3

### N83
- Location: DirectoryEntry#requireSymbolicLink (approx lines 112-118)
- Core relevance: symlink operations.
- Bug type: correctness
- Proposed change: invert isSymbolicLink check.
- Trigger conditions: readSymbolicLink on symlink.
- Expected symptom: NotLinkException on valid symlink.
- Why it’s hard: only on symlink operations.
- Static-analysis discoverability: low.
- Suggested detection: readSymbolicLink should succeed.
- Rank: 3/4/4

### N84
- Location: DirectoryEntry#file (approx lines 137-140)
- Core relevance: lookup correctness.
- Bug type: reliability
- Proposed change: remove checkState(exists()) guard.
- Trigger conditions: missing file entry.
- Expected symptom: null file used later, NPEs.
- Why it’s hard: only after non-existent lookups.
- Static-analysis discoverability: medium.
- Suggested detection: open missing file should throw NoSuchFileException.
- Rank: 3/4/3

### N85
- Location: Name#equals (approx lines 79-83)
- Core relevance: name equality and lookup.
- Bug type: correctness / canonicalization
- Proposed change: compare display strings instead of canonical.
- Trigger conditions: case-insensitive or normalized configs.
- Expected symptom: lookups fail for canonical-equal names.
- Why it’s hard: only on normalized configs.
- Static-analysis discoverability: low.
- Suggested detection: case-insensitive lookup tests.
- Rank: 5/5/4

### N86
- Location: Name#hashCode (approx lines 87-89)
- Core relevance: hashing of names in directory tables.
- Bug type: correctness / hash-equals
- Proposed change: use display hash instead of canonical hash.
- Trigger conditions: normalized/case-insensitive configs.
- Expected symptom: directory lookups miss entries.
- Why it’s hard: only with canonicalization.
- Static-analysis discoverability: low.
- Suggested detection: lookup with different case.
- Rank: 5/5/4

### N87
- Location: Name#displayComparator (approx lines 107-111)
- Core relevance: ordering of directory snapshots.
- Bug type: correctness
- Proposed change: compare by canonical form in displayComparator.
- Trigger conditions: listing directories with normalized names.
- Expected symptom: ordering differs from display string.
- Why it’s hard: only in sorted listings.
- Static-analysis discoverability: low.
- Suggested detection: directory listing order tests.
- Rank: 2/4/3

### N88
- Location: PathService#parsePath (approx lines 183-185)
- Core relevance: path parsing for Files.getPath.
- Bug type: correctness / empty segments
- Proposed change: remove NOT_EMPTY filtering in joiner inputs.
- Trigger conditions: parsePath("a", "", "b").
- Expected symptom: empty name components retained.
- Why it’s hard: only when passing empty segments.
- Static-analysis discoverability: low.
- Suggested detection: parsePath with empty segment should collapse.
- Rank: 3/4/4

### N89
- Location: PathService#toUri (approx lines 241-246)
- Core relevance: URI correctness.
- Bug type: correctness / link handling
- Proposed change: call Files.isDirectory(path) without NOFOLLOW_LINKS.
- Trigger conditions: path is symlink to directory.
- Expected symptom: URI trailing slash added for symlink.
- Why it’s hard: only with symlinks.
- Static-analysis discoverability: low.
- Suggested detection: toUri on symlink should reflect link.
- Rank: 3/4/3

### N90
- Location: PathService#compare (approx lines 230-234)
- Core relevance: Path ordering & equality.
- Bug type: correctness / canonicalization
- Proposed change: always use DISPLAY comparators even when canonical equality enabled.
- Trigger conditions: case-insensitive configs.
- Expected symptom: compareTo inconsistent with equals.
- Why it’s hard: only on specific configs.
- Static-analysis discoverability: medium.
- Suggested detection: compareTo consistent with equals in Windows config.
- Rank: 4/4/4

### N91
- Location: JimfsPath#startsWith (approx lines 169-175)
- Core relevance: path prefix logic.
- Bug type: correctness
- Proposed change: compare roots by reference (==) instead of equals.
- Trigger conditions: multiple Name instances with same root string.
- Expected symptom: startsWith false even when same root.
- Why it’s hard: only when roots are equivalent objects.
- Static-analysis discoverability: low.
- Suggested detection: startsWith on paths from same FS.
- Rank: 3/4/3

### N92
- Location: JimfsPath#endsWith (approx lines 189-193)
- Core relevance: suffix matching.
- Bug type: correctness
- Proposed change: for absolute otherPath, use startsWith instead of equality.
- Trigger conditions: absolute otherPath that is prefix.
- Expected symptom: endsWith true for prefix, not exact match.
- Why it’s hard: only on absolute-otherPath cases.
- Static-analysis discoverability: low.
- Suggested detection: endsWith on absolute path should require equality.
- Rank: 3/4/3

### N93
- Location: JimfsPath#resolve (approx lines 265-273)
- Core relevance: path resolution.
- Bug type: correctness
- Proposed change: when this is empty path, return this instead of other.
- Trigger conditions: resolve against empty path.
- Expected symptom: resolution yields empty path.
- Why it’s hard: only for empty path edge case.
- Static-analysis discoverability: low.
- Suggested detection: emptyPath.resolve("a") should be "a".
- Rank: 3/4/3

### N94
- Location: JimfsPath#resolveSibling (approx lines 287-294)
- Core relevance: sibling resolution.
- Bug type: correctness
- Proposed change: when parent is null, return this instead of otherPath.
- Trigger conditions: resolveSibling on root/empty path.
- Expected symptom: returns original path rather than sibling.
- Why it’s hard: only for root paths.
- Static-analysis discoverability: low.
- Suggested detection: root.resolveSibling("x") should be "x".
- Rank: 2/4/3

### N95
- Location: JimfsPath#toRealPath (approx lines 352-357)
- Core relevance: canonicalization.
- Bug type: correctness / link handling
- Proposed change: always pass FOLLOW_LINKS.
- Trigger conditions: toRealPath with NOFOLLOW_LINKS.
- Expected symptom: ignores NOFOLLOW_LINKS.
- Why it’s hard: only when options passed.
- Static-analysis discoverability: low.
- Suggested detection: toRealPath(NOFOLLOW_LINKS) should not resolve symlink.
- Rank: 3/4/3

### N96
- Location: PathMatchers#getPathMatcher (approx lines 49-61)
- Core relevance: glob matching used by clients.
- Bug type: correctness
- Proposed change: skip GlobToRegex conversion for "glob".
- Trigger conditions: FileSystem.getPathMatcher("glob:*.txt").
- Expected symptom: glob patterns treated as regex, mismatching.
- Why it’s hard: patterns may still "work" for simple cases.
- Static-analysis discoverability: low.
- Suggested detection: glob matching with "*" and "?".
- Rank: 4/5/4

### N97
- Location: PathMatchers.RegexPathMatcher#matches (approx lines 86-88)
- Core relevance: PathMatcher semantics.
- Bug type: correctness
- Proposed change: use `find()` instead of `matches()`.
- Trigger conditions: regex anchored vs unanchored.
- Expected symptom: partial matches accepted.
- Why it’s hard: depends on regex patterns.
- Static-analysis discoverability: low.
- Suggested detection: regex "^foo$" should not match "foobar".
- Rank: 3/4/4

### N98
- Location: GlobToRegex (STAR state, approx lines 269-281)
- Core relevance: glob path matching.
- Bug type: correctness
- Proposed change: treat "**" same as "*" (no directory crossing).
- Trigger conditions: glob patterns with "**".
- Expected symptom: patterns fail to match across directories.
- Why it’s hard: only for recursive globs.
- Static-analysis discoverability: low.
- Suggested detection: glob "**/a.txt" should match nested.
- Rank: 3/4/4

### N99
- Location: PathNormalization#compilePattern (approx lines 126-131)
- Core relevance: regex matching with normalization.
- Bug type: correctness
- Proposed change: ignore normalization flags (use 0).
- Trigger conditions: case-insensitive/canonical configs.
- Expected symptom: path matcher misses case-insensitive matches.
- Why it’s hard: only on normalized configs.
- Static-analysis discoverability: medium.
- Suggested detection: case-insensitive matcher on Windows config.
- Rank: 4/4/4

### N100
- Location: WindowsPathType#parsePath (approx lines 63-68)
- Core relevance: Windows path parsing.
- Bug type: correctness / API contract
- Proposed change: remove WORKING_DIR_WITH_DRIVE rejection.
- Trigger conditions: parse "C:foo\\bar".
- Expected symptom: treated as absolute path or wrong root.
- Why it’s hard: only on Windows configs.
- Static-analysis discoverability: medium.
- Suggested detection: parse of "C:foo" should throw.
- Rank: 4/4/4

## Top 10 recommended set
- N01: CREATE_NEW behavior regression in core open path; scorable.
- N06: Hard-link equality failure in isSameFile; subtle and high value.
- N09: Move-into-subdir cycle; correctness + data integrity.
- N31: Block allocation off-by-one; data corruption risk.
- N39: transferFrom at EOF incorrectly blocked; API contract drift.
- N46: Hard-link delete frees content; serious data loss.
- N59: register/close race; reliability issue.
- N63: default thread-pool leak; resource lifetime bug.
- N86: Name equality uses display; breaks canonicalization.
- N96: glob matcher treated as regex; user-facing correctness issue.
# Debugging Exercise Proposal (Jimfs) - 100 New Candidates

## Repo map
- Entry points / provider integration
  - jimfs/src/main/java/com/google/common/jimfs/Jimfs.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileSystems.java
  - jimfs/src/main/java/com/google/common/jimfs/SystemJimfsFileSystemProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileSystemProvider.java
  - Responsibilities: create file systems, expose provider to java.nio.file APIs, URI scheme.
- Core file system object model
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileSystem.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileStore.java
  - jimfs/src/main/java/com/google/common/jimfs/FileSystemView.java
  - jimfs/src/main/java/com/google/common/jimfs/FileSystemState.java
  - Responsibilities: root directories, working directory, file store/locks, open/close lifecycle.
- Path creation, parsing, normalization, equality
  - jimfs/src/main/java/com/google/common/jimfs/PathService.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsPath.java
  - jimfs/src/main/java/com/google/common/jimfs/PathType.java
  - jimfs/src/main/java/com/google/common/jimfs/UnixPathType.java
  - jimfs/src/main/java/com/google/common/jimfs/WindowsPathType.java
  - jimfs/src/main/java/com/google/common/jimfs/PathNormalization.java
  - Responsibilities: parse and normalize paths, compare paths, create URIs.
- File tree and directory structure
  - jimfs/src/main/java/com/google/common/jimfs/FileTree.java
  - jimfs/src/main/java/com/google/common/jimfs/Directory.java
  - jimfs/src/main/java/com/google/common/jimfs/DirectoryEntry.java
  - jimfs/src/main/java/com/google/common/jimfs/Name.java
  - Responsibilities: path lookup, symlink resolution, directory entry storage.
- File types and content storage
  - jimfs/src/main/java/com/google/common/jimfs/File.java
  - jimfs/src/main/java/com/google/common/jimfs/RegularFile.java
  - jimfs/src/main/java/com/google/common/jimfs/SymbolicLink.java
  - jimfs/src/main/java/com/google/common/jimfs/HeapDisk.java
  - jimfs/src/main/java/com/google/common/jimfs/Util.java
  - Responsibilities: file metadata, block allocation, content IO and locks.
- Channels and streams
  - jimfs/src/main/java/com/google/common/jimfs/JimfsFileChannel.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsAsynchronousFileChannel.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsInputStream.java
  - jimfs/src/main/java/com/google/common/jimfs/JimfsOutputStream.java
  - Responsibilities: FileChannel/AsyncFileChannel/InputStream/OutputStream behavior and locking.
- Attributes and attribute views
  - jimfs/src/main/java/com/google/common/jimfs/AttributeService.java
  - jimfs/src/main/java/com/google/common/jimfs/AttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/BasicAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/OwnerAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/PosixAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/UnixAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/DosAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/AclAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/UserDefinedAttributeProvider.java
  - jimfs/src/main/java/com/google/common/jimfs/StandardAttributeProviders.java
  - Responsibilities: file attribute storage, view composition, default attributes, copy behavior.
- Watch service
  - jimfs/src/main/java/com/google/common/jimfs/AbstractWatchService.java
  - jimfs/src/main/java/com/google/common/jimfs/PollingWatchService.java
  - jimfs/src/main/java/com/google/common/jimfs/WatchServiceConfiguration.java
  - Responsibilities: directory change polling and WatchKey event queues.
- Path matching / globbing
  - jimfs/src/main/java/com/google/common/jimfs/PathMatchers.java
  - jimfs/src/main/java/com/google/common/jimfs/GlobToRegex.java
  - Responsibilities: PathMatcher implementation for glob/regex patterns.
- Configuration and options
  - jimfs/src/main/java/com/google/common/jimfs/Configuration.java
  - jimfs/src/main/java/com/google/common/jimfs/Options.java
  - jimfs/src/main/java/com/google/common/jimfs/Feature.java
  - jimfs/src/main/java/com/google/common/jimfs/FileTimeSource.java
  - jimfs/src/main/java/com/google/common/jimfs/SystemFileTimeSource.java
  - Responsibilities: configuration for path types, block sizes, supported features, option parsing.
- URL/URI integration
  - jimfs/src/main/java/com/google/common/jimfs/PathURLConnection.java
  - jimfs/src/main/java/com/google/common/jimfs/Handler.java
  - Responsibilities: URL stream handler and URLConnection for jimfs URLs.

## Bug candidates (new 100, excluding prior 30)

### N01
- Location: FileSystemView#getOrCreateRegularFile (approx lines 293-309)
- Core relevance: core file open path for channels/streams.
- Bug type: correctness / option handling
- Proposed change: Move CREATE_NEW short-circuit after lookUpRegularFile so existing file is returned.
- Trigger conditions: Opening an existing file with CREATE_NEW.
- Expected symptom: File opens instead of FileAlreadyExistsException.
- Why it’s hard: Only shows with CREATE_NEW usage; common flows unaffected.
- Static-analysis discoverability: Medium.
- Suggested detection: Unit test expecting CREATE_NEW to fail on existing file.
- Rank (value/stealth/scorability): 4/3/5

### N02
- Location: FileSystemView#lookUpRegularFile (approx lines 316-326)
- Core relevance: regular file lookup for read/write.
- Bug type: correctness / type validation
- Proposed change: Return null instead of throwing if entry is not a regular file.
- Trigger conditions: Opening a directory path as a file.
- Expected symptom: Directory treated as missing; may be created or read as file.
- Why it’s hard: Only visible when mixed file types are used.
- Static-analysis discoverability: Medium.
- Suggested detection: Test opening a directory with READ and expecting NotRegularFile error.
- Rank: 4/4/4

### N03
- Location: FileSystemView#getOrCreateRegularFileWithWriteLock (approx lines 336-345)
- Core relevance: CREATE/CREATE_NEW behavior.
- Bug type: correctness / option handling
- Proposed change: Pass failIfExists=false to createFile even when CREATE_NEW set.
- Trigger conditions: CREATE_NEW on existing file.
- Expected symptom: Existing file opened instead of error.
- Why it’s hard: Same as N01 but only in race window.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent create with CREATE_NEW.
- Rank: 3/4/4

### N04
- Location: FileSystemView#newDirectoryStream (approx lines 125-136)
- Core relevance: directory listing path.
- Bug type: correctness / symlink handling
- Proposed change: Use NOFOLLOW_LINKS for lookup regardless of options.
- Trigger conditions: Listing a directory via a symlink.
- Expected symptom: NotDirectoryException or missing listing for symlinked dir.
- Why it’s hard: Only with symlinked dirs.
- Static-analysis discoverability: Low.
- Suggested detection: Integration test listing symlink to dir.
- Rank: 3/4/4

### N05
- Location: FileSystemView#link (approx lines 399-414)
- Core relevance: hard link creation.
- Bug type: data integrity / security boundary
- Proposed change: Remove same-file-system check.
- Trigger conditions: Linking across different Jimfs instances.
- Expected symptom: Cross-FS hard links corrupt link counts or crash later.
- Why it’s hard: Only appears in multi-FS scenarios.
- Static-analysis discoverability: Medium.
- Suggested detection: Test linking between two Jimfs instances.
- Rank: 4/4/4

### N06
- Location: FileSystemView#isSameFile (approx lines 181-191)
- Core relevance: Files.isSameFile, hard-link semantics.
- Bug type: correctness / hard links
- Proposed change: Compare DirectoryEntry equality instead of File identity.
- Trigger conditions: Two different hard links to same file.
- Expected symptom: isSameFile returns false for hard links.
- Why it’s hard: Only shows when hard links used.
- Static-analysis discoverability: Low.
- Suggested detection: Hard-link test for isSameFile.
- Rank: 4/4/5

### N07
- Location: FileSystemView#toRealPath (approx lines 201-221)
- Core relevance: path canonicalization.
- Bug type: correctness / path construction
- Proposed change: Skip reversing name list before creating path.
- Trigger conditions: toRealPath on nested paths.
- Expected symptom: Reversed path segments (e.g., /a/b -> /b/a).
- Why it’s hard: Only appears in toRealPath; many tests avoid it.
- Static-analysis discoverability: Medium.
- Suggested detection: Unit test for toRealPath ordering.
- Rank: 4/3/4

### N08
- Location: FileSystemView#deleteFile (approx lines 440-446)
- Core relevance: delete path.
- Bug type: concurrency / reliability
- Proposed change: Use readLock instead of writeLock.
- Trigger conditions: Concurrent delete with writes/creates.
- Expected symptom: Intermittent corruption or stale entries.
- Why it’s hard: Requires concurrency timing.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent delete/create stress test.
- Rank: 5/5/4

### N09
- Location: FileSystemView#copy (approx lines 531-536)
- Core relevance: move/copy directory correctness.
- Bug type: correctness / cycle prevention
- Proposed change: Remove checkNotAncestor for same-file-system move.
- Trigger conditions: Moving a directory into its own subtree.
- Expected symptom: Directory cycles or infinite traversal.
- Why it’s hard: Only with specific move target.
- Static-analysis discoverability: Medium.
- Suggested detection: Move dir into child and expect failure.
- Rank: 5/4/5

### N10
- Location: FileSystemView#copy (approx lines 542-547)
- Core relevance: overwrite semantics.
- Bug type: correctness / data loss
- Proposed change: When REPLACE_EXISTING, delete source entry instead of destination.
- Trigger conditions: copy with REPLACE_EXISTING.
- Expected symptom: Source disappears, destination unchanged.
- Why it’s hard: Only triggers with REPLACE_EXISTING.
- Static-analysis discoverability: Low.
- Suggested detection: Copy over existing file and verify source remains.
- Rank: 4/4/4

### N11
- Location: FileSystemView#readAttributes(String) (approx lines 728-731)
- Core relevance: attribute reads.
- Bug type: correctness / symlink handling
- Proposed change: Always use FOLLOW_LINKS ignoring passed options.
- Trigger conditions: NOFOLLOW_LINKS attribute reads on symlinks.
- Expected symptom: Attributes of target returned instead of link.
- Why it’s hard: Only for symlink + NOFOLLOW cases.
- Static-analysis discoverability: Medium.
- Suggested detection: readAttributes with NOFOLLOW_LINKS on symlink.
- Rank: 4/3/4

### N12
- Location: FileSystemView#setAttribute (approx lines 737-742)
- Core relevance: attribute writes.
- Bug type: correctness / symlink handling
- Proposed change: Always follow links when setting attributes.
- Trigger conditions: Setting attributes on symlink with NOFOLLOW_LINKS.
- Expected symptom: Target attributes modified.
- Why it’s hard: Only symlink + NOFOLLOW scenarios.
- Static-analysis discoverability: Low.
- Suggested detection: Set attribute on symlink with NOFOLLOW_LINKS.
- Rank: 4/4/4

### N13
- Location: FileSystemView#checkAccess (approx lines 389-391)
- Core relevance: access checks.
- Bug type: correctness / symlink handling
- Proposed change: Use NOFOLLOW_LINKS for lookup unconditionally.
- Trigger conditions: checkAccess on symlink to existing target.
- Expected symptom: NoSuchFileException for existing targets.
- Why it’s hard: Only shows on symlink paths.
- Static-analysis discoverability: Low.
- Suggested detection: checkAccess on symlink to file.
- Rank: 3/4/4

### N14
- Location: FileSystemView#createDirectory (approx lines 231-233)
- Core relevance: directory creation.
- Bug type: correctness / API contract
- Proposed change: Allow existing directory when CREATE is implied (mkdir -p).
- Trigger conditions: createDirectory on existing dir.
- Expected symptom: No exception (should throw).
- Why it’s hard: Common code assumes exception; failure masked.
- Static-analysis discoverability: Medium.
- Suggested detection: createDirectory on existing path should fail.
- Rank: 3/3/4

### N15
- Location: FileSystemView#readSymbolicLink (approx lines 372-382)
- Core relevance: symlink resolution.
- Bug type: correctness / API contract
- Proposed change: Resolve the link target to real path instead of returning stored target.
- Trigger conditions: readSymbolicLink on relative links.
- Expected symptom: Returns absolute path unexpectedly.
- Why it’s hard: Only fails when target is relative.
- Static-analysis discoverability: Medium.
- Suggested detection: readSymbolicLink returns exactly stored target.
- Rank: 4/3/4

### N16
- Location: FileTree#lookUp (approx lines 89-101)
- Core relevance: root lookup.
- Bug type: correctness / security boundary
- Proposed change: If root not found, return entry for workingDirectory instead of null.
- Trigger conditions: Absolute paths with unknown root.
- Expected symptom: Paths resolved under working directory instead of failing.
- Why it’s hard: Only multi-root configs show this.
- Static-analysis discoverability: Low.
- Suggested detection: Access missing root on multi-root FS.
- Rank: 4/4/4

### N17
- Location: FileTree#lookUp (approx lines 103-106)
- Core relevance: empty path handling.
- Bug type: correctness / edge-case input
- Proposed change: Do not replace empty names list with EMPTY_PATH_NAMES.
- Trigger conditions: Path created from empty string.
- Expected symptom: lookups fail or return null for "".
- Why it’s hard: Rare input.
- Static-analysis discoverability: Low.
- Suggested detection: getPath("") should resolve to working directory.
- Rank: 3/4/4

### N18
- Location: FileTree#lookUp (approx lines 131-141)
- Core relevance: symlink resolution.
- Bug type: correctness / path traversal
- Proposed change: Skip followSymbolicLink for intermediate components when NOFOLLOW_LINKS present.
- Trigger conditions: Path traversal with NOFOLLOW_LINKS in options.
- Expected symptom: Intermediate symlink treated as non-directory.
- Why it’s hard: Only when options include NOFOLLOW_LINKS.
- Static-analysis discoverability: Low.
- Suggested detection: Path traversal through symlink with NOFOLLOW_LINKS should work.
- Rank: 4/4/4

### N19
- Location: FileTree#followSymbolicLink (approx lines 176-182)
- Core relevance: symlink chain resolution.
- Bug type: correctness / symlink resolution
- Proposed change: Call lookUp with NOFOLLOW_LINKS, resolving only one hop.
- Trigger conditions: Nested symlinks.
- Expected symptom: Link chains stop early.
- Why it’s hard: Only link chains longer than one.
- Static-analysis discoverability: Low.
- Suggested detection: Symlink-to-symlink resolution test.
- Rank: 4/4/4

### N20
- Location: FileTree#getRealEntry (approx lines 194-202)
- Core relevance: "."/".." canonicalization.
- Bug type: correctness / path normalization
- Proposed change: Return entry directly for "." and ".." instead of parent entry.
- Trigger conditions: toRealPath on paths with "." or "..".
- Expected symptom: Real paths include "." or resolve to wrong parent.
- Why it’s hard: Only for special names; many tests skip.
- Static-analysis discoverability: Medium.
- Suggested detection: toRealPath on "/a/./b".
- Rank: 4/3/4

### N21
- Location: FileTree#isEmpty (approx lines 210-212)
- Core relevance: empty-path resolution.
- Bug type: correctness / edge-case input
- Proposed change: Treat a single empty name as non-empty.
- Trigger conditions: Path created from empty string.
- Expected symptom: Lookup fails for "".
- Why it’s hard: Rare input.
- Static-analysis discoverability: Low.
- Suggested detection: Files.exists(getPath("")) should be true.
- Rank: 3/4/4

### N22
- Location: Directory#remove (approx lines 309-325)
- Core relevance: link count & deletion semantics.
- Bug type: data integrity
- Proposed change: Remove decrementLinkCount call.
- Trigger conditions: Deleting or unlinking files.
- Expected symptom: link counts leak; delete-on-last-link fails.
- Why it’s hard: Only visible via hard links/open-delete patterns.
- Static-analysis discoverability: Low.
- Suggested detection: Hard-link delete tests.
- Rank: 4/4/4

### N23
- Location: Directory#expandIfNeeded (approx lines 262-266)
- Core relevance: directory performance and correctness.
- Bug type: performance / correctness
- Proposed change: Use `entryCount < resizeThreshold` to delay resizing.
- Trigger conditions: Large directories near capacity.
- Expected symptom: Very slow lookup/insert due to high collision rate.
- Why it’s hard: Performance-only regression.
- Static-analysis discoverability: Low.
- Suggested detection: Load test inserting many entries.
- Rank: 3/5/3

### N24
- Location: Directory#bucketIndex (approx lines 187-189)
- Core relevance: hash table indexing.
- Bug type: correctness / runtime error
- Proposed change: Replace bitmask with `% tableLength` (negative for negative hash).
- Trigger conditions: Names with negative hashCode.
- Expected symptom: ArrayIndexOutOfBoundsException.
- Why it’s hard: Only for certain hashes.
- Static-analysis discoverability: Medium.
- Suggested detection: Fuzz with random names and insert/remove.
- Rank: 4/3/4

### N25
- Location: Directory#iterator (approx lines 336-349)
- Core relevance: directory iteration & snapshots.
- Bug type: correctness / ordering
- Proposed change: Increment index before reading bucket 0.
- Trigger conditions: Iterating directories.
- Expected symptom: Missing entries in first bucket.
- Why it’s hard: Only some entries disappear; non-deterministic across hash.
- Static-analysis discoverability: Low.
- Suggested detection: Deterministic iteration test with known hashes.
- Rank: 4/4/4

### N26
- Location: Directory#link (approx lines 128-131)
- Core relevance: directory integrity.
- Bug type: correctness / security boundary
- Proposed change: Skip checkNotReserved(name, "link").
- Trigger conditions: User attempts to create "." or ".." entries.
- Expected symptom: Directory table corruption or traversal loops.
- Why it’s hard: Only triggered by invalid inputs.
- Static-analysis discoverability: Medium.
- Suggested detection: Attempt to create "." or ".." entry.
- Rank: 4/4/4

### N27
- Location: Directory#forcePut (approx lines 258-259)
- Core relevance: root/parent link updates.
- Bug type: correctness
- Proposed change: Use put(entry, false) so overwrite fails.
- Trigger conditions: Updating parent/self entries.
- Expected symptom: Parent/self links stale after moves.
- Why it’s hard: Only occurs in moves; symptoms indirect.
- Static-analysis discoverability: Low.
- Suggested detection: Move directory and check ".." resolution.
- Rank: 4/4/4

### N28
- Location: Directory#isEmpty (approx lines 102-105)
- Core relevance: delete semantics.
- Bug type: correctness / edge-case
- Proposed change: Return entryCount <= 2 instead of == 2.
- Trigger conditions: Internal entryCount underflows or mismatch.
- Expected symptom: Non-empty directories can be deleted.
- Why it’s hard: Needs inconsistent counters.
- Static-analysis discoverability: Low.
- Suggested detection: Delete non-empty dir should fail.
- Rank: 3/4/3

### N29
- Location: RegularFile#truncate (approx lines 236-243)
- Core relevance: file size semantics.
- Bug type: correctness / off-by-one
- Proposed change: Set lastPosition = size (not size - 1).
- Trigger conditions: Truncating to block boundary sizes.
- Expected symptom: One extra block kept; size metadata inconsistent.
- Why it’s hard: Only at boundaries; silent.
- Static-analysis discoverability: Low.
- Suggested detection: Truncate file to exact block boundary and verify block count.
- Rank: 4/4/4

### N30
- Location: RegularFile#truncate (approx lines 240-243)
- Core relevance: disk accounting.
- Bug type: correctness / off-by-one
- Proposed change: Free blocksToRemove + 1 blocks.
- Trigger conditions: Truncating files by small amounts.
- Expected symptom: Data loss beyond truncation point.
- Why it’s hard: Only visible when reading near end.
- Static-analysis discoverability: Low.
- Suggested detection: Truncate then read last block.
- Rank: 5/4/4

### N31
- Location: RegularFile#prepareForWrite (approx lines 253-258)
- Core relevance: write path.
- Bug type: correctness / allocation
- Proposed change: additionalBlocksNeeded = endBlockIndex - lastBlockIndex - 1.
- Trigger conditions: Write extends file by exactly one block.
- Expected symptom: ArrayIndexOutOfBounds or missing data.
- Why it’s hard: Only specific extension sizes.
- Static-analysis discoverability: Low.
- Suggested detection: Write that crosses block boundary by one byte.
- Rank: 5/4/4

### N32
- Location: RegularFile#prepareForWrite (approx lines 277)
- Core relevance: sparse writes.
- Bug type: correctness / size tracking
- Proposed change: Set size = end instead of pos when zeroing gap.
- Trigger conditions: Sparse write where pos > size.
- Expected symptom: File size jumps before actual write; gaps treated as data.
- Why it’s hard: Only with sparse writes.
- Static-analysis discoverability: Low.
- Suggested detection: Seek beyond EOF then write 0 bytes.
- Rank: 4/4/3

### N33
- Location: RegularFile#write(long, byte[], int, int) (approx lines 337-340)
- Core relevance: write path.
- Bug type: correctness / off-by-one
- Proposed change: Update size to endPos - 1.
- Trigger conditions: Writing to EOF.
- Expected symptom: File size under-reported by one.
- Why it’s hard: Only at boundaries; not immediately obvious.
- Static-analysis discoverability: Low.
- Suggested detection: Write N bytes and assert size==pos+N.
- Rank: 4/4/4

### N34
- Location: RegularFile#write(long, ByteBuffer) (approx lines 355-358)
- Core relevance: FileChannel write path.
- Bug type: correctness / buffer handling
- Proposed change: Use buf.capacity() instead of buf.remaining() for len.
- Trigger conditions: ByteBuffer with non-zero position.
- Expected symptom: Writes extra bytes or over-allocates blocks.
- Why it’s hard: Only for sliced buffers.
- Static-analysis discoverability: Medium.
- Suggested detection: Write with sliced buffer and verify length.
- Rank: 4/3/4

### N35
- Location: RegularFile#read(long) (approx lines 469-476)
- Core relevance: InputStream single-byte reads.
- Bug type: correctness / data integrity
- Proposed change: Return signed byte (block[off]) instead of unsigned.
- Trigger conditions: Reading bytes >= 128.
- Expected symptom: Negative values from read.
- Why it’s hard: Only for high-bit data.
- Static-analysis discoverability: Medium.
- Suggested detection: Write 0xFF and read; expect 255.
- Rank: 4/3/4

### N36
- Location: RegularFile#read(long, byte[], int, int) (approx lines 495-503)
- Core relevance: InputStream bulk reads.
- Bug type: correctness / bounds
- Proposed change: Use length(remaining) instead of length(offsetInBlock, remaining).
- Trigger conditions: Reads not aligned to block boundaries.
- Expected symptom: Reads wrong data or over-reads.
- Why it’s hard: Only on unaligned reads.
- Static-analysis discoverability: Low.
- Suggested detection: Read across block boundary and verify content.
- Rank: 4/4/4

### N37
- Location: RegularFile#read(long, ByteBuffer) (approx lines 517-534)
- Core relevance: FileChannel reads.
- Bug type: correctness / EOF semantics
- Proposed change: Treat bytesToRead==0 as readable and return 0 instead of -1.
- Trigger conditions: Read at EOF with zero remaining.
- Expected symptom: Infinite loops in callers expecting -1.
- Why it’s hard: Only EOF semantics.
- Static-analysis discoverability: Low.
- Suggested detection: Read at EOF should return -1.
- Rank: 4/4/4

### N38
- Location: RegularFile#read(long, Iterable<ByteBuffer>) (approx lines 545-557)
- Core relevance: scatter reads.
- Bug type: correctness / EOF semantics
- Proposed change: Return 0 when pos >= size instead of -1.
- Trigger conditions: Scatter read at EOF.
- Expected symptom: Callers treat as partial read.
- Why it’s hard: Only specific API.
- Static-analysis discoverability: Low.
- Suggested detection: Scatter read at EOF should return -1.
- Rank: 3/4/3

### N39
- Location: RegularFile#transferFrom (approx lines 409-413)
- Core relevance: FileChannel transferFrom.
- Bug type: correctness / API contract
- Proposed change: Return 0 when startPos >= size (disallow writes at EOF).
- Trigger conditions: transferFrom with position at EOF.
- Expected symptom: Data not written; return 0.
- Why it’s hard: Only in transferFrom use.
- Static-analysis discoverability: Medium.
- Suggested detection: transferFrom at EOF should write.
- Rank: 4/3/4

### N40
- Location: RegularFile#transferFrom (approx lines 458-459)
- Core relevance: FileChannel transferFrom.
- Bug type: correctness / size tracking
- Proposed change: Skip `size = currentPos` update.
- Trigger conditions: transferFrom that extends file.
- Expected symptom: Size remains old; data present but invisible.
- Why it’s hard: Only after transferFrom, size mismatches.
- Static-analysis discoverability: Low.
- Suggested detection: transferFrom then size() should reflect new length.
- Rank: 4/4/4

### N41
- Location: RegularFile#transferTo (approx lines 580-593)
- Core relevance: FileChannel transferTo.
- Bug type: correctness / non-blocking behavior
- Proposed change: Remove inner while loop and write once per buffer.
- Trigger conditions: Non-blocking dest writes partial buffers.
- Expected symptom: Data loss in transfers.
- Why it’s hard: Only for non-blocking dests.
- Static-analysis discoverability: Medium.
- Suggested detection: transferTo to channel that writes partials.
- Rank: 4/3/4

### N42
- Location: RegularFile#copyBlocksTo (approx lines 107-113)
- Core relevance: disk reuse.
- Bug type: correctness / off-by-one
- Proposed change: start = blockCount - count - 1.
- Trigger conditions: Copying blocks during free/cache.
- Expected symptom: Wrong blocks copied; data corruption.
- Why it’s hard: Only when cache used.
- Static-analysis discoverability: Low.
- Suggested detection: Create/delete files and compare contents after reuse.
- Rank: 5/4/4

### N43
- Location: RegularFile#transferBlocksTo (approx lines 117-119)
- Core relevance: cache transfer.
- Bug type: correctness / off-by-one
- Proposed change: truncateBlocks(blockCount - count - 1).
- Trigger conditions: Freeing blocks to cache.
- Expected symptom: One extra block removed; truncation too aggressive.
- Why it’s hard: Only for block cache reuse.
- Static-analysis discoverability: Low.
- Suggested detection: Truncate and validate last block retained.
- Rank: 4/4/4

### N44
- Location: RegularFile#truncateBlocks (approx lines 123-126)
- Core relevance: block cleanup.
- Bug type: data leakage
- Proposed change: Do not clear freed blocks (skip Util.clear).
- Trigger conditions: Block cache reuse.
- Expected symptom: Stale data visible in new files.
- Why it’s hard: Only after reuse under cache.
- Static-analysis discoverability: Medium.
- Suggested detection: Create file, delete, create new, check contents zeroed.
- Rank: 5/4/4

### N45
- Location: RegularFile#deleteContents (approx lines 219-221)
- Core relevance: delete semantics.
- Bug type: correctness / metadata leak
- Proposed change: Do not reset size to 0 after free.
- Trigger conditions: Delete then recreate file using same object (hard link).
- Expected symptom: Size reports stale non-zero.
- Why it’s hard: Only with hard links/open-delete.
- Static-analysis discoverability: Low.
- Suggested detection: Delete file with open stream; size should be 0 after close.
- Rank: 4/4/3

### N46
- Location: RegularFile#deleted (approx lines 205-209)
- Core relevance: hard links & deletion semantics.
- Bug type: data integrity
- Proposed change: Set deleted=true without checking links==0.
- Trigger conditions: Delete one hard link while others exist.
- Expected symptom: Remaining links lose content after close.
- Why it’s hard: Only with hard links.
- Static-analysis discoverability: Medium.
- Suggested detection: Create hard link, delete one, read from other.
- Rank: 5/4/4

### N47
- Location: RegularFile#closed (approx lines 194-197)
- Core relevance: open/close lifecycle.
- Bug type: correctness / resource cleanup
- Proposed change: Decrement openCount and deleteContents when openCount <= 1.
- Trigger conditions: Multiple open streams.
- Expected symptom: Content freed while still open.
- Why it’s hard: Only under multi-open usage.
- Static-analysis discoverability: Low.
- Suggested detection: Open two streams, close one; file should remain.
- Rank: 5/4/4

### N48
- Location: RegularFile#bytesToRead (approx lines 631-637)
- Core relevance: EOF behavior.
- Bug type: correctness / EOF semantics
- Proposed change: Treat available==0 as 0 instead of -1.
- Trigger conditions: Read at EOF.
- Expected symptom: Callers loop expecting -1.
- Why it’s hard: Only EOF edge.
- Static-analysis discoverability: Low.
- Suggested detection: Read at EOF returns -1.
- Rank: 4/4/4

### N49
- Location: RegularFile#blockIndex (approx lines 611-613)
- Core relevance: large file offsets.
- Bug type: numeric precision
- Proposed change: Use `(int) position / blockSize` (cast before division).
- Trigger conditions: position > Integer.MAX_VALUE.
- Expected symptom: Wrong block index for large files.
- Why it’s hard: Only large sizes.
- Static-analysis discoverability: Medium.
- Suggested detection: Large-file randomized read/write test.
- Rank: 4/3/3

### N50
- Location: RegularFile#length(int off, long max) (approx lines 623-625)
- Core relevance: block boundary operations.
- Bug type: correctness / bounds
- Proposed change: Use `min(disk.blockSize(), max)` ignoring offset.
- Trigger conditions: Non-zero offset in block.
- Expected symptom: Over-reads or overwrites next block.
- Why it’s hard: Only on unaligned offsets.
- Static-analysis discoverability: Medium.
- Suggested detection: Write with offset near block end.
- Rank: 4/3/4

### N51
- Location: RegularFile#sizeWithoutLocking (approx lines 146-148)
- Core relevance: size queries under lock.
- Bug type: correctness / visibility
- Proposed change: Remove this method and have callers use size() (locking mismatch in copy).
- Trigger conditions: Multi-threaded reads during writes.
- Expected symptom: Occasional stale size reads.
- Why it’s hard: Race-only.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent read/write with size assertions.
- Rank: 4/4/3

### N52
- Location: HeapDisk#allocate (approx lines 122-124)
- Core relevance: disk space enforcement.
- Bug type: correctness / off-by-one
- Proposed change: Use `>= maxBlockCount` to reject allocation early.
- Trigger conditions: Filling disk to exact capacity.
- Expected symptom: Out-of-space one block early.
- Why it’s hard: Only at full capacity.
- Static-analysis discoverability: Medium.
- Suggested detection: Allocate exactly max blocks.
- Rank: 3/3/3

### N53
- Location: HeapDisk#allocate (approx lines 127-128)
- Core relevance: block allocation.
- Bug type: correctness / allocation
- Proposed change: newBlocksNeeded = min(count - blockCache.blockCount(), 0).
- Trigger conditions: Cache empty, allocate new blocks.
- Expected symptom: No new blocks allocated; writes fail.
- Why it’s hard: Only when cache empty.
- Static-analysis discoverability: Low.
- Suggested detection: Write to empty FS should succeed.
- Rank: 4/4/4

### N54
- Location: HeapDisk#allocate (approx lines 137-138)
- Core relevance: disk accounting.
- Bug type: correctness / resource accounting
- Proposed change: Do not update allocatedBlockCount.
- Trigger conditions: Repeated file creation.
- Expected symptom: Over-allocation beyond max size.
- Why it’s hard: Appears as intermittent OOM.
- Static-analysis discoverability: Low.
- Suggested detection: Enforce max disk size in tests.
- Rank: 5/4/3

### N55
- Location: HeapDisk#free (approx lines 152-153)
- Core relevance: space reclaim.
- Bug type: resource leak
- Proposed change: Remove allocatedBlockCount decrement.
- Trigger conditions: File delete/truncate.
- Expected symptom: Usable space never increases.
- Why it’s hard: Gradual leak.
- Static-analysis discoverability: Low.
- Suggested detection: delete then check usable space.
- Rank: 4/5/3

### N56
- Location: HeapDisk#free (approx lines 147-151)
- Core relevance: cache behavior.
- Bug type: performance / memory
- Proposed change: Cache all freed blocks ignoring maxCachedBlockCount.
- Trigger conditions: Large deletes.
- Expected symptom: Memory bloat from cache.
- Why it’s hard: Performance-only, long-term.
- Static-analysis discoverability: Low.
- Suggested detection: Stress delete large files and monitor heap.
- Rank: 3/5/3

### N57
- Location: HeapDisk#getUnallocatedSpace (approx lines 116-118)
- Core relevance: space reporting.
- Bug type: correctness / accounting
- Proposed change: Subtract blockCache.blockCount as if allocated.
- Trigger conditions: Cache enabled.
- Expected symptom: Under-reported free space.
- Why it’s hard: Only when cache non-empty.
- Static-analysis discoverability: Low.
- Suggested detection: Free then check getUnallocatedSpace.
- Rank: 3/4/3

### N58
- Location: HeapDisk#toBlockCount (approx lines 83-85)
- Core relevance: capacity calculation.
- Bug type: correctness / rounding
- Proposed change: Use RoundingMode.CEILING.
- Trigger conditions: maxSize not multiple of blockSize.
- Expected symptom: Disk allows more data than configured.
- Why it’s hard: Only on non-aligned sizes.
- Static-analysis discoverability: Medium.
- Suggested detection: Configure non-aligned maxSize.
- Rank: 3/3/3

### N59
- Location: HeapDisk#createBlockCache (approx lines 89-95)
- Core relevance: memory footprint.
- Bug type: performance / memory
- Proposed change: Allocate blocks array of size maxCachedBlockCount (no cap).
- Trigger conditions: Large cache sizes.
- Expected symptom: High memory usage on startup.
- Why it’s hard: Only for large configs.
- Static-analysis discoverability: Medium.
- Suggested detection: Create FS with huge cache size.
- Rank: 3/3/3

### N60
- Location: HeapDisk#getTotalSpace (approx lines 107-108)
- Core relevance: space reporting.
- Bug type: correctness / overflow
- Proposed change: Cast after int multiplication (int overflow).
- Trigger conditions: Large max sizes.
- Expected symptom: Negative/incorrect total space.
- Why it’s hard: Only large configs.
- Static-analysis discoverability: Medium.
- Suggested detection: Large maxSize and validate totals.
- Rank: 3/3/3

### N61
- Location: FileSystemState#register (approx lines 81-86)
- Core relevance: resource lifecycle.
- Bug type: concurrency / race
- Proposed change: Increment registering after adding resource.
- Trigger conditions: Close concurrent with register.
- Expected symptom: Close loop exits early leaving resource open.
- Why it’s hard: Race-dependent.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent open/close stress test.
- Rank: 5/5/3

### N62
- Location: FileSystemState#unregister (approx lines 95-97)
- Core relevance: resource cleanup.
- Bug type: correctness / leak
- Proposed change: Skip removal if open==false.
- Trigger conditions: Close while resources being unregistered.
- Expected symptom: resources set never empties.
- Why it’s hard: Interleaving sensitive.
- Static-analysis discoverability: Low.
- Suggested detection: Close FS and assert resources empty.
- Rank: 4/5/3

### N63
- Location: FileSystemState#close (approx lines 109-112)
- Core relevance: shutdown order.
- Bug type: correctness / lifecycle
- Proposed change: Run onClose after closing resources.
- Trigger conditions: onClose expects resources still registered.
- Expected symptom: NullPointer or missed cleanup.
- Why it’s hard: Only when onClose uses resources.
- Static-analysis discoverability: Medium.
- Suggested detection: Close FS with custom onClose hook.
- Rank: 3/3/3

### N64
- Location: FileSystemState#close (approx lines 124-125)
- Core relevance: resource cleanup.
- Bug type: reliability / leak
- Proposed change: Remove resources.remove(resource) in finally.
- Trigger conditions: Resources that don’t unregister themselves.
- Expected symptom: Close loop spins or never completes.
- Why it’s hard: Only with misbehaving resources.
- Static-analysis discoverability: Low.
- Suggested detection: Close FS with dummy Closeable.
- Rank: 4/4/3

### N65
- Location: FileSystemState#checkOpen (approx lines 64-67)
- Core relevance: closed FS protection.
- Bug type: correctness
- Proposed change: Return silently instead of throwing ClosedFileSystemException.
- Trigger conditions: After FS close.
- Expected symptom: Operations continue on closed FS.
- Why it’s hard: Many tests assume exception.
- Static-analysis discoverability: Medium.
- Suggested detection: Attempt operation after close should fail.
- Rank: 4/3/4

### N66
- Location: JimfsFileSystem#getDefaultThreadPool (approx lines 295-313)
- Core relevance: async IO thread lifecycle.
- Bug type: resource leak
- Proposed change: Remove registration of Closeable that shuts down pool.
- Trigger conditions: Create and close many FS instances.
- Expected symptom: Thread leak.
- Why it’s hard: Only visible under long-running tests.
- Static-analysis discoverability: Medium.
- Suggested detection: Create/close FS in loop and check thread count.
- Rank: 4/4/3

### N67
- Location: JimfsFileSystem#toUri (approx lines 259-263)
- Core relevance: URI resolution.
- Bug type: correctness / path handling
- Proposed change: Use path without toAbsolutePath.
- Trigger conditions: toUri on relative paths.
- Expected symptom: Relative URIs or invalid paths.
- Why it’s hard: Only for relative paths.
- Static-analysis discoverability: Medium.
- Suggested detection: toUri on relative path should be absolute.
- Rank: 3/3/4

### N68
- Location: JimfsFileSystem#getPath (approx lines 253-256)
- Core relevance: path creation post-close.
- Bug type: correctness / lifecycle
- Proposed change: Remove state.checkOpen() before parsePath.
- Trigger conditions: getPath after close.
- Expected symptom: Paths created on closed FS.
- Why it’s hard: Only after close.
- Static-analysis discoverability: Low.
- Suggested detection: getPath after close should throw.
- Rank: 3/4/3

### N69
- Location: JimfsFileSystem#getFileStores (approx lines 243-246)
- Core relevance: FileStore enumeration.
- Bug type: correctness
- Proposed change: Return empty set instead of fileStore.
- Trigger conditions: FileStore enumeration APIs.
- Expected symptom: No FileStore available.
- Why it’s hard: Only when callers enumerate stores.
- Static-analysis discoverability: Medium.
- Suggested detection: getFileStores should contain one entry.
- Rank: 3/3/4

### N70
- Location: JimfsFileSystem#newWatchService (approx lines 284-286)
- Core relevance: watch service correctness.
- Bug type: correctness / view mismatch
- Proposed change: Create watch service using a new view with root working dir.
- Trigger conditions: Watch service on default view paths.
- Expected symptom: Keys never signal.
- Why it’s hard: Only appears in watch service tests.
- Static-analysis discoverability: Low.
- Suggested detection: Watch service integration test.
- Rank: 4/4/3

### N71
- Location: JimfsFileSystemProvider#newFileChannel (approx lines 141-147)
- Core relevance: file channel availability.
- Bug type: correctness / feature gating
- Proposed change: Invert FILE_CHANNEL support check.
- Trigger conditions: Opening FileChannel when feature supported.
- Expected symptom: UnsupportedOperationException on supported FS.
- Why it’s hard: Only when FileChannel used.
- Static-analysis discoverability: Medium.
- Suggested detection: newFileChannel on default config should work.
- Rank: 3/3/4

### N72
- Location: JimfsFileSystemProvider#newByteChannel (approx lines 160-166)
- Core relevance: SeekableByteChannel behavior.
- Bug type: correctness / API contract
- Proposed change: Always return JimfsFileChannel even when FILE_CHANNEL unsupported.
- Trigger conditions: FS without FILE_CHANNEL feature.
- Expected symptom: Unsupported operations in callers expecting byte channel.
- Why it’s hard: Only in custom configs.
- Static-analysis discoverability: Medium.
- Suggested detection: newByteChannel on FS without FILE_CHANNEL.
- Rank: 3/3/4

### N73
- Location: JimfsFileSystemProvider#newInputStream (approx lines 186-191)
- Core relevance: InputStream options validation.
- Bug type: correctness / option parsing
- Proposed change: Use getOptionsForOutputStream instead of input stream options.
- Trigger conditions: InputStream with invalid options.
- Expected symptom: WRITE options silently accepted.
- Why it’s hard: Only with invalid options.
- Static-analysis discoverability: Low.
- Suggested detection: newInputStream with WRITE should fail.
- Rank: 3/4/3

### N74
- Location: JimfsFileSystemProvider#newOutputStream (approx lines 197-202)
- Core relevance: OutputStream append behavior.
- Bug type: correctness
- Proposed change: Pass append=true regardless of APPEND option.
- Trigger conditions: Write without APPEND to existing file.
- Expected symptom: Always appends instead of overwriting/truncating.
- Why it’s hard: Only when caller expects overwrite.
- Static-analysis discoverability: Medium.
- Suggested detection: newOutputStream should overwrite by default.
- Rank: 4/3/4

### N75
- Location: JimfsFileSystemProvider#copy (approx lines 256-258)
- Core relevance: copy options validation.
- Bug type: correctness / option handling
- Proposed change: Use move options (includes NOFOLLOW_LINKS) for copy.
- Trigger conditions: COPY_ATTRIBUTES with NOFOLLOW_LINKS.
- Expected symptom: Wrong options set; copy may treat as move.
- Why it’s hard: Subtle option differences.
- Static-analysis discoverability: Low.
- Suggested detection: Files.copy with COPY_ATTRIBUTES should not include NOFOLLOW_LINKS.
- Rank: 3/4/3

### N76
- Location: JimfsFileStore#getRootDirectoryNames (approx lines 99-101)
- Core relevance: FS enumeration.
- Bug type: correctness / lifecycle
- Proposed change: Skip state.checkOpen for root enumeration.
- Trigger conditions: FS closed but getRootDirectories called.
- Expected symptom: No exception; stale data.
- Why it’s hard: Only after close.
- Static-analysis discoverability: Low.
- Suggested detection: getRootDirectories after close should fail.
- Rank: 3/4/3

### N77
- Location: JimfsFileStore#supportsFileAttributeView(String) (approx lines 250-252)
- Core relevance: attribute availability.
- Bug type: correctness
- Proposed change: Compare view names case-sensitively with lowercased input.
- Trigger conditions: Mixed-case view names.
- Expected symptom: Supported view reported unsupported.
- Why it’s hard: Only with mixed-case input.
- Static-analysis discoverability: Low.
- Suggested detection: supportsFileAttributeView("Basic") should work.
- Rank: 2/4/3

### N78
- Location: JimfsFileStore#setAttribute (approx lines 196-199)
- Core relevance: attribute mutation.
- Bug type: correctness / validation
- Proposed change: Pass create=true to attributes.setAttribute, allowing new attributes.
- Trigger conditions: setAttribute on unsupported attribute.
- Expected symptom: Silent creation of unsupported attributes.
- Why it’s hard: Only when invalid attributes are used.
- Static-analysis discoverability: Medium.
- Suggested detection: setAttribute on unsupported attribute should throw.
- Rank: 3/3/3

### N79
- Location: FileFactory#nextFileId (approx lines 46-48)
- Core relevance: file identity.
- Bug type: correctness / ID reuse
- Proposed change: Use get() instead of getAndIncrement.
- Trigger conditions: Create multiple files.
- Expected symptom: Duplicate file IDs, breaking fileKey uniqueness.
- Why it’s hard: Only manifests in attribute comparisons.
- Static-analysis discoverability: Medium.
- Suggested detection: Create multiple files and compare fileKey values.
- Rank: 4/3/4

### N80
- Location: File#copyAttributes (approx lines 246-249)
- Core relevance: attribute copying.
- Bug type: correctness / metadata
- Proposed change: Skip copyBasicAttributes; only copy attribute map.
- Trigger conditions: copy with COPY_ATTRIBUTES.
- Expected symptom: timestamps not copied.
- Why it’s hard: Only if tests check timestamps.
- Static-analysis discoverability: Medium.
- Suggested detection: Copy then compare timestamps.
- Rank: 3/3/4

### N81
- Location: File#setAttribute (approx lines 220-224)
- Core relevance: attribute storage.
- Bug type: correctness
- Proposed change: If attributes map is null, return without creating it.
- Trigger conditions: Set any attribute on new file.
- Expected symptom: Attribute silently not stored.
- Why it’s hard: Only visible when reading attributes later.
- Static-analysis discoverability: Low.
- Suggested detection: Set attribute then read it back.
- Rank: 4/4/4

### N82
- Location: File#getAttributeNames (approx lines 190-195)
- Core relevance: attribute enumeration.
- Bug type: correctness
- Proposed change: Return all keys for all views instead of per-view row.
- Trigger conditions: list attributes by view.
- Expected symptom: Attributes from other views leaked.
- Why it’s hard: Only with multiple views.
- Static-analysis discoverability: Medium.
- Suggested detection: Ensure owner view lists only owner attributes.
- Rank: 3/3/3

### N83
- Location: File#deleteAttribute (approx lines 228-231)
- Core relevance: attribute deletion.
- Bug type: correctness
- Proposed change: Create attributes table on delete when null.
- Trigger conditions: deleteAttribute on file with no attributes.
- Expected symptom: attributes table created unnecessarily; memory leak.
- Why it’s hard: Only shows as memory growth.
- Static-analysis discoverability: Low.
- Suggested detection: deleteAttribute should not allocate.
- Rank: 2/4/2

### N84
- Location: DirectoryEntry#requireDoesNotExist (approx lines 79-85)
- Core relevance: file creation semantics.
- Bug type: correctness / error handling
- Proposed change: Return without throwing when entry exists.
- Trigger conditions: create operations on existing paths.
- Expected symptom: Overwrite allowed where it should fail.
- Why it’s hard: Only when create options disallow overwrite.
- Static-analysis discoverability: Medium.
- Suggested detection: createDirectory on existing path should fail.
- Rank: 4/3/4

### N85
- Location: DirectoryEntry#requireDirectory (approx lines 95-101)
- Core relevance: directory validation.
- Bug type: correctness / error type
- Proposed change: Throw NoSuchFileException instead of NotDirectoryException.
- Trigger conditions: Path exists but is a file.
- Expected symptom: Misleading error type.
- Why it’s hard: Only error-type-sensitive tests catch.
- Static-analysis discoverability: Low.
- Suggested detection: Expect NotDirectoryException for file path.
- Rank: 2/4/3

### N86
- Location: DirectoryEntry#requireSymbolicLink (approx lines 112-118)
- Core relevance: symlink validation.
- Bug type: correctness
- Proposed change: Invert check (throw when it is a symlink).
- Trigger conditions: readSymbolicLink on symlink.
- Expected symptom: NotLinkException on valid symlink.
- Why it’s hard: Only when symlinks used.
- Static-analysis discoverability: Medium.
- Suggested detection: readSymbolicLink should succeed for symlink.
- Rank: 3/3/4

### N87
- Location: DirectoryEntry#file (approx lines 137-140)
- Core relevance: file access.
- Bug type: correctness / NPE
- Proposed change: Remove checkState(exists()) guard.
- Trigger conditions: Operations on missing entries.
- Expected symptom: NullPointerException later instead of NoSuchFileException.
- Why it’s hard: Error appears later, misleading stack trace.
- Static-analysis discoverability: Low.
- Suggested detection: Access missing file should throw NoSuchFileException.
- Rank: 3/4/4

### N88
- Location: Name#equals (approx lines 79-83)
- Core relevance: file lookup equality.
- Bug type: correctness / case handling
- Proposed change: Compare display instead of canonical.
- Trigger conditions: Case-insensitive or normalized FS configs.
- Expected symptom: Lookup becomes case-sensitive unexpectedly.
- Why it’s hard: Only for non-default configs.
- Static-analysis discoverability: Medium.
- Suggested detection: Case-insensitive FS should treat "A" and "a" equal.
- Rank: 4/3/4

### N89
- Location: Name#hashCode (approx lines 88-90)
- Core relevance: directory hashing.
- Bug type: correctness / hash-equals contract
- Proposed change: Use display.hashCode() instead of canonical hash.
- Trigger conditions: Case-insensitive configs.
- Expected symptom: HashMap lookup failures for names differing by case.
- Why it’s hard: Only with hash-based lookups.
- Static-analysis discoverability: Low.
- Suggested detection: Create file with name "Foo", lookup "foo".
- Rank: 4/4/4

### N90
- Location: Name#displayComparator (approx lines 107-109)
- Core relevance: directory snapshot ordering.
- Bug type: correctness
- Proposed change: Use canonical comparator for display ordering.
- Trigger conditions: Mixed-case names in display ordering.
- Expected symptom: Sorted order differs from user-visible names.
- Why it’s hard: Only visible in directory listings.
- Static-analysis discoverability: Low.
- Suggested detection: Snapshot ordering for case-insensitive config.
- Rank: 2/4/3

### N91
- Location: PathService#parsePath (approx lines 183-186)
- Core relevance: path parsing with multiple segments.
- Bug type: correctness / input handling
- Proposed change: Remove NOT_EMPTY filter when joining first/more.
- Trigger conditions: parsePath("a", "", "b").
- Expected symptom: Empty path segment preserved.
- Why it’s hard: Only when callers pass empty segments.
- Static-analysis discoverability: Low.
- Suggested detection: parsePath with empty segment should ignore it.
- Rank: 3/4/3

### N92
- Location: PathService#compare (approx lines 230-234)
- Core relevance: path ordering and equality.
- Bug type: correctness / config handling
- Proposed change: Always use DISPLAY comparators even when canonical equality is configured.
- Trigger conditions: Case-insensitive configs.
- Expected symptom: compareTo inconsistent with equals.
- Why it’s hard: Only in ordering-sensitive structures.
- Static-analysis discoverability: Medium.
- Suggested detection: TreeSet with paths that differ by case.
- Rank: 4/3/4

### N93
- Location: PathService#toUri (approx lines 241-245)
- Core relevance: URI generation.
- Bug type: correctness / symlink handling
- Proposed change: Use FOLLOW_LINKS for Files.isDirectory check.
- Trigger conditions: Symlink to directory.
- Expected symptom: URI path ends with "/" when link points to dir.
- Why it’s hard: Only symlink URIs.
- Static-analysis discoverability: Low.
- Suggested detection: URI for symlink should not force trailing "/".
- Rank: 3/4/3

### N94
- Location: JimfsPath#startsWith(Path) (approx lines 169-174)
- Core relevance: path comparison.
- Bug type: correctness / cross-FS
- Proposed change: Remove fileSystem equality check.
- Trigger conditions: Paths from different FS.
- Expected symptom: startsWith returns true across FS.
- Why it’s hard: Only when comparing across FS.
- Static-analysis discoverability: Medium.
- Suggested detection: startsWith should return false for different FS.
- Rank: 3/3/4

### N95
- Location: JimfsPath#endsWith(Path) (approx lines 189-193)
- Core relevance: path comparison.
- Bug type: correctness
- Proposed change: Compare suffix names even when other is absolute.
- Trigger conditions: endsWith absolute path.
- Expected symptom: endsWith returns true for unrelated absolute paths.
- Why it’s hard: Only when absolute path used.
- Static-analysis discoverability: Medium.
- Suggested detection: endsWith with absolute path should be equality only.
- Rank: 3/3/4

### N96
- Location: JimfsPath#resolve (approx lines 265-273)
- Core relevance: path resolution.
- Bug type: correctness
- Proposed change: If this is empty path, return this instead of other.
- Trigger conditions: resolve on empty path.
- Expected symptom: Result ignores other path.
- Why it’s hard: Only with empty path usage.
- Static-analysis discoverability: Low.
- Suggested detection: emptyPath.resolve("a") should be "a".
- Rank: 3/4/3

### N97
- Location: JimfsPath#resolveSibling (approx lines 290-294)
- Core relevance: path resolution.
- Bug type: correctness
- Proposed change: When parent is null, return this instead of other.
- Trigger conditions: resolveSibling on root/empty path.
- Expected symptom: Wrong sibling resolution.
- Why it’s hard: Only root/empty paths.
- Static-analysis discoverability: Low.
- Suggested detection: root.resolveSibling("x") should be "/x".
- Rank: 3/4/3

### N98
- Location: JimfsPath#toRealPath (approx lines 353-356)
- Core relevance: symlink resolution.
- Bug type: correctness / link options
- Proposed change: Always pass Options.FOLLOW_LINKS ignoring options.
- Trigger conditions: NOFOLLOW_LINKS real path.
- Expected symptom: Real path resolves symlink even when not requested.
- Why it’s hard: Only with NOFOLLOW.
- Static-analysis discoverability: Low.
- Suggested detection: toRealPath(NOFOLLOW_LINKS) should preserve symlink.
- Rank: 4/4/4

### N99
- Location: JimfsPath#compareTo (approx lines 414-417)
- Core relevance: path ordering across FS.
- Bug type: correctness
- Proposed change: Compare only by pathService (ignore FS URI).
- Trigger conditions: Sorting paths from different FS.
- Expected symptom: compareTo can return 0 for different FS.
- Why it’s hard: Only in cross-FS collections.
- Static-analysis discoverability: Medium.
- Suggested detection: Compare paths from different FS should be non-zero.
- Rank: 3/3/4

### N100
- Location: JimfsPath#equals (approx lines 421-423)
- Core relevance: path equality.
- Bug type: correctness / provider mismatch
- Proposed change: Return true when other is any Path with same toString.
- Trigger conditions: Paths from different providers with same string.
- Expected symptom: Equality across different FS instances.
- Why it’s hard: Only cross-provider comparisons.
- Static-analysis discoverability: Medium.
- Suggested detection: Path equality should be false across FS.
- Rank: 3/3/4

### N101
- Location: PathMatchers#getPathMatcher (approx lines 49-64)
- Core relevance: glob/regex matching.
- Bug type: correctness / parsing
- Proposed change: For syntax "glob", skip GlobToRegex conversion.
- Trigger conditions: glob patterns with "*", "?".
- Expected symptom: Patterns treated as regex; unexpected matches.
- Why it’s hard: Only for glob syntax; regex still works.
- Static-analysis discoverability: Medium.
- Suggested detection: glob:* should match path segments like glob, not regex.
- Rank: 4/3/4

### N102
- Location: PathMatchers.RegexPathMatcher#matches (approx lines 86-88)
- Core relevance: PathMatcher behavior.
- Bug type: correctness
- Proposed change: Use find() instead of matches().
- Trigger conditions: Regex intended to match full path.
- Expected symptom: Substring matches incorrectly.
- Why it’s hard: Only for patterns without anchors.
- Static-analysis discoverability: Medium.
- Suggested detection: regex "foo" should not match "/bar/foo/baz" unless specified.
- Rank: 3/3/4

### N103
- Location: GlobToRegex STAR state (approx lines 270-279)
- Core relevance: glob correctness.
- Bug type: correctness / glob semantics
- Proposed change: Treat "**" as "*" (non-crossing).
- Trigger conditions: Glob with "**" across directories.
- Expected symptom: Fails to match multi-directory paths.
- Why it’s hard: Only with "**" patterns.
- Static-analysis discoverability: Medium.
- Suggested detection: glob "**/a" should match "x/y/a".
- Rank: 4/3/4

### N104
- Location: GlobToRegex#appendNonSeparator (approx lines 141-147)
- Core relevance: glob correctness.
- Bug type: correctness / regex escaping
- Proposed change: Fail to escape ']' or '-' in separator set.
- Trigger conditions: Windows separators in glob patterns.
- Expected symptom: PatternSyntaxException or incorrect matching.
- Why it’s hard: Only on Windows-style configs.
- Static-analysis discoverability: Low.
- Suggested detection: glob with Windows separators.
- Rank: 3/4/3

### N105
- Location: PathNormalization#compilePattern (approx lines 125-131)
- Core relevance: case-insensitive matching.
- Bug type: correctness
- Proposed change: Ignore normalization flags (always 0).
- Trigger conditions: CASE_FOLD_* configurations.
- Expected symptom: PathMatcher becomes case-sensitive.
- Why it’s hard: Only in specific configs.
- Static-analysis discoverability: Medium.
- Suggested detection: PathMatcher on case-insensitive FS.
- Rank: 4/3/4

### N106
- Location: UnixPathType#parsePath (approx lines 44-54)
- Core relevance: path validation.
- Bug type: correctness / security
- Proposed change: Remove NUL character check.
- Trigger conditions: Path containing NUL.
- Expected symptom: Invalid paths accepted.
- Why it’s hard: Only with invalid inputs.
- Static-analysis discoverability: Medium.
- Suggested detection: path with NUL should throw InvalidPathException.
- Rank: 3/3/3

### N107
- Location: UnixPathType#toUriPath (approx lines 68-76)
- Core relevance: URI rendering.
- Bug type: correctness
- Proposed change: Do not append trailing "/" for directories.
- Trigger conditions: URI for directory.
- Expected symptom: Directory URIs missing trailing slash.
- Why it’s hard: Only for directory URIs.
- Static-analysis discoverability: Low.
- Suggested detection: Directory URI should end with "/".
- Rank: 2/4/3

### N108
- Location: WindowsPathType#parsePath (approx lines 63-67)
- Core relevance: Windows path validation.
- Bug type: correctness / API contract
- Proposed change: Remove WORKING_DIR_WITH_DRIVE check (allow "C:foo").
- Trigger conditions: Windows-style relative drive paths.
- Expected symptom: Treated as absolute or malformed root.
- Why it’s hard: Only in Windows configs.
- Static-analysis discoverability: Medium.
- Suggested detection: "C:foo" should throw InvalidPathException.
- Rank: 3/3/3

### N109
- Location: WindowsPathType#parseUriPath (approx lines 199-204)
- Core relevance: URI parsing.
- Bug type: correctness
- Proposed change: Do not strip leading slash for non-UNC paths.
- Trigger conditions: Non-UNC file URIs.
- Expected symptom: Root becomes "\\C:\\" or malformed.
- Why it’s hard: Only for URI-based path creation.
- Static-analysis discoverability: Low.
- Suggested detection: URI "jimfs://.../C:/x" parses correctly.
- Rank: 3/4/3

### N110
- Location: PathType#toUri (approx lines 187-200)
- Core relevance: URI creation.
- Bug type: correctness
- Proposed change: Use fileSystemUri.getPath() instead of composed path.
- Trigger conditions: FS with non-empty path component in URI.
- Expected symptom: Incorrect URI paths.
- Why it’s hard: Only for non-default URI forms.
- Static-analysis discoverability: Low.
- Suggested detection: toUri should include path element for file.
- Rank: 2/4/2

### N111
- Location: AttributeService#getViewName (approx lines 365-380)
- Core relevance: attribute parsing.
- Bug type: correctness / input validation
- Proposed change: Accept ":" at start or end instead of throwing.
- Trigger conditions: Invalid attribute strings like ":size".
- Expected symptom: Misparsed view/attribute.
- Why it’s hard: Only with invalid inputs.
- Static-analysis discoverability: Medium.
- Suggested detection: invalid attribute format should throw.
- Rank: 2/3/3

### N112
- Location: AttributeService#readAttributes (approx lines 318-327)
- Core relevance: attribute reads.
- Bug type: correctness / inheritance
- Proposed change: When attrs == "*", read only provider for view, ignore inherited.
- Trigger conditions: Requesting "posix:*" or "unix:*".
- Expected symptom: Missing inherited attributes (basic/owner).
- Why it’s hard: Only with inherited views.
- Static-analysis discoverability: Low.
- Suggested detection: "posix:*" should include basic attrs.
- Rank: 3/4/3

### N113
- Location: AttributeService#setAttributeInternal (approx lines 243-247)
- Core relevance: attribute writes.
- Bug type: correctness
- Proposed change: When using inherited provider, pass inherited view name instead of original.
- Trigger conditions: Setting "posix:permissions" via posix view.
- Expected symptom: Attribute stored under wrong view key.
- Why it’s hard: Only visible when reading by view name.
- Static-analysis discoverability: Low.
- Suggested detection: set "posix:permissions" then read via posix view.
- Rank: 4/4/4

### N114
- Location: AttributeService#createInheritedViews (approx lines 292-303)
- Core relevance: attribute view composition.
- Bug type: correctness / recursion
- Proposed change: Remove inheritedViews.containsKey check before recursion.
- Trigger conditions: Views with inheritance chains.
- Expected symptom: Stack overflow or redundant view creation.
- Why it’s hard: Only in deep inheritance.
- Static-analysis discoverability: Medium.
- Suggested detection: Build view for posix with inheritance.
- Rank: 3/3/3

### N115
- Location: AttributeService#getAttributeInternal (approx lines 213-220)
- Core relevance: attribute reads.
- Bug type: correctness / inheritance
- Proposed change: Stop after first inherited view regardless of attribute existence.
- Trigger conditions: Attribute only defined in later inherited view.
- Expected symptom: IllegalArgumentException for valid attributes.
- Why it’s hard: Only for certain view/attr combos.
- Static-analysis discoverability: Low.
- Suggested detection: readAttributes on posix/owner attributes.
- Rank: 3/4/3

### N116
- Location: BasicAttributeProvider#get (approx lines 60-73)
- Core relevance: basic attributes.
- Bug type: correctness
- Proposed change: "isOther" returns true for directories.
- Trigger conditions: Reading basic attributes.
- Expected symptom: Directories reported as "other".
- Why it’s hard: Only visible in attribute checks.
- Static-analysis discoverability: Low.
- Suggested detection: isOther for directory should be false.
- Rank: 2/4/3

### N117
- Location: BasicAttributeProvider.View#setTimes (approx lines 155-165)
- Core relevance: timestamp updates.
- Bug type: correctness / time
- Proposed change: If parameter is null, set time to now anyway.
- Trigger conditions: setTimes with null for some fields.
- Expected symptom: Unintended timestamp changes.
- Why it’s hard: Only when nulls passed.
- Static-analysis discoverability: Medium.
- Suggested detection: setTimes(null,null,null) should not change times.
- Rank: 3/3/4

### N118
- Location: OwnerAttributeProvider#defaultValues (approx lines 53-62)
- Core relevance: owner default config.
- Bug type: correctness
- Proposed change: If userProvidedOwner is UserPrincipal, treat as String and recreate.
- Trigger conditions: Config uses custom UserPrincipal.
- Expected symptom: Identity lost; equals mismatch.
- Why it’s hard: Only with custom principals.
- Static-analysis discoverability: Low.
- Suggested detection: Custom principal should be preserved.
- Rank: 3/4/3

### N119
- Location: OwnerAttributeProvider#set (approx lines 78-85)
- Core relevance: owner attribute.
- Bug type: correctness
- Proposed change: Accept any UserPrincipal without wrapping to JimfsUserPrincipal.
- Trigger conditions: Setting owner with external principal.
- Expected symptom: UnixAttributeProvider uid mapping inconsistent.
- Why it’s hard: Cross-module mismatch.
- Static-analysis discoverability: Medium.
- Suggested detection: Set owner, then read unix:uid stable across lookups.
- Rank: 4/3/4

### N120
- Location: PosixAttributeProvider#defaultValues (approx lines 93-101)
- Core relevance: permissions defaults.
- Bug type: correctness
- Proposed change: Accept Set values without type checking.
- Trigger conditions: Set contains non-PosixFilePermission values.
- Expected symptom: ClassCastException later.
- Why it’s hard: Fails later than configuration time.
- Static-analysis discoverability: Medium.
- Suggested detection: invalid permission set should throw early.
- Rank: 3/3/3

### N121
- Location: PosixAttributeProvider#get (approx lines 117-125)
- Core relevance: posix attributes.
- Bug type: correctness
- Proposed change: Return group name string instead of GroupPrincipal.
- Trigger conditions: Reading posix:group attribute.
- Expected symptom: ClassCastException in callers expecting GroupPrincipal.
- Why it’s hard: Only when caller expects type.
- Static-analysis discoverability: Medium.
- Suggested detection: readAttributes returns GroupPrincipal instance.
- Rank: 3/3/3

### N122
- Location: PosixAttributeProvider#set (approx lines 142-169)
- Core relevance: permissions set.
- Bug type: correctness
- Proposed change: When setting permissions, ignore create flag and allow create-only behavior.
- Trigger conditions: setAttribute on existing file.
- Expected symptom: UnsupportedOperationException or ignored set.
- Why it’s hard: Only for create flag usage.
- Static-analysis discoverability: Low.
- Suggested detection: setAttribute posix:permissions should work.
- Rank: 3/4/3

### N123
- Location: UnixAttributeProvider#getUniqueId (approx lines 85-92)
- Core relevance: unix uid/gid stability.
- Bug type: correctness / caching
- Proposed change: Use object.hashCode directly instead of stable cache.
- Trigger conditions: Recreating equivalent principals.
- Expected symptom: uid/gid changes across reads.
- Why it’s hard: Only across principal instances.
- Static-analysis discoverability: Low.
- Suggested detection: Same principal name should map to stable uid.
- Rank: 4/4/4

### N124
- Location: UnixAttributeProvider#get (approx lines 100-120)
- Core relevance: unix mode.
- Bug type: correctness
- Proposed change: Use owner permissions for gid calculation (swap uid/gid).
- Trigger conditions: Read unix:uid/gid.
- Expected symptom: uid/gid swapped or inconsistent.
- Why it’s hard: Only when unix view used.
- Static-analysis discoverability: Medium.
- Suggested detection: uid should be based on owner, gid on group.
- Rank: 3/3/4

### N125
- Location: DosAttributeProvider#set (approx in DosAttributeProvider)
- Core relevance: DOS attributes.
- Bug type: correctness / timestamp
- Proposed change: Update lastAccessTime instead of lastModifiedTime on attribute change.
- Trigger conditions: Setting DOS attributes.
- Expected symptom: lastModifiedTime not updated.
- Why it’s hard: Only when timestamps inspected.
- Static-analysis discoverability: Low.
- Suggested detection: Setting dos:hidden should update modified time.
- Rank: 3/4/3

### N126
- Location: UserDefinedAttributeProvider.View#read (approx lines 143-147)
- Core relevance: user attributes IO.
- Bug type: correctness / buffer bounds
- Proposed change: Only copy up to dst.remaining but still return full length.
- Trigger conditions: dst.remaining < attribute size.
- Expected symptom: Return value exceeds bytes written.
- Why it’s hard: Only with small buffers.
- Static-analysis discoverability: Low.
- Suggested detection: Read into small buffer should return bytes written.
- Rank: 3/4/3

### N127
- Location: UserDefinedAttributeProvider#set (approx lines 88-95)
- Core relevance: user attributes storage.
- Bug type: correctness / buffer handling
- Proposed change: Use buffer.array() without respecting position/limit.
- Trigger conditions: ByteBuffer with non-zero position.
- Expected symptom: Extra bytes stored.
- Why it’s hard: Only for sliced buffers.
- Static-analysis discoverability: Medium.
- Suggested detection: Write with sliced buffer; stored bytes should match remaining.
- Rank: 3/3/4

### N128
- Location: StandardAttributeProviders#get (approx in StandardAttributeProviders)
- Core relevance: attribute view registration.
- Bug type: correctness
- Proposed change: Return basic provider for unknown view names.
- Trigger conditions: Request unsupported view.
- Expected symptom: Silent fallback rather than error.
- Why it’s hard: Only for invalid view names.
- Static-analysis discoverability: Low.
- Suggested detection: Unsupported view should throw.
- Rank: 2/4/3

### N129
- Location: AbstractWatchService#enqueue (approx lines 80-83)
- Core relevance: watch event delivery.
- Bug type: correctness / lifecycle
- Proposed change: Enqueue keys even when watch service closed.
- Trigger conditions: Close service with pending events.
- Expected symptom: poll() returns keys after close.
- Why it’s hard: Only around shutdown.
- Static-analysis discoverability: Low.
- Suggested detection: ClosedWatchServiceException after close.
- Rank: 3/4/3

### N130
- Location: AbstractWatchService#poll (approx lines 95-98)
- Core relevance: watch API semantics.
- Bug type: correctness
- Proposed change: Remove checkOpen before polling.
- Trigger conditions: poll after close.
- Expected symptom: Returns null instead of throwing ClosedWatchServiceException.
- Why it’s hard: Only after close.
- Static-analysis discoverability: Low.
- Suggested detection: poll after close should throw.
- Rank: 2/4/3

### N131
- Location: AbstractWatchService.Key#post (approx lines 236-239)
- Core relevance: overflow handling.
- Bug type: correctness / metrics
- Proposed change: Drop event when queue full without incrementing overflow.
- Trigger conditions: Event queue overflow.
- Expected symptom: Missing OVERFLOW notifications.
- Why it’s hard: Only under heavy event load.
- Static-analysis discoverability: Low.
- Suggested detection: Force overflow and ensure overflow event emitted.
- Rank: 3/4/3

### N132
- Location: AbstractWatchService.Key#reset (approx lines 273-281)
- Core relevance: watch key lifecycle.
- Bug type: correctness
- Proposed change: Remove requeue when events are pending.
- Trigger conditions: Multiple events before reset.
- Expected symptom: Pending events never delivered.
- Why it’s hard: Only under high activity.
- Static-analysis discoverability: Low.
- Suggested detection: Post multiple events and verify queueing.
- Rank: 4/4/3

### N133
- Location: PollingWatchService#register (approx lines 103-109)
- Core relevance: watch service scheduling.
- Bug type: correctness / scheduling
- Proposed change: Start polling only if snapshots.size() > 1.
- Trigger conditions: First registration.
- Expected symptom: No events for first key.
- Why it’s hard: Only in single-watched directory.
- Static-analysis discoverability: Medium.
- Suggested detection: Watch one directory and modify; expect events.
- Rank: 4/3/4

### N134
- Location: PollingWatchService#cancelled (approx lines 137-142)
- Core relevance: watch service cleanup.
- Bug type: correctness
- Proposed change: Stop polling when snapshots size == 1.
- Trigger conditions: Two keys; cancelling one stops polling for other.
- Expected symptom: Remaining key stops receiving events.
- Why it’s hard: Only when multiple keys and cancellations.
- Static-analysis discoverability: Low.
- Suggested detection: Two keys; cancel one and verify other still receives events.
- Rank: 4/4/3

### N135
- Location: PollingWatchService#close (approx lines 149-156)
- Core relevance: resource cleanup.
- Bug type: resource leak
- Proposed change: Remove pollingService.shutdown().
- Trigger conditions: Close watch service.
- Expected symptom: Background polling thread remains.
- Why it’s hard: Only visible via thread leaks.
- Static-analysis discoverability: Medium.
- Suggested detection: Close watch service and check thread count.
- Rank: 3/3/3

### N136
- Location: PollingWatchService#takeSnapshot (approx lines 196-198)
- Core relevance: watch semantics.
- Bug type: correctness
- Proposed change: Snapshot workingDirectory entries instead of modified times.
- Trigger conditions: ENTRY_MODIFY detection.
- Expected symptom: Modify events never fired.
- Why it’s hard: Only visible via watch service tests.
- Static-analysis discoverability: Low.
- Suggested detection: Modify file and expect ENTRY_MODIFY.
- Rank: 4/4/3

### N137
- Location: PollingWatchService.Snapshot#postChanges (approx lines 217-233)
- Core relevance: watch events.
- Bug type: correctness
- Proposed change: Swap created/deleted set differences.
- Trigger conditions: Create or delete files in watched dir.
- Expected symptom: CREATE events reported as DELETE and vice versa.
- Why it’s hard: Only under watch testing.
- Static-analysis discoverability: Medium.
- Suggested detection: Create file and expect ENTRY_CREATE.
- Rank: 4/3/4

### N138
- Location: WatchServiceConfiguration#polling (approx lines 42-44)
- Core relevance: polling interval.
- Bug type: performance / correctness
- Proposed change: Allow interval=0 and scheduleAtFixedRate with zero.
- Trigger conditions: polling(0, unit).
- Expected symptom: Busy loop; CPU spike.
- Why it’s hard: Only for invalid config.
- Static-analysis discoverability: Medium.
- Suggested detection: Validate interval must be >0.
- Rank: 3/3/3

### N139
- Location: JimfsFileChannel#implCloseChannel (approx lines 640-652)
- Core relevance: channel lifecycle.
- Bug type: resource leak
- Proposed change: Remove fileSystemState.unregister(this).
- Trigger conditions: Close channels repeatedly.
- Expected symptom: FileSystemState resources leak; close hangs.
- Why it’s hard: Only under many channels.
- Static-analysis discoverability: Medium.
- Suggested detection: Open/close many channels then close FS.
- Rank: 4/3/3

### N140
- Location: JimfsFileChannel#read(ByteBuffer) (approx lines 158-162)
- Core relevance: read path.
- Bug type: correctness / metadata
- Proposed change: Update lastModifiedTime instead of lastAccessTime on reads.
- Trigger conditions: Any read via FileChannel.
- Expected symptom: Reads appear as writes.
- Why it’s hard: Only visible by timestamp inspection.
- Static-analysis discoverability: Low.
- Suggested detection: Read should only update access time.
- Rank: 3/4/3

### N141
- Location: JimfsFileChannel#write(ByteBuffer) (approx lines 265-268)
- Core relevance: write path.
- Bug type: correctness / metadata
- Proposed change: Update lastAccessTime instead of lastModifiedTime on writes.
- Trigger conditions: Writes via FileChannel.
- Expected symptom: Modified time not updated.
- Why it’s hard: Only visible by timestamps.
- Static-analysis discoverability: Low.
- Suggested detection: Write should update lastModifiedTime.
- Rank: 3/4/3

### N142
- Location: JimfsFileChannel#position(long) (approx lines 415-417)
- Core relevance: channel position.
- Bug type: correctness
- Proposed change: If newPosition > size, clamp to size.
- Trigger conditions: Seek beyond EOF then write.
- Expected symptom: Writes occur at EOF instead of sparse offsets.
- Why it’s hard: Only with sparse writes.
- Static-analysis discoverability: Medium.
- Suggested detection: position beyond EOF should be respected.
- Rank: 4/3/4

### N143
- Location: JimfsAsynchronousFileChannel#read (approx lines 132-145)
- Core relevance: async IO correctness.
- Bug type: correctness / API contract
- Proposed change: Delegate to channel.read(dst) (relative) instead of positional read.
- Trigger conditions: Async read with explicit position.
- Expected symptom: Reads from current position, not requested offset.
- Why it’s hard: Only in async positional reads.
- Static-analysis discoverability: Medium.
- Suggested detection: Async positional read should not change position.
- Rank: 4/3/4

### N144
- Location: JimfsAsynchronousFileChannel#write (approx lines 158-169)
- Core relevance: async IO correctness.
- Bug type: correctness
- Proposed change: Update channel position after positional write.
- Trigger conditions: Async positional writes.
- Expected symptom: Channel position changes unexpectedly.
- Why it’s hard: Only in async usage.
- Static-analysis discoverability: Medium.
- Suggested detection: Async positional write should not move position.
- Rank: 4/3/4

### N145
- Location: JimfsInputStream#readInternal (approx lines 98-106)
- Core relevance: InputStream semantics.
- Bug type: correctness / metadata
- Proposed change: Always update lastAccessTime even when read returns -1.
- Trigger conditions: Read at EOF.
- Expected symptom: Access time updated on EOF reads.
- Why it’s hard: Subtle timestamp behavior.
- Static-analysis discoverability: Low.
- Suggested detection: EOF read should not change access time.
- Rank: 2/4/3

### N146
- Location: JimfsInputStream#skip (approx lines 124-127)
- Core relevance: InputStream semantics.
- Bug type: correctness
- Proposed change: Skip uses file.size() without lock and without bounds check.
- Trigger conditions: Concurrent writes and skips.
- Expected symptom: Skip beyond EOF or negative skip.
- Why it’s hard: Concurrency-dependent.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent write while skipping.
- Rank: 3/4/3

### N147
- Location: JimfsOutputStream#writeInternal (approx lines 83-87)
- Core relevance: OutputStream semantics.
- Bug type: correctness
- Proposed change: Use file.size() instead of sizeWithoutLocking for append.
- Trigger conditions: Append with concurrent writes.
- Expected symptom: Slowdowns or inconsistent position.
- Why it’s hard: Only under concurrency.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent append writes should preserve ordering.
- Rank: 3/4/3

### N148
- Location: JimfsOutputStream#close (approx lines 103-108)
- Core relevance: resource cleanup.
- Bug type: resource leak
- Proposed change: Remove file.closed() call.
- Trigger conditions: Streams closed after delete.
- Expected symptom: File contents never freed.
- Why it’s hard: Only after delete with open streams.
- Static-analysis discoverability: Medium.
- Suggested detection: Open, delete, close; disk space should be freed.
- Rank: 4/3/3

### N149
- Location: JimfsOutputStream#write(int) (approx lines 55-61)
- Core relevance: OutputStream behavior.
- Bug type: correctness / append
- Proposed change: For append, use file.size() (locked) but do not update pos.
- Trigger conditions: Multiple writes in append mode.
- Expected symptom: Each write overwrites previous append byte.
- Why it’s hard: Only in append mode with multiple writes.
- Static-analysis discoverability: Low.
- Suggested detection: Append writes should increase file size.
- Rank: 4/4/4

### N150
- Location: JimfsFileSystemProvider#isHidden (approx lines 295-309)
- Core relevance: hidden-file semantics.
- Bug type: correctness / platform behavior
- Proposed change: Use DOS hidden check even when DOS view unsupported.
- Trigger conditions: Unix-like config without dos view.
- Expected symptom: isHidden always false for dotfiles.
- Why it’s hard: Only with dotfiles.
- Static-analysis discoverability: Low.
- Suggested detection: isHidden on ".foo" should be true on Unix config.
- Rank: 3/4/3

### N151
- Location: PathURLConnection#connect (approx lines 76-94)
- Core relevance: URLConnection behavior.
- Bug type: correctness / content length
- Proposed change: For directories, use Files.size(path) instead of listing bytes length.
- Trigger conditions: URLConnection to directory.
- Expected symptom: content-length header incorrect.
- Why it’s hard: Only URLConnection usage.
- Static-analysis discoverability: Low.
- Suggested detection: Directory URLConnection content-length should match listing bytes.
- Rank: 2/4/2

### N152
- Location: PathURLConnection#connect (approx lines 104-106)
- Core relevance: HTTP date headers.
- Bug type: correctness / time
- Proposed change: Format last-modified using local timezone instead of GMT.
- Trigger conditions: Any URLConnection access.
- Expected symptom: Incorrect last-modified header.
- Why it’s hard: Subtle timezone issue.
- Static-analysis discoverability: Medium.
- Suggested detection: last-modified header should be GMT.
- Rank: 2/3/2

### N153
- Location: Util#nextPowerOf2 (approx lines 35-41)
- Core relevance: table sizing.
- Bug type: correctness / allocation
- Proposed change: Return 0 when n == 0.
- Trigger conditions: Initialization with 0 capacity.
- Expected symptom: zero-length arrays, division errors.
- Why it’s hard: Only for edge inputs.
- Static-analysis discoverability: Medium.
- Suggested detection: Directory table expansion with n=0.
- Rank: 3/3/3

### N154
- Location: Util#checkNotNegative (approx lines 47-48)
- Core relevance: validation.
- Bug type: correctness
- Proposed change: Allow -1 (use > -1 instead of >= 0).
- Trigger conditions: Negative positions or sizes.
- Expected symptom: Silent acceptance of invalid values.
- Why it’s hard: Only for invalid inputs.
- Static-analysis discoverability: Low.
- Suggested detection: Negative position should throw.
- Rank: 2/4/3

### N155
- Location: InternalCharMatcher (approx in InternalCharMatcher)
- Core relevance: glob/path matching.
- Bug type: correctness / character classes
- Proposed change: Return true for all chars when anyOf set empty.
- Trigger conditions: Empty separator set.
- Expected symptom: Matcher treats all characters as separators.
- Why it’s hard: Only in unusual configs.
- Static-analysis discoverability: Low.
- Suggested detection: PathMatchers with empty separator set.
- Rank: 2/4/2

### N156
- Location: Java8Compatibility.clear (approx in Java8Compatibility)
- Core relevance: ByteBuffer reuse.
- Bug type: correctness
- Proposed change: Do not clear buffers after transfer.
- Trigger conditions: transferTo on repeated calls.
- Expected symptom: Subsequent writes use wrong positions.
- Why it’s hard: Only in repeated operations.
- Static-analysis discoverability: Low.
- Suggested detection: transferTo called twice should produce same result.
- Rank: 3/4/3

### N157
- Location: DowngradedSeekableByteChannel#position (approx in DowngradedSeekableByteChannel)
- Core relevance: compatibility channel.
- Bug type: correctness / position tracking
- Proposed change: Return cached position instead of delegating to channel.
- Trigger conditions: External writes update position.
- Expected symptom: Stale position values.
- Why it’s hard: Only for downgraded channels.
- Static-analysis discoverability: Low.
- Suggested detection: Use newByteChannel when FILE_CHANNEL unsupported.
- Rank: 3/4/3

### N158
- Location: DowngradedDirectoryStream#iterator (approx in DowngradedDirectoryStream)
- Core relevance: directory listing.
- Bug type: correctness / resource handling
- Proposed change: Close underlying stream after first iterator call.
- Trigger conditions: Iterate over entries.
- Expected symptom: Missing entries or closed stream.
- Why it’s hard: Only when using downgraded stream.
- Static-analysis discoverability: Low.
- Suggested detection: Directory listing on FS without secure dir stream.
- Rank: 3/4/3

### N159
- Location: JimfsFileSystemProvider#newAsynchronousFileChannel (approx lines 176-182)
- Core relevance: async channel creation.
- Bug type: correctness
- Proposed change: Use executor parameter even when null (do not fallback).
- Trigger conditions: newAsynchronousFileChannel with executor null.
- Expected symptom: NullPointerException on async ops.
- Why it’s hard: Only in default executor path.
- Static-analysis discoverability: Medium.
- Suggested detection: newAsynchronousFileChannel(null) should work.
- Rank: 3/3/3

### N160
- Location: JimfsFileSystem#supportedFileAttributeViews (approx lines 249-251)
- Core relevance: attribute support discovery.
- Bug type: correctness
- Proposed change: Return empty set when file store closed.
- Trigger conditions: After close or during close.
- Expected symptom: Clients think no attributes supported.
- Why it’s hard: Only around close timing.
- Static-analysis discoverability: Low.
- Suggested detection: supportedFileAttributeViews should throw when closed.
- Rank: 2/4/2

### N161
- Location: JimfsFileSystemProvider#move (approx lines 272-273)
- Core relevance: move semantics.
- Bug type: correctness / option handling
- Proposed change: Call copy with Options.getCopyOptions instead of getMoveOptions.
- Trigger conditions: Move with NOFOLLOW_LINKS or ATOMIC_MOVE.
- Expected symptom: Wrong options; ATOMIC_MOVE not rejected.
- Why it’s hard: Only specific options.
- Static-analysis discoverability: Low.
- Suggested detection: move with ATOMIC_MOVE should throw.
- Rank: 3/4/3

### N162
- Location: PathService#emptyPath (approx lines 114-121)
- Core relevance: empty path identity.
- Bug type: correctness / caching
- Proposed change: Recreate empty path on each call (remove cached field).
- Trigger conditions: Path equality/hash in collections.
- Expected symptom: Increased allocations, subtle identity assumptions break.
- Why it’s hard: Performance-only; semantic ok.
- Static-analysis discoverability: Low.
- Suggested detection: Profiling or identity-based tests.
- Rank: 2/5/2

### N163
- Location: PathService#name (approx lines 126-137)
- Core relevance: name normalization.
- Bug type: correctness
- Proposed change: Treat empty string as Name.SELF instead of Name.EMPTY.
- Trigger conditions: Paths with empty segments.
- Expected symptom: Empty segments normalized to ".".
- Why it’s hard: Only on edge inputs.
- Static-analysis discoverability: Medium.
- Suggested detection: parsePath("") should produce empty name, not ".".
- Rank: 3/3/3

### N164
- Location: PathService#hash (approx lines 207-224)
- Core relevance: hash/equals contract.
- Bug type: correctness
- Proposed change: Exclude root from hash in canonical form.
- Trigger conditions: Paths on different roots.
- Expected symptom: Hash collisions across roots.
- Why it’s hard: Only with multi-root config.
- Static-analysis discoverability: Medium.
- Suggested detection: HashSet with paths on different roots.
- Rank: 3/3/3

### N165
- Location: PathType#parseUriPath (abstract; used by Unix/Windows)
- Core relevance: URI parsing.
- Bug type: correctness
- Proposed change: Allow empty uriPath without returning emptyPath.
- Trigger conditions: URI with empty path.
- Expected symptom: Invalid empty path accepted.
- Why it’s hard: Rare URI forms.
- Static-analysis discoverability: Low.
- Suggested detection: fromUri with empty path should error.
- Rank: 2/4/2

### N166
- Location: UnixPathType#parsePath (approx lines 39-47)
- Core relevance: path parsing.
- Bug type: correctness / root detection
- Proposed change: Treat any path starting with "//" as having a null root.
- Trigger conditions: Paths with double leading slash.
- Expected symptom: Absolute paths become relative.
- Why it’s hard: Only for "//" prefix.
- Static-analysis discoverability: Low.
- Suggested detection: "//a" should be absolute.
- Rank: 3/4/3

### N167
- Location: WindowsPathType#parsePath (approx lines 96-104)
- Core relevance: Windows root handling.
- Bug type: correctness
- Proposed change: Do not append trailing "\\" to root.
- Trigger conditions: Root-only paths.
- Expected symptom: Path rendering missing separator.
- Why it’s hard: Only on root paths.
- Static-analysis discoverability: Low.
- Suggested detection: toString of root should end with "\\".
- Rank: 2/4/2

### N168
- Location: PathMatchers#getPathMatcher (approx lines 55-63)
- Core relevance: syntax parsing.
- Bug type: correctness
- Proposed change: Lowercase the entire pattern instead of syntax only.
- Trigger conditions: Case-sensitive regex patterns.
- Expected symptom: Pattern meaning changes.
- Why it’s hard: Only for patterns with uppercase letters.
- Static-analysis discoverability: Low.
- Suggested detection: regex "[A-Z]" should remain case-sensitive.
- Rank: 3/4/3

### N169
- Location: GlobToRegex#appendSeparator (approx lines 128-137)
- Core relevance: glob matching.
- Bug type: correctness
- Proposed change: Always use single separator even when multiple configured.
- Trigger conditions: Windows config with "/" and "\\" separators.
- Expected symptom: Glob fails for alternate separator.
- Why it’s hard: Only on Windows config.
- Static-analysis discoverability: Medium.
- Suggested detection: glob should match paths with either separator.
- Rank: 3/3/3

### N170
- Location: AttributeService#setInitialAttributes (approx lines 155-168)
- Core relevance: file creation defaults.
- Bug type: correctness / defaults
- Proposed change: Apply user-provided attrs before default values.
- Trigger conditions: Files created with explicit attributes.
- Expected symptom: Defaults overwrite user-specified attributes.
- Why it’s hard: Only when defaults and explicit attrs overlap.
- Static-analysis discoverability: Medium.
- Suggested detection: Explicit owner/group should override defaults.
- Rank: 4/3/4

### N171
- Location: AttributeService#readAttributes (approx lines 330-332)
- Core relevance: attribute reads.
- Bug type: correctness
- Proposed change: Use getAttribute(file, view, attr) without checking existence first, causing NPE on null.
- Trigger conditions: Read attribute that exists transiently.
- Expected symptom: NPE instead of IllegalArgumentException.
- Why it’s hard: Only under concurrent attribute deletes.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent delete while reading attributes.
- Rank: 3/4/3

### N172
- Location: BasicAttributeProvider.Attributes (approx lines 183-191)
- Core relevance: attribute snapshot consistency.
- Bug type: correctness / timing
- Proposed change: Read times from file without synchronization (remove sync).
- Trigger conditions: Concurrent updates.
- Expected symptom: Inconsistent timestamps across attributes.
- Why it’s hard: Only under concurrent updates.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent time updates and attribute reads.
- Rank: 3/4/3

### N173
- Location: OwnerAttributeProvider.View#getOwner (approx lines 113-115)
- Core relevance: owner view semantics.
- Bug type: correctness
- Proposed change: Return default owner instead of stored attribute.
- Trigger conditions: Owner set explicitly.
- Expected symptom: getOwner ignores setOwner.
- Why it’s hard: Only if owner is changed.
- Static-analysis discoverability: Medium.
- Suggested detection: setOwner then getOwner should return same.
- Rank: 3/3/4

### N174
- Location: PosixAttributeProvider.View#setPermissions (approx in PosixAttributeProvider)
- Core relevance: permissions enforcement.
- Bug type: correctness
- Proposed change: Accept null permissions and clear to empty set.
- Trigger conditions: setPermissions(null).
- Expected symptom: File becomes unusable; permissions empty.
- Why it’s hard: Only with invalid input.
- Static-analysis discoverability: Low.
- Suggested detection: setPermissions(null) should throw.
- Rank: 2/4/3

### N175
- Location: UnixAttributeProvider#attributes (approx in UnixAttributeProvider)
- Core relevance: unix view enumeration.
- Bug type: correctness
- Proposed change: Include "ctime" twice and omit "nlink".
- Trigger conditions: unix:* attribute read.
- Expected symptom: Missing nlink attribute.
- Why it’s hard: Only for unix view users.
- Static-analysis discoverability: Low.
- Suggested detection: unix:* should include nlink.
- Rank: 2/4/2

### N176
- Location: DosAttributeProvider#get (approx in DosAttributeProvider)
- Core relevance: DOS attribute view.
- Bug type: correctness
- Proposed change: Return default false for attributes even when set.
- Trigger conditions: Setting dos:hidden or dos:readonly.
- Expected symptom: getAttributes always false.
- Why it’s hard: Only DOS view users.
- Static-analysis discoverability: Medium.
- Suggested detection: set dos:hidden then read back.
- Rank: 3/3/3

### N177
- Location: AclAttributeProvider#set (approx in AclAttributeProvider)
- Core relevance: ACL attributes.
- Bug type: correctness / validation
- Proposed change: Accept null ACL list and store null.
- Trigger conditions: Setting ACLs.
- Expected symptom: NPE on read or permission checks.
- Why it’s hard: Only ACL view used.
- Static-analysis discoverability: Low.
- Suggested detection: setAcl(null) should throw.
- Rank: 2/4/2

### N178
- Location: UserDefinedAttributeProvider#attributes (approx lines 60-69)
- Core relevance: user attribute enumeration.
- Bug type: correctness
- Proposed change: Return empty set when attributes exist (always empty).
- Trigger conditions: list user attributes.
- Expected symptom: list() returns empty even with attributes.
- Why it’s hard: Only for user-defined attrs.
- Static-analysis discoverability: Medium.
- Suggested detection: set user attr then list should include it.
- Rank: 3/3/3

### N179
- Location: StandardAttributeProviders#get (approx in StandardAttributeProviders)
- Core relevance: provider registry.
- Bug type: correctness
- Proposed change: Map "posix" to OwnerAttributeProvider by mistake.
- Trigger conditions: posix view requested.
- Expected symptom: posix view missing permissions/group.
- Why it’s hard: Only when posix view used.
- Static-analysis discoverability: Low.
- Suggested detection: posix view should expose permissions/group.
- Rank: 4/4/3

### N180
- Location: JimfsFileSystemProvider#createLink (approx lines 221-228)
- Core relevance: hard link creation.
- Bug type: correctness / cross-FS
- Proposed change: Remove file system equality check for existing path.
- Trigger conditions: Linking across different FS.
- Expected symptom: link to foreign FS allowed.
- Why it’s hard: Only multi-FS.
- Static-analysis discoverability: Medium.
- Suggested detection: createLink across FS should fail.
- Rank: 4/3/4

### N181
- Location: JimfsFileSystemProvider#createSymbolicLink (approx lines 232-239)
- Core relevance: symlink creation.
- Bug type: correctness
- Proposed change: Require link and target belong to same FS (currently ok); bug: skip check.
- Trigger conditions: Symlink to path from another FS.
- Expected symptom: Symlink target invalid or resolves incorrectly.
- Why it’s hard: Only multi-FS.
- Static-analysis discoverability: Medium.
- Suggested detection: Symlink across FS should fail.
- Rank: 3/3/3

### N182
- Location: JimfsFileSystemProvider#isSameFile (approx lines 277-292)
- Core relevance: Files.isSameFile semantics.
- Bug type: correctness
- Proposed change: Return true when both paths do not exist.
- Trigger conditions: isSameFile on missing paths.
- Expected symptom: Returns true; spec should throw.
- Why it’s hard: Only when paths missing.
- Static-analysis discoverability: Medium.
- Suggested detection: isSameFile on missing should throw NoSuchFileException.
- Rank: 3/3/3

### N183
- Location: JimfsFileSystemProvider#isHidden (approx lines 295-309)
- Core relevance: hidden file semantics.
- Bug type: correctness
- Proposed change: Use path.getFileName().toString() even when path has zero names (root).
- Trigger conditions: isHidden on root.
- Expected symptom: NullPointerException or false positives.
- Why it’s hard: Only root path usage.
- Static-analysis discoverability: Low.
- Suggested detection: isHidden on root should be false.
- Rank: 2/4/2

### N184
- Location: JimfsFileSystemProvider#checkAccess (approx lines 318-320)
- Core relevance: access checks.
- Bug type: correctness / modes ignored
- Proposed change: Ignore AccessMode args; always allow if exists.
- Trigger conditions: AccessMode.WRITE on read-only configs (future).
- Expected symptom: Access allowed when should fail.
- Why it’s hard: Depends on future permission features.
- Static-analysis discoverability: Low.
- Suggested detection: AccessMode checks should enforce permissions if supported.
- Rank: 2/3/2

### N185
- Location: JimfsFileChannel#transferFrom (approx lines 553-557)
- Core relevance: transferFrom with append.
- Bug type: correctness / append
- Proposed change: Use size() (locking) but do not update position after transfer.
- Trigger conditions: APPEND + transferFrom.
- Expected symptom: position stale, next write overwrites data.
- Why it’s hard: Only in append + transferFrom.
- Static-analysis discoverability: Medium.
- Suggested detection: Append transferFrom should advance position.
- Rank: 4/3/4

### N186
- Location: JimfsFileChannel#size (approx lines 433-438)
- Core relevance: size queries.
- Bug type: correctness / concurrency
- Proposed change: Remove read lock, read sizeWithoutLocking directly.
- Trigger conditions: concurrent writes.
- Expected symptom: size values torn/stale.
- Why it’s hard: Race-dependent.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent read/write size consistency.
- Rank: 3/4/3

### N187
- Location: JimfsFileChannel#lock (approx lines 606-610)
- Core relevance: lock semantics.
- Bug type: correctness / interrupt handling
- Proposed change: Call end(false) unconditionally even on success.
- Trigger conditions: lock() call.
- Expected symptom: ClosedByInterruptException incorrectly thrown.
- Why it’s hard: Only in lock usage.
- Static-analysis discoverability: Low.
- Suggested detection: lock() should succeed without interruption.
- Rank: 3/4/3

### N188
- Location: JimfsAsynchronousFileChannel#closedChannelFuture (approx lines 184-187)
- Core relevance: async error signaling.
- Bug type: correctness
- Proposed change: Return completed future with null instead of ClosedChannelException.
- Trigger conditions: Async operation on closed channel.
- Expected symptom: Callers treat as success with null.
- Why it’s hard: Only for closed channels.
- Static-analysis discoverability: Medium.
- Suggested detection: Async on closed channel should fail.
- Rank: 3/3/3

### N189
- Location: JimfsInputStream#available (approx lines 136-138)
- Core relevance: InputStream semantics.
- Bug type: correctness
- Proposed change: Return (int) file.size() ignoring current pos.
- Trigger conditions: available() after reading some bytes.
- Expected symptom: available over-reports remaining bytes.
- Why it’s hard: Only via available() method.
- Static-analysis discoverability: Medium.
- Suggested detection: available() should decrease after reads.
- Rank: 2/3/3

### N190
- Location: JimfsOutputStream#write(byte[], int, int) (approx lines 72-76)
- Core relevance: OutputStream correctness.
- Bug type: correctness / bounds
- Proposed change: Use checkPositionIndexes(off, len, b.length) (wrong end index).
- Trigger conditions: Writes with offset/length.
- Expected symptom: Accepts invalid ranges or rejects valid ones.
- Why it’s hard: Only with offsets.
- Static-analysis discoverability: Medium.
- Suggested detection: write with offset should succeed.
- Rank: 3/3/3

### N191
- Location: PathURLConnection#getHeaderField (approx lines 138-146)
- Core relevance: header retrieval.
- Bug type: correctness
- Proposed change: Return first header without lowercasing name.
- Trigger conditions: Requesting mixed-case header.
- Expected symptom: Header lookup fails for non-lowercase keys.
- Why it’s hard: Only for mixed-case usage.
- Static-analysis discoverability: Low.
- Suggested detection: getHeaderField("Content-Type") should work.
- Rank: 2/4/2

### N192
- Location: PathService#createPathMatcher (approx lines 257-261)
- Core relevance: PathMatcher creation.
- Bug type: correctness
- Proposed change: Pass displayNormalizations regardless of equalityUsesCanonicalForm.
- Trigger conditions: Case-insensitive configs.
- Expected symptom: PathMatcher mismatches actual path equality rules.
- Why it’s hard: Only for case-insensitive configs.
- Static-analysis discoverability: Medium.
- Suggested detection: glob matches should be case-insensitive when configured.
- Rank: 4/3/4

### N193
- Location: FileSystemView#snapshotModifiedTimes (approx lines 165-168)
- Core relevance: watch service snapshots.
- Bug type: correctness / stale snapshot
- Proposed change: Skip updating lastAccessTime for directory entries.
- Trigger conditions: Watch service polling.
- Expected symptom: ENTRY_MODIFY events missed for timestamp changes.
- Why it’s hard: Only with watch service and timestamp-based logic.
- Static-analysis discoverability: Low.
- Suggested detection: Watch service should emit modify on write.
- Rank: 4/4/3

### N194
- Location: FileSystemView#link (approx lines 432-434)
- Core relevance: metadata.
- Bug type: correctness / timestamps
- Proposed change: Update lastAccessTime instead of lastModifiedTime on link creation.
- Trigger conditions: Creating hard links.
- Expected symptom: Modified time unchanged.
- Why it’s hard: Timestamp-only difference.
- Static-analysis discoverability: Low.
- Suggested detection: Link creation should update modified time.
- Rank: 3/4/3

### N195
- Location: FileSystemView#delete (approx lines 456-460)
- Core relevance: delete semantics.
- Bug type: correctness / metadata
- Proposed change: Remove file.deleted() call after unlink.
- Trigger conditions: Delete regular file with open streams.
- Expected symptom: Content not freed after last close.
- Why it’s hard: Only after open-delete.
- Static-analysis discoverability: Low.
- Suggested detection: Delete open file; contents should free after close.
- Rank: 4/4/3

### N196
- Location: FileSystemView#checkEmpty (approx lines 498-501)
- Core relevance: directory delete semantics.
- Bug type: correctness
- Proposed change: Treat directory as empty if entryCount <= 3.
- Trigger conditions: Directory with one child.
- Expected symptom: Non-empty directory deletion allowed.
- Why it’s hard: Only for small directories.
- Static-analysis discoverability: Low.
- Suggested detection: Delete non-empty dir should fail.
- Rank: 4/4/3

### N197
- Location: FileSystemView#copy (approx lines 552-558)
- Core relevance: move within same FS.
- Bug type: correctness / metadata
- Proposed change: Skip updating destParent lastModifiedTime.
- Trigger conditions: Move file within same FS.
- Expected symptom: Parent directory modified time unchanged.
- Why it’s hard: Only timestamp-related.
- Static-analysis discoverability: Low.
- Suggested detection: Parent mtime should update on move.
- Rank: 2/4/3

### N198
- Location: JimfsFileStore#copyWithoutContent (approx lines 152-156)
- Core relevance: copy semantics.
- Bug type: correctness
- Proposed change: Skip attributes.copyAttributes call.
- Trigger conditions: COPY_ATTRIBUTES or move across FS.
- Expected symptom: Attributes lost.
- Why it’s hard: Only if attributes inspected.
- Static-analysis discoverability: Medium.
- Suggested detection: Copy should preserve basic attributes at least.
- Rank: 3/3/4

### N199
- Location: FileFactory#copyWithoutContent (approx lines 73-75)
- Core relevance: copy semantics.
- Bug type: correctness / timestamps
- Proposed change: Use original file creation time instead of now.
- Trigger conditions: Copy operations.
- Expected symptom: Copied file has same creation time as source.
- Why it’s hard: Timestamp semantics subtle.
- Static-analysis discoverability: Medium.
- Suggested detection: Copied file should have new creation time unless COPY_ATTRIBUTES.
- Rank: 2/3/3

### N200
- Location: JimfsFileSystemProvider#newFileSystem(Path, Map) (approx lines 91-105)
- Core relevance: ZipFileSystem integration.
- Bug type: correctness / error handling
- Proposed change: Catch Exception and return null instead of throwing UnsupportedOperationException.
- Trigger conditions: Non-zip path passed to newFileSystem(Path,...).
- Expected symptom: NullPointerException downstream.
- Why it’s hard: Only when non-zip path used.
- Static-analysis discoverability: Medium.
- Suggested detection: newFileSystem(Path) should throw UnsupportedOperationException on non-zip.
- Rank: 3/3/3

## Top 10 recommended set
- N08: delete under readLock (race; core path, high stealth).
- N09: move directory into subtree (integrity and traversal failures).
- N31: missing block allocation (boundary correctness; subtle).
- N42: copyBlocksTo off-by-one (corruption via cache reuse).
- N46: delete ignores hard links (data loss across links).
- N61: register increment after add (close race, hard to spot).
- N66: default thread pool leak (resource leak, long-running tests).
- N88: Name.equals uses display (breaks canonicalization).
- N91: parsePath leaves empty components (edge-case input).
- N101: glob treated as regex (pattern semantics drift).
