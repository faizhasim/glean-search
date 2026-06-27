# Architecture

## Overview

Glean Search is a Raycast extension implemented as a single-view React component. It delegates all Glean interactions to the official Glean CLI, which is auto-downloaded on first use.

```mermaid
flowchart TB
    subgraph User["User"]
        A["Raycast search field"]
    end

    subgraph Extension["Raycast Extension"]
        B["search-glean.tsx<br/>UI / State"]
        C["glean.ts<br/>useGlean hook"]
        D["cli.ts<br/>Binary management"]
        E["auth.ts<br/>OAuth flow"]
    end

    subgraph CLI["glean CLI"]
        F["glean search"]
        G["glean auth login"]
        H["glean auth status"]
    end

    subgraph External["External"]
        I["GitHub Releases<br/>glean-cli binary"]
        J["Glean Cloud API"]
        K["Browser<br/>OAuth"]
        L["~/.glean/config.json<br/>Server URL cache"]
    end

    A --> B
    B --> C
    C --> D
    C --> E
    D --> I
    E --> G --> K
    E --> H
    E --> L
    C --> F --> J
```

## Modules

### `src/search-glean.tsx`

The Raycast command entry point. A React component that renders the search interface and handles user interactions. It uses the `useGlean` hook to manage extension lifecycle.

**States handled:**

- Initializing -- CLI binary resolving, auth checking
- CLI not found -- prompts user to retry or check network
- Not authenticated -- email form or OAuth flow
- Auth error -- displays error with retry
- Authenticated search -- results list with actions

### `src/lib/glean.ts`

The `useGlean` React hook that orchestrates the extension lifecycle:

1. **CLI discovery** -- calls `resolveGleanCli()` on mount to find or download the binary
2. **Auth checking** -- calls `checkGleanAuth()` to verify the current session
3. **Sign-in** -- calls `signInToGlean()` when the user initiates authentication
4. **Search** -- executes `glean search <query>` via `execFile`, parses JSON output

The hook exposes a stable API (`search`, `signIn`, `recheckAuth`, `retryCliDiscovery`) via `useCallback` to avoid unnecessary re-renders.

### `src/lib/cli.ts`

Manages the Glean CLI binary lifecycle:

- **Resolution** -- attempts common paths in order, falling back to auto-download
- **Auto-download** -- fetches the latest release from GitHub, verifies SHA-256, marks errors persistently to avoid repeated failure loops
- **Caching** -- stores version and checksum in `cli-info.json` alongside the binary

The download uses Node.js `https` module with redirect following (up to 10 hops) and streams directly to disk.

The resolution chain follows this priority:

```mermaid
flowchart LR
    P["$PATH"] --> H["Homebrew<br/>brew --prefix glean"]
    H --> C["~/.glean/cache/glean"]
    C --> G["GitHub Releases<br/>auto-download"]
    G --> E["Error<br/>persistent flag set"]

    P -->|found| OK["Use system binary"]
    H -->|found| OK
    C -->|found| OK
    G -->|success| OK
    G -->|failure| E

    style OK fill:#1e3a2f,stroke:#2ecc71,color:#e0e0e0
    style E fill:#3a1e1e,stroke:#e74c3c,color:#e0e0e0
```

### `src/lib/auth.ts`

Handles Glean authentication:

- **Config reading** -- reads `~/.glean/config.json` to discover the Glean server URL
- **Auth check** -- runs `glean auth status` and parses the output to determine authentication state
- **Sign-in** -- spawns `glean auth login` with stdin closed, captures the OAuth URL from CLI output, and opens it via Raycast's `open()` API. Falls back to an email prompt if `GLEAN_SERVER_URL` is not yet cached.

### `src/lib/types.ts`

Shared TypeScript interfaces:

- `GleanResult` -- a single search result (title, URL, document, snippets)
- `GleanSearchResponse` -- the full API response (results, cursor, pagination)
- `AuthInfo` -- authentication state (authenticated, unauthenticated, error)
- `GleanSnippet` -- a text snippet with MIME type
- `GleanDocument` -- document metadata (datasource, doc type, title, URL)

## Data flow

### Search

```mermaid
sequenceDiagram
    participant User
    participant UI as search-glean.tsx
    participant Hook as useGlean
    participant CLI as glean CLI
    participant API as Glean API

    User->>UI: type query
    UI->>Hook: search(query, signal)
    Hook->>CLI: execFile("glean search", [query])
    CLI->>API: HTTP request
    API-->>CLI: JSON results
    CLI-->>Hook: stdout
    Hook-->>UI: GleanResult[]
    UI-->>User: render list
```

### Authentication

```mermaid
sequenceDiagram
    participant User
    participant UI as search-glean.tsx
    participant Auth as auth.ts
    participant CLI as glean CLI
    participant Config as ~/.glean/config.json
    participant Browser
    participant Keyring

    alt Server URL cached
        Config-->>Auth: server_url found
        Auth->>CLI: spawn with GLEAN_SERVER_URL
        Note over CLI: skips email prompt
    else First time
        User-->>Auth: enter email
        Auth->>CLI: pipe email to stdin
        CLI->>Config: save server_url
    end

    CLI->>Browser: open OAuth URL
    User->>Browser: authenticate
    Browser-->>CLI: OAuth callback
    CLI->>Keyring: store token
    CLI-->>Auth: exit 0
    Auth-->>UI: authenticated
```

## Design decisions

```mermaid
flowchart LR
    D1["CLI delegation<br/>over library"] --> R1["Reduced maintenance<br/>API compatibility"]
    D2["Auto-download<br/>over bundling"] --> R2["Smaller package<br/>Independent updates"]
    D3["React hooks<br/>for state"] --> R3["Stable API surface<br/>No unnecessary re-renders"]
    D4["AbortController<br/>for search"] --> R4["No stale results<br/>on fast typing"]

    style D1 fill:#2e3440,stroke:#81a1c1,color:#e0e0e0
    style D2 fill:#2e3440,stroke:#81a1c1,color:#e0e0e0
    style D3 fill:#2e3440,stroke:#81a1c1,color:#e0e0e0
    style D4 fill:#2e3440,stroke:#81a1c1,color:#e0e0e0
    style R1 fill:#1e3a2f,stroke:#2ecc71,color:#e0e0e0
    style R2 fill:#1e3a2f,stroke:#2ecc71,color:#e0e0e0
    style R3 fill:#1e3a2f,stroke:#2ecc71,color:#e0e0e0
    style R4 fill:#1e3a2f,stroke:#2ecc71,color:#e0e0e0
```

- **CLI delegation** over library integration -- the Glean CLI is the official client, reducing maintenance burden and ensuring API compatibility
- **Auto-download** over bundling -- keeping the binary out of the extension repository avoids bloating the package and allows independent updates
- **React hooks** for state management -- using `useCallback` and `useRef` avoids unnecessary re-renders while keeping the API surface stable
- **AbortController** for search cancellation -- prevents stale results from appearing when the user types quickly
