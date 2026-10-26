# Smart Traffic Monitoring & Incident Detection on AWS

A cloud-deployed web application that analyses road-camera video with computer vision. It counts and classifies vehicles, detects congestion, stopped vehicles, overspeeding and wrong-direction driving, stores the results in a database, and shows them on a live dashboard.

Built on top of an existing YOLOv8 + ByteTrack traffic-analysis pipeline, extended into a scalable AWS architecture.

## Features

- **Vehicle detection and classification:** car, motorcycle, bus and truck (YOLOv8) with persistent IDs (ByteTrack)
- **Counting:** each vehicle counted once as it crosses a line, with lane-wise and direction-aware counts
- **Incident detection:**
  - congestion level (free, moderate, heavy)
  - stopped or abnormal vehicles, tagged as "in queue" during congestion
  - overspeed (requires calibration) and wrong-direction vehicles
- **Evidence:** snapshot image for each incident and an annotated output video
- **Web dashboard:** upload a video or give a stream URL, track progress, view the traffic chart and incident list
- **REST API** with API-key authentication

## Architecture

```
User browser
    |
Application Load Balancer
    |
EC2 instances in an Auto Scaling Group (Docker: FastAPI + YOLO + OpenCV)
    |                    |                    |
RDS PostgreSQL          S3                CloudWatch
(statistics, incidents) (videos, snapshots) (logs, metrics, alarms)

Also: VPC + Security Groups, IAM roles, Secrets Manager, ECR, CloudFormation
```

| AWS service | Purpose |
|---|---|
| EC2 + Auto Scaling | Runs the application and AI processing; scales on CPU and replaces failed servers |
| Application Load Balancer | Single entry point; health checks |
| RDS PostgreSQL | Jobs, traffic statistics (every 5 s) and incidents; encrypted, private subnets |
| S3 | Original and annotated video, incident snapshots; private, encrypted, auto-expiry |
| CloudWatch | Container logs, custom metrics, dashboard, alarms |
| IAM + Secrets Manager | Instance role with least privilege; API key and DB password stored as secrets |
| ECR | Stores the Docker image |
| CloudFormation | Creates the whole stack from one template (Infrastructure as Code) |

## Tech stack

Python, YOLOv8 (Ultralytics), ByteTrack, OpenCV, EasyOCR, FastAPI, SQLAlchemy, PostgreSQL, Docker, AWS (EC2, ALB, Auto Scaling, RDS, S3, CloudWatch, IAM, Secrets Manager, ECR, CloudFormation), HTML/JavaScript, Chart.js

## Project structure

```
cloud/            Server code: API, jobs, database, storage, incident detection, dashboard
deploy/           CloudFormation template and deploy scripts
tests/            Automated tests
Dockerfile        Container image
docker-compose.yml  Local run with PostgreSQL
README-v4.md      Documentation of the original detection pipeline
*.py              Original detection pipeline (detector, tracker, counter, speed, plates)
```

## How incident detection works

- **Stopped vehicle:** a vehicle on the road that stays within a small distance (relative to its own size) for 8 seconds.
- **Congestion:** based on how many vehicles are on the road and what share of them move slowly. A new level must hold for 3 seconds before it switches, to avoid flicker.
- All thresholds can be changed in `IncidentConfig` in `cloud/incidents.py`.

## Run locally

Requirements: Docker Desktop.

```bash
APP_API_KEY=choose-a-key docker compose up --build
```

Open http://localhost:8000, enter the key, and upload a video.

Run the tests (no models or AWS needed):

```bash
pip install -r requirements-dev.txt
pytest
```

## Deploy to AWS

Requirements: an AWS account, AWS CLI v2 configured (`aws configure`), Docker Desktop, and Git Bash on Windows.

```bash
./deploy/01-push-image.sh        # build the image and push it to ECR
./deploy/02-deploy-stack.sh      # create the whole AWS stack, prints the app URL and API key
./deploy/03-teardown.sh --delete-ecr   # delete everything when finished
```

Options for the deploy script, set as environment variables:

- `ALLOWED_CIDR` limits who can reach the load balancer (defaults to your current IP)
- `MAX_SIZE` sets the maximum number of servers
- `INSTANCE_TYPE` selects the server size
- `ALERT_EMAIL` receives CloudWatch alarm emails

The stack costs roughly US$0.25 per hour at demo size (an estimate; check AWS pricing). Always tear it down when finished.

## Security

- API-key authentication on all API routes
- Private database; security-group chain (load balancer to app to database); no SSH port (use SSM Session Manager)
- Secrets in Secrets Manager, not in code or environment settings
- IMDSv2 required; S3 fully private and encrypted
- Uploads checked for type, size and that they are readable video
- Stream URLs restricted to public addresses (SSRF protection)
- Files served through short-lived signed links

## Limitations

- Congestion and stopped-vehicle thresholds are heuristics and need tuning per camera.
- Speed estimates are only meaningful after calibration; uncalibrated runs raise no overspeed alerts.
- Number-plate reading is weak without a dedicated plate-detection model.
- Processing runs on CPU, so it is slower than real time.
- A job runs on the server that received it. Extra servers add concurrency, not speed for one video. An SQS queue would remove this limit.
- Prototype security: one shared API key and plain HTTP by default. Provide an ACM certificate for HTTPS in real use.

## Future work

- SQS job queue so any server can take any job
- GPU instances for real-time streams
- HTTPS with a domain and per-user login (Cognito)
- Dedicated plate-detection model and a larger YOLO model
- Email or SMS alerts for high-severity incidents (SNS)

## Screenshots

Add screenshots of the dashboard, the incident list and the AWS console here.

## License and credits

YOLOv8 is provided by Ultralytics under AGPL-3.0. Check the licenses of all dependencies before any commercial use.

Author: s0ulm8
