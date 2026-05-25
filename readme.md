# Blog Support

A Go-based CLI tool for managing Hugo blog posts. Automates post creation, image processing, and publishing via S3.

## Features

- Automated post creation with date-based directory structure
- Image resizing (1024x1024) and format conversion (HEIC/WEBP/AVIF → JPG)
- S3 image upload with automatic URL replacement in markdown
- Multiple posts per day support
- Batch publishing for all posts in a year

## Prerequisites

- Docker
- AWS account with S3 bucket

## Setup

1. Copy `.env.sample` to `.env` and fill in your values:
   ```bash
   cp .env.sample .env
   ```

2. Build the dev image:
   ```bash
   docker compose build
   ```

## Configuration

| Variable | Required | Description |
|---|---|---|
| `POST_DIR` | Yes | Root directory for blog posts (e.g. `content/post`) |
| `S3_BUCKET_NAME` | Yes | AWS S3 bucket name |
| `REMOTE_IMG_BASE_URL` | Yes | Base URL for S3 images |
| `AWS_REGION` | No | AWS region |
| `S3_KEY_PREFIX` | No | Prefix for S3 object keys |

## Usage

Run commands via Docker Compose:

```bash
# Create post for today
docker compose run --rm dev ./main

# Create post for a specific date
docker compose run --rm dev ./main 10/23
docker compose run --rm dev ./main 2025/10/23

# Create second post on the same day
docker compose run --rm dev ./main -N 2 10/23

# Publish all posts for current year
docker compose run --rm dev ./main -P

# Publish posts for a specific year
docker compose run --rm dev ./main -P 2025
```

## Workflow

1. Place source images in `{POST_DIR}/YYYY/MM/DD/img_src/`
2. Run the create command — images are resized and referenced in `index.md`
3. Edit `index.md` with your content
4. Run the publish command — images are uploaded to S3, local `img/` directories are removed

### Post directory structure

```
{POST_DIR}/
  YYYY/
    MM/
      DD/
        index.md
        img/       # deleted after publish
        img_src/   # deleted after publish
      DD_2/        # second post on same day
        index.md
        img/
        img_src/
```

## Build

```bash
# Build ARM64 image and push to registry
task build

# Build AMD64 image
task amd
```
