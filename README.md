# RADAR: Government Contract Scraping & ML Platform 📊🤖

> **Note:** This is a showcase repository demonstrating my architectural refactoring and Machine Learning integration work at the **Illinois Hispanic Chamber of Commerce (IHCC)**. The full source code is proprietary and private.

## Overview
RADAR is an intelligence platform built to monitor, scrape, and classify government contracts. It operates as a complex asynchronous pipeline, consuming external APIs, extracting textual features, and categorizing business opportunities.

My core contribution was dismantling a monolithic `main.py` structure, implementing an automated testing strategy, and integrating scikit-learn classification models that drastically reduced the dependency on LLM APIs, saving the business ~50% in operational AI costs.

## Technical Stack
- **Backend Framework**: FastAPI, Starlette
- **Machine Learning**: scikit-learn, spaCy (NLP), TF-IDF Feature Extraction
- **Asynchronous Workers**: Celery, Redis
- **Testing**: Pytest, conftest
- **Web Scraping**: Playwright
- **Database**: Qdrant Vector Cloud

## Key Engineering Achievements

### 1. Refactoring the FastAPI Monolith
**Problem:** The original `main.py` contained over 2,500 lines of heavily coupled routing, business, and data access logic, making the addition of new endpoints treacherous.
**Solution:**
- Transitioned to an `APIRouter` pattern. Split the monolith into 8 distinct modular routers (e.g., `auth`, `scrapers`, `hybrid jobs`).
- Decoupled business logic into a dedicated `services/` layer, allowing routes to function purely as HTTP controllers.

### 2. Machine Learning Pipeline Integration (Cost Reduction)
**Problem:** The platform was previously sending raw contract text to OpenAI's GPT models simply to categorize the business vertical and NAICS codes, incurring significant API latency and costs.
**Solution:**
- Integrated traditional ML classification models (`.pkl` fast classifiers) trained on historical data.
- **Results:** Achieved **98.1% accuracy** on business category prediction and **74% accuracy** on NAICS code prediction.
- This hybrid (fast model first, LLM later) approach **cut GPT API costs by ~50%**.

### 3. Comprehensive Test Suite Engineering
**Problem:** The project lacked automated safety nets, relying entirely on manual testing.
**Solution:**
- Built the entire automated test suite from scratch using `pytest`.
- Created robust fixtures in `conftest.py` for database mocking and authentication bypassing.
- Engineered hybrid job tests, NLP evaluation scripts, and resolved complex dependency clashes (e.g., `pywin32` and Starlette versioning).

### 4. Deprecating Technical Debt
- Identified and removed obsolete ETL pipelines (`etl_pipeline.py`, `csv_jobs.py`).
- Relocated 15+ "floating" standalone bash/python scripts into a structured legacy module.
- Added a Qdrant Singleton pattern to avoid connection pool exhaustion under heavy Celery loads.

---
*Developed by Mario Adrian Ornelas Cortes during tenure at IHCC.*
