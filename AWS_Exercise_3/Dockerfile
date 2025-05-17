# Use official Python image
FROM python:3.9

# Set the working directory inside the container
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . .

# Install system dependencies
RUN apt-get update && apt-get install -y libpq-dev gcc

# Install the Python dependencies
RUN pip install --upgrade pip && pip install -r requirements.txt

# Set Flask environment variables
ENV FLASK_APP=crudapp.py
ENV FLASK_RUN_HOST=0.0.0.0

# Run database migrations
RUN flask db init && flask db migrate -m 'entries table' && flask db upgrade

# Expose port 80 for the application
EXPOSE 80

# Start Gunicorn with the Flask app (use Gunicorn for production)
CMD ["gunicorn", "--bind", "0.0.0.0:80", "crudapp:app"]
