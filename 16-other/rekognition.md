# AWS Rekognition

> **Status (2026):** Rekognition **people pathing** was discontinued on Oct 31, 2025. **Streaming Video** (streaming events with Kinesis Video Streams) and **Bulk/Batch Image Analysis** (batch content moderation) are no longer available to new customers since Apr 30, 2026 (accounts using them in the last 12 months keep access). Alternative: process frames/images with the Rekognition Image APIs via Lambda/Step Functions. Other features are unaffected.

- Rekognition is a deep learning based image and video analysis product
- It can identify objects, people, text, activities, content moderation, face detection, face analysis, face comparison, pathing (discontinued) and much more
- The product is per as you use per image or per minute in case of video
- Integrates with application and can be invoked event-driven
- Can analyse live video streams integrating with Kinesis Video Streams (stream processors - closed to new customers since Apr 2026)
- Image APIs (synchronous): `DetectLabels`, `DetectFaces`, `CompareFaces`, `DetectText`, `DetectModerationLabels`, `RecognizeCelebrities`, `DetectProtectiveEquipment` (PPE)
- Video APIs (asynchronous, video in S3): `StartLabelDetection`, `StartFaceDetection`, `StartContentModeration`, `StartSegmentDetection` (shots, black frames, end credits) - completion notification via SNS, results via `Get*` APIs
- **Face collections**: index faces (`IndexFaces` - stores face vectors, not images) and search (`SearchFacesByImage`) - user vectors to aggregate multiple faces per user
- **Face Liveness**: detect that a real person (not a photo/video spoof) is in front of the camera during identity verification
- **Custom Labels**: train a custom object/scene detection model with a small labeled dataset (e.g. detect our product logos, manufacturing defects) - pay per inference hour
- **Custom Moderation** adapters to tune content moderation to our policies; integrates with Amazon Augmented AI (A2I) for human review of low-confidence results
- Typical serverless pipeline: S3 upload -> EventBridge/S3 event -> Lambda -> Rekognition -> store labels in DynamoDB/OpenSearch
- Exam: "moderate user-uploaded images", "identity verification with selfie vs ID photo", "detect PPE on construction sites" -> Rekognition; documents/forms text -> Textract instead
