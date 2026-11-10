# Voice Assistant

A browser-based voice assistant. Speak or type a question, get a concise answer from OpenAI's GPT model, and hear the reply read aloud. The backend is a Flask server; the frontend is plain HTML, CSS, and JavaScript with jQuery and Bootstrap.

## Features

- **Voice input:** record a question with the microphone button; the audio is sent to the server for transcription.
- **Text input:** type a message and press Enter instead.
- **GPT responses:** replies are generated with OpenAI's `gpt-5-nano` model and kept short (2–3 sentences).
- **Spoken replies:** every answer is synthesized to speech and plays automatically, with a replay button on each message.
- **Voice selection:** choose from several English, French, and Italian voices.
- **Light and dark themes:** toggle the interface theme from the page.

## How It Works

```
Browser ──(recorded audio)──▶ /speech-to-text ──▶ transcript
   │
   └──(text message + voice)──▶ /process-message
                                    ├─▶ OpenAI Chat Completions → reply text
                                    └─▶ Text-to-speech → base64 WAV audio
   ◀────────── JSON: { openaiResponseText, openaiResponseSpeech } ──────────┘
```

1. The browser records audio with the `MediaRecorder` API and posts the raw bytes to `/speech-to-text`.
2. The server transcribes the audio and returns the text.
3. The text is sent to `/process-message` along with the selected voice.
4. The server requests a reply from OpenAI, converts it to speech, and returns both the text and the audio (base64-encoded).
5. The browser displays the reply and plays the audio.

## Project Structure

```
.
├── server.py            # Flask routes: page, speech-to-text, and message processing
├── worker.py            # Integrations: speech-to-text, text-to-speech, and OpenAI calls
├── requirements.txt     # Python dependencies
├── templates/
│   └── index.html       # Single-page UI
├── static/
│   ├── script.js        # Recording, API calls, and chat rendering
│   └── style.css        # Layout and theme styles
├── models/              # Optional: local IBM Watson speech containers (see below)
│   ├── stt/             # Speech-to-text container build
│   └── tts/             # Text-to-speech container build
├── certs/rootCA.crt     # Root certificate used by the Docker image (see Docker notes)
├── Dockerfile
└── LICENSE
```

## Prerequisites

- Python 3.10 or later
- An OpenAI API key
- A modern browser with microphone access (Chrome, Edge, Firefox, or Safari)
- Access to the speech services described under [Speech Services](#speech-services)

## Getting Started

1. **Clone the repository**

   ```bash
   git clone https://github.com/AlokHosamaneJP/voice-chatapp-openai.git
   cd voice-chatapp-openai
   ```

2. **Create a virtual environment and install dependencies**

   ```bash
   python3 -m venv venv
   source venv/bin/activate        # Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. **Set your OpenAI API key**

   ```bash
   export OPENAI_API_KEY="your-api-key"        # Windows (PowerShell): $env:OPENAI_API_KEY="your-api-key"
   ```

4. **Run the server**

   ```bash
   python server.py
   ```

5. **Open the app** at [http://localhost:8000](http://localhost:8000).

Browsers only allow microphone access on `localhost` or over HTTPS, so use `localhost` when testing locally.

## Usage

- Click the **microphone** button to start recording, then click it again to stop. The transcribed question is sent automatically.
- Type in the text box and press **Enter** to send a typed message.
- Use the **Choose a Voice** menu to pick the voice for spoken replies.
- Click the **volume** icon beside any message to replay its audio.
- Use the **Toggle light/dark mode** switch to change the theme.

## API Reference

| Method | Endpoint | Body | Response |
| --- | --- | --- | --- |
| `GET` | `/` | — | The web interface |
| `POST` | `/speech-to-text` | Raw audio bytes | `{ "text": "transcribed speech" }` |
| `POST` | `/process-message` | `{ "userMessage": "...", "voice": "en-US_MichaelV3Voice" }` | `{ "openaiResponseText": "...", "openaiResponseSpeech": "<base64 WAV>" }` |

`voice` may be `"default"` or an empty string to use the default voice.

## Speech Services

Speech-to-text and text-to-speech are implemented in `worker.py`, which calls IBM Watson speech HTTP endpoints. The default configuration uses hosted endpoints that may not be publicly available, so you may need one of the following:

- **Local Watson containers:** build and run the speech containers in `models/stt` and `models/tts`, then point the request URLs in `worker.py` to `http://localhost:1080` (STT) and `http://localhost:1081` (TTS). Each folder has its own README with build and run instructions. This requires an IBM entitlement key.
- **Alternative providers:** replace the functions in `worker.py` with another speech provider. The rest of the app only depends on the function signatures and the JSON responses listed above.

## Docker

```bash
docker build -t voice-chatapp .
docker run --rm -p 8000:8000 -e OPENAI_API_KEY="your-api-key" voice-chatapp
```

> **Note:** The `Dockerfile` installs a local root certificate and sets a placeholder `OPENAI_API_KEY` for a restricted network environment. Remove those steps before deploying elsewhere, and supply your real key at runtime as shown above. The container serves on port 8000, so the `-p 8000:8000` mapping is required.

## Configuration

| Setting | Where | Default |
| --- | --- | --- |
| `OPENAI_API_KEY` | Environment variable | — (required) |
| OpenAI model | `worker.py` (`model=`) | `gpt-5-nano` |
| Reply length limit | `worker.py` (`max_completion_tokens`, system prompt) | 1000 tokens; 2–3 sentences |
| Server port | `server.py` (`app.run(port=...)`) | `8000` |

## Security Notes

- Never commit your API key. The `.gitignore` already excludes `.env` files, so you can store the key there if you load it with a library such as `python-dotenv`.
- `server.py` runs Flask in its default development mode and binds to all interfaces (`0.0.0.0`). Use a production WSGI server and restrict access before exposing the app publicly.
- Recorded audio and transcripts are sent to the configured speech and OpenAI services. Review those providers' data policies before using the app with sensitive information.

## Troubleshooting

- **Microphone does not respond:** confirm the page is served from `localhost` or HTTPS and that the browser has microphone permission.
- **No transcript returned:** check the terminal for errors from the speech endpoint; the hosted endpoints in `worker.py` may be unavailable.
- **OpenAI errors:** confirm `OPENAI_API_KEY` is set in the same shell that runs `server.py`.

## License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.
