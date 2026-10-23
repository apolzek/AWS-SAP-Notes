# Amazon Mechanical Turk

- Provides a managed human task outsourcing system
- Provides a set of APIs and a marketplace to outsource jobs to human beings
- Allows requesters to post Human Intelligence Tasks (HITs) to a marketplace
- Tasks are completed by workers, who earn money for it
- Pay per task, perfect for tasks suited to humans rather than ML
- Qualification: worker attribute, we can require a test. A qualification can be a requirement to complete HITs
- Great for data collection, manual processing, image classification
- Still available (2026), with no major new features; pay-per-task + MTurk fee on top of the reward
- Concepts: **Requester** (publishes HITs), **Worker**, **HIT** (task), **Assignment** (one worker's answer to a HIT, a HIT can require N assignments for consensus)
- **Sandbox** environment to test HITs without paying
- Integrations: API/SDK, can be orchestrated with Step Functions (`.waitForTaskToken` callback) or SWF

## MTurk vs Alternatives (Exam)

- **SageMaker Ground Truth**: managed **data labeling** for ML (images, text, video, 3D point clouds) with active learning/auto-labeling; workforce can be MTurk (public), a private workforce or vendors
- **SageMaker Ground Truth Plus**: fully managed labeling done by an AWS expert workforce
- **Amazon Augmented AI (A2I)**: human review of low-confidence ML predictions (Textract, Rekognition, custom models)
- Rule of thumb: generic human micro-tasks => MTurk; ML training data => Ground Truth; human-in-the-loop review of ML inference => A2I
