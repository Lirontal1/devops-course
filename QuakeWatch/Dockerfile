# Use Python 3.11 slim image as base
FROM python:3.11-slim

# Set working directory in the container
WORKDIR /app

# Install system dependencies required for matplotlib
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements file
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .
COPY dashboard.py .
COPY utils.py .

# Copy templates and static directories
COPY templates/ templates/
COPY static/ static/

# Create logs directory
RUN mkdir -p logs

# Expose the Flask port
EXPOSE 5000

# Set environment variable to disable buffering for real-time logging
ENV PYTHONUNBUFFERED=1

# Run the application
CMD ["python", "app.py"]
