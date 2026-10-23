# Elastic Transcoder and AWS Elemental MediaConvert

> **Status (2026):** **Amazon Elastic Transcoder is discontinued** - end of support on **November 13, 2025** (console and resources no longer accessible). **AWS Elemental MediaConvert** is the only answer for file-based transcoding. AWS Elemental **MediaStore** was also discontinued on the same date (use S3 or MediaPackage as origin).

- MediaConvert a new AWS product replacing Elastic Transcoder
- MediaConvert is a superset of features provided by Transcoder
- Both of the systems are file based video transcoding systems
- Serverless - pay for resources used
- For both we add jobs to pipelines (ET) or queues (MC)
- Files are loaded from S3, processed and stored back to S3
- MC supports EventBridge for job signalling => processes in MC can generate an event in an EventBridge bus
- ET/MC architecture example:

![ET/MC architecture](images/ElasticTranscoder&MediaConvert.png)

- We use these products when we need to convert media in a serverless, event-driven media processing pipeline

## Choosing between ET and MC

- ET is legacy (now discontinued), by default we should chose MC
- MC supports more codecs, design for larger volume and parallel processing
- MC supports reserved pricing
- MC is cheaper per minute than ET was (e.g. ~$0.0075/min vs $0.015/min basic tier)

## MediaConvert Details

- Jobs submitted to **queues**: on-demand (pay per minute of output, by tier: Basic / Professional) or **reserved queues** (fixed monthly price, 12 month commitment, for steady volume)
- **Job templates** and **output presets** for repeatable settings
- Outputs: MP4, **HLS**, **DASH**, CMAF, MSS; codecs H.264, H.265/HEVC, AV1, VP9, ProRes, etc.; captions, audio normalization, thumbnails/frame capture, watermarking
- **QVBR** (quality-defined variable bitrate) to reduce storage/CDN costs
- DRM via **SPEKE** (Speke API with key providers), encryption of outputs
- **Accelerated transcoding** for long/large files (parallelizes a job)
- Events to **EventBridge** (job `COMPLETE`, `ERROR`, progress) => trigger Lambda/Step Functions/SNS
- Typical serverless VOD pipeline: S3 upload => EventBridge/S3 event => Lambda (or Step Functions) => MediaConvert job => S3 output => CloudFront distribution (+ DynamoDB for metadata)

## Related Media Services

- **MediaLive**: live video encoding (broadcast-grade), pairs with **MediaPackage** (just-in-time packaging/origin, DRM, time-shift) and CloudFront
- **MediaTailor**: server-side ad insertion and channel assembly
- **Amazon IVS** (Interactive Video Service): managed low-latency live streaming for interactive apps (simplest live streaming answer)
- **Kinesis Video Streams**: ingest video from devices/cameras for ML/analytics (not for viewer delivery at scale)
