# Adding Deepgram & Cartesia to AnythingLLM

## System Architecture Overview

AnythingLLM has **TWO SEPARATE** audio systems:

### 1. STT (Speech-to-Text) for Chat Input
- **Location**: Frontend only (`frontend/src/components/WorkspaceChat/ChatContainer/PromptInput/SpeechToText/`)
- **Technology**: Browser's native Web Speech API via `react-speech-recognition`
- **Use Case**: Real-time voice typing in chat
- **Note**: Currently NO backend provider system - it's all client-side

### 2. Whisper for Audio File Transcription
- **Location**: Collector service (`collector/utils/WhisperProviders/`)
- **Technology**: Provider-based system (local models or OpenAI API)
- **Use Case**: Transcribing uploaded audio/video files to documents

### 3. TTS (Text-to-Speech) for Chat Responses
- **Location**: Backend (`server/utils/TextToSpeech/`)
- **Technology**: Provider-based system
- **Use Case**: Reading AI responses aloud

---

## Integration Points

### For DEEPGRAM (STT - two integration paths):

#### Option A: Real-time Chat STT (Replace Web Speech API)
**Files to modify:**
```
frontend/src/components/WorkspaceChat/ChatContainer/PromptInput/SpeechToText/index.jsx
```

**Approach:**
- Add Deepgram WebSocket streaming
- Replace `react-speech-recognition` with Deepgram SDK
- Keep same UI/UX (microphone button, CTRL+M)

#### Option B: Audio File Transcription Provider
**Files to create/modify:**
1. `collector/utils/WhisperProviders/DeepgramWhisper.js` (new)
2. `collector/index.js` - Add Deepgram to provider selection
3. `server/utils/helpers/updateENV.js` - Add Deepgram env vars
4. `frontend/src/pages/GeneralSettings/AudioPreference/whisper.jsx` (check if exists)

### For CARTESIA (TTS):

**Files to create/modify:**

1. **Backend Provider** (new file):
```
server/utils/TextToSpeech/cartesia/index.js
```

2. **Backend Provider Router** (modify):
```
server/utils/TextToSpeech/index.js
```
Add case for 'cartesia'

3. **Frontend Options Component** (new file):
```
frontend/src/components/TextToSpeech/CartesiaOptions/index.jsx
```

4. **Frontend Provider List** (modify):
```
frontend/src/pages/GeneralSettings/AudioPreference/tts.jsx
```
Add Cartesia to PROVIDERS array

5. **Environment Variables** (modify):
```
server/utils/helpers/updateENV.js
```
Add Cartesia API key and voice settings

6. **Logo** (add):
```
frontend/src/media/ttsproviders/cartesia.png
```

---

## Step-by-Step Implementation Pattern

### TTS Provider (Cartesia Example):

#### Step 1: Backend Provider Class
```javascript
// server/utils/TextToSpeech/cartesia/index.js
class CartesiaTTS {
  constructor() {
    if (!process.env.TTS_CARTESIA_API_KEY)
      throw new Error("No Cartesia API key was set.");
    
    this.apiKey = process.env.TTS_CARTESIA_API_KEY;
    this.voice = process.env.TTS_CARTESIA_VOICE_ID ?? "default-voice";
    this.model = process.env.TTS_CARTESIA_MODEL ?? "sonic-english";
  }

  async ttsBuffer(textInput) {
    try {
      // Implement Cartesia API call here
      // Return Buffer of audio data
      const response = await fetch('https://api.cartesia.ai/tts', {
        method: 'POST',
        headers: {
          'X-API-Key': this.apiKey,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          text: textInput,
          voice_id: this.voice,
          model_id: this.model,
        })
      });
      
      const arrayBuffer = await response.arrayBuffer();
      return Buffer.from(arrayBuffer);
    } catch (e) {
      console.error('[CartesiaTTS]', e);
      return null;
    }
  }
}

module.exports = { CartesiaTTS };
```

#### Step 2: Register Provider in Router
```javascript
// server/utils/TextToSpeech/index.js
function getTTSProvider() {
  const provider = process.env.TTS_PROVIDER || "openai";
  switch (provider) {
    case "openai":
      const { OpenAiTTS } = require("./openAi");
      return new OpenAiTTS();
    case "elevenlabs":
      const { ElevenLabsTTS } = require("./elevenLabs");
      return new ElevenLabsTTS();
    case "generic-openai":
      const { GenericOpenAiTTS } = require("./openAiGeneric");
      return new GenericOpenAiTTS();
    case "cartesia":  // ADD THIS
      const { CartesiaTTS } = require("./cartesia");
      return new CartesiaTTS();
    default:
      throw new Error("ENV: No TTS_PROVIDER value found in environment!");
  }
}
```

#### Step 3: Add Environment Variable Validation
```javascript
// server/utils/helpers/updateENV.js
// Find the TTS section and add:
CartesiaTTSKey: {
  envKey: "TTS_CARTESIA_API_KEY",
  checks: [],
},
CartesiaTTSVoiceId: {
  envKey: "TTS_CARTESIA_VOICE_ID",
  checks: [],
},
CartesiaTTSModel: {
  envKey: "TTS_CARTESIA_MODEL",
  checks: [],
},
```

#### Step 4: Frontend Options Component
```jsx
// frontend/src/components/TextToSpeech/CartesiaOptions/index.jsx
import React from "react";

export default function CartesiaOptions({ settings }) {
  return (
    <div className="w-full flex flex-col gap-y-4">
      <div className="w-full flex items-start gap-4">
        <div className="flex flex-col w-60">
          <label className="text-white text-sm font-semibold block mb-4">
            Cartesia API Key
          </label>
          <input
            type="password"
            name="TTSCartesiaKey"
            className="bg-theme-settings-input-bg text-white placeholder:text-theme-settings-input-placeholder text-sm rounded-lg focus:border-white block w-full p-2.5"
            placeholder="Cartesia API Key"
            defaultValue={settings?.TTSCartesiaKey ? "*".repeat(20) : ""}
            required={true}
            autoComplete="off"
          />
        </div>
        
        <div className="flex flex-col w-60">
          <label className="text-white text-sm font-semibold block mb-4">
            Voice ID
          </label>
          <input
            type="text"
            name="TTSCartesiaVoiceId"
            className="bg-theme-settings-input-bg text-white placeholder:text-theme-settings-input-placeholder text-sm rounded-lg focus:border-white block w-full p-2.5"
            placeholder="Voice ID"
            defaultValue={settings?.TTSCartesiaVoiceId}
          />
        </div>

        <div className="flex flex-col w-60">
          <label className="text-white text-sm font-semibold block mb-4">
            Model
          </label>
          <select
            name="TTSCartesiaModel"
            defaultValue={settings?.TTSCartesiaModel || "sonic-english"}
            className="bg-theme-settings-input-bg text-white text-sm rounded-lg block w-full p-2.5"
          >
            <option value="sonic-english">Sonic English</option>
            <option value="sonic-multilingual">Sonic Multilingual</option>
          </select>
        </div>
      </div>
    </div>
  );
}
```

#### Step 5: Register in Frontend Provider List
```jsx
// frontend/src/pages/GeneralSettings/AudioPreference/tts.jsx
import CartesiaIcon from "@/media/ttsproviders/cartesia.png";
import CartesiaOptions from "@/components/TextToSpeech/CartesiaOptions";

const PROVIDERS = [
  // ... existing providers ...
  {
    name: "Cartesia",
    value: "cartesia",
    logo: CartesiaIcon,
    options: (settings) => <CartesiaOptions settings={settings} />,
    description: "Ultra-fast, realistic text to speech with Cartesia.",
  },
];
```

---

## For Deepgram STT Provider (Audio File Transcription):

### Files to Create/Modify:

1. **Backend Provider**:
```javascript
// collector/utils/WhisperProviders/DeepgramWhisper.js
const fs = require("fs");
const FormData = require("form-data");
const axios = require("axios");

class DeepgramWhisper {
  constructor({ options }) {
    if (!options.deepgramApiKey) 
      throw new Error("No Deepgram API key was set.");
    
    this.apiKey = options.deepgramApiKey;
    this.model = options.deepgramModel || "nova-2";
    this.#log("Initialized.");
  }

  #log(text, ...args) {
    console.log(`\x1b[36m[DeepgramWhisper]\x1b[0m ${text}`, ...args);
  }

  async processFile(fullFilePath) {
    try {
      const formData = new FormData();
      formData.append('file', fs.createReadStream(fullFilePath));

      const response = await axios.post(
        `https://api.deepgram.com/v1/listen?model=${this.model}&smart_format=true`,
        formData,
        {
          headers: {
            'Authorization': `Token ${this.apiKey}`,
            ...formData.getHeaders(),
          },
        }
      );

      const transcript = response.data?.results?.channels?.[0]?.alternatives?.[0]?.transcript;

      if (!transcript) {
        return {
          content: "",
          error: "No content was able to be transcribed.",
        };
      }

      return { content: transcript, error: null };
    } catch (error) {
      this.#log(`Could not get response from Deepgram`, error.message);
      return { content: "", error: error.message };
    }
  }
}

module.exports = { DeepgramWhisper };
```

2. **Collector Service Router**:
```javascript
// collector/index.js (find the whisper provider selection)
// Add case for 'deepgram'
```

3. **Environment Variables**:
```javascript
// server/utils/helpers/updateENV.js
DeepgramWhisperApiKey: {
  envKey: "DEEPGRAM_API_KEY",
  checks: [],
},
DeepgramWhisperModel: {
  envKey: "DEEPGRAM_MODEL",
  checks: [],
},
```

---

## Testing Checklist

### For TTS (Cartesia):
- [ ] Backend class returns valid audio buffer
- [ ] API key validation works
- [ ] Frontend settings page shows Cartesia option
- [ ] Settings are saved correctly to .env
- [ ] Audio plays in chat when TTS is triggered
- [ ] Error handling for invalid API key

### For STT (Deepgram):
- [ ] Audio files are transcribed correctly
- [ ] Supports multiple audio formats (mp3, wav, m4a)
- [ ] Error handling for failed transcriptions
- [ ] Settings page allows Deepgram configuration

---

## Environment Variables to Add

Add these to `docker/.env` and `server/.env.example`:

```bash
# Cartesia TTS
TTS_CARTESIA_API_KEY=your_cartesia_api_key
TTS_CARTESIA_VOICE_ID=default_voice_id
TTS_CARTESIA_MODEL=sonic-english

# Deepgram STT (for file transcription)
WHISPER_PROVIDER=deepgram
DEEPGRAM_API_KEY=your_deepgram_api_key
DEEPGRAM_MODEL=nova-2
```

---

## Summary of Changes

### Cartesia TTS (Full Integration):
1. Create backend provider class
2. Register in backend router
3. Create frontend options component
4. Add to frontend provider list
5. Add environment variable validation
6. Add logo image
7. Update .env.example

### Deepgram STT (Two Options):

**Option A - Replace Real-time Chat STT:**
- Modify existing component to use Deepgram streaming
- More complex, affects chat UX

**Option B - Add as Whisper Provider (for file uploads):**
- Create provider class (like OpenAI Whisper)
- Add to collector service
- Add frontend settings
- Simpler, follows existing pattern

---

## Recommended Approach

1. **Start with Cartesia TTS** - It's simpler and follows the established pattern
2. **Then add Deepgram as Whisper provider** - For audio file transcription
3. **Optionally** - Replace real-time STT later if needed

Both services will coexist nicely:
- Cartesia for speaking responses
- Deepgram for transcribing uploaded audio files
- Browser STT (current) for real-time chat voice input
