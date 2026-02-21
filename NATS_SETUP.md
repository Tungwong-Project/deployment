# NATS JetStream Setup Guide

This guide explains how to use NATS JetStream for video processing queue in the Tungwong project.
 
## Overview

NATS JetStream is used as a message queue for video upload and processing workflow:

1. **Video Upload API** publishes video processing requests to NATS
2. **Video Worker** subscribes to process videos with FFmpeg for HLS streaming

## NATS Connection

### Connection URL
```
nats://localhost:4222  (from host)
nats://nats:4222       (from Docker containers)
```

### Monitoring Dashboard
```
http://localhost:8222
```

## Stream and Consumer Setup

### Creating a Stream for Video Processing

```bash
# Connect to NATS container
docker-compose exec nats sh

# Create stream using NATS CLI
nats stream add VIDEO_PROCESSING \
  --subjects "video.upload,video.process" \
  --storage file \
  --retention limits \
  --max-msgs=-1 \
  --max-bytes=-1 \
  --max-age=7d \
  --max-msg-size=10485760 \
  --discard old
```

### Creating a Consumer for Video Worker

```bash
nats consumer add VIDEO_PROCESSING VIDEO_WORKER \
  --filter "video.process" \
  --deliver all \
  --ack explicit \
  --wait 30s \
  --max-deliver 3 \
  --replay instant
```

## Go Client Examples

### Installing NATS Client

```bash
go get github.com/nats-io/nats.go
```

### Publisher (Video Upload API)

```go
package main

import (
    "encoding/json"
    "log"
    "github.com/nats-io/nats.go"
)

type VideoProcessRequest struct {
    VideoID      string `json:"video_id"`
    SourcePath   string `json:"source_path"`
    OutputPath   string `json:"output_path"`
    UserID       string `json:"user_id"`
    Quality      []string `json:"quality"` // ["1080p", "720p", "480p"]
}

func PublishVideoProcessRequest(nc *nats.Conn, req VideoProcessRequest) error {
    js, err := nc.JetStream()
    if err != nil {
        return err
    }

    data, err := json.Marshal(req)
    if err != nil {
        return err
    }

    _, err = js.Publish("video.process", data)
    if err != nil {
        return err
    }

    log.Printf("Published video processing request for video_id: %s", req.VideoID)
    return nil
}

func main() {
    // Connect to NATS
    nc, err := nats.Connect("nats://nats:4222")
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    // Example: Publish a request
    req := VideoProcessRequest{
        VideoID:    "video-123",
        SourcePath: "/uploads/videos/raw/video-123.mp4",
        OutputPath: "/uploads/videos/hls/video-123",
        UserID:     "user-456",
        Quality:    []string{"1080p", "720p", "480p"},
    }

    if err := PublishVideoProcessRequest(nc, req); err != nil {
        log.Fatal(err)
    }
}
```

### Consumer (Video Worker)

```go
package main

import (
    "encoding/json"
    "log"
    "os/exec"
    "fmt"
    "github.com/nats-io/nats.go"
)

type VideoProcessRequest struct {
    VideoID      string   `json:"video_id"`
    SourcePath   string   `json:"source_path"`
    OutputPath   string   `json:"output_path"`
    UserID       string   `json:"user_id"`
    Quality      []string `json:"quality"`
}

func processVideo(req VideoProcessRequest) error {
    log.Printf("Processing video: %s", req.VideoID)

    // Create output directory
    outputDir := req.OutputPath
    exec.Command("mkdir", "-p", outputDir).Run()

    // FFmpeg command for HLS conversion
    for _, quality := range req.Quality {
        scale := getScaleForQuality(quality)
        bitrate := getBitrateForQuality(quality)
        
        outputFile := fmt.Sprintf("%s/%s.m3u8", outputDir, quality)
        
        cmd := exec.Command("ffmpeg",
            "-i", req.SourcePath,
            "-vf", fmt.Sprintf("scale=%s", scale),
            "-c:v", "libx264",
            "-b:v", bitrate,
            "-c:a", "aac",
            "-b:a", "128k",
            "-hls_time", "10",
            "-hls_playlist_type", "vod",
            "-hls_segment_filename", fmt.Sprintf("%s/%s_%%03d.ts", outputDir, quality),
            outputFile,
        )

        output, err := cmd.CombinedOutput()
        if err != nil {
            log.Printf("FFmpeg error for %s: %v\nOutput: %s", quality, err, output)
            return err
        }

        log.Printf("Successfully created %s variant for video %s", quality, req.VideoID)
    }

    // Create master playlist
    if err := createMasterPlaylist(outputDir, req.Quality); err != nil {
        return err
    }

    log.Printf("Video processing complete: %s", req.VideoID)
    return nil
}

func getScaleForQuality(quality string) string {
    scales := map[string]string{
        "1080p": "-2:1080",
        "720p":  "-2:720",
        "480p":  "-2:480",
        "360p":  "-2:360",
    }
    return scales[quality]
}

func getBitrateForQuality(quality string) string {
    bitrates := map[string]string{
        "1080p": "5000k",
        "720p":  "2800k",
        "480p":  "1400k",
        "360p":  "800k",
    }
    return bitrates[quality]
}

func createMasterPlaylist(outputDir string, qualities []string) error {
    // Implementation for creating HLS master playlist
    // This combines all quality variants into one m3u8 file
    return nil
}

func main() {
    // Connect to NATS
    nc, err := nats.Connect("nats://nats:4222")
    if err != nil {
        log.Fatal(err)
    }
    defer nc.Close()

    // Create JetStream context
    js, err := nc.JetStream()
    if err != nil {
        log.Fatal(err)
    }

    // Subscribe to video processing queue
    _, err = js.Subscribe("video.process", func(msg *nats.Msg) {
        var req VideoProcessRequest
        if err := json.Unmarshal(msg.Data, &req); err != nil {
            log.Printf("Error unmarshaling message: %v", err)
            msg.Nak() // Negative acknowledgment
            return
        }

        // Process the video
        if err := processVideo(req); err != nil {
            log.Printf("Error processing video: %v", err)
            msg.Nak() // Will retry based on consumer config
            return
        }

        // Acknowledge successful processing
        msg.Ack()
    }, nats.Durable("VIDEO_WORKER"), nats.ManualAck())

    if err != nil {
        log.Fatal(err)
    }

    log.Println("Video worker started, waiting for messages...")
    select {} // Keep running
}
```

## Stream Configuration Details

### Subject Naming Convention

```
video.upload          - Raw video uploaded
video.process         - Video ready for processing
video.processed       - Video processing complete
video.failed          - Video processing failed
video.thumbnail       - Generate thumbnail
video.metadata        - Extract metadata
```

### Message Payload Example

```json
{
  "video_id": "d94b4f37-4086-42fd-b088-cfbcbff262d8",
  "source_path": "/uploads/videos/raw/video-123.mp4",
  "output_path": "/uploads/videos/hls/video-123",
  "user_id": "user-456",
  "quality": ["1080p", "720p", "480p"],
  "thumbnail_time": "00:00:05",
  "callback_url": "http://video-api:3003/api/videos/process-complete",
  "metadata": {
    "title": "My Video",
    "duration": 300,
    "size_bytes": 52428800
  }
}
```

## FFmpeg HLS Output Structure

```
/uploads/videos/hls/video-123/
├── 1080p.m3u8
├── 1080p_000.ts
├── 1080p_001.ts
├── 1080p_002.ts
├── 720p.m3u8
├── 720p_000.ts
├── 720p_001.ts
├── 480p.m3u8
├── 480p_000.ts
├── master.m3u8
└── thumbnail.jpg
```

## Monitoring and Management

### View Stream Info
```bash
docker-compose exec nats nats stream info VIDEO_PROCESSING
```

### View Consumer Info
```bash
docker-compose exec nats nats consumer info VIDEO_PROCESSING VIDEO_WORKER
```

### View Messages
```bash
docker-compose exec nats nats stream view VIDEO_PROCESSING
```

### Purge Stream (careful!)
```bash
docker-compose exec nats nats stream purge VIDEO_PROCESSING
```

## Error Handling

### Retry Configuration
- Max deliveries: 3 attempts
- Wait time: 30 seconds between retries
- After max retries: Message goes to dead letter (configure separately)

### Dead Letter Queue
```bash
# Create dead letter stream
nats stream add VIDEO_PROCESSING_DLQ \
  --subjects "video.dlq" \
  --storage file \
  --retention limits \
  --max-age=30d
```

## Performance Tuning

### For High Volume
```bash
# Increase max messages and bytes
nats stream add VIDEO_PROCESSING \
  --subjects "video.*" \
  --storage file \
  --max-msgs=1000000 \
  --max-bytes=100GB \
  --replicas=3  # For clustering
```

### Worker Scaling
```bash
# Scale video workers
docker-compose up -d --scale video-worker=5
```

## Security

### Enable Authentication
Update `nats.conf`:
```
authorization {
    user: video_service
    password: your_secure_password
}
```

Connect with credentials:
```go
nc, err := nats.Connect("nats://nats:4222",
    nats.UserInfo("video_service", "your_secure_password"))
```

## Testing

### Publish Test Message
```bash
docker-compose exec nats nats pub video.process '{
  "video_id": "test-123",
  "source_path": "/test/video.mp4",
  "output_path": "/test/output",
  "user_id": "test-user",
  "quality": ["720p"]
}'
```

### Monitor Messages
```bash
# Subscribe to all video subjects
docker-compose exec nats nats sub "video.>"
```

## Troubleshooting

### Check NATS is Running
```bash
docker-compose ps nats
docker-compose logs nats
```

### Check Connections
```bash
curl http://localhost:8222/connz
```

### Check Stream Health
```bash
curl http://localhost:8222/jsz?streams=true
```

## Best Practices

1. **Idempotency**: Ensure video processing is idempotent (can be safely retried)
2. **Progress Updates**: Publish progress messages during long processing
3. **Cleanup**: Delete processed source files after successful conversion
4. **Monitoring**: Track processing time and failure rates
5. **Logging**: Log all processing steps for debugging
6. **Validation**: Validate video file before processing
7. **Resource Management**: Limit concurrent FFmpeg processes

## References

- [NATS JetStream Documentation](https://docs.nats.io/nats-concepts/jetstream)
- [NATS Go Client](https://github.com/nats-io/nats.go)
- [FFmpeg HLS Documentation](https://ffmpeg.org/ffmpeg-formats.html#hls-2)
