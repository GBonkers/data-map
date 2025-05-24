# DataMap Backend

Backend service for the DataMap application.

## Setup

1. Install dependencies:
```bash
poetry install
```

2. Run the development server:
```bash
poetry run uvicorn main:app --reload
```

## Development

This project uses:
- FastAPI
- Poetry for dependency management
- PostgreSQL database 