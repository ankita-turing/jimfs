# Debugging Exercise Proposal (Jimfs)

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

## Bug candidates

### B01
- Location: jimfs/src/main/java/com/google/common/jimfs/FileTree.java, lookUpLast(...) around lines 150-169
- Core relevance: central to all path lookup; every file operation resolves paths here.
- Bug type: correctness / symlink handling / security
- Proposed change: In lookUpLast, follow symlinks even when NOFOLLOW_LINKS is present (invert the LinkOption check).
- Trigger conditions: Any API call using LinkOption.NOFOLLOW_LINKS on a symlink path (readAttributes, delete, etc.).
- Expected symptom: Operations affect the symlink target instead of the link itself.
- Why it is hard: Only visible for symlink + NOFOLLOW_LINKS combinations; normal usage still works.
- Static-analysis discoverability: Medium; requires semantic knowledge of LinkOption.
- Suggested detection: Unit test for Files.readAttributes(link, NOFOLLOW_LINKS) and delete(link, NOFOLLOW_LINKS).
- Rank (value/stealth/scorability): 5/4/5

### B02
- Location: jimfs/src/main/java/com/google/common/jimfs/FileTree.java, followSymbolicLink(...) around lines 176-183
- Core relevance: symlink resolution is part of every path lookup.
- Bug type: correctness / boundary / reliability
- Proposed change: Change the depth check from >= MAX_SYMBOLIC_LINK_DEPTH to > MAX_SYMBOLIC_LINK_DEPTH.
- Trigger conditions: Symlink chains of depth exactly 40 (or cycles near the threshold).
- Expected symptom: Excessive symlink depth allowed; potentially very deep recursion before failure.
- Why it is hard: Rare input and failure can look like a generic IO error.
- Static-analysis discoverability: Low to medium; off-by-one in guard logic is subtle.
- Suggested detection: Property test generating chains of N symlinks and expecting failure at threshold.
- Rank (value/stealth/scorability): 3/4/4

### B03
- Location: jimfs/src/main/java/com/google/common/jimfs/FileSystemView.java, copy(...) around lines 563-576
- Core relevance: Files.copy/move use this path; core I/O behavior.
- Bug type: data integrity / API contract drift
- Proposed change: When COPY_ATTRIBUTES is requested and same file system, use BASIC instead of ALL attributes.
- Trigger conditions: Files.copy(source, target, COPY_ATTRIBUTES) with non-basic attribute views.
- Expected symptom: Attributes like posix/dos/user are silently dropped.
- Why it is hard: Only appears when callers check non-basic attributes post-copy.
- Static-analysis discoverability: Low; code still looks reasonable.
- Suggested detection: Integration test copying a file with POSIX permissions or DOS attributes.
- Rank (value/stealth/scorability): 4/4/5 (likely too easy unless attribute-copy tests removed)

### B04
- Location: jimfs/src/main/java/com/google/common/jimfs/FileSystemView.java, lockSourceAndCopy(...) around lines 673-679
- Core relevance: copy/move across file systems; interacts with delete and open count.
- Bug type: concurrency / data integrity
- Proposed change: Remove sourceFile.opened() from lockSourceAndCopy.
- Trigger conditions: Copy/move while another thread deletes the source file.
- Expected symptom: Copy occasionally ends up empty or truncated because source content is freed mid-copy.
- Why it is hard: Race-dependent, non-deterministic, hard to reproduce.
- Static-analysis discoverability: Low; looks like an innocuous removal.
- Suggested detection: Concurrent copy/delete stress test with verification of copied content.
- Rank (value/stealth/scorability): 5/5/4

### B05
- Location: jimfs/src/main/java/com/google/common/jimfs/FileSystemView.java, checkDeletable(...) around lines 474-493
- Core relevance: delete semantics are part of core file ops.
- Bug type: correctness / edge-case input
- Proposed change: Flip the working-directory-relative-path guard to only reject absolute paths.
- Trigger conditions: Deleting the working directory using a relative path ("" or ".").
- Expected symptom: Relative deletes unexpectedly succeed (or absolute deletes unexpectedly fail).
- Why it is hard: This is an odd, OS-specific edge case.
- Static-analysis discoverability: Medium; requires knowledge of Unix behavior.
- Suggested detection: Unit test that delete(".") fails while delete("/work") succeeds.
- Rank (value/stealth/scorability): 3/4/4

### B06
- Location: jimfs/src/main/java/com/google/common/jimfs/FileSystemView.java, copy(...) around lines 601-614
- Core relevance: Files.copy is a hot path under test suites and builds.
- Bug type: performance regression (non-obvious)
- Proposed change: Move sourceFile.copyContentTo(copyFile) inside the file store lock section.
- Trigger conditions: Concurrent file operations or large file copies.
- Expected symptom: Severe lock contention and slowdowns under parallel copy/delete workloads.
- Why it is hard: Behavior remains correct; only performance degrades under load.
- Static-analysis discoverability: Low; requires performance intuition.
- Suggested detection: Load test that copies many files in parallel and measures throughput.
- Rank (value/stealth/scorability): 4/5/4

### B07
- Location: jimfs/src/main/java/com/google/common/jimfs/FileSystemView.java, open(...) around lines 355-363
- Core relevance: file opens and truncation are on the hot path for streams/channels.
- Bug type: correctness / option handling
- Proposed change: Check TRUNCATE_EXISTING + READ instead of TRUNCATE_EXISTING + WRITE.
- Trigger conditions: Opening a file write-only with TRUNCATE_EXISTING (default output stream case).
- Expected symptom: File is not truncated; stale data remains after write.
- Why it is hard: Only visible when writing shorter content than the existing file.
- Static-analysis discoverability: Medium; small condition change.
- Suggested detection: Unit test that open(TRUNCATE_EXISTING, WRITE) zeroes file.
- Rank (value/stealth/scorability): 3/3/4 (likely too easy unless output-stream tests removed)

### B08
- Location: jimfs/src/main/java/com/google/common/jimfs/RegularFile.java, write(long pos, byte b) around lines 288-299
- Core relevance: core write path used by channels and streams.
- Bug type: off-by-one / correctness
- Proposed change: Update size only when pos > size (not >=).
- Trigger conditions: Writing a byte at exactly EOF (append case).
- Expected symptom: File size fails to grow by one; last byte not visible to readers.
- Why it is hard: Only triggers on boundary; many tests do not cover exact EOF writes.
- Static-analysis discoverability: Low to medium.
- Suggested detection: Unit test that writing a single byte at size() increases length.
- Rank (value/stealth/scorability): 3/4/4

### B09
- Location: jimfs/src/main/java/com/google/common/jimfs/RegularFile.java, prepareForWrite(...) around lines 261-278
- Core relevance: used by all write paths and sparse writes.
- Bug type: data integrity / security (stale data exposure)
- Proposed change: Off-by-one in gap zeroing so one byte between old size and new pos is not zeroed.
- Trigger conditions: Seek past EOF then write, then read the gap.
- Expected symptom: Read exposes previous data from reused blocks instead of zeros.
- Why it is hard: Only occurs with sparse writes and subsequent reads of the gap.
- Static-analysis discoverability: Low; off-by-one is subtle.
- Suggested detection: Property test for sparse writes expecting zero-filled gaps.
- Rank (value/stealth/scorability): 4/5/4

### B10
- Location: jimfs/src/main/java/com/google/common/jimfs/RegularFile.java, transferFrom(...) around lines 433-446
- Core relevance: used by FileChannel.transferFrom, a core API.
- Bug type: resource leak / reliability
- Proposed change: Remove disk.free(...) in the "read 0 bytes" path after allocating a new block.
- Trigger conditions: transferFrom on a non-blocking channel that returns 0 when no data is available.
- Expected symptom: Disk space leaks and eventually "out of disk space" even for small files.
- Why it is hard: Requires specific channel behavior; leak is gradual.
- Static-analysis discoverability: Low.
- Suggested detection: Repeated transferFrom on a non-blocking channel with no data; check free space.
- Rank (value/stealth/scorability): 4/4/4

### B11
- Location: jimfs/src/main/java/com/google/common/jimfs/RegularFile.java, transferTo(...) around lines 598-599
- Core relevance: transferTo is part of FileChannel API.
- Bug type: API contract drift
- Proposed change: Return bytesToRead directly (may be -1) instead of max(bytesToRead, 0).
- Trigger conditions: transferTo with position >= size.
- Expected symptom: Returns -1 instead of 0, breaking callers that expect non-negative.
- Why it is hard: Only at EOF and only for transferTo.
- Static-analysis discoverability: Medium; return value looks reasonable without spec knowledge.
- Suggested detection: Unit test for transferTo at EOF.
- Rank (value/stealth/scorability): 3/3/5 (likely too easy unless FileChannel tests removed)

### B12
- Location: jimfs/src/main/java/com/google/common/jimfs/HeapDisk.java, allocate(...) around lines 127-135
- Core relevance: central block allocation for all regular files.
- Bug type: caching/invalidation / data integrity
- Proposed change: Use blockCache.copyBlocksTo(...) instead of transferBlocksTo(...) when reusing cached blocks.
- Trigger conditions: Block cache enabled and multiple files allocate cached blocks.
- Expected symptom: Two files share the same block array, causing cross-file data corruption.
- Why it is hard: Requires cache reuse and specific write patterns; corruption may appear far away.
- Static-analysis discoverability: Low; both methods look plausible.
- Suggested detection: Stress test creating, deleting, and rewriting many files; compare contents.
- Rank (value/stealth/scorability): 4/5/4

### B13
- Location: jimfs/src/main/java/com/google/common/jimfs/HeapDisk.java, free(...) around lines 146-154
- Core relevance: affects free space accounting for all files.
- Bug type: caching / resource accounting
- Proposed change: Decrement allocatedBlockCount only for blocks not cached (treat cached blocks as still allocated).
- Trigger conditions: Files created/deleted with cache enabled.
- Expected symptom: Usable space shrinks over time; premature out-of-space errors.
- Why it is hard: Gradual degradation; looks like capacity tuning issue.
- Static-analysis discoverability: Low.
- Suggested detection: Loop create/delete while checking getUsableSpace stays stable.
- Rank (value/stealth/scorability): 4/5/4

### B14
- Location: jimfs/src/main/java/com/google/common/jimfs/HeapDisk.java, getTotalSpace() around lines 107-108
- Core relevance: file store space reporting is used by external tools and tests.
- Bug type: numeric precision / overflow
- Proposed change: Compute maxBlockCount * blockSize as int then cast to long.
- Trigger conditions: Large maxSize or blockSize leading to int overflow.
- Expected symptom: Total space reported negative or incorrect; quota logic misbehaves.
- Why it is hard: Only visible at large configurations; normal defaults unaffected.
- Static-analysis discoverability: Medium; requires overflow awareness.
- Suggested detection: Unit test with large maxSize values.
- Rank (value/stealth/scorability): 3/4/4

### B15
- Location: jimfs/src/main/java/com/google/common/jimfs/Directory.java, snapshot() around lines 149-159
- Core relevance: directory listing and watch service rely on snapshots.
- Bug type: correctness / data integrity
- Proposed change: Remove the reserved-name check so "." and ".." are included.
- Trigger conditions: Listing directories or using watch service snapshots.
- Expected symptom: Users see "." and ".." entries; watch service may emit spurious events.
- Why it is hard: Looks like a minor difference from POSIX behavior.
- Static-analysis discoverability: Low to medium.
- Suggested detection: Unit test expecting directory listings to exclude "." and "..".
- Rank (value/stealth/scorability): 3/3/4 (likely too easy unless directory tests removed)

### B16
- Location: jimfs/src/main/java/com/google/common/jimfs/Directory.java, put(...) around lines 236-252
- Core relevance: every file creation and link operation goes through Directory.put.
- Bug type: data integrity / lifecycle
- Proposed change: Remove entry.file().incrementLinkCount() for new entries.
- Trigger conditions: Creating files or links, then deleting while open.
- Expected symptom: File contents freed prematurely; open streams see data loss.
- Why it is hard: Only appears when link counts matter (hard links, open-delete).
- Static-analysis discoverability: Low.
- Suggested detection: Create file, open stream, delete file, keep reading; should remain valid.
- Rank (value/stealth/scorability): 4/4/4

### B17
- Location: jimfs/src/main/java/com/google/common/jimfs/FileSystemView.java, snapshotWorkingDirectoryEntries() around lines 140-146
- Core relevance: directory listing and watch service depend on access time updates.
- Bug type: time metadata mix-up
- Proposed change: Update lastModifiedTime instead of lastAccessTime when snapshotting.
- Trigger conditions: Listing directories or polling watch service.
- Expected symptom: Directory modified time changes on read; time-based tools behave oddly.
- Why it is hard: Only visible if timestamps are inspected; no functional failure.
- Static-analysis discoverability: Low to medium.
- Suggested detection: Unit test verifying lastAccessTime changes but lastModifiedTime does not.
- Rank (value/stealth/scorability): 4/4/4

### B18
- Location: jimfs/src/main/java/com/google/common/jimfs/FileSystemState.java, close() around lines 113-136
- Core relevance: governs all resource cleanup when file system is closed.
- Bug type: concurrency / resource leak
- Proposed change: Change loop condition to (registering > 0 && !resources.isEmpty()).
- Trigger conditions: Concurrent register calls during close.
- Expected symptom: Some resources never closed; file system appears closed but threads/streams remain.
- Why it is hard: Race condition, intermittent and hard to reproduce.
- Static-analysis discoverability: Low.
- Suggested detection: Stress test concurrent open/close and verify all resources are closed.
- Rank (value/stealth/scorability): 4/5/3

### B19
- Location: jimfs/src/main/java/com/google/common/jimfs/FileSystemState.java, register(...) around lines 81-88
- Core relevance: all streams/channels register here.
- Bug type: concurrency / lifecycle
- Proposed change: Remove the second checkOpen() after incrementing registering.
- Trigger conditions: Registering resources concurrently with close().
- Expected symptom: Resources registered after close never get closed.
- Why it is hard: Requires tight interleaving; easy to miss in review.
- Static-analysis discoverability: Low.
- Suggested detection: Concurrent test that closes FS while opening channels and asserts no leaked resources.
- Rank (value/stealth/scorability): 4/4/3

### B20
- Location: jimfs/src/main/java/com/google/common/jimfs/PathService.java, name(...) around lines 135-137
- Core relevance: every path component uses this normalization.
- Bug type: correctness / API contract drift
- Proposed change: Swap display and canonical normalization when creating Name objects.
- Trigger conditions: Configurations with normalization or case folding (OSX/Windows-like).
- Expected symptom: Path equality and lookup mismatch; case-insensitive behavior breaks.
- Why it is hard: Only appears in non-default configurations.
- Static-analysis discoverability: Low to medium.
- Suggested detection: Unit test for case-insensitive lookups and equality.
- Rank (value/stealth/scorability): 4/4/4 (likely too easy unless PathService tests removed)

### B21
- Location: jimfs/src/main/java/com/google/common/jimfs/PathService.java, hash(...) around lines 213-224
- Core relevance: JimfsPath hashing used in maps/sets across the system.
- Bug type: correctness / hash-equals contract
- Proposed change: Always use canonical hash even when equalityUsesCanonicalForm is false.
- Trigger conditions: Using JimfsPath in HashMap/HashSet on Unix-like config.
- Expected symptom: Hash-based collections behave inconsistently (lookups fail).
- Why it is hard: Only appears in hash-based collections; equality still correct.
- Static-analysis discoverability: Low.
- Suggested detection: Property test for HashMap get/put with paths that differ by display case.
- Rank (value/stealth/scorability): 4/5/4

### B22
- Location: jimfs/src/main/java/com/google/common/jimfs/PathService.java, createPath(...) around lines 167-173
- Core relevance: all path creation, normalization, and parsing.
- Bug type: input validation / edge case
- Proposed change: Remove filtering of empty names (NOT_EMPTY).
- Trigger conditions: Paths with repeated separators or trailing separators.
- Expected symptom: Paths that should be equivalent become different; lookups fail or create empty-name entries.
- Why it is hard: Only visible on edge-case inputs.
- Static-analysis discoverability: Low.
- Suggested detection: Tests for "a//b" and "a/b/" equivalence.
- Rank (value/stealth/scorability): 3/4/4

### B23
- Location: jimfs/src/main/java/com/google/common/jimfs/JimfsPath.java, normalize() around lines 208-215
- Core relevance: normalization affects most path operations (resolve, toRealPath).
- Bug type: security / correctness
- Proposed change: Allow ".." above root for absolute paths (change condition so ".." is retained).
- Trigger conditions: normalize("/../secret") or similar absolute paths.
- Expected symptom: Normalized path escapes root; can resolve to invalid locations.
- Why it is hard: Only seen with explicit normalization; not obvious in static review.
- Static-analysis discoverability: Medium.
- Suggested detection: Unit test ensuring "/../x" normalizes to "/x" not "../x".
- Rank (value/stealth/scorability): 5/4/5

### B24
- Location: jimfs/src/main/java/com/google/common/jimfs/JimfsPath.java, relativize(...) around lines 320-327
- Core relevance: used by Path.relativize across many APIs.
- Bug type: correctness / canonical vs display mismatch
- Proposed change: Use reference equality (==) instead of equals for name comparison.
- Trigger conditions: Case-insensitive or normalized file system configurations.
- Expected symptom: Incorrect relative paths with extra ".." segments.
- Why it is hard: Only appears on specific configurations and name variants.
- Static-analysis discoverability: Low to medium.
- Suggested detection: Unit test for relativize with same canonical names but different case.
- Rank (value/stealth/scorability): 3/4/4

### B25
- Location: jimfs/src/main/java/com/google/common/jimfs/Options.java, getLinkOptions(...) around lines 62-64
- Core relevance: affects link-follow behavior across most file operations.
- Bug type: API contract drift / security
- Proposed change: Invert logic so NOFOLLOW_LINKS is ignored (always FOLLOW_LINKS).
- Trigger conditions: Any caller passing LinkOption.NOFOLLOW_LINKS.
- Expected symptom: Symlinks always followed; link-specific operations behave on target.
- Why it is hard: Only surfaces when NOFOLLOW_LINKS is explicitly used.
- Static-analysis discoverability: Medium.
- Suggested detection: Unit test for NOFOLLOW_LINKS in Files.isSymbolicLink / readAttributes.
- Rank (value/stealth/scorability): 4/3/5 (likely too easy unless link-option tests removed)

### B26
- Location: jimfs/src/main/java/com/google/common/jimfs/Options.java, getOptionsForOutputStream(...) around lines 115-123
- Core relevance: output streams are common in file operations.
- Bug type: correctness / data integrity
- Proposed change: Drop TRUNCATE_EXISTING from default write options (no options provided).
- Trigger conditions: Files.newOutputStream(path) with existing file.
- Expected symptom: File is not truncated; old bytes remain after write.
- Why it is hard: Only visible when new content is shorter than old content.
- Static-analysis discoverability: Low to medium.
- Suggested detection: Unit test writing shorter data to an existing file and asserting length.
- Rank (value/stealth/scorability): 4/4/5 (likely too easy unless output-stream tests removed)

### B27
- Location: jimfs/src/main/java/com/google/common/jimfs/JimfsFileChannel.java, read(ByteBuffer dst, long position) around lines 214-233
- Core relevance: FileChannel random access operations.
- Bug type: correctness / API contract drift
- Proposed change: Use this.position instead of the method argument "position".
- Trigger conditions: Calling FileChannel.read(dst, position) with explicit offset.
- Expected symptom: Reads occur from current channel position instead of requested offset.
- Why it is hard: Only appears when mixing sequential and positional reads.
- Static-analysis discoverability: Low.
- Suggested detection: Unit test that positional read does not change or depend on channel position.
- Rank (value/stealth/scorability): 4/4/5 (likely too easy unless FileChannel tests removed)

### B28
- Location: jimfs/src/main/java/com/google/common/jimfs/JimfsFileChannel.java, write(ByteBuffer src, long position) around lines 320-373
- Core relevance: FileChannel random access writes.
- Bug type: API contract drift / correctness
- Proposed change: Update channel position after positional write (even when append is false).
- Trigger conditions: Writing with explicit position while other code relies on channel position.
- Expected symptom: Channel position changes unexpectedly; subsequent reads/writes go to wrong offset.
- Why it is hard: Only visible when positional and sequential IO are mixed.
- Static-analysis discoverability: Low.
- Suggested detection: Unit test that positional writes do not move channel position.
- Rank (value/stealth/scorability): 4/4/5 (likely too easy unless FileChannel tests removed)

### B29
- Location: jimfs/src/main/java/com/google/common/jimfs/AbstractWatchService.java, Key.pollEvents() around lines 262-267
- Core relevance: watch service event delivery and backpressure.
- Bug type: reliability / performance
- Proposed change: Use overflow.get() instead of overflow.getAndSet(0).
- Trigger conditions: Event queue overflows at least once.
- Expected symptom: OVERFLOW event keeps recurring on every poll even after queue drains.
- Why it is hard: Only appears under overflow conditions and looks like normal activity.
- Static-analysis discoverability: Low.
- Suggested detection: Stress test that forces overflow and asserts it is reported once.
- Rank (value/stealth/scorability): 3/4/4

### B30
- Location: jimfs/src/main/java/com/google/common/jimfs/PollingWatchService.java, Snapshot.postChanges(...) around lines 238-245
- Core relevance: watch service correctness for directory monitoring.
- Bug type: time metadata mix-up / correctness
- Proposed change: Compare lastAccessTime instead of lastModifiedTime when emitting ENTRY_MODIFY events.
- Trigger conditions: Reads of files in watched directory (access time changes).
- Expected symptom: Spurious ENTRY_MODIFY events on reads; true modifications may be missed.
- Why it is hard: Only visible via WatchService and timing; not deterministic.
- Static-analysis discoverability: Low.
- Suggested detection: WatchService integration test that reads files without modification and asserts no modify events.
- Rank (value/stealth/scorability): 3/4/4 (likely too easy unless watch-service tests removed)

## Top 10 recommended set
- B01: Core path resolution + symlink semantics; security-impacting and very scorable.
- B03: Attribute copy loss is subtle and cross-module; good data-integrity teaching point.
- B04: Concurrency race in copy/delete; high educational value and realistic.
- B06: Performance regression via lock scope; useful for profiling/triage skills.
- B09: Sparse write gap leak; highlights zero-fill invariants and data safety.
- B12: Cache aliasing causes cross-file corruption; excellent for deep debugging.
- B17: Timestamp mix-up in directory snapshots; subtle time-metadata bug.
- B23: Path normalization escapes root; clear security boundary issue.
- B27: Positional read uses channel position; spec nuance and core API contract.
- B29: WatchService overflow handling; reliability and backpressure behavior.
