<template>
  <div class="container">
    <h1>Screen Capture and Streaming</h1>
    <div class="status">
      <p>Status: <span :class="isConnected ? 'connected' : 'disconnected'">{{ isConnected ? "Connected" : "Disconnected" }}</span></p>
      <p v-if="isRecordingActive">Recording Time: {{ formattedTime }}</p>
    </div>
    <video ref="videoRef" autoplay></video>
    <div class="buttons">
      <button @click="startScreenCapture" :disabled="isRecordingActive">Start Capture</button>
      <button @click="pauseRecording" :disabled="!isRecordingActive || isPaused">Pause</button>
      <button @click="resumeRecording" :disabled="!isRecordingActive || !isPaused">Resume</button>
      <button @click="stopScreenCapture" :disabled="!isRecordingActive">Stop Capture</button>
      <button @click="downloadRecording" :disabled="chunks.length === 0">Download Recording</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      websocket: null,
      mediaRecorder: null,
      chunks: [],
      isRecordingActive: false,
      isPaused: false,
      isConnected: false,
      timer: null,
      elapsedTime: 0,
    };
  },
  computed: {
    formattedTime() {
      const minutes = String(Math.floor(this.elapsedTime / 60)).padStart(2, "0");
      const seconds = String(this.elapsedTime % 60).padStart(2, "0");
      return `${minutes}:${seconds}`;
    },
  },
  mounted() {
    this.initializeWebSocket();
  },
  methods: {
    initializeWebSocket() {
      this.websocket = new WebSocket("ws://localhost:3001");

      this.websocket.onopen = () => {
        this.isConnected = true;
        console.log("Connected to server");
      };

      this.websocket.onclose = () => {
        this.isConnected = false;
        console.log("Disconnected from server");
      };
    },
    async startScreenCapture() {
      try {
        const stream = await navigator.mediaDevices.getDisplayMedia({ video: true });
        this.$refs.videoRef.srcObject = stream;

        this.mediaRecorder = new MediaRecorder(stream, { mimeType: "video/webm" });
        this.isRecordingActive = true;
        this.elapsedTime = 0;
        this.timer = setInterval(() => this.elapsedTime++, 1000);

        this.mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) {
            this.chunks.push(event.data);

            if (this.websocket && this.websocket.readyState === WebSocket.OPEN) {
              this.websocket.send(event.data);
            }
          }
        };

        this.mediaRecorder.start(100);
      } catch (error) {
        if (error.name === "NotAllowedError") {
          console.error("Screen capture permission was denied.");
        } else {
          console.error("Error capturing screen:", error);
        }
      }
    },
    pauseRecording() {
      if (this.mediaRecorder && this.mediaRecorder.state === "recording") {
        this.mediaRecorder.pause();
        this.isPaused = true;
      }
    },
    resumeRecording() {
      if (this.mediaRecorder && this.mediaRecorder.state === "paused") {
        this.mediaRecorder.resume();
        this.isPaused = false;
      }
    },
    stopScreenCapture() {
      if (this.mediaRecorder) {
        this.mediaRecorder.stop();
        this.isRecordingActive = false;
        this.isPaused = false;
        clearInterval(this.timer);
      }
      this.$refs.videoRef.srcObject.getTracks().forEach((track) => track.stop());
    },
    downloadRecording() {
      const blob = new Blob(this.chunks, { type: "video/webm" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.style.display = "none";
      a.href = url;
      a.download = "screen_recording.webm";
      document.body.appendChild(a);
      a.click();
      window.URL.revokeObjectURL(url);
    },
  },
  beforeDestroy() {
    if (this.websocket) this.websocket.close();
    clearInterval(this.timer);
  },
};
</script>

<style scoped>
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
  font-family: Arial, sans-serif;
  padding: 20px;
}

h1 {
  font-size: 24px;
  margin-bottom: 15px;
}

.status {
  display: flex;
  gap: 20px;
  font-size: 14px;
  margin-bottom: 10px;
}

.status p {
  margin: 0;
}

.connected {
  color: green;
}

.disconnected {
  color: red;
}

video {
  width: 80%;
  border: 2px solid #333;
  margin: 15px 0;
}

.buttons {
  display: flex;
  gap: 10px;
}

button {
  padding: 10px 15px;
  font-size: 14px;
  cursor: pointer;
  border: none;
  border-radius: 5px;
  background-color: #007bff;
  color: white;
  transition: background-color 0.3s;
}

button:hover:not(:disabled) {
  background-color: #0056b3;
}

button:disabled {
  background-color: #999;
  cursor: not-allowed;
}
</style>

