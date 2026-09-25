# JusticeX: Legal Case Analytics & Recommendation System

JusticeX helps citizens who may be selected as **lay judges** in Taiwan understand real court cases. Taiwan introduced its Citizen Judge system in 2023, and most people chosen to sit on a trial have no legal background. JusticeX turns published criminal judgments into structured, searchable cases, recommends similar past cases, and lets users compare their own reading of a case with the actual verdict.

This repository contains the **Django REST API backend**. The mobile client lives in [JusticeX-frontend](https://github.com/ylin1202/JusticeX-frontend).

> Capstone project, Bachelor of Information Management, National Taipei University of Business (2023)

## Highlights

* **Automated Judgment Pipeline**: New judgments are scraped from the Judicial Yuan judgment website on a schedule, then a GPT model extracts case titles, summaries, sentences, and structured crime features with no manual work.
* **Few-Shot Structured Extraction**: The extraction prompt includes three worked examples, so the model returns the same structure for every judgment and the output can be written straight into the database.
* **Similar Case Recommendation**: For each case, the API computes cosine similarity over its crime features and returns the three most similar judgments of the same crime type.
* **Four Crime Types, Eight Features Each**: Theft, homicide, robbery, and driving offences, each described by eight binary features (for example, *weapon used*, *prior record*, *hit and run*).
* **Citizen Judge Practice**: Users record their own view of a case's features and sentence, then compare it with the crowd and with the real verdict once enough responses exist.
* **Crime Feature Trends**: Aggregates feature totals per crime type to power charts in the app.
* **JWT Authentication & Email Verification**: Custom JWT authentication keyed on email, with 8-character verification codes sent by SMTP for registration and password reset.

## Tech Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Backend API** | Django · Django REST Framework | RESTful API, serializers, request handling |
| **Authentication** | djangorestframework-simplejwt | JWT access tokens with a custom email-based authenticator |
| **Recommendation** | pandas · NumPy · scikit-learn | Feature matrix construction and cosine similarity |
| **Database** | MySQL on AWS RDS | Judgments, crime features, users, comments |
| **Scraping** | Scrapy · ScrapeOps | Scheduled judgment collection and job monitoring |
| **AI Extraction** | GPT (few-shot prompting) | Titles, summaries, sentences, and crime features |
| **Deployment** | AWS | Hosting for the API and database |
| **Client** | React Native (TypeScript) | Mobile app ([JusticeX-frontend](https://github.com/ylin1202/JusticeX-frontend)) |

## System Architecture

```
+---------------------------+        +----------------------------------+
|  Judicial Yuan Judgment   |        |      ScrapeOps Scheduler         |
|         Website           |        |  (job scheduling & monitoring)   |
+-------------+-------------+        +----------------+-----------------+
              |                                       |
              +-------------------+-------------------+
                                  v
                  +-------------------------------+
                  |        Scrapy Spider          |
                  |  (fetch new judgments)        |
                  +---------------+---------------+
                                  |
                                  v
                  +-------------------------------+
                  |   GPT Extraction (few-shot)   |
                  | • title / summary / sentence  |
                  | • crime features              |
                  +---------------+---------------+
                                  |
                                  v
+----------------------------------------------------------------------+
|                         MySQL (AWS RDS)                              |
|----------------------------------------------------------------------|
| • verdict, crime                                                     |
| • theft_feature / homicide_feature / robbery_feature / driving_feature|
| • account, comment, comment_<crime>, reply, like, saved, quiz        |
+-----------------------------------+----------------------------------+
                                    |
                                    v
                  +-------------------------------+
                  |   Django REST API (this repo) |
                  |-------------------------------|
                  | • /api/auth      JWT + email  |
                  | • /api/verdict   cases, recs  |
                  | • /api/comment   user views   |
                  | • /api/account   profile      |
                  +---------------+---------------+
                                  |
                          JWT Bearer / JSON
                                  |
                                  v
                  +-------------------------------+
                  |  React Native Mobile Client   |
                  +-------------------------------+
```

## Data Flow: Similar Case Recommendation

```
          GET /api/verdict/get_verdict/?verdict_id=<id>
                              │
                              ▼
          Look up the case's crime type (theft / homicide / robbery / driving)
                              │
                              ▼
          Load that crime type's feature table,
          excluding cases the user has already reviewed
                              │
                              ▼
          Build a feature matrix (pandas DataFrame)
                              │
                              ▼
          scikit-learn cosine_similarity → rank by similarity
                              │
                              ▼
          Drop the case itself, keep the top 3
                              │
                              ▼
          Return the case details + 3 recommended cases (in ranked order)
```

Cases a user has already reviewed are excluded from recommendations, so each suggestion is a new case to practise on. Those cases can still be opened directly through search.

## Core Engineering Highlights

### Reliable AI Extraction with Few-Shot Prompting
The first version of the extraction prompt returned results in inconsistent formats, which broke the automated flow into the database. The prompt was redesigned with three worked examples of judgments and their correctly extracted fields, giving the model a clear standard to follow. After re-running the extraction, the output followed a consistent structure and could be stored and compared automatically.

### Feature-Based Recommendation
Each crime type has its own feature table, so similarity is only computed between comparable cases. Features are binary, which keeps the vectors simple and makes the recommendations easy to explain to non-lawyers: two cases are similar because they share the same circumstances.

### Crowd vs. Verdict Comparison
When a user comments on a case, they also record which features they believe apply and the sentence they would give. Once a case has more than five responses, the API returns the aggregated feature counts and the distribution of proposed sentences, so users can see how their judgement compares with others and with the court's decision.

## Repository Structure

```
├── api/
│   ├── auth/            # Registration, login, email verification codes
│   ├── verdict/         # Case listing, search, recommendations, trends
│   ├── comment/         # User reviews, replies, likes, crowd statistics
│   ├── account/         # Profile, saved cases, quizzes, notifications
│   ├── models.py        # Database models (MySQL)
│   └── ...
├── core/
│   ├── settings.py      # Django settings
│   └── urls.py          # Root URL routing
├── utils/
│   ├── token.py         # Custom JWT authentication (email as user ID)
│   ├── handlers.py      # Custom exception handler
│   └── response_helpers.py
├── manage.py
├── Pipfile
└── Pipfile.lock
```

## Quickstart

**Install dependencies** (Python 3.11)
```
pipenv install
pipenv install djangorestframework-simplejwt
```

**Configure the database and email settings** in `core/settings.py` (or environment variables), then start the server:
```
pipenv run python manage.py runserver
```
