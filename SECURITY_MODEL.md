# Mila security model

Scope: `cog-declan/mila` at release 1.9.8. No infrastructure repository: Mila has no backend. The only infrastructure code is the optional self-hosted transcription server example in `docker/` and `docs/self-hosted-server/k8s/`. `SECURITY.md` is a disclosure policy; its "Good to know" notes (local transcription, user-installed LLM CLI, EdDSA-signed Sparkle updates) are folded in below.

## Maintainer decisions
- **Attackers:** all of them matter. Nothing is ruled out.
- **Out of scope / accepted risk:** nothing. Everything is in scope.
- **Assets:** all treated equally. No asset is singled out as critical.

Scan findings keep the scanner's severity unless these decisions change.

## System overview and trust boundaries
Mila is a native macOS app (Swift/SwiftUI). It is **not sandboxed** (`com.apple.security.app-sandbox = false`). It has JIT and `disable-library-validation`, network client, and microphone/audio-input entitlements. It also asks for screen recording and accessibility permissions. It runs as the logged-in user.

Trust boundaries:
1. **Imported files → app.** These come from other people: `.milashare` zips, `.milaconfig` JSON, and shared speaker profiles.
2. **Network → app.** These cross the network: the Sparkle appcast and updates, model downloads (Hugging Face), the PyTorch wheels for diarization, the Claude CLI installer, the Remote API transcription endpoint, and the OpenAI-compatible client.
3. **App → subprocesses.** Mila starts these: Python/pyannote diarization, the LLM CLIs (claude etc.) with user-defined extra args, `git` for the Obsidian sync, `ditto` for zips, and the bundle relocator helper.
4. **App store → MCP clients.** `mila-mcp` serves recordings to Claude and other MCP clients. Transcript text is attacker-influenced, because anyone speaking in a meeting controls it.
5. **MDM / configuration profile → app.** `ManagedSettings` enforces forced values.

## Components
| Name | Code |
|---|---|
| App Shell | `Mila/App/`, `Mila/Views/`, `Mila/Resources/` (Info.plist, entitlements) |
| Recording | `Mila/Audio/`, `Mila/Dictation/`, `Mila/VoiceMemos/`, `Mila/MCP/LiveTranscriptSidecarWriter.swift` |
| Transcription | `Mila/Transcription/`, `Packages/TranscriptionCore/` (whisper.cpp, ModelManager, diarization bootstrap, RemoteWhisperEngine) |
| LLM & Integrations | `Mila/Actions/` (LLMRunner, Claude*, OpenAIClient, RecordingSummarizer, LiveAISession, Obsidian*, DiagnosticReporter) |
| Store & Sharing | `Mila/Models/` (RecordingStore, RecordingShareImporter/ShareManifest, MilaConfig/Importer, ManagedSettings, Keychain/ClaudeTokenStore, speaker profiles), `Mila/Actions/ZipArchiver.swift`, `RecordingShareExporter.swift` |
| MCP Server | `MilaMCP/`, `Packages/MilaKit/` (MCPAccessGate, MilaStoreReader, MilaMCPToolHandlers, TranscriptFileWriter) |
| Release & CI | `.github/workflows/`, `scripts/`, `site/`, `project.yml` (Sparkle feed `https://island-io.github.io/mila/appcast.xml`, `SUPublicEDKey`) |
| Self-hosted Server | `docker/speaches-hebrew/`, `docs/self-hosted-server/` (example k8s manifests) |

## Entry points and who can reach them
- **Opening a `.milashare` / `.milaconfig` / speaker-profile file.** Anyone who can send the user a file can reach this (document types in `Info.plist`).
- **Audio content.** Anyone speaking in a recorded meeting or audio played into it can reach this. Their words become transcript text, which flows into the LLM prompts, Live AI and MCP tools.
- **Network responses.** A network attacker or a compromised upstream can reach these: the appcast, model files (pinned SHA-256 in `ModelManager.swift`), PyTorch wheels, the Claude installer, and Remote API responses.
- **MCP stdio.** Any MCP client the user configures can reach this. Reads are gated by `mcp-access.json`; `save_transcript` has a second gate.
- **Local files under `~/Library/Application Support/Mila/`.** Any process running as the user can reach these. Per `docs/mcp.md`, the MCP gate is consent, not a privilege boundary.
- **CI workflows.** PR authors and contributors can reach these. Release signing keys are in CI secrets.

## Assets
All are equal priority:
- Recordings, transcripts, summaries.
- Voice-profile embeddings (biometric).
- Keychain credentials: Remote API keys and the Claude setup token.
- Integrity of the app, its update channel and the downloaded models/wheels/CLI.
- The user's files (writes by `save_transcript`, the Obsidian export and git sync).

## Attacker profiles
All are in scope:
1. Sender of a malicious shared or config file.
2. Meeting participant injecting instructions through speech (prompt injection into the LLM CLI or MCP clients).
3. Network attacker or compromised upstream (Hugging Face, PyTorch, the GitHub Pages appcast).
4. Malicious MCP client or LLM tool output.
5. Other software running as the same user.
6. Malicious contributor or PR targeting CI and release secrets.
7. An MDM admin pushing hostile managed values (low).

## Existing controls
- **`.milashare` import.** The central directory is validated before `ditto -x`: fixed entry names, regular files, size caps. Extracted entries and sizes are re-checked, and audio SHA-256 is matched against the manifest (`RecordingShareImporter`, `ZipArchiver.listEntries`). Known gap noted in `CLAUDE.md`: a deflate stream whose header claims a small size can still inflate large.
- **MCP.** `MCPAccessGate` fails closed and is re-read on every call. There is a separate file-export consent. Trashed recordings are excluded in `MilaStoreReader`.
- **Downloads and updates.** Model downloads are verified by pinned SHA-256. Sparkle updates are EdDSA-signed.
- **Secrets.** API keys live in the Keychain, and a managed `apiKey` never touches the Keychain. LLM argv is redacted in logs (`LLMRunner`).
- **Managed settings.** These use forced-only `CFPreferences` values with setter guards (`ManagedField`).
- **CI.** CodeQL and OpenSSF Scorecard workflows run.

## Out of scope / accepted risk
None. Everything is in scope.

## Open questions
- Is the deployed Sparkle feed and signing key managed outside this repo (`island-io` GitHub Pages)?
- Are the PyTorch wheels and the Claude CLI installer integrity-pinned? (Unverified.)
