FROM python:3.12-slim

WORKDIR /app

# Prevent Python from writing pyc files
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Copy dependency file first for layer caching
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt

# Copy app source code
COPY ./app .

# Run as non-root user
RUN addgroup --system app && adduser --system --ingroup app app
USER app

# Expose FastAPI port
EXPOSE 8000

# Run FastAPI
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]