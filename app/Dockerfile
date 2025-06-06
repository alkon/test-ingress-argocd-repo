# Use a suitable Python base image
FROM python:3.9-slim-buster

# Set working directory inside the container to /app
WORKDIR /app

# Copy only requirements.txt first to leverage Docker layer caching.
# Since the build context is 'app', requirements.txt is directly in '.'
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy your application code.
# main.py is also directly in '.' because the build context is 'app'.
COPY main.py .

# Expose the port your Flask app listens on (assuming 5000 as discussed)
EXPOSE 5000

# Command to run your Flask application using Gunicorn
# This assumes your Flask app instance is named 'app' in main.py
# Make sure 'gunicorn' is in your requirements.txt
#CMD ["gunicorn", "-b", "0.0.0.0:5000", "main:app"]

CMD ["python", "main.py"]