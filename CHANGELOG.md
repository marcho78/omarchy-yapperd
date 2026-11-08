# Changelog

All notable changes to omarchy-yapperd. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow
[Semantic Versioning](https://semver.org/).

## [Unreleased]

### Changed
- Opening a room now warms the send path in the background. Sync uses lazy
  member loading, so the SDK's first encrypted send downloads the room's
  member list and queries device keys for members it has never seen before it
  can share the room key and post the event; those requests now happen while
  the room is being read, and Enter only has to do the share and the send.
  Small rooms also have untracked members' keys fetched up front.
- Requests that take longer than 400 ms are logged as `slow request` with
  their command name; `RUST_LOG=omarchy_yapperd=debug` logs every request
  with its duration, and every message send with how long it took.

### Fixed
- The community status is cached for 30 s (60 s for a failure) and concurrent
  lookups share one request; the alias lookup also gives up after one attempt
  instead of three. `community_status` used to resolve the space alias on
  every call, and an alias the homeserver cannot resolve (a 502 after 10 s)
  meant each refresh cost ~30 s. Once eight of those were in flight, the
  connection stopped reading new request lines, so an unrelated send sat
  unread until a slow lookup finished. That was the "sometimes Enter takes a
  while" case.
- At most 32 requests run at once per connection (was 8): a burst of slow
  optional requests no longer stops a send from being read.
- The first room-list request after a restart is answered from a snapshot
  taken before the initial sync instead of waiting behind that sync's store
  writes; once the sync has run, the list is rebuilt as before.
- The room list is built with eight rooms in flight rather than one at a
  time. The first call after a start asks each room for its latest activity
  (classic sync carries no recency stamp), which was 1.6 s of serial round
  trips on this account.
- The room list no longer re-parses every cached event of every room on each
  refresh. The cache scan that counts unreads hidden behind a read receipt
  left on paged history now runs only when the SDK's own counter cannot
  answer (it is zero and something arrived after the receipt); fully read
  rooms and rooms the counter already counts are skipped.

## [1.0.2]

### Security
- `packaging/PKGBUILD` no longer exports `RUSTUP_TOOLCHAIN=stable`, which
  overrode the pinned `rust-toolchain.toml` and let a source build fetch a
  moving compiler. The packaged Arch compiler, or the pinned release under
  rustup, builds the daemon.

## [1.0.1]

### Security
- Media downloads (attachments, thumbnails, avatars, link-preview images) go
  through a bounded fetcher: refused when the server declares more than the
  cap, dropped the moment they exceed it (100 MiB for files, 16 MiB for
  thumbnails), abandoned after 300 s. Attachments whose event declares a size
  over the cap are never requested.
- ffmpeg runs by absolute path with a 120 s deadline, `-t 900`, local-file
  input only, a pinned demuxer for recordings, and is killed if the daemon
  drops the request; voice notes that are not audio by their bytes are not
  handed to it.
- Cached attachments get an extension from an allowlist per kind; images
  from their bytes only. A file declared `text/html` is `.bin`, not `.html`.
- Message edits are applied only when the edit comes from the original
  message's sender, live and from the cache.
- The local search scan has one 45 s deadline and covers at most 40 rooms.
- Socket request lines are capped at 1 MiB and eight requests run at once per
  connection.
- The reply-preview cache is bounded; reaction keys, waveforms, community
  cards, bridge names and link-preview titles are capped in length.
- Release workflow actions are pinned to commits and the Rust release is
  pinned in `rust-toolchain.toml`.

## [1.0.0]

### Fixed
- On stop the daemon now ends the sync loop and closes its SQLite stores
  before exiting, so no `-wal` and `-shm` files are left for the next start
  to recover.

### Changed
- First stable release; the plugin's install, update and removal flows are
  built around the two packages (`omarchy-yapperd`, `omarchy-yapperd-bin`).

## [0.22.0]

### Added
- A Release workflow that builds the daemon for x86_64 and aarch64 on every
  tag, attaches the tarballs with a SHA256SUMS file and a build provenance
  attestation to the GitHub release.
- `packaging/bin/PKGBUILD`: the `omarchy-yapperd-bin` package installs that
  prebuilt binary through pacman without a compile. It replaces and is
  replaced by the source package.
- `scripts/release.sh` waits for the workflow, records the checksums, and tags
  the packaging commit `pkg-vX.Y.Z` for the plugin to pin.

## [0.21.1]

### Changed
- The source PKGBUILD depends on `rust` rather than the virtual `cargo`, so
  makepkg no longer asks "rust or rustup?".

## [0.21.0]

### Added
- Omarchy community space, directory browsing, bounded request retries.

[Unreleased]: https://github.com/marcho78/omarchy-yapperd/compare/v1.0.2...HEAD
[1.0.2]: https://github.com/marcho78/omarchy-yapperd/compare/v1.0.1...v1.0.2
[1.0.1]: https://github.com/marcho78/omarchy-yapperd/compare/v1.0.0...v1.0.1
[1.0.0]: https://github.com/marcho78/omarchy-yapperd/compare/v0.22.0...v1.0.0
[0.22.0]: https://github.com/marcho78/omarchy-yapperd/compare/v0.21.1...v0.22.0
[0.21.1]: https://github.com/marcho78/omarchy-yapperd/compare/v0.21.0...v0.21.1
[0.21.0]: https://github.com/marcho78/omarchy-yapperd/compare/v0.20.0...v0.21.0
