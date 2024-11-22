<template>
  <div class="app-container">
    <div class="header">
      <h1>📚 Bookstore Support Channel! Say hi to Jenny 👋 🤖</h1>
    </div>
    <div v-if="!isConnected" class="connection-container">
      <textarea v-model="instructions" placeholder="Enter assistant instructions..."></textarea>
      <div class="controls-container">
        <button @click="toggleConnection">
          {{ isConnected ? 'Reset' : 'Start' }}
        </button>
      </div>
    </div>
    <div v-else>
      <div class="controls-container">
        <button @click="toggleConnection">
          {{ isConnected ? 'Reset' : 'Start' }}
        </button>
        <template v-if="isConnected">
          <button @click="toggleMute" :class="{ 'muted': isMuted }">
            {{ isMuted ? 'Unmute' : 'Mute' }}
          </button>
          <button @click="toggleInstructions" class="instructions-toggle">
            {{ isInstructionsCollapsed ? 'Show Instructions' : 'Hide Instructions' }}
          </button>
        </template>
      </div>
      <div class="instructions-container" v-show="!isInstructionsCollapsed">
        <textarea v-model="instructions" placeholder="Enter assistant instructions..."></textarea>
        <button @click="updateInstructions">Update Instructions</button>
      </div>
      <div class="messages-container">
        <h3>Conversation</h3>
        <div v-for="(message, index) in messages" :key="index" 
             :class="['message', message.role.toLowerCase()]">
          <div class="message-header">{{ message.role }}</div>
          <div class="message-content">
            {{ message.content[0].transcript }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { RealtimeClient } from '@openai/realtime-api-beta'

export default {
  data() {
    return {
      isConnected: false,
      isMuted: true,
      isInstructionsCollapsed: false,
      messages: [],
      client: null,
      audioContext: null,
      source: null,
      processor: null,
      inputSampleRate: null,
      instructions: `You are Jenny, a warm and friendly customer service representative for our bookstore.
Your goal is to provide exceptional customer support by following this structured approach:

1. Begin by introducing yourself, warmly greeting the customer and asking how you can help them today.
2. Once they explain their need, collect their identification details:
   - Full name. Ensure correct spelling, it's really important to correctly spell customer's name.
   - Email address
   - Order number (if relevant)
3. After verifying their information, assist with their request with a caring and solution-focused attitude

Also verify each step with the customer to ensure accuracy.

You can help customers with:
- Book recommendations
- Order status and tracking
- Returns and exchanges
- Store hours and locations
- Loyalty program information
- General inquiries about books and availability

Maintain a cheerful, helpful tone throughout the conversation. Show genuine interest in resolving their concerns and go above and beyond to ensure customer satisfaction. Remember to thank them for their patience during verification steps.`,
    }
  },
  methods: {
    async initializeRealtimeClient() {
      try {
        this.client = new RealtimeClient({
          apiKey: import.meta.env.VITE_OPENAI_API_KEY,
          dangerouslyAllowAPIKeyInBrowser: true,
          model: 'gpt-4o-realtime-preview-2024-10-01'
        });

        // Set initial session parameters with user-provided instructions
        this.client.updateSession({ instructions: this.instructions });
        this.client.updateSession({ voice: 'alloy' });
        this.client.updateSession({
          turn_detection: { type: 'server_vad' }, // Enable server-side VAD
          input_audio_transcription: { model: 'whisper-1' },
        });

        // Set up event handling
        this.client.on('conversation.item.completed', ({ item }) => {
          if (item.type === 'message' && item.role === 'assistant') {
            this.messages.push({
              role: 'Assistant',
              content: item.content
            });
            console.log('Received message:', this.messages);

            // Always play audio response regardless of mute state
            if (item.formatted?.audio) {
              this.playAudioResponse(item.formatted.audio);
            }
          }
        });

        await this.client.connect();
        this.isConnected = true;
        this.isMuted = false; // Start muted by default
        this.isInstructionsCollapsed = true; // Hide instructions after connecting

        this.client.sendUserMessageContent([{ type: 'input_text', text: `A customer has just called. Please answer the phone.` }]);


        // Set up audio recording after connecting
        await this.setupAudioRecording();
      } catch (error) {
        console.error('Failed to initialize realtime client:', error);
      }
    },
    async setupAudioRecording() {
      try {
        const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
        this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
        this.inputSampleRate = this.audioContext.sampleRate;
        this.source = this.audioContext.createMediaStreamSource(stream);

        // Create a ScriptProcessorNode to process audio data
        this.processor = this.audioContext.createScriptProcessor(4096, 1, 1);
        this.source.connect(this.processor);
        this.processor.connect(this.audioContext.destination);

        this.processor.onaudioprocess = (event) => {
          if (this.isMuted) return; // Do not process audio if muted
          const inputData = event.inputBuffer.getChannelData(0);

          // Resample the audio data to 24000Hz
          const resampledData = this.resampleAudioBuffer(inputData, this.inputSampleRate, 24000);

          // Convert to Int16Array
          const int16Data = this.floatTo16BitPCM(resampledData);

          // Send to the API
          this.client.appendInputAudio(int16Data);
        };
      } catch (error) {
        console.error('Error setting up audio recording:', error);
      }
    },
    resampleAudioBuffer(audioBuffer, inputSampleRate, outputSampleRate) {
      const sampleRatio = inputSampleRate / outputSampleRate;
      const newLength = Math.round(audioBuffer.length / sampleRatio);
      const resampledBuffer = new Float32Array(newLength);

      for (let i = 0; i < newLength; i++) {
        const samplePosition = i * sampleRatio;
        const first = Math.floor(samplePosition);
        const second = first + 1;
        const interpolation = samplePosition - first;

        if (second < audioBuffer.length) {
          resampledBuffer[i] = audioBuffer[first] + (audioBuffer[second] - audioBuffer[first]) * interpolation;
        } else {
          resampledBuffer[i] = audioBuffer[first];
        }
      }
      return resampledBuffer;
    },
    floatTo16BitPCM(input) {
      const output = new Int16Array(input.length);
      for (let i = 0; i < input.length; i++) {
        let s = Math.max(-1, Math.min(1, input[i]));
        output[i] = s < 0 ? s * 0x8000 : s * 0x7FFF;
      }
      return output;
    },
    toggleInstructions() {
      this.isInstructionsCollapsed = !this.isInstructionsCollapsed;
    },
    toggleMute() {
      this.isMuted = !this.isMuted;
      if (!this.isMuted) {
        this.startRecording();
      } else {
        this.stopRecording();
      }
    },
    startRecording() {
      if (this.audioContext && this.processor) {
        this.isMuted = false;
      }
    },
    stopRecording() {
      if (this.audioContext && this.processor) {
        this.isMuted = true;
        // No need to call createResponse() since server-side VAD handles it
      }
    },
    playAudioResponse(audioData) {
      // Create a new audio context for each playback
      const audioContext = new (window.AudioContext || window.webkitAudioContext)();

      // Convert Int16Array to Float32Array
      const float32Data = new Float32Array(audioData.length);
      for (let i = 0; i < audioData.length; i++) {
        float32Data[i] = audioData[i] / 32768.0;
      }

      // Create and set up the audio buffer
      const audioBuffer = audioContext.createBuffer(1, float32Data.length, 24000);
      audioBuffer.copyToChannel(float32Data, 0);

      // Create and play the audio source
      const source = audioContext.createBufferSource();
      source.buffer = audioBuffer;
      source.connect(audioContext.destination);
      source.start(0);
    },
    updateInstructions() {
      if (this.client) {
        this.client.updateSession({ instructions: this.instructions });
      }
    },
    async resetConnection() {
      // Clean up existing connection
      if (this.client) {
        await this.client.disconnect();
        this.client = null;
      }
      // Reset state
      this.isConnected = false;
      this.messages = [];
      this.isInstructionsCollapsed = false;
      this.isMuted = true;
    },
    async toggleConnection() {
      if (this.isConnected) {
        await this.resetConnection();
      } else {
        await this.initializeRealtimeClient();
      }
    },
  }
}
</script>

<style scoped>
.app-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  color: #2c3e50;
}

.header {
  text-align: center;
  margin-bottom: 2rem;
}

.header h1 {
  font-weight: 600;
  color: #1a73e8;
}

button {
  background: #1a73e8;
  color: white;
  border: none;
  padding: 12px 24px;
  border-radius: 8px;
  font-weight: 500;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

button:hover {
  background: #1557b0;
  transform: translateY(-1px);
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.instructions-container textarea,
.connection-container textarea {
  width: 100%;
  min-height: 300px;
  height: 50vh;
  max-height: 600px;
  padding: 1rem;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  font-size: 0.95rem;
  line-height: 1.5;
  transition: border-color 0.2s ease;
  background: #f8f9fa;
  resize: vertical;
}

.instructions-container textarea:focus,
.connection-container textarea:focus {
  border-color: #1a73e8;
  outline: none;
  box-shadow: 0 0 0 2px rgba(26, 115, 232, 0.2);
}

.messages-container {
  border: none;
  background: #f8f9fa;
  border-radius: 12px;
  padding: 1.5rem;
  max-height: 600px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
  overflow-y: auto;
  margin-top: 1rem;
}

.message {
  margin-bottom: 1rem;
  padding: 1rem;
  border-radius: 12px;
  animation: fadeIn 0.3s ease;
  display: block;
  width: auto;
}

.message.assistant {
  background: #e8f0fe;
  margin-right: 2rem;
}

.message.user {
  background: white;
  margin-left: 2rem;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.message-header {
  font-weight: 600;
  color: #1a73e8;
  margin-bottom: 0.5rem;
  font-size: 0.9rem;
}

.message-content {
  color: #4a4a4a;
  line-height: 1.5;
}

button.muted {
  background-color: #dc3545;
}

button.muted:hover {
  background-color: #c82333;
}

.instructions-toggle {
  background-color: #34495e !important;
}

.instructions-toggle:hover {
  background-color: #2c3e50 !important;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.controls-container {
  display: flex;
  justify-content: center;
  gap: 1rem;
  margin: 1rem 0;
}
</style>
