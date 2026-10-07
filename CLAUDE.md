# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Go-based command-line tool for managing Hugo blog posts. It automates the creation of new posts, image processing, and publishing workflows by handling image resizing, S3 uploads, and markdown generation.

## Repository Structure

```
blog-support/
  app/                  # Go application source
    main.go
    model/post_model.go
    Dockerfile
    Dockerfile.dev
    go.mod / go.sum
  compose.yaml          # Docker Compose (dev service only)
  Taskfile.yaml         # Build tasks
  .env                  # Environment variables (not committed)
```

## Core Architecture

### Entry Point (app/main.go)
- CLI accepts two modes:
  - Create new post: `./main [date] [-N <number>]` (date format MM/DD or YYYY/MM/DD as positional argument)
  - Publish posts: `./main -P [year]` (publishes all posts for a year, defaults to current year)
  - Default: creates post for current date if no date specified

### Post Model (app/model/post_model.go)
The entire business logic lives in a single model file with the following workflow:

**Creating a New Post:**
1. `CreateNewPost()` - Initializes Post struct with date-based directory structure (YYYY/MM/DD or YYYY/MM/DD_N for multiple posts per day)
2. `CreatePost()` - Orchestrates:
   - `createDirectory()` - Creates post dir with `img/` and `img_src/` subdirectories
   - `createIndexFile()` - Generates index.md with frontmatter (title, slug, date, image, categories, tags)
   - `resizeImages()` - Processes images from `img_src/` using ImageMagick (resize to 1024x1024, convert HEIC/WEBP/AVIF to JPG, rename IMG_* files to i*, append image references to index.md)

**Publishing Posts:**
1. `PublishYearPosts()` - Entry point for batch publishing
2. `scanPostDirectories()` - Walks directory tree to find all posts with `img/` directories
3. `publishPost()` - For each post:
   - `removeUnusedImages()` - Deletes images not referenced in index.md (except cover.*)
   - `uploadImagesToS3()` - Uploads all images to S3 with path structure: `{S3_KEY_PREFIX}/{YYYY/MM/DD}/img/{filename}`
   - `replaceImageLinks()` - Regex-replaces local image paths with S3 URLs, adds `?d=300x300` query param; removes `image:` frontmatter line if no cover image exists
   - `removeImageDirectories()` - Cleans up local `img/` and `img_src/` directories

### Configuration (app/model/post_model.go:LoadEnv)
Required environment variables (loaded from .env via godotenv):
- `POST_DIR` - Root directory for blog posts (e.g., content/post) — **required**
- `S3_BUCKET_NAME` - AWS S3 bucket for image storage — **required**
- `REMOTE_IMG_BASE_URL` - Base URL for accessing images (used in markdown replacement) — **required**
- `AWS_REGION` - AWS region
- `S3_KEY_PREFIX` - Prefix for S3 object keys

## Key Commands

### Development (Docker)
```bash
# Open shell in dev container
docker compose run --rm dev /bin/ash

# Create post for today
docker compose run --rm dev ./main

# Create post for specific date
docker compose run --rm dev ./main 10/23
docker compose run --rm dev ./main 2025/10/23

# Create second post for same day
docker compose run --rm dev ./main -N 2 10/23

# Publish all posts for current year
docker compose run --rm dev ./main -P

# Publish posts for specific year
docker compose run --rm dev ./main -P 2025
```

### Build & Deploy
```bash
# Build Docker image for ARM64 and push to registry
task build

# Build AMD64 image (requires env vars: AMD64_BUILDER, REGISTRY, REMOTE_HOST)
task amd
```

### Testing
No test files exist in the codebase.

## Directory Structure Conventions

Posts follow this structure:
```
{POST_DIR}/
  YYYY/
    MM/
      DD/          # Single post per day
        index.md
        img/       # Processed images (deleted after publish)
        img_src/   # Source images (deleted after publish)
      DD_2/        # Second post on same day
        index.md
        img/
        img_src/
```

After publishing, only index.md remains with remote image references.

## Important Technical Details

### Image Processing Pipeline
- Uses ImageMagick (magick command) for resizing - must be installed in container
- Supported input formats: HEIC, WEBP, AVIF, JPG, JPEG, PNG
- Output: JPG (1024x1024) except PNG stays as PNG
- Filename transformation: `IMG_1234.heic` → `i1234.jpg`
- Cover images (cover.jpg or cover.png) are preserved during cleanup

### AWS Integration
- Uses AWS SDK v2 for Go
- Credentials loaded via standard AWS config chain (env vars, ~/.aws/credentials, IAM roles)
- S3 uploads use PutObject with default ACL settings

### Regex Patterns
- Image link replacement: `\]\((img/[^)]+)\)` → `]({REMOTE_IMG_BASE_URL}/{dir}/$1?d=300x300)`
- Cover image frontmatter: `image: (img/cover\..{3})` → `image: {REMOTE_IMG_BASE_URL}/{dir}/$1?d=300x300`
- No cover image: `image: img/cover.xxx` line is removed from frontmatter entirely

## Docker Environment

The production Dockerfile (`app/Dockerfile`) uses multi-stage build:
- Builder: golang:1.24-alpine
- Runtime: alpine:3.21 with ImageMagick (imagemagick-jpeg, imagemagick-heic, imagemagick-webp) + timezone data
- CGO disabled for static binary compilation
- Entrypoint: `./main` (no wrapper script)

The dev Dockerfile (`app/Dockerfile.dev`) uses golang:1.24-alpine with ImageMagick installed. Source code is volume-mounted from `./app`.

## Development Workflow

1. Set up .env file (copy from .env.sample)
2. Place source images in `{POST_DIR}/YYYY/MM/DD/img_src/`
3. Run tool via Docker Compose to create post and process images
4. Edit generated index.md with content
5. Run publish command to upload to S3 and finalize
