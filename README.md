# Trueset_Showcase

# TrueSet — Full Technical Breakdown

---

## What is TrueSet

TrueSet is a data annotation platform. AI companies need labeled training data to build computer vision models — for example, images of cars where every car has a box drawn around it. TrueSet manages the entire pipeline: clients upload raw images, a workforce of vetted annotators labels them, and a consensus algorithm combines their work to produce high-quality ground truth data. Think Scale AI but built lean as a startup.

---

## The Three Users

**Admin (you)** — manages everything via Django admin panel. Approves/rejects worker applications, monitors submissions, adjusts pay rates and quality thresholds, views client enquiries from the landing page.

**Client** — an AI company. Logs into the portal, uploads a zip of images, defines what labels they want (car, truck, pedestrian etc), and downloads the completed annotations as JSON files ready for their training pipeline.

**Worker** — an annotator. Applies through the landing page, you approve them manually, they log into the Flutter app and draw bounding boxes around objects in images. They get paid per annotation based on their quality level.

---

<img width="1916" height="896" alt="image" src="https://github.com/user-attachments/assets/59b9091b-924a-4d28-ac37-abaac2ad3a13" />
<img width="1919" height="904" alt="image" src="https://github.com/user-attachments/assets/0adf8b92-7d58-4fff-96f2-6643c6e4375e" />
<img width="1917" height="909" alt="image" src="https://github.com/user-attachments/assets/5f3f9c15-131f-406f-9ad0-e7ff3b292a8e" />
<img width="1914" height="915" alt="image" src="https://github.com/user-attachments/assets/5304e635-f07c-41f8-a324-26ea80bc2c95" />


## Tech Stack — What and Why

### Django (Python) — Backend
The core of the system. Handles all business logic, data storage, authentication, and serves the API. Django was chosen because it has a built-in admin panel which means you get a full management dashboard for free — no need to build one. It handles the database, user management, file uploads, and all the annotation processing logic.

### PostgreSQL — Database
Stores everything — users, projects, tasks, annotations, config values. PostgreSQL was chosen over simpler databases like SQLite because it supports `SELECT FOR UPDATE` which is critical for the task locking system (explained below).

### Django REST Framework — API Layer
Sits on top of Django and turns it into a proper REST API that Flutter and Next.js can talk to. Handles request parsing, serialization (converting database objects to JSON), and permission enforcement.

### JWT Authentication (djangorestframework-simplejwt)
JSON Web Tokens. When a worker or client logs in, Django issues a token — a signed string that proves who they are. Flutter stores this token and sends it with every request. The token contains the user's role (WORKER or CLIENT) so Flutter can route them to the right screen without an extra API call. Tokens expire after 60 minutes, refresh tokens last 1 day. Old refresh tokens are blacklisted after rotation so stolen tokens can't be reused.

### Flutter — Worker + Client Portal
Cross-platform framework from Google. You write one codebase and it runs on Android, iOS, and web. The worker annotation app runs in Chrome (web) and Android. Flutter was chosen because you only need to build once. The UI uses a custom glass morphism design system — semi-transparent cards with warm shadows — consistent with the landing page aesthetic.

### Next.js — Landing Page
React framework with server-side rendering. The landing page needed Next.js specifically because Flutter web renders to a canvas element which search engines cannot read, so it has terrible SEO. Next.js produces proper HTML that Google can index, which means TrueSet can be found through search.

---

<img width="1919" height="878" alt="image" src="https://github.com/user-attachments/assets/fc9f0ed9-c570-4d1e-a056-488502e81306" />
<img width="1909" height="907" alt="image" src="https://github.com/user-attachments/assets/6138c5a4-9360-49a7-8e4e-8db19b0e55be" />
<img width="1916" height="830" alt="image" src="https://github.com/user-attachments/assets/c9db65c3-e6cb-48d0-b338-08df0dd8f3b1" />
<img width="1894" height="906" alt="image" src="https://github.com/user-attachments/assets/731b5cdc-3056-446e-b36c-ee947bed1108" />


## Key Features — Technical Detail

### Task Locking System
The biggest engineering challenge in any annotation platform is race conditions — two workers grabbing the same image simultaneously. TrueSet solves this with database-level locking using PostgreSQL's `SELECT FOR UPDATE SKIP LOCKED`. When a worker requests tasks, Django locks exactly 5 rows in the database atomically. Other workers skip those locked rows and get different tasks. Locks expire after 30 minutes (configurable) so if a worker abandons a task it gets released automatically.

### Weighted Consensus Algorithm
This is TrueSet's core quality mechanism and main differentiator from competitors. Instead of majority voting (where 2 out of 3 workers wins), TrueSet weights each annotation by the worker's reputation score. A level 3 worker with a high reputation has more influence on the final answer than a new level 1 worker.

The process:
1. Multiple workers annotate the same image independently
2. Their bounding boxes are clustered using IoU (Intersection over Union) — a geometric measure of how much two boxes overlap
3. Within each cluster, a weighted average box is calculated using reputation scores as weights
4. Entropy is calculated across all annotations — high disagreement = high entropy = low confidence
5. When confidence crosses the threshold (default 85%) and minimum annotations are met (default 3), the task is marked complete

### Gold Standard Tasks
Hidden quality control. Some tasks have known correct answers that you set. Workers don't know which tasks are gold standard. When a worker submits on a gold standard task, their answer is compared to the known correct answer using IoU. 80%+ match gives +5 reputation, below that gives -10. This continuously measures worker quality without them gaming the system.

### Shard System
Tasks are divided into 3 difficulty tiers (shards) based on entropy score — how much workers disagree on them. Easy tasks (low entropy) go to shard 0, hard disputed tasks go to shard 2. Level 1 workers only see shard 0, level 2 see shards 0 and 1, level 3 see all shards. This means your best workers spend their time on the hardest tasks where quality matters most.

### Worker Level System
Workers start at level 1 and progress based on reputation score. Level 1 earns $4 per annotation, level 2 earns $8, level 3 earns $15 (all configurable). Workers are only paid when the task confidence crosses the threshold — meaning bad annotations don't get paid. This creates a natural incentive for quality.

### Registration Approval Flow
Workers don't self-register. They submit an application through the landing page modal which creates a `RegistrationRequest` record. You see it in Django admin and approve or reject it. Only on approval is a real user account created. This ensures no unknown people ever touch client data.

### Contact Submission System
The landing page has a contact modal with two modes — "Start a project" (client lead) and "Become an annotator" (worker application). Submissions go from the browser to Next.js, Next.js forwards them server-to-server to Django, Django saves them to the `ContactSubmission` model. You see all leads in Django admin.

### Staged Download System (planned)
Clients uploading 50,000 images don't have to wait for all of them to be annotated before seeing results. Every N completed images unlocks a download stage. Clients can start training their model on completed batches while annotation continues.

---

## Security

- **JWT with rotation and blacklisting** — stolen refresh tokens are invalidated
- **Role-based permissions** — `IsWorker` and `IsClient` permission classes enforce at the API level, not just in the view logic
- **Task authorization** — workers can only submit on tasks assigned to them
- **Zip path traversal protection** — uploaded zip files have filenames sanitized with `os.path.basename` to prevent directory traversal attacks
- **Rate limiting** — login and registration endpoints are throttled to 5 requests/minute for anonymous users to prevent brute force
- **Atomic database operations** — pay calculations use Django's `F()` expressions to prevent race conditions where two simultaneous requests could double-count earnings
- **Input validation** — bounding box coordinates are validated server-side against image dimensions before saving

---

## Monorepo Structure

```
trueset/
├── trueset_backend/    Django — API, business logic, database
├── trueset_web/        Flutter — worker + client portal app  
└── landing/            Next.js — marketing + lead capture site
```

Three separate deployable units in one git repository. In production:
- Backend → Railway (Python hosting)
- Landing → Vercel (Next.js hosting, free tier)
- Flutter web → served from same domain at `/portal`

All three communicate only through the Django REST API. The landing page and Flutter portal are completely independent frontends — swapping either one out doesn't affect the other.

---

<img width="1888" height="905" alt="image" src="https://github.com/user-attachments/assets/13e4677a-02c9-4767-a8fa-6ef49e12f140" />
<img width="1919" height="874" alt="image" src="https://github.com/user-attachments/assets/e043db96-9604-44a1-b96f-e85d4d1c53b9" />
<img width="1919" height="902" alt="image" src="https://github.com/user-attachments/assets/6b8e8e50-7c12-4465-93e5-29e63ecc3c52" />



## What's Built vs What's Left

**Built:**
- Full worker annotation flow end to end
- Consensus algorithm with IoU clustering and reputation weighting
- Gold standard quality control
- JWT auth with role-based routing
- Worker registration approval flow
- Task locking with race condition protection
- Client project creation and zip upload
- Annotation export as JSON zip
- Landing page with contact form wired to backend
- Django admin for full platform management

**Left before launch:**
- Media file hosting (S3 or whitenoise)
- Deployment
- Email notifications (post-domain purchase)
.
