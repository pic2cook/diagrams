# Pic2Cook

[![Python](https://img.shields.io/badge/python-3.12-blue.svg?logo=python&logoColor=white)](https://www.python.org/) [![Node](https://img.shields.io/badge/node-24-green.svg?logo=nodedotjs&logoColor=white)](https://nodejs.org/) [![Last Commit](https://img.shields.io/github/last-commit/pic2cook/diagrams/main?path=pic2cook%2FREADME.md&label=last%20updated&logo=github&logoColor=white)](https://github.com/pic2cook/diagrams/blob/main/pic2cook/README.md)

## Architecture Diagram

```mermaid
---
config:
  theme: "neutral"
---
flowchart TB
    subgraph Users["User"]
        Browser["Browser"]
    end

    subgraph GCP["Google Cloud Platform"]

        subgraph Network["Networking"]
            CloudCDN["Google Cloud CDN"]
            LB["Load Balancer"]
        end

        subgraph CloudRun["Cloud Run"]
            WebService["pic2cook-web<br/>Next.js"]
            APIService["pic2cook-api<br/>FastAPI"]
            WorkerService["pic2cook-worker<br/>Python"]
        end

        subgraph VPCNetwork["VPC Network"]
            subgraph CloudSQL["Cloud SQL"]
                PostgreSQL[("PostgreSQL<br/>Private IP")]
            end

            subgraph Memorystore["Memorystore"]
                Redis[("Redis<br/>Private IP")]
            end
        end

        subgraph Storage["Storage"]
            GCS["Cloud Storage"]
            ArtifactRegistry["Artifact Registry"]
            SecretManager["Secret Manager<br/>(Secrets)"]
        end

        subgraph AI["AIaaS"]
            subgraph VertexAI["Vertex AI"]
                Gemini["Gemini 2.5 Flash<br/>(Recipe Generation)"]
                Imagen["Imagen 4 Ultra<br/>(Image Generation)"]
                Embeddings["text-multilingual-embedding-002<br/>(Semantic Search)"]
            end
            VisionAPI["Vision API<br/>(Food Detection)"]
        end

        subgraph IAM["IAM"]
            WorkloadIdentity["Workload Identity Federation"]
            ServiceAccount["Service Account"]
        end
    end

    subgraph CICD["CI/CD"]
        GitHub["GitHub Actions"]
    end

    %% Flow
    Browser --> LB
    LB --> WebService
    LB --> CloudCDN
    CloudCDN --> GCS

    %% Service to Service (Internal)
    WebService <-->|Authenticated<br/>Internal Call| APIService

    %% Backend logic
    APIService -.->|Direct VPC Egress| PostgreSQL
    APIService -.->|Direct VPC Egress| Redis
    WorkerService -.->|Direct VPC Egress| PostgreSQL
    WorkerService -.->|Direct VPC Egress| Redis

    APIService --> Embeddings

    WorkerService --> VisionAPI
    WorkerService --> Gemini
    WorkerService --> Imagen

    %% Storage & Secrets
    APIService --> GCS
    WorkerService --> GCS
    WebService --> GCS
    SecretManager -.->|Mount| WebService
    SecretManager -.->|Mount| APIService
    SecretManager -.->|Mount| WorkerService

    %% CI/CD
    GitHub --> ArtifactRegistry
    GitHub --> CloudRun
    GitHub <--> WorkloadIdentity
    WorkloadIdentity --> ServiceAccount
```

## Analysis Request Flow

```mermaid
sequenceDiagram
    actor User
    participant Web as Web Client (Next.js)
    participant API as API Server (FastAPI)
    participant Redis as Redis (JobQueue & PubSub)
    participant Worker as Analysis Worker
    participant Vision as Vision API
    participant Gemini as Vertex AI (Gemini)
    participant GCS as Cloud Storage
    participant DB as PostgreSQL

    User->>Web: Uploads food photo
    Web->>API: POST /api/recipes/analyze (image)
    activate API
    
    API->>GCS: Upload Image
    activate GCS
    GCS-->>API: Image URL
    deactivate GCS

    API->>Redis: Create Job (Pending) & Enqueue
    activate Redis
    Redis-->>API: OK
    deactivate Redis

    API-->>Web: Returns job_id
    deactivate API

    Web->>API: GET /api/analysis/sse?job_id={job_id}
    activate API
    Note right of Web: Subscribes to Server-Sent Events

    API->>Redis: Subscribe to job channel
    
    Worker->>Redis: Dequeue Job
    activate Worker
    
    Worker->>Redis: Update Job (Processing)
    Redis-->>API: Pub/Sub Message (Processing)
    API-->>Web: SSE Event (Processing)

    par Vision Analysis (If enabled)
            Worker->>Vision: Analyze Image
            activate Vision
            Vision-->>Worker: Labels & Confidence
            deactivate Vision
            Worker->>Redis: Update Job (Vision Done)
    and Gemini Analysis
            Worker->>Gemini: Analyze Food & Generate Recipe
            activate Gemini
            Gemini-->>Worker: Recipe Data
            deactivate Gemini
    end

    Worker->>Redis: Update Job (Recipe Structuring)
    Redis-->>API: Pub/Sub Message (Recipe Structuring)
    API-->>Web: SSE Event (Recipe Structuring)

    Worker->>Gemini: Generate Showcase Image (If enabled)
    activate Gemini
    Gemini-->>Worker: Image Base64
    deactivate Gemini
    
    Worker->>GCS: Upload Showcase Image
    activate GCS
    GCS-->>Worker: URL
    deactivate GCS
    
    Worker->>DB: Save Recipe
    activate DB
    DB-->>Worker: Recipe ID
    deactivate DB

    Worker->>Redis: Update Job (Succeeded)
    deactivate Worker

    Redis-->>API: Pub/Sub Message (Succeeded)
    API-->>Web: SSE Event (Completed, recipe_id)
    deactivate API

    Web->>Web: Redirect to /recipes/{recipe_id}
```

## Project Dependencies (Auto-generated)

### Backend (Python/FastAPI)
| Library | Constraint |
| :--- | :--- |
| fastapi | >=0.121.3 |
| uvicorn[standard | latest |

### Frontend (Node.js/Next.js)
| Package | Version |
| :--- | :--- |
| @google-cloud/opentelemetry-cloud-trace-exporter | ^3.0.0 |
| @microsoft/clarity | ^1.0.2 |
| @opentelemetry/api | ^1.9.0 |
| @opentelemetry/instrumentation-fetch | ^0.208.0 |
| @opentelemetry/instrumentation-http | ^0.208.0 |
| @opentelemetry/resources | ^2.2.0 |
| @opentelemetry/sdk-node | ^0.208.0 |
| @opentelemetry/sdk-trace-base | ^2.2.0 |
| @opentelemetry/semantic-conventions | ^1.38.0 |
| @radix-ui/react-accordion | ^1.2.12 |
| @radix-ui/react-alert-dialog | ^1.1.15 |
| @radix-ui/react-avatar | ^1.1.11 |
| @radix-ui/react-checkbox | ^1.3.3 |
| @radix-ui/react-collapsible | ^1.1.12 |
| @radix-ui/react-dialog | ^1.1.15 |
| @radix-ui/react-dropdown-menu | ^2.1.16 |
| @radix-ui/react-popover | ^1.1.15 |
| @radix-ui/react-progress | ^1.1.8 |
| @radix-ui/react-scroll-area | ^1.2.10 |
| @radix-ui/react-select | ^2.2.6 |
| @radix-ui/react-separator | ^1.1.8 |
| @radix-ui/react-slider | ^1.3.6 |
| @radix-ui/react-slot | ^1.2.4 |
| @radix-ui/react-tooltip | ^1.2.8 |
| @serwist/turbopack | ^10.0.0-preview.14 |
| @t3-oss/env-nextjs | ^0.13.8 |
| @tabler/icons-react | ^3.35.0 |
| @tanstack/react-devtools | ^0.8.4 |
| @tanstack/react-form-devtools | ^0.2.5 |
| @tanstack/react-form | ^1.27.2 |
| @tanstack/react-query-devtools | ^5.91.1 |
| @tanstack/react-query | ^5.90.12 |
| ahooks | ^3.9.6 |
| axios | ^1.13.2 |
| better-auth | ^1.4.6 |
| canvas-confetti | ^1.9.4 |
| class-variance-authority | ^0.7.1 |
| clsx | ^2.1.1 |
| es-toolkit | ^1.42.0 |
| import-in-the-middle | ^2.0.0 |
| jotai-devtools | ^0.13.0 |
| jotai-location | ^0.6.2 |
| jotai | ^2.16.0 |
| luxon | ^3.7.2 |
| motion | ^12.23.25 |
| next-intl | ^4.5.8 |
| next | ^16.0.8 |
| react-dom | ^19.2.1 |
| react-icons | ^5.5.0 |
| react-markdown | ^10.1.0 |
| react | ^19.2.1 |
| remark-gfm | ^4.0.1 |
| require-in-the-middle | ^8.0.1 |
| serwist | ^10.0.0-preview.14 |
| sonner | ^2.0.7 |
| tailwind-merge | ^3.4.0 |
| zod | ^4.1.13 |
