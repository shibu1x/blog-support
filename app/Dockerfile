FROM golang:1.24-alpine AS builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o main .

FROM alpine:3.21

RUN apk add --no-cache imagemagick imagemagick-jpeg imagemagick-heic imagemagick-webp tzdata

WORKDIR /app

COPY --from=builder /app/main .

ENTRYPOINT ["./main"]
