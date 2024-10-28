---
title: 'From Headlines to YouTube: Crafting an AI-driven News Video Generator'
categories: [AI, llm]
date: 2024-10-27 12:00:00
tags: [AI, llm]
image: "/assets/img/ai_driven_news_video_generator/poster.png"
---

As content creation becomes more dynamic and competitive, automation tools powered by artificial intelligence (AI) are transforming the way creators, publishers, and news agencies generate media. In this tutorial, we’ll walk through how to build an **AI-driven news video generator** using SwarmZero AI and Livepeer AI. 

This system automates the process of fetching news, narrating it, generating visuals, and uploading engaging video summaries to YouTube. With a seamless pipeline that incorporates news aggregation, audio narration, image generation, and video editing, this project provides a versatile tool for creators looking to produce informative content with minimal manual effort.

![video-news-ai](https://github.com/user-attachments/assets/f51d175a-4b40-4556-aacb-202183e8ff07)


---

## Overview of Our AI-Powered Workflow

The system we’re building is designed to simplify news video creation by integrating AI agents into a cohesive workflow. Below is a breakdown of each component in our pipeline:

### 1. **News Aggregator Agent**
   - Fetches the latest news from external APIs, such as TheNewsAPI, based on specified categories (e.g., business, technology, sports).
   
### 2. **Audio Narration Agent**
   - Converts the news descriptions into speech, adding an engaging human-like voice to each segment.

### 3. **Script Writer Agent**
   - Write a short visual script based on the summarization from the top news stories.

### 4. **Scene Prompt Generator & Image Generator**
   - Creates prompts for generating relevant visuals, then generates these images, providing a dynamic visual representation of the news story.

### 5. **Image-to-Video Generator**
   - Combines images and narration into short video segments using Livepeer AI.

### 6. **Video Editor Agent**
   - Merges multiple video segments into a complete, polished video, suitable for uploading.

### 7. **YouTube Upload Agent**
   - Automates the upload of the generated video to YouTube, completing the content creation process.

![video-news-ai workflow](https://github.com/user-attachments/assets/c2976ada-4672-46f1-a931-3af0036b90e9)

The system also incorporates SwarmZero AI, which allows users to choose their preferred large language model (LLM) from OpenAI (like GPT-4), Anthropic, or other open-source models by simply defining the model name in the configuration. 

---

## Prerequisites

To get started, ensure you have the following:

- Python 3.11
- API Keys for OpenAI or your chosen LLM provider
- Livepeer API Key
- Google API Client for YouTube
- TheNewsAPI Key

### Setting Up the Google API Client
1. Go to the [Google API Console](https://console.cloud.google.com/) and create a new project.
2. Enable the YouTube Data API v3.
3. Set up OAuth 2.0 credentials:
   - Configure the consent screen.
   - Add valid redirect URIs (`http://localhost:8088/` and `http://localhost:8088/flowName=GeneralOAuthFlow`).
4. Download the `client_secret.json` file and place it in the project’s root directory for secure OAuth authentication.

---

## Project Setup

### Step 1: Clone the Repository

Begin by cloning the project repository and navigating to the project directory:

```bash
git clone https://github.com/your-repo/video-news-ai.git
cd video-news-ai
```

### Step 2: Define Environment Variables
Define the backend environment variables in `.env`:
```
MODEL=gpt-4-turbo-preview
OPENAI_API_KEY=
MISTRAL_API_KEY=
ANTHROPIC_API_KEY=
ENVIRONMENT=dev
SWARMZERO_LOG_LEVEL=INFO
SWARMZERO_DATABASE_URL=
PINECONE_API_KEY=
LIVEPEER_API_KEY=
NEWS_API_KEY=
```
Next, set up the frontend environment in `ui/.env`:
```
NEXT_PUBLIC_BACKEND_API=http://127.0.0.1:8000/chat/
```
### Step 3: Run app
Docker simplifies setup and ensures all dependencies are in place. Run the following command to start the application:
`docker-compose up`

The application will be available at `http://localhost:3000`

## Using the Application
The interface is simple and intuitive. Here’s how to get started:
### 1. Input a Prompt

- You can start with something like, `Create a short video from the two latest news`
- Optionally, specify categories like `business, tech, or sports`

### 2. Generate the Video

- Click on the **Generate** button and wait a few minutes while the system processes the request

### 3. Check your Youtube Channel

- Once complete, the video is automatically uploaded to your YouTube channel, where you can view and share it.

## Potential improvement:
1. Implement a cron job for scheduled video publishing.
2. Expand distribution to TikTok and Twitter.
3. Add advanced editing features and human-like TTS.
4. Optimize the prompt to ensure the tool runs with stability.

---
With just a few clicks, you’ve created a fully automated news video that’s ready to engage your audience!

Please checkout the [GitHub](https://github.com/hgky95/video-news-ai) for more details.

Enjoying the project? Don’t forget to star it <a href="https://github.com/hgky95/video-news-ai">Star me here ⭐ !</a>