# Kinesis Video Streams

- Allows us to ingest live video data from producers
- Producers can be security cameras, smartphones, cars, drones, time-serialised audio, thermal, depth and RADAR data
- Consumers can access the data frame-by-frame or as needed
- Kinesis can persist and encrypt data in-transit and at rest
- We can not access data directly via storage, only via APIs!
- Data is not stored in its original format. Data is indexed and stored in a structured way
- Kinesis Video Streams integrates with other AWS services such as Rekognition and Connect
- Producers use the KVS Producer SDK (C, C++, Java, Android) or GStreamer plugin; data organized in **fragments** (sequence of frames) with producer/server timestamps
- Retention: from 0 (no storage, real-time only) up to 10 years (configurable per stream); encrypted at rest with KMS
- Consumption: `GetMedia` (real-time), `GetMediaForFragmentList`, **HLS** and **MPEG-DASH** streaming sessions for playback in browsers/apps, `GetClip` (MP4 clip), `GetImages` (extract images/thumbnails from video)
- **KVS with WebRTC**: ultra-low-latency two-way real-time media (peer-to-peer, signaling channels, STUN/TURN) for camera viewing, video calls, IoT devices; can also ingest WebRTC to a stream for storage
- Edge Agent / Image generation to S3 for ML pipelines
- Integrations: Rekognition Video (stream processors - closed to new customers since Apr 2026, use `GetImages` + Rekognition Image), Amazon Connect (call audio streaming for real-time transcription with Transcribe), SageMaker (custom ML on frames), Lambda
- One stream per device/producer (e.g. one per camera); pricing per GB ingested, consumed and stored
- KVS vs Kinesis Data Streams: KVS = binary time-encoded media (video/audio/radar), KDS = generic data records (max 1 MB) <span style="color: red;">EXAM</span>
