# Protected HLS Proof of Concept

This experiment demonstrates how to stream a short test video through HLS while preventing direct, permanent access to the underlying media URLs.

> Use only synthetic, public-domain, or otherwise authorized media for this experiment.

## Goal

Build a small local proof of concept that shows:

1. A short test video can be generated or imported.
2. FFmpeg converts it to a single 720p HLS stream.
3. The HLS files live in a non-public/private storage directory.
4. The application issues a short-lived signed playback URL.
5. Every manifest/segment request validates the token.
6. An expired, missing, or modified token returns `403 Forbidden`.
7. The React frontend plays the protected HLS stream using `hls.js`.

## Architecture

```text
Generate/import 30-60 second test clip
                |
                v
              FFmpeg
                |
                v
             720p HLS
                |
                v
      private storage directory
                |
      +---------+---------+
      |                   |
 master.m3u8         segment files
      |                   |
      +---------+---------+
                |
                v
         Protected Express API
                |
        HMAC signed token
                |
                v
             hls.js
                |
                v
             Browser
```

## Suggested repository layout

```text
experiments/protected-hls-demo/
  README.md
  server/
    index.mjs
    signing.mjs
  client/
    ProtectedHlsDemo.tsx
  scripts/
    generate-test-video.sh
    encode-hls.sh
  storage/
    .gitkeep
```

The generated video and HLS segments should not be committed to Git.

## Test video

Generate a synthetic video with FFmpeg so the experiment has no copyright dependency.

Example concept:

```bash
ffmpeg \
  -f lavfi -i "testsrc2=size=1280x720:rate=30" \
  -f lavfi -i "sine=frequency=1000:sample_rate=48000" \
  -t 30 \
  -c:v libx264 \
  -c:a aac \
  test-source.mp4
```

Then encode it to HLS:

```bash
ffmpeg \
  -i test-source.mp4 \
  -vf "scale=-2:720" \
  -c:v libx264 \
  -preset veryfast \
  -crf 22 \
  -c:a aac \
  -b:a 128k \
  -hls_time 4 \
  -hls_playlist_type vod \
  -hls_segment_filename "storage/test-video/segment_%03d.ts" \
  storage/test-video/master.m3u8
```

## Media identity

Do not use a storage-provider URL as the identity of the video.

Use an application-owned ID:

```text
media_id = test-video
storage_key = test-video/master.m3u8
```

The API translates `media_id` into the private storage path.

## Signed playback token

For the experiment, sign these values:

```text
media_id
expires
```

Conceptually:

```text
payload = "test-video:1760000000"

signature = HMAC_SHA256(
  payload,
  SERVER_SECRET
)
```

Playback URL:

```text
/stream/test-video/master.m3u8?expires=1760000000&sig=<signature>
```

The secret must remain server-side and must never be shipped in the React/Vite bundle.

## API flow

### Request playback session

```http
GET /api/playback/test-video
```

Server checks whether the caller is allowed to watch the media, then creates a short-lived token.

Example response:

```json
{
  "mediaId": "test-video",
  "playbackUrl": "/stream/test-video/master.m3u8?expires=...&sig=...",
  "expiresAt": "..."
}
```

For this local experiment, use a short lifetime such as 60 seconds so expiration is easy to test.

## Protect every HLS request

It is not enough to protect only `master.m3u8`.

The playlist references segment URLs. Those segment requests must also require authorization.

The server should validate:

```text
1. signature exists
2. expiration exists
3. expiration is still in the future
4. signature matches media_id + expiration
5. requested path belongs to that media_id
6. requested file exists
```

Otherwise return:

```http
HTTP/1.1 403 Forbidden
```

## Playlist rewriting

Because HLS segment URLs are normally relative, the protected server can rewrite playlist entries so the authorization parameters are propagated to segment requests.

Example generated playlist:

```text
#EXTM3U
#EXT-X-TARGETDURATION:4
#EXTINF:4.000,
segment_000.ts?expires=...&sig=...
#EXTINF:4.000,
segment_001.ts?expires=...&sig=...
```

For a production CDN, signed cookies or CDN-native token authentication may be preferable to appending a token to every segment URL.

## React / hls.js test player

The existing project already includes `hls.js`, so the demo component can:

```text
1. call /api/playback/test-video
2. receive the signed manifest URL
3. initialize Hls
4. loadSource(playbackUrl)
5. attachMedia(videoElement)
```

Native-HLS browsers can assign the URL directly to the `<video>` element.

## Security test checklist

Run these tests deliberately.

### Valid playback

```text
GET /api/playback/test-video
       |
       v
signed URL
       |
       v
video plays
```

Expected: `200` and successful playback.

### Missing token

```text
/stream/test-video/master.m3u8
```

Expected: `403 Forbidden`.

### Modified signature

Change one character in `sig`.

Expected: `403 Forbidden`.

### Expired token

Wait until the 60-second test token expires and reload.

Expected: `403 Forbidden`.

### Wrong media ID

Take a valid token for `test-video` and request another media ID.

Expected: `403` or `404`.

### Direct storage request

Attempt to access the physical storage directory directly.

Expected: impossible from the browser because the storage directory is not mounted as public static content.

## Public vs protected comparison

For educational testing, the demo can expose two routes:

```text
/demo/public
/demo/protected
```

The public demo intentionally shows the weakness of a permanent static media URL.

The protected demo shows that the manifest and segments require a valid short-lived authorization token.

Do not use the intentionally public route for real/private media.

## Important limitation

Signed URLs do not make video impossible to copy.

They primarily protect against:

- permanent URL sharing;
- casual hotlinking;
- unauthorized embedding;
- direct origin access;
- bandwidth theft;
- long-lived leaked media URLs.

A user who is legitimately allowed to display video on their device can still potentially capture it. Premium-content systems may additionally use DRM and forensic watermarking.

## Production evolution

Once this local experiment works, the same model can evolve to:

```text
React / mobile app
       |
       v
Playback authorization API
       |
       v
short-lived token
       |
       v
CDN (Bunny / CloudFront / another provider)
       |
       v
private object storage
```

Keep these abstractions provider-independent:

```text
media_id
storage_key
playback authorization
HLS/DASH manifest
```

Do not make a Bunny-specific or storage-provider-specific URL the permanent identity of the video.

## Recommended next Codex task

Implement this proof of concept without modifying the main MovieVerse player flow:

1. Create an isolated Express demo server.
2. Add HMAC signing and verification.
3. Add protected manifest/segment routes.
4. Add the synthetic-video generation script.
5. Add the 720p HLS encoding script.
6. Add an isolated React `ProtectedHlsDemo` component using the existing `hls.js` dependency.
7. Add automated/manual tests for valid, invalid, modified, and expired tokens.
8. Keep generated media and secrets out of Git.

The first implementation should remain local-only and provider-independent. Bunny/CDN integration can be tested after the protection model is proven locally.
