# Google Workspace App Contract

This contract covers Gmail and Google Drive connectors that write to Memory through the generic app integration APIs.

Read the core Memory contract first:

- [Memory core contract](../../memory/README.md)
- [Memory app integration guide](../../memory/app-integration.md)
- [Principals and membership contract](../../memory/principals-and-membership.md)

Then use these Google Workspace-specific documents and examples:

- [Google Workspace boundary](boundary.md)
- [Gmail bootstrap example](examples/gmail-bootstrap.json)
- [Gmail ingestion example](examples/gmail-ingestion.json)
- [Drive folder bootstrap example](examples/drive-folder-bootstrap.json)
- [Drive file ingestion example](examples/drive-file-ingestion.json)

## Local connector model

The Google Workspace connector owns Google OAuth, sync scheduling, Google API calls, export / text extraction, and Google-side cursors.

Memory owns MemorySpace, Source, RawEvidence, derived memory, Policy, membership, and retrieval authorization.

Memory must not store Google OAuth tokens, refresh tokens, history cursors, page tokens, file revision cursors, Google API credentials, or exported file binaries as connector state. The connector sends deterministic extracted text and provenance metadata to Memory through `POST /v1/memory-spaces/{space_id}/ingestions`.

## Source types

Initial Source types:

- `google_gmail_message`: extracted text from Gmail messages or bounded message threads.
- `google_drive_file`: deterministic extracted text from Drive files selected by a bound folder or file source.

These are provenance types. They do not cause Memory to branch structurally for Google Workspace.

## Space ownership

Gmail uses a user-owned MemorySpace by default.

Drive folder ownership depends on the folder purpose:

- personal folder: user-owned MemorySpace
- team knowledge folder: team-owned MemorySpace
- organization knowledge folder: organization-owned MemorySpace

Google folder sharing is not Memory membership. The connector or platform must sync explicit Memory membership / MemoryView state for the range Memory should expose. Cross-space reads use MemoryView or owner-containment scope, never arbitrary app-passed space lists.
