<script>
export default {
  data() {
    return {
      messages: [],
      input: "",
      answering: false,
      rendering: false,
      aiBuffer: "",
      aiInterval: null,
      selectedFiles: [],
    };
  },
  methods: {
    async getAnswer(question) {
      let answer = null;
      await fetch("https://payload.vextapp.com/hook/D3C3T4ZEUB/catch/hello", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Apikey: `Api-Key ${import.meta.env.VITE_API_KEY_RAG}`,
        },
        body: JSON.stringify({ payload: question }),
      })
        .then((response) => {
          if (response.ok) {
            answer = response.json();
          } else {
            throw new Error("Request failed with status " + response.status);
          }
        })
        .catch((error) => console.error("Error making API call:", error));
      return answer;
    },
    async sendMessage() {
      if (this.input.trim() === "" || this.rendering || this.answering) return;
      let question = this.input;
      this.input = "";
      this.messages.push({ role: "user", text: question });
      // Get answer from API
      this.answering = true;
      let answer = await this.getAnswer(question);
      this.answering = false;
      let answerText = answer.text ?? "I'm tired, sorry.";
      // Lazy load effect
      this.rendering = true;
      this.aiBuffer = "";
      let i = 0;
      this.aiInterval = setInterval(() => {
        if (i < answerText.length) {
          this.aiBuffer += answerText[i];
          i++;
        } else {
          clearInterval(this.aiInterval);
          this.messages.push({ role: "ai", text: this.aiBuffer });
          this.rendering = false;
          this.aiBuffer = "";
        }
      }, 30);
      this.input = "";
    },
    handleFileChange(e) {
      this.selectedFiles = Array.from(e.target.files);
    },
    async submitFiles() {
      if (!this.selectedFiles.length) return;
      const form = new FormData();
      form.append("file", this.selectedFiles[0]);
      form.append("dataset_id", "f107b40a-b2ad-41e1-8d30-6e9055d0a7ce");

      fetch("https://api.vextapp.com/openapi/v2/data-source", {
        method: "POST",
        headers: {
          accept: "application/json",
          Apikey: `Api-Key ${import.meta.env.VITE_API_KEY_RAG}`,
        },
        body: form,
      })
        .then((res) => res.json())
        .then(() => alert("File uploaded"))
        .catch((err) => alert(err));
    },
  },
  beforeDestroy() {
    if (this.aiInterval) clearInterval(this.aiInterval);
  },
};
</script>

<template>
  <div class="chat-container">
    <div class="chat-left-panel">
      <form class="upload-form" @submit.prevent="submitFiles">
        <label class="upload-label">
          <input
            type="file"
            accept=".pdf,.xlsx"
            class="upload-input"
            multiple
            @change="handleFileChange"
          />
          <span class="upload-btn">
            <svg
              width="22"
              height="22"
              viewBox="0 0 24 24"
              fill="none"
              stroke="#1976d2"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
            >
              <path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4" />
              <polyline points="17 8 12 3 7 8" />
              <line x1="12" y1="3" x2="12" y2="15" />
            </svg>
            <span>Choose PDF/XLSX</span>
          </span>
        </label>
        <transition name="fade">
          <button v-if="selectedFiles.length" type="submit" class="submit-btn">
            Submit Files
          </button>
        </transition>
      </form>
      <transition name="fade">
        <div v-if="selectedFiles.length" class="selected-files">
          <h4>Selected Files:</h4>
          <ul>
            <li v-for="file in selectedFiles" :key="file.name">
              <svg
                width="16"
                height="16"
                viewBox="0 0 24 24"
                fill="none"
                stroke="#42a5f5"
                stroke-width="2"
                stroke-linecap="round"
                stroke-linejoin="round"
              >
                <rect x="3" y="3" width="18" height="18" rx="2" ry="2" />
                <polyline points="3 7 12 13 21 7" />
              </svg>
              <span class="file-name">{{ file.name }}</span>
            </li>
          </ul>
        </div>
      </transition>
      <div class="upload-info">
        Upload documents to help Marcus answer your questions with more details
        and context from your files.
      </div>
    </div>
    <div class="chat-main">
      <div class="chat-window">
        <div
          v-for="(msg, idx) in messages"
          :key="idx"
          :class="['message', msg.role]"
        >
          <template v-if="msg.role === 'user'">
            <span class="user-label">You:</span>
            <span class="msg-text">{{ msg.text }}</span>
          </template>
          <template v-else-if="msg.role === 'ai'">
            <img src="/ai-avatar.png" alt="AI Avatar" class="avatar" />
            <div class="ai-content">
              <span class="ai-label">Marcus</span>
              <span class="msg-text">{{ msg.text }}</span>
            </div>
          </template>
        </div>
        <div v-if="answering" class="message ai answering">
          <img src="/ai-avatar.png" alt="AI Avatar" class="avatar" />
          <div class="ai-content">
            <span class="ai-label">Marcus</span>
            <span class="msg-text">
              <span class="typing-indicator">
                <span class="dot"></span>
                <span class="dot"></span>
                <span class="dot"></span>
              </span>
            </span>
          </div>
        </div>
        <div v-if="rendering" class="message ai">
          <img src="/ai-avatar.png" alt="AI Avatar" class="avatar" />
          <div class="ai-content">
            <span class="ai-label">Marcus</span>
            <span class="msg-text">{{ aiBuffer }}</span>
          </div>
        </div>
      </div>
      <form class="chat-input" @submit.prevent="sendMessage">
        <textarea
          v-model="input"
          placeholder="Type your question..."
          rows="2"
          @keydown.enter.exact.prevent="sendMessage"
        ></textarea>
        <button type="submit" :disabled="answering || rendering">Send</button>
      </form>
    </div>
  </div>
</template>

<style scoped>
.cursor {
  display: inline-block;
  width: 1ch;
  animation: blink 1s steps(1) infinite;
}
@keyframes blink {
  0%,
  50% {
    opacity: 1;
  }
  51%,
  100% {
    opacity: 0;
  }
}
html,
body,
#app {
  height: 100%;
  min-height: 100vh;
  margin: 0;
  padding: 0;
  overflow: hidden;
  font-family: "Inter", "Segoe UI", Arial, sans-serif;
  background: linear-gradient(135deg, #f9f9f9 0%, #e3f2fd 100%);
}
.chat-container {
  display: flex;
  width: 70vw;
  height: 700px;
  margin: 0 auto;
  border-radius: 24px;
  box-shadow: 0 8px 32px rgba(44, 62, 80, 0.1);
  background: #f4f7fb;
  overflow: hidden;
}
.chat-left-panel {
  width: 220px;
  background: #fff;
  border-radius: 24px 0 0 24px;
  box-shadow: 0 8px 32px rgba(44, 62, 80, 0.08);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: flex-start;
  padding: 40px 16px;
}
.chat-left-panel h3 {
  margin-bottom: 18px;
  color: #1976d2;
  font-size: 1.08em;
  font-weight: 600;
  text-align: center;
}
.upload-form {
  width: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}
.upload-label {
  display: flex;
  flex-direction: column;
  align-items: center;
  cursor: pointer;
  margin-bottom: 8px;
}
.upload-input {
  display: none;
}
.upload-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  background: #e3f2fd;
  color: #1976d2;
  border-radius: 10px;
  padding: 10px 22px;
  font-size: 1em;
  font-weight: 500;
  margin-top: 8px;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(44, 62, 80, 0.1);
  border: 1.5px solid #42a5f5;
  transition: background 0.2s, border 0.2s;
}
.upload-btn:hover {
  background: #bbdefb;
  border: 1.5px solid #1976d2;
}
.upload-info {
  margin-top: 14px;
  color: #797979;
  font-size: 0.98em;
  text-align: center;
}
.dataset-status {
  margin-top: 18px;
  font-size: 0.98em;
  color: #333;
  text-align: center;
}
.status-label {
  font-weight: 600;
  color: #1976d2;
}
.status-value {
  font-weight: 500;
  color: #42a5f5;
}
.chat-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 100%;
}
.chat-window {
  flex: 1;
  padding: 48px 48px 40px 48px;
  display: flex;
  flex-direction: column;
  gap: 24px;
  background: #fff;
  border-radius: 0 24px 0 0;
  box-shadow: 0 2px 16px rgba(44, 62, 80, 0.1);
  overflow-y: auto;
  overflow-x: hidden;
}
.message {
  padding: 8px 16px;
  border-radius: 16px;
  max-width: 60%;
  word-break: break-word;
  font-size: 0.95em;
  display: flex;
  align-items: center;
  gap: 8px;
  box-shadow: 0 2px 8px rgba(44, 62, 80, 0.08);
  transition: background 0.2s;
}
.message.user {
  background: #e3f2fd;
  align-self: flex-end;
  color: #1976d2;
  border-bottom-right-radius: 6px;
  border-top-right-radius: 24px;
  border-top-left-radius: 24px;
}
.message.ai {
  background: #f0f4ff;
  align-self: flex-start;
  color: #333;
  border-bottom-left-radius: 6px;
  border-top-right-radius: 24px;
  border-top-left-radius: 24px;
  display: flex;
  align-items: flex-start;
  gap: 8px;
}
.message.ai.answering {
  background: linear-gradient(90deg, #e3f2fd 60%, #bbdefb 100%);
  color: #1976d2;
  box-shadow: 0 2px 12px rgba(25, 118, 210, 0.12);
  opacity: 0.85;
  position: relative;
  border: none;
}
.message.ai.answering .msg-text {
  font-style: italic;
  font-size: 1.05em;
  letter-spacing: 2px;
  color: #1976d2;
  animation: pulse 1.2s infinite;
}
@keyframes pulse {
  0% {
    opacity: 0.5;
  }
  50% {
    opacity: 1;
  }
  100% {
    opacity: 0.5;
  }
}
.message.ai .ai-content {
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  align-items: flex-start;
}
.user-label {
  font-weight: 600;
  color: #1976d2;
  font-size: 1em;
}
.ai-label {
  font-weight: 600;
  color: #1565c0;
  font-size: 0.95em;
  display: block;
  text-align: left;
  line-height: 22px;
}
.msg-text {
  display: block;
  text-align: left;
  line-height: 22px;
}
.avatar {
  width: 42px;
  height: 42px;
  border-radius: 50%;
  margin-right: 6px;
  object-fit: cover;
  vertical-align: middle;
  background: #e3eafc;
  border: 1.5px solid #42a5f5;
  align-self: flex-start;
}
.ai-content {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: flex-start;
  min-height: 40px;
}
.chat-input {
  display: flex;
  gap: 12px;
  padding: 32px 48px;
  border-top: 1px solid #e0e0e0;
  background: #f4f7fb;
  box-shadow: 0 -2px 8px rgba(44, 62, 80, 0.04);
}
.chat-input textarea {
  flex: 1;
  resize: none;
  border-radius: 16px;
  border: 1.5px solid #42a5f5;
  padding: 8px;
  font-size: 0.95em;
  background: #fff;
  color: #222;
  box-shadow: 0 1px 4px rgba(44, 62, 80, 0.04);
  transition: border 0.2s;
}
.chat-input textarea:hover {
  border: 1.5px solid #42a5f5;
  box-shadow: 0 0 0 2px #42a5f533;
  background: #fff;
}
.chat-input textarea:focus {
  border: 1.5px solid #42a5f5;
  outline: none;
}
.chat-input button {
  background: linear-gradient(90deg, #1976d2 60%, #42a5f5 100%);
  color: #fff;
  border: none;
  border-radius: 16px;
  padding: 0 20px;
  font-size: 0.95em;
  font-weight: 600;
  cursor: pointer;
  box-shadow: 0 2px 8px rgba(44, 62, 80, 0.1);
  transition: background 0.2s, box-shadow 0.2s;
}
.chat-input button:hover {
  background: linear-gradient(90deg, #1565c0 60%, #1976d2 100%);
  box-shadow: 0 4px 16px rgba(44, 62, 80, 0.16);
}
.submit-btn {
  margin-top: 8px;
  background: linear-gradient(90deg, #42a5f5 60%, #1976d2 100%);
  color: #fff;
  border-radius: 10px;
  padding: 10px 24px;
  font-size: 1em;
  font-weight: 600;
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 8px rgba(44, 62, 80, 0.1);
  transition: background 0.2s;
}
.submit-btn:disabled {
  background: #bdbdbd;
  cursor: not-allowed;
}
.selected-files {
  margin-top: 10px;
  width: 100%;
  text-align: left;
  background: #f5f7fa;
  border-radius: 8px;
  padding: 10px 12px;
  box-shadow: 0 1px 4px rgba(44, 62, 80, 0.06);
}
.selected-files h4 {
  margin-bottom: 6px;
  font-size: 1em;
  color: #1976d2;
}
.selected-files ul {
  list-style: none;
  padding: 0;
  margin: 0;
}
.selected-files li {
  font-size: 0.98em;
  color: #333;
  margin-bottom: 4px;
  display: flex;
  align-items: center;
  gap: 6px;
}
.file-name {
  font-size: 0.98em;
  color: #1976d2;
}
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter,
.fade-leave-to {
  opacity: 0;
}
.typing-indicator {
  display: inline-block;
  margin-right: 10px;
  vertical-align: middle;
}
.dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  margin: 0 2px;
  background: #42a5f5;
  border-radius: 50%;
  opacity: 0.6;
  animation: typing 1.2s infinite;
}
.dot:nth-child(2) {
  animation-delay: 0.2s;
}
.dot:nth-child(3) {
  animation-delay: 0.4s;
}
@keyframes typing {
  0%,
  80%,
  100% {
    opacity: 0.6;
    transform: scale(1);
  }
  40% {
    opacity: 1;
    transform: scale(1.3);
  }
}
.typing-text {
  font-style: italic;
  color: #1976d2;
  font-size: 1em;
  vertical-align: middle;
}
</style>
