# omnistorage-github

[![Go Reference](https://pkg.go.dev/badge/github.com/grokify/omnistorage-github.svg)](https://pkg.go.dev/github.com/grokify/omnistorage-github)
[![Go Report Card](https://goreportcard.com/badge/github.com/grokify/omnistorage-github)](https://goreportcard.com/report/github.com/grokify/omnistorage-github)

GitHub repository backend for [omnistorage](https://github.com/grokify/omnistorage).

## Features

- Read and write files to any branch of a GitHub repository
- Batch multiple file operations into a single atomic commit
- List files in directories with prefix filtering
- Get file metadata (size, SHA1 hash)
- Delete files from the repository
- Configurable commit messages and author
- Support for GitHub Enterprise
- Automatic registration with omnistorage registry

## Installation

```bash
go get github.com/grokify/omnistorage-github
```

## Quick Start

### Reading Files

```go
package main

import (
    "context"
    "io"
    "log"
    "os"

    "github.com/grokify/omnistorage-github/backend/github"
)

func main() {
    backend, err := github.New(github.Config{
        Owner:  "grokify",
        Repo:   "omnistorage",
        Branch: "master",
        Token:  os.Getenv("GITHUB_TOKEN"),
    })
    if err != nil {
        log.Fatal(err)
    }
    defer backend.Close()

    ctx := context.Background()

    // Read a file
    r, err := backend.NewReader(ctx, "README.md")
    if err != nil {
        log.Fatal(err)
    }
    defer r.Close()

    data, _ := io.ReadAll(r)
    log.Println(string(data))
}
```

### Writing Files

```go
// Write a new file (creates a commit)
w, err := backend.NewWriter(ctx, "docs/example.txt")
if err != nil {
    log.Fatal(err)
}
w.Write([]byte("Hello, GitHub!"))
w.Close() // Commits the file

// Update an existing file (creates another commit)
w2, _ := backend.NewWriter(ctx, "docs/example.txt")
w2.Write([]byte("Updated content"))
w2.Close()

// Delete a file (creates a commit)
backend.Delete(ctx, "docs/example.txt")
```

### Batch Operations

For multiple file operations in a single commit, use the batch API:

```go
// Create a batch
batch, err := backend.NewBatch(ctx, "Update multiple files")
if err != nil {
    log.Fatal(err)
}

// Queue operations
batch.Write("file1.txt", []byte("content for file 1"))
batch.Write("file2.txt", []byte("content for file 2"))
batch.Delete("old-file.txt")

// Commit all changes atomically
if err := batch.Commit(); err != nil {
    log.Fatal(err)
}
```

The batch API uses the Git Trees and Commits API to create a single commit with all changes, which is more efficient than individual writes when updating multiple files.

### Custom Commit Messages

```go
backend, err := github.New(github.Config{
    Owner:         "grokify",
    Repo:          "my-repo",
    Token:         os.Getenv("GITHUB_TOKEN"),
    CommitMessage: "[bot] Update {path}",
    CommitAuthor: &github.CommitAuthor{
        Name:  "My Bot",
        Email: "bot@example.com",
    },
})
```

## Using the Registry

```go
import (
    "github.com/grokify/omnistorage"
    _ "github.com/grokify/omnistorage-github/backend/github" // Register backend
)

backend, err := omnistorage.Open("github", map[string]string{
    "owner":  "grokify",
    "repo":   "omnistorage",
    "branch": "master",
    "token":  os.Getenv("GITHUB_TOKEN"),
})
```

## Configuration

### Config Struct

```go
type Config struct {
    Owner     string // Repository owner (required)
    Repo      string // Repository name (required)
    Branch    string // Branch name (default: "main")
    Token     string // GitHub personal access token (required)
    BaseURL   string // API base URL (default: "https://api.github.com/")
    UploadURL string // Upload URL (default: "https://uploads.github.com/")

    // CommitMessage is the commit message template.
    // Use {path} as a placeholder for the file path.
    // Default: "Update {path} via omnistorage"
    CommitMessage string

    // CommitAuthor is the author for commits.
    // If nil, uses the authenticated user.
    CommitAuthor *CommitAuthor
}
```

### GitHub Enterprise

```go
backend, err := github.New(github.Config{
    Owner:     "myorg",
    Repo:      "myrepo",
    Branch:    "main",
    Token:     os.Getenv("GITHUB_TOKEN"),
    BaseURL:   "https://github.example.com/api/v3/",
    UploadURL: "https://github.example.com/uploads/",
})
```

### Environment Variables

Configuration can also be loaded from environment variables:

```go
cfg := github.ConfigFromEnv()
backend, err := github.New(cfg)
```

Supported environment variables:

| Variable | Fallback | Description |
|----------|----------|-------------|
| `OMNISTORAGE_GITHUB_OWNER` | `GITHUB_OWNER` | Repository owner |
| `OMNISTORAGE_GITHUB_REPO` | `GITHUB_REPO` | Repository name |
| `OMNISTORAGE_GITHUB_BRANCH` | - | Branch name (default: "main") |
| `OMNISTORAGE_GITHUB_TOKEN` | `GITHUB_TOKEN` | Personal access token |
| `OMNISTORAGE_GITHUB_BASE_URL` | `GITHUB_API_URL` | API base URL |
| `OMNISTORAGE_GITHUB_UPLOAD_URL` | - | Upload URL |
| `OMNISTORAGE_GITHUB_COMMIT_MESSAGE` | - | Commit message template |
| `OMNISTORAGE_GITHUB_COMMIT_AUTHOR_NAME` | - | Commit author name |
| `OMNISTORAGE_GITHUB_COMMIT_AUTHOR_EMAIL` | - | Commit author email |

## Supported Operations

| Operation | Supported | Notes |
|-----------|-----------|-------|
| `NewReader` | Yes | Reads file content via Contents API |
| `NewWriter` | Yes | Creates/updates files (each write = 1 commit) |
| `NewBatch` | Yes | Atomic multi-file commits via Git Trees API |
| `Exists` | Yes | Checks if file/directory exists |
| `Delete` | Yes | Deletes files (each delete = 1 commit) |
| `List` | Yes | Lists files via Trees API |
| `Stat` | Yes | Returns size and SHA1 hash |
| `Copy` | No | Returns `ErrNotSupported` |
| `Move` | No | Returns `ErrNotSupported` |
| `Mkdir` | No | Returns `ErrNotSupported` (directories are implicit) |
| `Rmdir` | No | Returns `ErrNotSupported` (directories are implicit) |

## Limitations

- **File size**: GitHub Contents API only supports files up to 1MB. Larger files require the Git Blobs API (not yet implemented).
- **Rate limits**: GitHub API has rate limits (5,000 requests/hour for authenticated users).
- **Commits per write**: Each `NewWriter.Close()` creates a separate commit. For bulk operations, use `NewBatch()` to combine multiple operations into a single commit.

## API Reference

See [pkg.go.dev](https://pkg.go.dev/github.com/grokify/omnistorage-github) for full documentation.

## Related Projects

- [omnistorage](https://github.com/grokify/omnistorage) - Core storage abstraction library
- [omnistorage-google](https://github.com/grokify/omnistorage-google) - Google Drive and GCS backends

## License

MIT License - see [LICENSE](LICENSE) for details.
