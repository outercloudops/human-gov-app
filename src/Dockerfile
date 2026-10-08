# Use Python as a base image
FROM python:3.8-slim-buster

# Set working directory
WORKDIR /app

# Copy requirements and install dependencies
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Copy the Flask application
COPY . /app

# Start Gunicorn
CMD ["gunicorn", "--workers", "1", "--bind", "0.0.0.0:8000", "humangov:app"]