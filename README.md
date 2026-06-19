Ironman AI Coach — Strava + n8n + Postgres + Ollama + Slack

An automated endurance-coaching workflow that listens for new Strava activities, stores structured workout data in Postgres, compares the new workout against recent training history, asks a local Ollama model for a short coach read, saves the recommendation, and posts the result to Slack.

This project is built to show a practical, low-cost example of agentic AI: the system reacts to a real-world event, gathers context from a database, uses an LLM to make a recommendation, and sends the answer where the athlete already works.

Important: This is coaching guidance, not medical advice. Use your own judgment and adjust based on how you feel.

What this workflow does

When a new Strava activity is created, the workflow:

Receives the activity from the Strava Trigger.
Cleans the raw Strava payload into normalized workout fields.
Upserts the workout into a workouts table in Postgres.
Pulls the last 28 days of training history.
Pulls the most recent similar workouts by sport.
Pulls tomorrow's planned training session from a weekly template table.
Builds a compact context object for the AI coach.
Sends that context to a local Ollama chat model.
Formats the AI response for Slack.
Saves the coach recommendation to Postgres.
Posts the coach check-in to a Slack channel.
Architecture
Strava
  |
  v
n8n Strava Trigger
  |
  v
Clean Workout Data
  |
  v
Postgres - Upsert Workout
  |
  v
Postgres - Last 28 Days
  |
  v
Postgres - Similar Workouts
  |
  v
Postgres - Tomorrow Plan
  |
  v
Prepare AI Coach Context
  |
  v
Ollama Local LLM
  |
  v
Format Slack Message
  |
  v
Postgres - Save Recommendation
  |
  v
Slack
Tech stack
n8n — workflow automation
Strava API — workout trigger/data source
Postgres — workout history and recommendations database
Ollama — local LLM runtime
Qwen 2.5 Coder 3B — local model used by the workflow
Slack — coach message delivery
Prerequisites

You need:

A machine or VM that can run Docker
Docker and Docker Compose
A Strava account
A Slack workspace where you can create an app or bot
Basic command-line comfort
A GitHub account if you plan to publish this repo

Recommended VM size:

2+ CPU cores
4 GB RAM minimum
8 GB RAM recommended if running Ollama locally
20+ GB disk
Step 1 — Clone the repo
git clone https://github.com/YOUR_USERNAME/ironman-ai-coach.git
cd ironman-ai-coach
Step 2 — Create the project folders
mkdir -p ./n8n-data
mkdir -p ./postgres-data
mkdir -p ./ollama-data
Step 3 — Create a Docker Compose file

Create docker-compose.yml:

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL}
      - WEBHOOK_URL=${WEBHOOK_URL}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - ./n8n-data:/home/node/.n8n
    depends_on:
      - postgres
      - ollama

  postgres:
    image: postgres:16
    container_name: postgres
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - ./postgres-data:/var/lib/postgresql/data

  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ./ollama-data:/root/.ollama
Step 4 — Create your .env file

Create .env:

N8N_HOST=localhost
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678/

N8N_ENCRYPTION_KEY=change-this-to-a-long-random-string

POSTGRES_DB=ironman_coach
POSTGRES_USER=ironman_user
POSTGRES_PASSWORD=change-this-password

Generate a strong encryption key:

openssl rand -hex 32

Use that value for N8N_ENCRYPTION_KEY.

Do not commit your .env file to GitHub.

Step 5 — Start the stack
docker compose up -d

Check that everything started:

docker ps

Open n8n:

http://localhost:5678
Step 6 — Pull the Ollama model

The workflow uses qwen2.5-coder:3b.

docker exec -it ollama ollama pull qwen2.5-coder:3b

Test it:

docker exec -it ollama ollama run qwen2.5-coder:3b
Step 7 — Create the Postgres tables

Connect to Postgres:

docker exec -it postgres psql -U ironman_user -d ironman_coach

Run this SQL:

CREATE TABLE IF NOT EXISTS workouts (
  id BIGSERIAL PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'strava',
  source_activity_id TEXT NOT NULL UNIQUE,
  start_time TIMESTAMP,
  sport TEXT,
  sub_sport TEXT,
  workout_name TEXT,
  distance_miles NUMERIC,
  duration_minutes NUMERIC,
  pace_per_mile TEXT,
  avg_heart_rate NUMERIC,
  max_heart_rate NUMERIC,
  avg_power NUMERIC,
  calories NUMERIC,
  elevation_gain_feet NUMERIC,
  training_load NUMERIC,
  effort_label TEXT DEFAULT 'unknown',
  raw_json JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_workouts_start_time
ON workouts (start_time DESC);

CREATE INDEX IF NOT EXISTS idx_workouts_sport
ON workouts (sport);

CREATE TABLE IF NOT EXISTS coach_recommendations (
  id BIGSERIAL PRIMARY KEY,
  source_activity_id TEXT REFERENCES workouts(source_activity_id),
  recommendation_date TIMESTAMP DEFAULT NOW(),
  summary TEXT,
  trend_analysis TEXT,
  next_workout_recommendation TEXT,
  risk_flags TEXT,
  slack_message TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS weekly_training_template (
  id BIGSERIAL PRIMARY KEY,
  day_of_week TEXT UNIQUE NOT NULL,
  planned_sport TEXT NOT NULL,
  planned_focus TEXT NOT NULL,
  description TEXT
);

Insert a starter weekly training plan:

INSERT INTO weekly_training_template
(day_of_week, planned_sport, planned_focus, description)
VALUES
('Monday', 'run', 'aerobic', 'Easy aerobic run. Keep it controlled.'),
('Tuesday', 'bike', 'endurance', 'Steady endurance bike. Mostly easy effort.'),
('Wednesday', 'swim', 'technique', 'Technique-focused swim with relaxed pacing.'),
('Thursday', 'run', 'long run', 'Long run day. Keep effort sustainable.'),
('Friday', 'swim', 'easy', 'Easy swim or recovery-focused session.'),
('Saturday', 'bike', 'long bike', 'Long endurance bike. Prioritize fueling and steady effort.'),
('Sunday', 'recovery', 'rest', 'Recovery, rest, mobility, or optional easy session.')
ON CONFLICT (day_of_week)
DO UPDATE SET
  planned_sport = EXCLUDED.planned_sport,
  planned_focus = EXCLUDED.planned_focus,
  description = EXCLUDED.description;

Exit Postgres:

\q
Step 8 — Create a Slack app

Go to the Slack API dashboard and create a new app for your workspace.

Add the bot permissions your workflow needs:

chat:write
channels:read
groups:read

Install the app to your workspace.

Copy the bot token. It usually starts with:

xoxb-

You will paste this into n8n as a Slack credential. Do not put this token in GitHub.

Step 9 — Create a Strava API app

Go to your Strava API settings and create an application.

You will need:

Client ID
Client Secret
OAuth redirect/callback URL

When using n8n locally, your callback URL may look like:

http://localhost:5678/rest/oauth2-credential/callback

When using a public domain or Cloudflare Tunnel, it may look like:

https://your-domain.example.com/rest/oauth2-credential/callback

The exact callback URL must match what n8n shows in the Strava credential setup screen.

Do not commit your Strava client secret to GitHub.

Step 10 — Import the n8n workflow

In n8n:

Go to Workflows.
Select Import from File.
Upload the exported workflow JSON.
Open the imported workflow.
Reconnect each credential:
Strava OAuth2 credential
Postgres credential
Slack API credential
Ollama API credential

The workflow JSON should not contain your real credentials. n8n exports usually reference credential names and IDs, but you still need to create credentials inside your own n8n instance.

Step 11 — Configure the Ollama credential in n8n

If n8n and Ollama are running in the same Docker Compose network, use:

http://ollama:11434

If n8n is running outside Docker and Ollama is exposed locally, use:

http://localhost:11434

Model name:

qwen2.5-coder:3b
Step 12 — Configure the Postgres credential in n8n

If n8n and Postgres are running in the same Docker Compose network:

Host: postgres
Port: 5432
Database: ironman_coach
User: ironman_user
Password: your password from .env
Step 13 — Configure Slack output

Open the Slack - Send Coach Message node.

Set the channel where you want the AI coach messages to go.

Test the node with a simple message before activating the whole workflow.

Step 14 — Test the workflow

Before activating it fully:

Confirm Postgres is reachable from n8n.
Confirm Ollama responds from n8n.
Confirm Slack can send a test message.
Confirm the Strava credential connects.
Run a manual workflow test if you have sample Strava activity data.

Once everything works, activate the workflow.

The workflow triggers when a new Strava activity is created.

Example Slack output
🏊‍♂️🚴‍♂️🏃 Ironman Coach Check-In

Workout Summary
You completed a steady run with useful aerobic work.

Trend
Compared to recent similar workouts, this gives the coach more data to understand your running load.

Coach Read
Moderate. Nothing here screams panic, but watch cumulative fatigue.

Tomorrow's Recommendation
Follow the planned bike endurance session, but keep it mostly easy if your legs feel heavy.

One Thing to Watch
Pay attention to heart rate drift during the next session.
