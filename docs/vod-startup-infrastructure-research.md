# VOD Startup Infrastructure Research

## Purpose

This note summarizes our research discussion about how an authorized video-on-demand startup can ingest source media, transcode it into multiple qualities, manage audio/subtitles and media IDs, stream through HLS/DASH, estimate costs, and grow infrastructure incrementally.

> This architecture assumes the operator has the rights/authorization to ingest and distribute the media.

## Core media pipeline

A typical VOD processing flow is:

```text
Authorized source / upload / import
        |
        v
Ingestion worker
        |
        v
Source container (e.g. MKV)
        |
        v
ffprobe / media inspection
        |
        +--> identify video tracks
        +--> identify audio/languages
        +--> identify subtitle tracks
        |
        v
Assign internal media ID
        |
        v
Job queue
        |
        v
Transcoding workers
   |       |       |
 1080p    720p    480p
   \       |       /
        audio tracks
        subtitles
            |
            v
       HLS / DASH / CMAF
            |
            v
       Object storage
            |
            v
            CDN
            |
            v
       Adaptive player
```

## Source inspection

An MKV is a container and can contain several streams, for example:

```text
Video: HEVC 1080p
Audio: English DTS 5.1
Audio: French AC3 5.1
Subtitle: English ASS
Subtitle: French SRT
```

`ffprobe` can inspect these tracks. The application should record track metadata separately from the physical media files.

Example logical schema:

```text
media
- id
- title
- duration
- status
- manifest_url

media_tracks
- media_id
- type (video/audio/subtitle)
- language
- codec
- channels
```

## Adaptive bitrate transcoding

Rather than stream the original MKV directly, generate an adaptive bitrate ladder such as:

| Rendition | Example bitrate |
|---|---:|
| 1080p | ~3.5-5 Mbps |
| 720p | ~1.8-2.5 Mbps |
| 480p | ~0.8-1.2 Mbps |

The exact ladder should be determined by source quality, codec, target devices, and desired quality/cost tradeoff.

Audio should normally remain separate from the video renditions. This prevents duplicating every video quality for every language.

Example:

```text
Video
- 1080p
- 720p
- 480p

Audio
- English AAC
- French AAC
- Japanese AAC

Subtitles
- English WebVTT
- French WebVTT
```

A player can then select 720p video + French audio + English subtitles without requiring a unique combined file.

## HLS/DASH packaging

A packaged HLS asset could look roughly like:

```text
/media/{media_id}/
  master.m3u8
  video/
    1080p/
      index.m3u8
      segment001.m4s
      ...
    720p/
    480p/
  audio/
    en/
    fr/
  subtitles/
    en.vtt
    fr.vtt
```

The player chooses segments based on measured network conditions. Common player technologies include hls.js, Shaka Player, AVPlayer, and Android Media3/ExoPlayer.

## Media IDs

The media ID normally belongs in the application/database rather than being embedded into the encoded video itself.

Example:

```text
media_id = 847291
manifest = /media/847291/master.m3u8
```

The API can expose the appropriate playback manifest while keeping storage layout independent from human-readable titles.

## Security considerations

Treat all incoming media as untrusted input.

- Isolate ingestion and FFmpeg workers in restricted containers/VMs.
- Validate file sizes and formats.
- Keep ingestion directories inaccessible from the public web/CDN.
- Restrict worker permissions.
- Verify generated outputs before publishing them.
- Use signed playback URLs/tokens where appropriate.

## Cost drivers

The major cost categories are:

1. Video delivery/CDN bandwidth.
2. Encoded media storage.
3. Transcoding.
4. Origin infrastructure.
5. Application/database/search infrastructure.
6. Backups and replication.
7. Monitoring/security.

At meaningful streaming scale, delivery bandwidth is often substantially more important than ordinary website/API traffic.

A useful capacity formula is:

```text
monthly video traffic
≈ average concurrent viewers
  × average delivered bitrate
  × seconds/month
  ÷ 8
```

For example, 8,000 concurrent streams averaging 3 Mbps represents roughly 24 Gbps of continuous delivery and several petabytes per month.

This is why CDN pricing and negotiated bandwidth become extremely important at scale.

## Large catalog considerations

A catalog such as tens of thousands of movies plus hundreds of thousands of episodes should not normally be provisioned on day one.

Infrastructure and catalog should grow incrementally.

A mature catalog can also use hot/warm/cold strategies:

```text
HOT
new/popular content
high cache coverage

WARM
moderate demand

COLD
rarely viewed content
cheaper origin/archive storage
```

The source master may be archived to inexpensive storage after successful transcoding, or removed when the content-management requirements permit it.

## What early-stage companies commonly do

Research into current VOD platforms and public company case studies showed a recurring pattern:

### Stage 1: managed video

Early teams commonly outsource the complicated media layer to a managed service such as Mux, Cloudflare Stream, Bunny Stream, or a managed AWS architecture.

The startup concentrates on:

```text
frontend
backend/API
accounts
catalog metadata
search
recommendations
subscriptions/ads
CMS
analytics
```

while the video provider handles some or all of:

```text
upload/import
transcoding
adaptive bitrate ladder
HLS/DASH packaging
storage
CDN delivery
playback IDs
```

This reduces engineering and operational overhead before product-market fit.

### Stage 2: managed/hybrid

Once traffic and catalog size become meaningful, the company may separate its API, database, cache, and search infrastructure while continuing to outsource the media pipeline.

### Stage 3: selective self-hosting

When managed video bills become significant and predictable, bringing selected components in-house can become economical.

A common candidate is transcoding:

```text
source
  |
  v
self-hosted FFmpeg CPU/GPU workers
  |
  v
HLS/CMAF output
  |
  v
object storage
  |
  v
CDN
```

Dedicated encoding hardware is economically attractive when utilization is consistently high.

### Stage 4: large-scale infrastructure

At multi-petabyte delivery scale, companies can justify more specialized infrastructure:

- dedicated origin clusters
- large storage clusters
- custom FFmpeg pipelines
- HEVC/AV1 optimization
- origin shielding
- negotiated CDN contracts
- multiple CDNs
- QoE monitoring
- custom capacity planning

The key principle is to reach this architecture progressively rather than building it before the traffic exists.

## Why not begin with one giant dedicated server?

Putting everything on one machine can look inexpensive:

```text
single server
|- API
|- PostgreSQL
|- Redis
|- FFmpeg
|- media storage
|- HLS origin
`- downloads/uploads
```

but creates several problems:

- one hardware failure can take down everything;
- FFmpeg can consume CPU needed by the API;
- encoding disk I/O competes with playback/origin I/O;
- the database competes for memory and disk resources;
- migration becomes harder as the service grows.

Small startups can still begin with very few machines, but the software should keep the responsibilities logically separated so they can later be moved independently.

## Practical startup architecture

A cost-conscious first version could be:

```text
Users
  |
  v
CDN / managed video delivery
  |
  +-------------------+
  |                   |
Frontend/API      Managed video service
  |                   |
  v                   +--> transcode
PostgreSQL             +--> package
Redis/jobs             +--> store
                      `--> deliver
```

Potential application stack:

- Backend: Go, Node.js, or Python
- Database: PostgreSQL
- Cache/jobs: Redis
- Video: Bunny Stream, Cloudflare Stream, Mux, or equivalent
- Search: PostgreSQL initially; dedicated search later if required
- Playback: HLS/DASH-capable web/mobile player

The startup may only need one or two modest application/database servers initially because the video platform handles the expensive media work.

## Managed-platform examples from the research

### Mux

Public Mux case studies illustrate companies choosing managed video because building encoding, storage, playback, and delivery internally would consume engineering resources better spent on their product.

Mux pricing is structured around video input, storage, and delivery usage, with volume arrangements available as companies grow.

### Cloudflare Stream

Cloudflare Stream provides upload/import, encoding, adaptive bitrate streaming, HLS/DASH playback, storage, CDN delivery, video UIDs, and signed playback functionality.

Its pricing model is based primarily on minutes stored and minutes delivered rather than requiring customers to assemble separate encoding, storage, and CDN products.

### Bunny Stream

Bunny Stream is particularly interesting for cost-sensitive startups because it combines video encoding, storage, playback, and CDN delivery. Its published pricing has historically emphasized inexpensive per-GB storage/delivery and included transcoding, making it worth comparing for bootstrapped products.

### AWS managed architecture

A common AWS VOD architecture is:

```text
S3 upload
   |
   v
Lambda / Step Functions
   |
   v
MediaConvert
   |
   v
S3 encoded output
   |
   v
CloudFront
   |
   v
Viewer
```

This gives significantly more infrastructure control but introduces more individual services and operational complexity.

## A useful real-world progression

The case studies suggest this progression:

| Company stage | Likely strategy |
|---|---|
| Prototype | Managed video |
| First customers | Managed video |
| Growing catalog | Managed or hybrid |
| Large predictable video bill | Selective self-hosting |
| Hundreds of TB / PB delivery | Dedicated origins/storage + volume CDN |
| Very large streamer | Highly customized pipeline and CDN strategy |

An important example from the research was CoStar Group: after operating with third-party video SaaS, it eventually moved more video processing into an AWS architecture using MediaConvert and related AWS services, reporting major cost savings. The lesson is not that every startup should immediately self-host; rather, once usage is sufficiently large and predictable, engineering investment can reduce per-unit infrastructure cost.

## Recommended growth strategy

Do not buy infrastructure for an eventual massive catalog before demand exists.

Instead:

```text
START
managed video
small API/database footprint
hundreds/thousands of authorized titles
        |
        v
MEASURE
average watch time
average bitrate
GB/TB delivered
storage/title
new titles/month
transcode hours/month
popular-title concentration
        |
        v
OPTIMIZE
replace the expensive component first
        |
        v
SCALE
add encoding/storage/origin nodes only as required
```

The most important measurements before designing custom infrastructure are:

- average source and encoded size;
- average content duration;
- average delivered bitrate;
- average viewer watch time;
- peak concurrent streams;
- monthly delivered TB/PB;
- new content ingested per month;
- transcoding hours per month;
- percentage of traffic generated by the hottest titles;
- CDN cache hit ratio.

Those numbers tell the company whether it should optimize storage, encoding, CDN delivery, or application infrastructure first.

## Main conclusion

For an early authorized VOD startup, the strongest recurring pattern is:

**Start managed, keep application data/business logic portable, measure real usage, and progressively own expensive infrastructure only when scale makes doing so economically rational.**

Building the final multi-petabyte architecture before the audience and catalog exist usually creates unnecessary fixed cost and operational complexity.

## References discussed

- Mux pricing and customer case studies: https://www.mux.com/docs/pricing/overview
- Cloudflare Stream documentation: https://developers.cloudflare.com/stream/
- Cloudflare Stream pricing: https://developers.cloudflare.com/stream/pricing/
- Bunny Stream: https://bunny.net/stream/
- AWS Video on Demand guidance: https://docs.aws.amazon.com/wellarchitected/latest/streaming-media-lens/scenario-video-on-demand-streaming.html
- AWS Video on Demand solution: https://docs.aws.amazon.com/solutions/video-on-demand-on-aws/
- AWS MediaConvert pricing: https://aws.amazon.com/mediaconvert/pricing/

Pricing changes over time, so current provider pricing should always be rechecked before making purchasing decisions.
