FROM python:3.8-slim

WORKDIR /app

COPY ./requirements.txt /app/requirements.txt

RUN apt-get update \
    && apt-get install gcc -y \
    && apt-get clean

RUN pip install -r /app/requirements.txt \
    && rm -rf /root/.cache/pip

COPY . /app/

# Force cache invalidation - build timestamp  
RUN echo "Build timestamp: $(date)" > /app/build-info.txt

# Start the FastAPI application with uvicorn
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]