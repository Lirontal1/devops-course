![Screenshot of QuakeWatch](static/experts-logo.svg)

# QuakeWatch

**QuakeWatch** is a Flask-based web application designed to display real-time and historical earthquake data. It visualizes earthquake statistics with interactive graphs and provides detailed information sourced from the USGS Earthquake API. Built using an object‑oriented design and modular structure, QuakeWatch separates templates, utility functions, and route definitions, making it both scalable and maintainable. The application is also containerized with Docker for easy deployment.

## Features

- **Real-Time & Historical Data:** Fetches earthquake data from the USGS API.
- **Interactive Graphs:** Displays earthquake counts over various time periods (e.g., last 30 days, 5-year view) using Matplotlib.
- **Top Earthquake Events:** Shows the top 5 worldwide earthquakes (last 30 days) by magnitude.
- **Recent Earthquake Details:** Highlights the most recent earthquake event.
- **RESTful Endpoints:** Provides endpoints for health checks, status, connectivity tests, and raw data.
- **Clean UI:** Built with Bootstrap 5, featuring a professional navigation bar with a logo.
- **Dockerized:** Easily containerized for streamlined deployment.

## Project Structure

```
QuakeWatch/
├── app.py                  # Application factory and entry point
├── dashboard.py            # Blueprint & route definitions using OOP style
├── utils.py                # Helper functions and custom Jinja2 filters
├── requirements.txt        # Python dependencies
├── static/
│   └── experts-logo.svg    # Logo file used in the UI
└── templates/              # Jinja2 HTML templates
    ├── base.html           # Base template with common layout and navigation
    ├── main_page.html      # Home page content
    └── graph_dashboard.html# Dashboard view with graphs and earthquake details
```

## Installation

### Locally

1. **Clone the Repository:**

   ```bash
   git clone https://github.com/yourusername/QuakeWatch.git
   cd QuakeWatch
   ```

2. **Set Up a Virtual Environment (optional but recommended):**

   ```bash
   python -m venv venv
   source venv/bin/activate   # On Windows: venv\Scripts\activate
   ```

3. **Install Dependencies:**

   ```bash
   pip install -r requirements.txt
   ```

## Running the Application Locally

1. **Start the Flask Application:**

   ```bash
   python app.py
   ```

2. **Access the Application:**

   Open your browser and visit [http://127.0.0.1:5000](http://127.0.0.1:5000) to view the dashboard.

## Running with Docker

### Prerequisites

- Docker installed on your system ([Get Docker](https://docs.docker.com/get-docker/))
- Docker Compose installed (included with Docker Desktop)

### Option 1: Using Docker Compose (Recommended)

1. **Build and Start the Container:**

   ```bash
   docker-compose up -d
   ```

   This will build the image if it doesn't exist and start the container in detached mode.

2. **Access the Application:**

   Open your browser and visit [http://localhost:5000](http://localhost:5000)

3. **View Logs:**

   ```bash
   docker-compose logs -f
   ```

4. **Stop the Container:**

   ```bash
   docker-compose down
   ```

### Option 2: Using Docker Commands Directly

1. **Build the Docker Image:**

   ```bash
   docker build -t quakewatch:latest .
   ```

   To tag the image for Docker Hub (replace `yourusername` with your Docker Hub username):

   ```bash
   docker build -t yourusername/quakewatch:latest .
   ```

2. **Run the Container:**

   ```bash
   docker run -d -p 5000:5000 --name quakewatch-app quakewatch:latest
   ```

   With volume mount for logs:

   ```bash
   docker run -d -p 5000:5000 -v $(pwd)/logs:/app/logs --name quakewatch-app quakewatch:latest
   ```

3. **Access the Application:**

   Open your browser and visit [http://localhost:5000](http://localhost:5000)

4. **View Logs:**

   ```bash
   docker logs -f quakewatch-app
   ```

5. **Stop and Remove the Container:**

   ```bash
   docker stop quakewatch-app
   docker rm quakewatch-app
   ```

### Pushing to Docker Hub

1. **Login to Docker Hub:**

   ```bash
   docker login
   ```

2. **Tag the Image (if not already tagged):**

   ```bash
   docker tag quakewatch:latest yourusername/quakewatch:latest
   ```

3. **Push to Docker Hub:**

   ```bash
   docker push yourusername/quakewatch:latest
   ```

### Pulling from Docker Hub

Once the image is published to Docker Hub, others can pull and run it:

```bash
docker pull yourusername/quakewatch:latest
docker run -d -p 5000:5000 yourusername/quakewatch:latest
```


## Custom Jinja2 Filter

The project includes a custom filter `timestamp_to_str` that converts epoch timestamps to human-readable strings. This filter is registered during application initialization and is used in the templates to format earthquake event times.

## Known Issues

- **SSL Warning:** You might see a warning regarding LibreSSL when using urllib3. This is informational and does not affect the functionality of the application.
- **Matplotlib Backend:** The application forces Matplotlib to use the `Agg` backend for headless rendering. Ensure this setting is applied before any Matplotlib imports to avoid GUI-related errors.
