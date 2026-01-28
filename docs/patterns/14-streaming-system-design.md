# Video Streaming System Design (Netflix/YouTube)

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        VIDEO STREAMING ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────┐     ┌──────────────┐     ┌─────────────────────────────┐     │
│   │  User   │────►│  CloudFront  │────►│     Origin (S3 / Media)    │     │
│   │ (App)   │     │    (CDN)     │     │                             │     │
│   └─────────┘     └──────────────┘     └─────────────────────────────┘     │
│        │                                                                    │
│        │ API Calls                                                          │
│        ▼                                                                    │
│   ┌──────────────────────────────────────────────────────────────────┐     │
│   │                      API Gateway / ALB                            │     │
│   └───────────────────────────┬──────────────────────────────────────┘     │
│                               │                                             │
│   ┌───────────────────────────┼──────────────────────────────────────┐     │
│   │                    MICROSERVICES                                  │     │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │     │
│   │  │  User   │ │ Content │ │ Search  │ │Recommend│ │ Billing │    │     │
│   │  │ Service │ │ Service │ │ Service │ │ Service │ │ Service │    │     │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘    │     │
│   └──────────────────────────────────────────────────────────────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Video Upload & Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     VIDEO UPLOAD FLOW                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Creator                                                               │
│      │                                                                  │
│      │ 1. Request upload URL                                            │
│      ▼                                                                  │
│   ┌──────────────┐                                                      │
│   │ Upload API   │──► Returns Pre-signed S3 URL                        │
│   └──────────────┘                                                      │
│      │                                                                  │
│      │ 2. Direct upload to S3 (bypass servers)                         │
│      ▼                                                                  │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐           │
│   │  S3 Bucket   │────►│ S3 Event     │────►│    SQS       │           │
│   │  (Raw Video) │     │ Notification │     │   Queue      │           │
│   └──────────────┘     └──────────────┘     └──────┬───────┘           │
│                                                     │                   │
│      3. Transcoding Pipeline                        ▼                   │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │              TRANSCODING WORKERS (ECS/Lambda)                │      │
│   │  ┌─────────────────────────────────────────────────────┐    │      │
│   │  │  FFmpeg / AWS Elemental MediaConvert                │    │      │
│   │  │                                                      │    │      │
│   │  │  Input: video.mp4 (4K, 10GB)                        │    │      │
│   │  │                    │                                 │    │      │
│   │  │                    ▼                                 │    │      │
│   │  │  Output:  ┌──────────────────────────────────┐      │    │      │
│   │  │           │ 4K   (2160p) - 15 Mbps          │      │    │      │
│   │  │           │ 1080p (Full HD) - 5 Mbps        │      │    │      │
│   │  │           │ 720p  (HD) - 2.5 Mbps           │      │    │      │
│   │  │           │ 480p  (SD) - 1 Mbps             │      │    │      │
│   │  │           │ 360p  (Low) - 0.5 Mbps          │      │    │      │
│   │  │           │ + HLS segments (10 sec chunks)   │      │    │      │
│   │  │           │ + Thumbnails                     │      │    │      │
│   │  │           └──────────────────────────────────┘      │    │      │
│   │  └─────────────────────────────────────────────────────┘    │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Upload API (Pre-signed URL)

```java
@RestController
@RequestMapping("/api/upload")
public class UploadController {
    
    @Autowired
    private S3Client s3Client;
    
    @PostMapping("/presigned-url")
    public UploadUrlResponse getUploadUrl(@RequestBody UploadRequest request) {
        String videoId = UUID.randomUUID().toString();
        String key = "raw-videos/" + videoId + "/" + request.getFileName();
        
        // Generate pre-signed URL (valid for 1 hour)
        PutObjectRequest putRequest = PutObjectRequest.builder()
            .bucket("video-uploads-bucket")
            .key(key)
            .contentType(request.getContentType())
            .build();
        
        PresignedPutObjectRequest presignedRequest = s3Presigner.presignPutObject(
            PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofHours(1))
                .putObjectRequest(putRequest)
                .build()
        );
        
        return new UploadUrlResponse(videoId, presignedRequest.url().toString());
    }
}
```

### S3 Event → SQS → Transcoding

```json
// S3 Event Notification Configuration
{
  "QueueConfigurations": [{
    "QueueArn": "arn:aws:sqs:us-east-1:123456789:video-processing-queue",
    "Events": ["s3:ObjectCreated:*"],
    "Filter": {
      "Key": {
        "FilterRules": [{
          "Name": "prefix",
          "Value": "raw-videos/"
        }]
      }
    }
  }]
}
```

### Transcoding Worker

```java
@Service
public class TranscodingService {
    
    @SqsListener("video-processing-queue")
    public void processVideo(S3Event event) {
        String bucket = event.getBucket();
        String key = event.getKey();
        String videoId = extractVideoId(key);
        
        // Update status
        videoRepository.updateStatus(videoId, "PROCESSING");
        
        // Transcode to multiple resolutions
        List<String> resolutions = List.of("1080p", "720p", "480p", "360p");
        
        for (String resolution : resolutions) {
            transcodeToResolution(bucket, key, videoId, resolution);
        }
        
        // Generate thumbnails
        generateThumbnails(bucket, key, videoId);
        
        // Update status
        videoRepository.updateStatus(videoId, "READY");
    }
    
    private void transcodeToResolution(String bucket, String key, 
                                        String videoId, String resolution) {
        // AWS MediaConvert Job
        CreateJobRequest jobRequest = CreateJobRequest.builder()
            .role(mediaConvertRole)
            .settings(JobSettings.builder()
                .inputs(List.of(Input.builder()
                    .fileInput("s3://" + bucket + "/" + key)
                    .build()))
                .outputGroups(List.of(OutputGroup.builder()
                    .outputGroupSettings(OutputGroupSettings.builder()
                        .hlsGroupSettings(HlsGroupSettings.builder()
                            .destination("s3://video-output/" + videoId + "/" + resolution + "/")
                            .segmentLength(10)
                            .build())
                        .build())
                    .build()))
                .build())
            .build();
        
        mediaConvertClient.createJob(jobRequest);
    }
}
```

---

## 2. Video Streaming (HLS/DASH)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ADAPTIVE BITRATE STREAMING (ABR)                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   User clicks PLAY                                                      │
│        │                                                                │
│        │ 1. Request manifest file                                       │
│        ▼                                                                │
│   ┌──────────────────────────────────────────────────────────────┐     │
│   │              master.m3u8 (HLS Manifest)                       │     │
│   │  ┌────────────────────────────────────────────────────────┐  │     │
│   │  │  #EXTM3U                                               │  │     │
│   │  │  #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080│ │     │
│   │  │  1080p/playlist.m3u8                                   │  │     │
│   │  │  #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720│  │     │
│   │  │  720p/playlist.m3u8                                    │  │     │
│   │  │  #EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480│  │     │
│   │  │  480p/playlist.m3u8                                    │  │     │
│   │  └────────────────────────────────────────────────────────┘  │     │
│   └──────────────────────────────────────────────────────────────┘     │
│        │                                                                │
│        │ 2. Player picks quality based on bandwidth                     │
│        ▼                                                                │
│   ┌──────────────────────────────────────────────────────────────┐     │
│   │   Video Player (Client)                                       │     │
│   │   ┌─────────────────────────────────────────────────────┐    │     │
│   │   │  Bandwidth: 8 Mbps → Pick 1080p                     │    │     │
│   │   │  Bandwidth: 3 Mbps → Pick 720p                      │    │     │
│   │   │  Bandwidth: 1 Mbps → Pick 480p                      │    │     │
│   │   │                                                      │    │     │
│   │   │  Adaptive: Switch quality mid-stream!               │    │     │
│   │   └─────────────────────────────────────────────────────┘    │     │
│   └──────────────────────────────────────────────────────────────┘     │
│        │                                                                │
│        │ 3. Request video chunks from CDN                               │
│        ▼                                                                │
│   ┌──────────────────────────────────────────────────────────────┐     │
│   │   CDN (CloudFront) - Edge Location                            │     │
│   │                                                               │     │
│   │   segment_001.ts ──► segment_002.ts ──► segment_003.ts       │     │
│   │      (10 sec)           (10 sec)           (10 sec)          │     │
│   │                                                               │     │
│   │   Cache HIT → Serve from edge (5ms latency)                  │     │
│   │   Cache MISS → Fetch from S3 origin, then cache              │     │
│   └──────────────────────────────────────────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### HLS Manifest Files

**master.m3u8** (Main manifest):
```
#EXTM3U
#EXT-X-VERSION:3

#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080,CODECS="avc1.640028,mp4a.40.2"
1080p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720,CODECS="avc1.64001f,mp4a.40.2"
720p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=854x480,CODECS="avc1.64001e,mp4a.40.2"
480p/playlist.m3u8

#EXT-X-STREAM-INF:BANDWIDTH=500000,RESOLUTION=640x360,CODECS="avc1.640015,mp4a.40.2"
360p/playlist.m3u8
```

**1080p/playlist.m3u8** (Resolution-specific):
```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:10
#EXT-X-MEDIA-SEQUENCE:0

#EXTINF:10.0,
segment_000.ts
#EXTINF:10.0,
segment_001.ts
#EXTINF:10.0,
segment_002.ts
#EXTINF:10.0,
segment_003.ts
#EXT-X-ENDLIST
```

### Video Playback API

```java
@RestController
@RequestMapping("/api/videos")
public class VideoController {
    
    @GetMapping("/{videoId}/stream")
    public StreamUrlResponse getStreamUrl(@PathVariable String videoId,
                                          @AuthenticationPrincipal User user) {
        // Check subscription
        if (!subscriptionService.hasAccess(user, videoId)) {
            throw new AccessDeniedException("Subscription required");
        }
        
        // Get video metadata
        Video video = videoRepository.findById(videoId)
            .orElseThrow(() -> new NotFoundException("Video not found"));
        
        // Generate signed CloudFront URL (expires in 4 hours)
        String manifestUrl = cloudFrontService.createSignedUrl(
            "https://cdn.mystreaming.com/" + videoId + "/master.m3u8",
            Duration.ofHours(4)
        );
        
        // Track view event
        analyticsService.trackViewStart(user.getId(), videoId);
        
        return new StreamUrlResponse(manifestUrl, video.getTitle(), video.getDuration());
    }
    
    @PostMapping("/{videoId}/progress")
    public void updateProgress(@PathVariable String videoId,
                               @RequestBody ProgressRequest request,
                               @AuthenticationPrincipal User user) {
        // Save watch progress for "Continue Watching"
        watchHistoryService.updateProgress(user.getId(), videoId, request.getPosition());
    }
}
```

---

## 3. CDN Configuration (CloudFront)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      CLOUDFRONT DISTRIBUTION                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   User Request: cdn.mystreaming.com/video123/720p/segment_005.ts       │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                    Edge Location                             │      │
│   │                                                              │      │
│   │   1. Check Cache                                             │      │
│   │      ├── HIT  → Return cached segment (5ms)                 │      │
│   │      └── MISS → Fetch from origin                           │      │
│   │                                                              │      │
│   │   2. Origin Fetch (if MISS)                                 │      │
│   │      └── S3: video-bucket/video123/720p/segment_005.ts      │      │
│   │                                                              │      │
│   │   3. Cache for future requests                              │      │
│   │      └── TTL: 24 hours (video segments don't change)        │      │
│   │                                                              │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
│   Cache Settings:                                                       │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │  *.m3u8 (manifests)  → TTL: 1 second (for live streaming)   │      │
│   │  *.ts (segments)     → TTL: 86400 seconds (1 day)           │      │
│   │  thumbnails/*        → TTL: 604800 seconds (7 days)         │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### CloudFront Signed URLs (DRM Light)

```java
@Service
public class CloudFrontService {
    
    private final CloudFrontSigner signer;
    private final String keyPairId;
    private final PrivateKey privateKey;
    
    public String createSignedUrl(String url, Duration validDuration) {
        Instant expiration = Instant.now().plus(validDuration);
        
        return signer.getSignedUrlWithCannedPolicy(
            SignedUrlRequest.builder()
                .url(url)
                .keyPairId(keyPairId)
                .privateKey(privateKey)
                .expiresAt(expiration)
                .build()
        );
    }
    
    // For premium content - IP restricted
    public String createSignedUrlWithPolicy(String url, String clientIp) {
        String policy = """
            {
                "Statement": [{
                    "Resource": "%s*",
                    "Condition": {
                        "DateLessThan": {"AWS:EpochTime": %d},
                        "IpAddress": {"AWS:SourceIp": "%s/32"}
                    }
                }]
            }
            """.formatted(url, Instant.now().plusSeconds(14400).getEpochSecond(), clientIp);
        
        return signer.getSignedUrlWithCustomPolicy(policy, keyPairId, privateKey);
    }
}
```

---

## 4. Database Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DATABASE ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                    USER SERVICE                              │      │
│   │   PostgreSQL / Aurora                                        │      │
│   │   ┌───────────────────────────────────────────────────┐     │      │
│   │   │  users: id, email, password_hash, subscription    │     │      │
│   │   │  profiles: id, user_id, name, avatar, preferences │     │      │
│   │   │  watch_history: user_id, video_id, progress, ts   │     │      │
│   │   └───────────────────────────────────────────────────┘     │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                   CONTENT SERVICE                            │      │
│   │   PostgreSQL + Elasticsearch                                 │      │
│   │   ┌───────────────────────────────────────────────────┐     │      │
│   │   │  videos: id, title, description, duration, status │     │      │
│   │   │  video_metadata: video_id, resolution, url, size  │     │      │
│   │   │  genres: id, name                                  │     │      │
│   │   │  video_genres: video_id, genre_id                  │     │      │
│   │   └───────────────────────────────────────────────────┘     │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                 RECOMMENDATION SERVICE                       │      │
│   │   Redis (Cache) + ML Model Storage                          │      │
│   │   ┌───────────────────────────────────────────────────┐     │      │
│   │   │  user_preferences: user_id → genre weights        │     │      │
│   │   │  similar_videos: video_id → [related_video_ids]   │     │      │
│   │   │  trending: region → [video_ids]                   │     │      │
│   │   └───────────────────────────────────────────────────┘     │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                   ANALYTICS SERVICE                          │      │
│   │   Kafka + ClickHouse / Redshift                             │      │
│   │   ┌───────────────────────────────────────────────────┐     │      │
│   │   │  view_events: user_id, video_id, timestamp, etc.  │     │      │
│   │   │  Billions of events per day                       │     │      │
│   │   └───────────────────────────────────────────────────┘     │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Schema Design

```sql
-- Users & Profiles
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    subscription_tier VARCHAR(50) DEFAULT 'free',
    subscription_expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    name VARCHAR(100) NOT NULL,
    avatar_url VARCHAR(500),
    maturity_rating VARCHAR(20) DEFAULT 'adult',
    language VARCHAR(10) DEFAULT 'en',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Videos & Content
CREATE TABLE videos (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    duration_seconds INT NOT NULL,
    release_year INT,
    maturity_rating VARCHAR(20),
    thumbnail_url VARCHAR(500),
    status VARCHAR(50) DEFAULT 'processing', -- processing, ready, failed
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE video_resolutions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id UUID REFERENCES videos(id),
    resolution VARCHAR(10) NOT NULL, -- 1080p, 720p, 480p
    bitrate_kbps INT NOT NULL,
    manifest_url VARCHAR(500) NOT NULL,
    file_size_bytes BIGINT
);

-- Watch History (for Continue Watching)
CREATE TABLE watch_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id UUID REFERENCES profiles(id),
    video_id UUID REFERENCES videos(id),
    progress_seconds INT DEFAULT 0,
    completed BOOLEAN DEFAULT FALSE,
    last_watched_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(profile_id, video_id)
);

-- Index for "Continue Watching" query
CREATE INDEX idx_watch_history_profile_recent 
ON watch_history(profile_id, last_watched_at DESC) 
WHERE completed = FALSE;
```

---

## 5. Live Streaming Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      LIVE STREAMING (Twitch-like)                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Streamer (OBS)                                                        │
│        │                                                                │
│        │ RTMP Push                                                      │
│        ▼                                                                │
│   ┌──────────────────┐                                                  │
│   │  Ingest Server   │  (AWS IVS / MediaLive)                          │
│   │  (RTMP Endpoint) │                                                  │
│   └────────┬─────────┘                                                  │
│            │                                                            │
│            ▼                                                            │
│   ┌──────────────────┐     ┌──────────────────┐                        │
│   │   Transcoder     │────►│  Origin Server   │                        │
│   │  (Real-time)     │     │  (HLS Packager)  │                        │
│   └──────────────────┘     └────────┬─────────┘                        │
│                                      │                                  │
│            ┌─────────────────────────┼─────────────────────────┐       │
│            ▼                         ▼                         ▼       │
│   ┌──────────────┐        ┌──────────────┐        ┌──────────────┐    │
│   │  CDN Edge    │        │  CDN Edge    │        │  CDN Edge    │    │
│   │  (US-East)   │        │  (EU-West)   │        │  (Asia)      │    │
│   └──────┬───────┘        └──────┬───────┘        └──────┬───────┘    │
│          │                       │                       │             │
│          ▼                       ▼                       ▼             │
│      Viewers                 Viewers                 Viewers           │
│                                                                         │
│   Latency: 2-5 seconds (Low Latency HLS)                               │
│   Latency: <1 second (WebRTC for ultra-low)                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Live vs VOD Comparison

| Aspect | VOD (Netflix) | Live (Twitch) |
|--------|---------------|---------------|
| **Latency** | Not critical | Critical (2-30 sec) |
| **Encoding** | Pre-transcoded | Real-time |
| **Manifest** | Static | Dynamic (updates every segment) |
| **CDN Cache** | Long TTL | Short TTL (1-3 sec) |
| **Protocol** | HLS/DASH | LL-HLS, WebRTC, RTMP |
| **Scale** | Predictable | Spiky (events) |

### AWS IVS (Interactive Video Service)

```java
@Service
public class LiveStreamService {
    
    @Autowired
    private IvsClient ivsClient;
    
    public StreamKeyResponse createChannel(String channelName) {
        // Create IVS Channel
        CreateChannelResponse response = ivsClient.createChannel(
            CreateChannelRequest.builder()
                .name(channelName)
                .latencyMode(ChannelLatencyMode.LOW) // Low latency mode
                .type(ChannelType.STANDARD)
                .build()
        );
        
        // Create Stream Key
        CreateStreamKeyResponse streamKey = ivsClient.createStreamKey(
            CreateStreamKeyRequest.builder()
                .channelArn(response.channel().arn())
                .build()
        );
        
        return new StreamKeyResponse(
            response.channel().ingestEndpoint(),  // rtmps://ingest.ivs.us-east-1.amazonaws.com:443/app/
            streamKey.streamKey().value(),        // sk_us-east-1_xxxxx
            response.channel().playbackUrl()      // https://xxxxx.us-east-1.playback.live-video.net/api/video/v1/...
        );
    }
}
```

---

## 6. Recommendation System

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     RECOMMENDATION ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                    DATA SOURCES                              │      │
│   │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │      │
│   │  │ Watch    │  │ Search   │  │ Ratings  │  │ Profile  │    │      │
│   │  │ History  │  │ Queries  │  │ & Likes  │  │ Prefs    │    │      │
│   │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │      │
│   └───────┼─────────────┼─────────────┼─────────────┼───────────┘      │
│           │             │             │             │                   │
│           └─────────────┴──────┬──────┴─────────────┘                   │
│                                ▼                                        │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │              KAFKA / KINESIS (Event Stream)                  │      │
│   └────────────────────────────┬────────────────────────────────┘      │
│                                │                                        │
│           ┌────────────────────┼────────────────────┐                   │
│           ▼                    ▼                    ▼                   │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐             │
│   │ Batch ML     │    │ Real-time    │    │ A/B Testing  │             │
│   │ Training     │    │ Scoring      │    │ Framework    │             │
│   │ (SageMaker)  │    │ (Lambda)     │    │              │             │
│   └──────────────┘    └──────────────┘    └──────────────┘             │
│           │                    │                                        │
│           └────────────────────┼────────────────────────────────┐      │
│                                ▼                                 │      │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                    REDIS (Recommendations Cache)             │      │
│   │                                                              │      │
│   │  user:123:recommendations → [video_ids with scores]         │      │
│   │  video:456:similar → [related_video_ids]                    │      │
│   │  trending:us → [video_ids]                                  │      │
│   │                                                              │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Recommendation Types

```java
@RestController
@RequestMapping("/api/recommendations")
public class RecommendationController {
    
    // "Because you watched X" - Content-based
    @GetMapping("/similar/{videoId}")
    public List<Video> getSimilarVideos(@PathVariable String videoId) {
        return recommendationService.getSimilarVideos(videoId, 10);
    }
    
    // "Continue Watching" - Watch history
    @GetMapping("/continue-watching")
    public List<VideoWithProgress> getContinueWatching(@AuthenticationPrincipal User user) {
        return watchHistoryService.getIncomplete(user.getProfileId(), 10);
    }
    
    // "Trending Now" - Popularity-based
    @GetMapping("/trending")
    public List<Video> getTrending(@RequestParam String region) {
        return recommendationService.getTrending(region, 20);
    }
    
    // "Top Picks for You" - Collaborative filtering
    @GetMapping("/personalized")
    public List<Video> getPersonalized(@AuthenticationPrincipal User user) {
        return recommendationService.getPersonalized(user.getProfileId(), 20);
    }
    
    // "New Releases" - Recency-based
    @GetMapping("/new-releases")
    public List<Video> getNewReleases() {
        return videoRepository.findRecentlyAdded(30);
    }
}
```

---

## 7. Search Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      SEARCH ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   User Search: "stranger things season 2"                               │
│                       │                                                 │
│                       ▼                                                 │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                  SEARCH SERVICE                              │      │
│   │                                                              │      │
│   │  1. Query Understanding                                      │      │
│   │     └── Tokenize, spell-check, synonyms                     │      │
│   │                                                              │      │
│   │  2. Search Elasticsearch                                     │      │
│   │     └── Multi-field: title, description, cast, genres       │      │
│   │                                                              │      │
│   │  3. Apply Filters                                           │      │
│   │     └── Maturity rating, availability, language             │      │
│   │                                                              │      │
│   │  4. Personalized Ranking                                    │      │
│   │     └── Boost based on user preferences                     │      │
│   │                                                              │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                       │                                                 │
│                       ▼                                                 │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │              ELASTICSEARCH CLUSTER                           │      │
│   │                                                              │      │
│   │  Index: videos                                               │      │
│   │  {                                                           │      │
│   │    "title": "Stranger Things",                              │      │
│   │    "title.autocomplete": "stranger things",                 │      │
│   │    "description": "A group of kids...",                     │      │
│   │    "genres": ["sci-fi", "horror", "drama"],                 │      │
│   │    "cast": ["Millie Bobby Brown", "Finn Wolfhard"],         │      │
│   │    "release_year": 2016,                                    │      │
│   │    "maturity_rating": "TV-14",                              │      │
│   │    "popularity_score": 95.5                                 │      │
│   │  }                                                           │      │
│   │                                                              │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Scale Numbers (Netflix-like)

```
┌────────────────────────────────────────────────────────┐
│                   SCALE METRICS                         │
├────────────────────────────────────────────────────────┤
│  Users:              200M+ subscribers                  │
│  Concurrent Streams: 10M+ peak                         │
│  Bandwidth:          15% of global internet traffic    │
│  Videos:             15,000+ titles                    │
│  Storage:            Petabytes of video                │
│  CDN Requests:       Billions per day                  │
│  Upload Processing:  Hours of video per minute         │
└────────────────────────────────────────────────────────┘
```

---

## 9. AWS Services Stack

| Component | AWS Service | Purpose |
|-----------|-------------|---------|
| **Upload** | S3 + Pre-signed URLs | Direct client upload |
| **Transcoding** | Elemental MediaConvert | Multi-resolution encoding |
| **Storage** | S3 + S3 Glacier | Video storage + archive |
| **CDN** | CloudFront | Global edge caching |
| **Live Streaming** | IVS / MediaLive | Real-time video |
| **DRM** | AWS Media Services | Content protection |
| **API** | API Gateway + ECS | Backend services |
| **Database** | Aurora + DynamoDB | Metadata + sessions |
| **Cache** | ElastiCache (Redis) | Recommendations, sessions |
| **Search** | OpenSearch | Full-text search |
| **Analytics** | Kinesis + Redshift | Event streaming + warehouse |
| **ML** | SageMaker | Recommendations model |
| **Queue** | SQS | Async processing |

---

## 10. Key Interview Points

1. **CDN is Critical** - Video segments cached at edge = low latency
2. **Pre-signed URLs** - Upload directly to S3, bypass servers
3. **Adaptive Bitrate (ABR)** - HLS/DASH switches quality based on bandwidth
4. **Transcoding Pipeline** - Async processing, multiple resolutions
5. **Signed URLs** - Time-limited access for content protection
6. **Polyglot Persistence** - PostgreSQL (metadata), Redis (cache), Elasticsearch (search)
7. **Event-Driven** - Kafka/Kinesis for analytics, async processing

---

## 11. Common Follow-up Questions

**Q: How to handle viral videos?**
- CDN auto-scales at edge
- Origin shield to reduce origin load
- Pre-warm popular content to edges

**Q: How to reduce buffering?**
- Adaptive bitrate streaming
- Predictive pre-fetching
- Multiple CDN providers (multi-CDN)

**Q: How to protect content (DRM)?**
- Widevine (Android/Chrome), FairPlay (iOS/Safari), PlayReady (Windows)
- Signed URLs + short expiry
- Token-based authentication

**Q: How to handle global users?**
- Multi-region deployment
- CloudFront 400+ edge locations
- Region-specific content catalogs
