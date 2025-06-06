# Use a suitable Python base image
FROM python:3.9-slim-buster

# Set working directory inside the container to /app
WORKDIR /app

# Copy only requirements.txt first to leverage Docker layer caching.
# Make sure gunicorn is listed in your requirements.txt
COPY requirements.txt .

# Install Python dependencies, including Gunicorn
RUN pip install --no-cache-dir -r requirements.txt

# Copy your Flask application code
COPY main.py .

# Expose the port your Flask app listens on.
# This should match the port configured in main.py (5000).
EXPOSE 5000

# Command to run your Flask application using Gunicorn.
# This is the stable, production-ready way to serve a Flask app.
# "main:app" refers to the 'app' Flask instance in your 'main.py' file.
CMD ["gunicorn", "-b", "0.0.0.0:5000", "main:app"]

# The previous CMD ["python", "main.py"] uses Flask's development server,
# which is not suitable for production deployments and can cause instability.