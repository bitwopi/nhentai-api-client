# nhentai API Client (Go)

Type-safe Go client for the [nhentai.net](https://nhentai.net) REST API v2.

Generated from the official OpenAPI 3.1 specification with `oapi-codegen`.

## Requirements

- Go 1.22+
- Optional: API key from [account settings](https://nhentai.net/user/settings#apikeys) for higher rate limits and private endpoints

## Installation

```bash
go get github.com/bitwopi/nhentai-api-client
```

Or copy the `api/` package into your project.

## Quick start

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"

	"github.com/bitwopi/nhentai-api-client/api"
)

func main() {
	client, err := api.NewClientWithResponses(
		"https://nhentai.net",
		api.WithRequestEditorFn(func(ctx context.Context, req *http.Request) error {
			// Required: descriptive User-Agent
			req.Header.Set("User-Agent", "MyApp/1.0 (https://github.com/you/project)")

			// Optional: API key for higher limits / private methods
			// req.Header.Set("Authorization", "Key YOUR_API_KEY")
			return nil
		}),
	)
	if err != nil {
		log.Fatal(err)
	}

	// Get a gallery by ID
	resp, err := client.GetGalleryWithResponse(context.Background(), 177013, nil)
	if err != nil {
		log.Fatal(err)
	}
	if resp.StatusCode() != 200 {
		log.Fatalf("status %d: %s", resp.StatusCode(), string(resp.Body))
	}

	g := resp.JSON200
	fmt.Println("ID:", g.Id)
	fmt.Println("Title:", g.Title.English)
	fmt.Println("Pages:", g.NumPages)
}
```

## Authentication

Most public endpoints work without auth. For favorites, profile, and higher rate limits:

1. Create an API key: https://nhentai.net/user/settings#apikeys
2. Send header: `Authorization: Key YOUR_API_KEY`

Always set a descriptive `User-Agent`:

```text
AppName/version (contact or project URL)
```

Example with both headers:

```go
client, err := api.NewClientWithResponses(
	"https://nhentai.net",
	api.WithRequestEditorFn(func(ctx context.Context, req *http.Request) error {
		req.Header.Set("User-Agent", "MyApp/1.0 (https://github.com/you/myapp)")
		req.Header.Set("Authorization", "Key YOUR_API_KEY")
		return nil
	}),
)
```

## Common examples

### Search

```go
query := "futanari"
page := 1

params := &api.SearchGalleriesParams{
	Query: &query,
	Page:  &page,
}

resp, err := client.SearchGalleriesWithResponse(context.Background(), params)
if err != nil {
	log.Fatal(err)
}
if resp.JSON200 != nil {
	for _, item := range resp.JSON200.Result {
		fmt.Println(item.Id, item.Title.English)
	}
}
```

### List newest galleries

```go
page := 1
perPage := 25

params := &api.GetAllGalleriesParams{
	Page:    &page,
	PerPage: &perPage,
}

resp, err := client.GetAllGalleriesWithResponse(context.Background(), params)
```

### Random gallery

```go
resp, err := client.GetRandomGalleryWithResponse(context.Background())
if err != nil {
	log.Fatal(err)
}
fmt.Printf("%+v\n", resp.JSON200)
```

### Related galleries

```go
resp, err := client.GetRelatedGalleriesWithResponse(context.Background(), 177013)
```

### CDN / image servers config

```go
resp, err := client.GetConfigWithResponse(context.Background())
// or
resp, err := client.GetCdnConfigWithResponse(context.Background())
```

### Favorites (requires API key)

```go
// Check
resp, err := client.CheckFavoriteWithResponse(context.Background(), 177013)

// Add
resp, err := client.AddToFavoritesWithResponse(context.Background(), 177013)

// Remove
resp, err := client.RemoveFromFavoritesWithResponse(context.Background(), 177013)

// List
params := &api.GetFavoritesParams{Page: &page}
resp, err := client.GetFavoritesWithResponse(context.Background(), params)
```

## Handling responses

Methods ending with `WithResponse` return a typed envelope:

```go
resp, err := client.GetGalleryWithResponse(ctx, id, params)
if err != nil {
	// network / transport error
	return err
}

switch resp.StatusCode() {
case 200:
	gallery := resp.JSON200
	_ = gallery
case 404:
	// resp.JSON404 may contain ErrorResponse
case 429:
	// rate limited
default:
	log.Printf("unexpected %d: %s", resp.StatusCode(), string(resp.Body))
}
```

Useful fields on the response object:

| Field | Description |
|-------|-------------|
| `StatusCode()` | HTTP status |
| `Body` | Raw response body |
| `HTTPResponse` | Underlying `*http.Response` |
| `JSON200`, `JSON404`, … | Parsed body for that status (if defined in the spec) |

## Regenerating the client

The client is generated from an OpenAPI spec with nicer `operationId` values.

```bash
# Install generator
go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest

# Config (oapi-codegen.yaml)
# package: api
# output: api/client.gen.go
# generate:
#   models: true
#   client: true

oapi-codegen -config oapi-codegen.yaml nhentai-openapi-nice.yaml
```

Or via `go generate`:

```go
//go:generate go run github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen -config oapi-codegen.yaml nhentai-openapi-nice.yaml
```

Runtime dependency:

```bash
go get github.com/oapi-codegen/runtime@latest
go mod tidy
```

## Project layout

```text
.
├── api/
│   ├── client.gen.go      # generated client + models
│   ├── generate.go        # //go:generate directive
│   ├── oapi-codegen.yaml  # generator config
│   └── nhentai-openapi-nice.yaml
├── go.mod
└── README.md
```

## Notes

- Official docs: https://nhentai.net/api/v2/docs
- Changelog: https://nhentai.net/api/v2/changelog
- Prefer the official API over HTML scraping
- Respect rate limits (anonymous limits are lower than authenticated)
- This is an unofficial client library; nhentai is not affiliated with this repository

## License

MIT
