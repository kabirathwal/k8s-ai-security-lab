# Image for chat-api.
# Build: docker build -t chat-api:0.1.0 app/

# Base image: slim Debian with Python, pinned to an exact version so builds are repeatable.
FROM python:3.13.16-slim

# Don't write .pyc files, and print logs straight away (so `kubectl logs` shows them immediately).
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# All following commands run in /app.
WORKDIR /app

# Copy requirements first: Docker caches this layer, so code changes don't reinstall packages.
COPY requirements.txt .
# --no-cache-dir: don't keep pip's download cache in the image (smaller image).
RUN pip install --no-cache-dir -r requirements.txt

COPY main.py .

# Create an unprivileged user and run as it, so the app isn't root inside the container.
RUN useradd --uid 10001 --no-create-home --shell /usr/sbin/nologin appuser
USER 10001

# Documents the port the app listens on (the Service/Deployment do the real wiring).
EXPOSE 8000

# Start the web server. 0.0.0.0 = listen on all interfaces, so the pod IP is reachable.
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
