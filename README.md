# Realtime API - WebRTC
In this titirual we will create a talking agent that can litsen to what you are saying, handle interruptions and respond.
We will use [OpenAI RealTime API](https://platform.openai.com/docs/guides/realtime-webrtc)

## You Tube Video
What the tutorial on YouTube here


## Get started
We will use Flask to build a quick server that will serve a HTML page, so get started with a virtual environment

```ssh
virtualenv realenv
source realenv/bin/activate
```

Then install required packages
```ssh
pip install Flask httpx
```

That is it, then create Flask folderr structure 
|
|______ app.py
|
|______ templates
    |_____ index.html


## Flask APP
The flask app simply needs to get a `EPHEMERAL_KEY` from OpenAI using your API_KEY. This is a key that you can saely use on your client side without exposing your real API key.
The view function will look like so: 

```python
@app.route("/session", methods=["GET"])
def session_endpoint():
    openai_api_key = os.environ.get("OPENAI_API_KEY")
    if not openai_api_key:
        return jsonify({"error": "OPENAI_API_KEY not set"}), 500

    # Make a synchronous POST request to the OpenAI realtime sessions endpoint
    with httpx.Client() as client:
        r = client.post(
            "https://api.openai.com/v1/realtime/sessions",
            headers={
                "Authorization": f"Bearer {openai_api_key}",
                "Content-Type": "application/json",
            },
            json={
                "model": "gpt-4o-realtime-preview-2024-12-17",
                "voice": "verse",
            },
        )
        data = r.json()
        print(data)
        return jsonify(data)
```

The whole Flask code must look like so: 



```python
import os
import httpx
from flask import Flask, render_template, jsonify

# Initialize the Flask application.
app = Flask("voice_app")

# Serve the HTML page at the root route.
@app.route("/")
def index():
    try:
        return render_template("index.html")
    except Exception as e:
        return "index.html not found", 404

# The /session endpoint
@app.route("/session", methods=["GET"])
def session_endpoint():
    openai_api_key = os.environ.get("OPENAI_API_KEY")
    if not openai_api_key:
        return jsonify({"error": "OPENAI_API_KEY not set"}), 500

    # Make a synchronous POST request to the OpenAI realtime sessions endpoint
    with httpx.Client() as client:
        r = client.post(
            "https://api.openai.com/v1/realtime/sessions",
            headers={
                "Authorization": f"Bearer {openai_api_key}",
                "Content-Type": "application/json",
            },
            json={
                "model": "gpt-4o-realtime-preview-2024-12-17",
                "voice": "verse",
            },
        )
        data = r.json()
        print(data)
        return jsonify(data)

if __name__ == "__main__":
    # Run the Flask app on port 8116
    app.run(host="0.0.0.0", port=8116, debug=True)

```

## Index HTML File

We want our HTML to have some visualisations, so help the user sort of feel engaged with the app, so w will draw voice vosualisations and add them to the code, the final code looks like:


```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Real-Time Voice App with WebRTC</title>
  <!-- Bootstrap CSS -->
  <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
  <style>
    /* Custom styles for the visualizers */
    .visualizer {
      width: 100%;
      height: 150px;
      background-color: #f5f5f5;
      border: 1px solid #ddd;
      margin-bottom: 20px;
    }
  </style>
</head>
<body class="p-3">
  <div class="container">
    <h1 class="mb-4">Real-Time Voice App with WebRTC</h1>
    <div class="mb-3">
      <p><strong>Local Input (Blue):</strong></p>
      <canvas id="localVisualizer" class="visualizer"></canvas>
    </div>
    <div class="mb-3">
      <p><strong>Remote Audio Visualization (Red):</strong></p>
      <canvas id="backendVisualizer" class="visualizer"></canvas>
    </div>
  </div>

  <!-- Bootstrap Bundle with Popper -->
  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>

  <script>
    /**
     * Initialize the WebRTC connection with the OpenAI realtime API.
     * This function obtains an ephemeral key from the backend, creates an RTCPeerConnection,
     * sets up local audio capture and visualization, and performs the SDP exchange.
     */
    async function init() {
      try {
        // 1. Get an ephemeral key from your server (ensure your backend provides the /session endpoint)
        const tokenResponse = await fetch("/session");
        const tokenData = await tokenResponse.json();
        const EPHEMERAL_KEY = tokenData.client_secret.value;
        console.log("Ephemeral key received:", EPHEMERAL_KEY);

        // 2. Create a new RTCPeerConnection.
        const pc = new RTCPeerConnection();

        // 3. Set up remote audio playback.
        const audioEl = document.createElement("audio");
        audioEl.autoplay = true;
        document.body.appendChild(audioEl);
        pc.ontrack = e => {
          if (e.streams && e.streams[0]) {
            audioEl.srcObject = e.streams[0];
            // Instead of a placeholder, start the remote visualizer with the received stream.
            startBackendVisualizer(e.streams[0]);
          }
        };

        // 4. Get the microphone stream.
        const micStream = await navigator.mediaDevices.getUserMedia({ audio: true });
        // Add the microphone track to the RTCPeerConnection.
        pc.addTrack(micStream.getTracks()[0]);
        // Start local visualization using the same micStream.
        startLocalVisualizer(micStream);

        // 5. Set up a data channel for events.
        const dc = pc.createDataChannel("oai-events");
        dc.addEventListener("message", (e) => {
          console.log("Data Channel message:", e.data);
        });

        // 6. Create an SDP offer.
        const offer = await pc.createOffer();
        await pc.setLocalDescription(offer);
        console.log("SDP offer created and set as local description.");

        // 7. Send the SDP offer to the OpenAI realtime API.
        const baseUrl = "https://api.openai.com/v1/realtime";
        const model = "gpt-4o-realtime-preview-2024-12-17";
        const sdpResponse = await fetch(`${baseUrl}?model=${model}`, {
          method: "POST",
          body: offer.sdp,
          headers: {
            Authorization: `Bearer ${EPHEMERAL_KEY}`,
            "Content-Type": "application/sdp"
          },
        });

        // 8. Get the SDP answer and set it as the remote description.
        const answer = {
          type: "answer",
          sdp: await sdpResponse.text(),
        };
        await pc.setRemoteDescription(answer);
        console.log("SDP answer received and set as remote description. WebRTC connection established.");
      } catch (error) {
        console.error("Error initializing WebRTC connection:", error);
      }
    }

    /**
     * Start local visualization of the microphone audio (blue waveform).
     * Uses the Web Audio API to draw a continuous waveform on the local visualizer canvas.
     * @param {MediaStream} stream - The microphone audio stream.
     */
    function startLocalVisualizer(stream) {
      const localCanvas = document.getElementById('localVisualizer');
      const localCtx = localCanvas.getContext('2d');
      localCanvas.width = localCanvas.offsetWidth;
      localCanvas.height = localCanvas.offsetHeight;

      const audioContext = new (window.AudioContext || window.webkitAudioContext)();
      const analyser = audioContext.createAnalyser();
      analyser.fftSize = 2048;
      const bufferLength = analyser.frequencyBinCount;
      const dataArray = new Uint8Array(bufferLength);
      const source = audioContext.createMediaStreamSource(stream);
      source.connect(analyser);

      function draw() {
        requestAnimationFrame(draw);
        analyser.getByteTimeDomainData(dataArray);
        localCtx.fillStyle = '#f5f5f5';
        localCtx.fillRect(0, 0, localCanvas.width, localCanvas.height);
        localCtx.lineWidth = 2;
        localCtx.strokeStyle = '#007bff';
        localCtx.beginPath();
        const sliceWidth = localCanvas.width / dataArray.length;
        let x = 0;
        for (let i = 0; i < dataArray.length; i++) {
          const v = dataArray[i] / 128.0;
          const y = v * localCanvas.height / 2;
          if (i === 0) {
            localCtx.moveTo(x, y);
          } else {
            localCtx.lineTo(x, y);
          }
          x += sliceWidth;
        }
        localCtx.lineTo(localCanvas.width, localCanvas.height / 2);
        localCtx.stroke();
      }
      draw();
    }

    /**
     * Visualizes the remote (backend) audio stream on the backend visualizer canvas (red waveform).
     * Uses the Web Audio API to continuously draw the waveform from the remote audio stream.
     * @param {MediaStream} remoteStream - The remote audio stream from the RTCPeerConnection.
     */
    function startBackendVisualizer(remoteStream) {
      const backendCanvas = document.getElementById('backendVisualizer');
      const backendCtx = backendCanvas.getContext('2d');
      backendCanvas.width = backendCanvas.offsetWidth;
      backendCanvas.height = backendCanvas.offsetHeight;

      const audioContext = new (window.AudioContext || window.webkitAudioContext)();
      const analyser = audioContext.createAnalyser();
      analyser.fftSize = 2048;
      const bufferLength = analyser.frequencyBinCount;
      const dataArray = new Uint8Array(bufferLength);
      const source = audioContext.createMediaStreamSource(remoteStream);
      source.connect(analyser);

      function draw() {
        requestAnimationFrame(draw);
        analyser.getByteTimeDomainData(dataArray);
        backendCtx.fillStyle = '#f5f5f5';
        backendCtx.fillRect(0, 0, backendCanvas.width, backendCanvas.height);
        backendCtx.lineWidth = 2;
        backendCtx.strokeStyle = '#ff0000';
        backendCtx.beginPath();
        const sliceWidth = backendCanvas.width / dataArray.length;
        let x = 0;
        for (let i = 0; i < dataArray.length; i++) {
          const v = dataArray[i] / 128.0;
          const y = v * backendCanvas.height / 2;
          if (i === 0) {
            backendCtx.moveTo(x, y);
          } else {
            backendCtx.lineTo(x, y);
          }
          x += sliceWidth;
        }
        backendCtx.lineTo(backendCanvas.width, backendCanvas.height / 2);
        backendCtx.stroke();
      }
      draw();
    }

    // Call init() on page load.
    init().catch(err => console.error("Error initializing connection:", err));
  </script>
</body>
</html>

```

## Running the APP
The app needs permission to use a MIC. For security web browsers do not allow this over insecure connections, which is what you will have when running a dev server, to get around that --- do this:

```sh
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --unsafely-treat-insecure-origin-as-secure="http://102.218.215.158:8116" --user-data-dir=/tmp/chrome_dev
```

This will open an insecure browser that will allow for this connection. Replace the IP with your IP address or use locahost. 
