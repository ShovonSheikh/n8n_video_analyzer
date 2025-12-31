# n8n Video Processing Workflow System

## Overview

This is an automated video processing system built with n8n that creates voiceover videos in Banglish (Bengali written in English). The system consists of three interconnected workflows that handle video analysis, script generation, audio processing, and final video rendering.

---

## System Architecture

The system uses three workflows that work together:

1. **Video to Script** - Analyzes videos and generates voiceover scripts
2. **Video Creator** - Renders final videos with voiceovers
3. **Audio Receiver** - Receives audio files from Telegram and triggers video creation

### Workflow Flow Diagram

```
Video Upload (Google Drive)
    ↓
Video to Script Workflow
    ├─ Analyzes video with Gemini AI
    ├─ Generates Banglish script
    └─ Sends script to Telegram
    ↓
User Records Audio
    ↓
Audio Receiver Workflow
    ├─ Receives audio from Telegram
    ├─ Uploads to Google Drive
    └─ Triggers Video Creator
    ↓
Video Creator Workflow
    ├─ Renders video with Creatomate
    └─ Sends download link via Telegram
```

---

## Workflow 1: Video to Script

### Purpose
Automatically analyzes uploaded videos and generates synchronized Banglish voiceover scripts that match the video duration.

### How It Works

#### 1. **Video Detection** (Google Drive Trigger)
- Monitors a specific Google Drive folder (`n8n-test`)
- Triggers when new video files are uploaded
- Polls every minute for changes

#### 2. **Duplicate Prevention** (Code in JavaScript1)
```javascript
// Prevents processing the same file multiple times
- Maintains a list of processed files in workflow static data
- Only processes files created in the last 2 minutes
- Keeps track of up to 100 recently processed files
```

#### 3. **Video Download** (Download Video1)
- Downloads the video file from Google Drive
- Prepares it for AI analysis

#### 4. **AI Video Analysis** (Analyze video1)
Uses Google Gemini Flash to analyze the video and extract:
- Video duration in seconds
- Timestamp-based scene descriptions
- Visual details (characters, actions, environment)
- Sequence of events
- Emotional moments and peaks
- Audio cues (if present)

**Output Format:**
```
Video Duration: X seconds

[0-5 seconds]: Opening scene description...
[5-12 seconds]: Next event...
[12-25 seconds]: Main action...
[25-40 seconds]: Climax moment...
[40-45 seconds]: Closing...

Key Visual Elements:
- Color palettes
- Important objects
- Character details

Emotional Peaks:
- Timing of key moments
- Where pauses are needed
```

#### 5. **Script Generation** (Generate Script)
Creates a Banglish voiceover script with strict requirements:
- **Word count limit**: 2.5 words per second × video duration
- **Natural speaking pace**: Conversational Banglish
- **Emotional words**: "Bhaba jay!", "ekdom alada", etc.
- **Timeline sync**: Descriptions match actual video timing
- **No formatting**: Pure script text, no labels or timestamps

#### 6. **Telegram Delivery** (Send Script to Telegram)
- Sends formatted script to Telegram chat
- Prompts user to record voiceover audio
- Uses Markdown formatting for readability

#### 7. **Workflow Pause** (Store Video Info & HTTP Request)
```javascript
// Stores workflow resume information
{
  "resumeUrl": "webhook_url",
  "executionId": "exec_id",
  "videoId": "drive_file_id",
  "videoName": "filename.mp4",
  "status": "waiting_for_audio"
}
```
- Saves to JSONBin for persistence
- Pauses execution waiting for audio response

#### 8. **Resume Point** (Wait for Audio Response)
- Waits for the Audio Receiver workflow to send audio
- Resumes when webhook is called
- Downloads original video file again
- Searches for corresponding audio file
- Passes data to Video Creator workflow

### Key Features
- **Smart duplicate detection**: Won't process the same video twice
- **Duration-aware scripting**: Scripts perfectly match video length
- **Emotional pacing**: Identifies key moments for impactful narration
- **Persistent state**: Can resume after system restarts

---

## Workflow 2: Video Creator

### Purpose
Renders final videos by combining base video footage with voiceover audio using Creatomate API.

### How It Works

#### 1. **Workflow Trigger** (When Executed by Another Workflow)
Receives input data:
```json
{
  "base_audio": "https://drive.google.com/...",
  "base_video": "https://drive.google.com/...",
  "chat_id": "telegram_user_id"
}
```

#### 2. **Video Rendering Request** (HTTP Request)
```javascript
POST https://api.creatomate.com/v2/renders

Headers:
- Authorization: Bearer {api_key}

Body:
{
  "template_id": "4da8294a-562b-4ed2-811d-ed3b60973579",
  "modifications": {
    "base_video.source": "video_url",
    "base_audio.source": "audio_url"
  }
}
```
- Initiates video rendering with Creatomate
- Returns render job ID

#### 3. **Processing Wait Loop** (Wait → HTTP Request1 → Switch)
```
While status is "processing":
    Wait 10 seconds
    Check render status
    If succeeded → Send download link
    If failed → Send error message
    If still processing → Loop again
```

**Status Flow:**
- `planned` → Initial state
- `transcribing` → Processing audio
- `waiting` → Queued for rendering
- `rendering` → Creating video
- `succeeded` → Ready for download ✓
- `failed` → Error occurred ✗

#### 4. **Success Notification** (Send a text message1)
When render succeeds:
```
The video is generated. You can view that through this link:

```Link
https://cdn.creatomate.com/renders/...
```
```

#### 5. **Error Handling** (Send a text message)
If rendering fails:
```
There's an error in the render. 
Please visit the API log to see more.
```

### Key Features
- **Automatic retry loop**: Checks status every 10 seconds
- **Multiple status handling**: Accounts for all rendering states
- **Direct download links**: Users get CDN-hosted video URLs
- **Error reporting**: Alerts users to rendering failures

---

## Workflow 3: Audio Receiver

### Purpose
Receives audio voiceovers from Telegram, uploads them to Google Drive, and triggers the video creation process.

### How It Works

#### 1. **Telegram Listener** (Telegram Trigger)
- Listens for all incoming messages
- Captures audio files, documents, and voice messages
- Extracts file metadata and user information

#### 2. **Instant Confirmation** (Send Confirmation Message)
```
✅ Audio file received! Processing your video now...
```
- Provides immediate user feedback
- Confirms audio was received successfully

#### 3. **Audio Type Validation** (Check if Audio)
```javascript
if (message.document.mime_type.contains("audio")) {
    // Process audio
} else {
    // Ignore non-audio messages
}
```

#### 4. **Resume URL Retrieval** (HTTP Request)
```javascript
GET https://api.jsonbin.io/v3/b/{bin_id}/latest
Headers: X-Master-Key

// Retrieves stored workflow state:
{
  "resumeUrl": "webhook_url",
  "executionId": "exec_id", 
  "videoId": "drive_file_id",
  "videoName": "filename.mp4",
  "status": "waiting_for_audio"
}
```

#### 5. **State Validation** (Code in JavaScript)
```javascript
// Validates workflow state
- Checks if resume URL exists
- Verifies status is "waiting_for_audio"
- Extracts audio file ID from Telegram message
- Throws errors if workflow not ready
```

#### 6. **Audio Download Pipeline**
**Get File Path** → Gets Telegram file path
```javascript
POST https://api.telegram.org/bot{token}/getFile
Body: { "file_id": "audio_file_id" }
```

**Download Audio** → Downloads from Telegram servers
```javascript
GET https://api.telegram.org/file/bot{token}/{file_path}
Response: Binary audio data
```

#### 7. **Google Drive Upload** (Upload file)
```javascript
// Names file to match video
original_video.mp4 → original_video_audio.ogg

// Uploads to same folder
Folder: n8n-test (1htTAPNhr1q_rnNwoZ0HHeTWALgf-rzoc)
```

#### 8. **Workflow Resume** (Download Audio1 → Send to Workflow 1)
```javascript
// Triggers the paused Video to Script workflow
POST {resumeUrl}
Content-Type: audio/ogg
Body: Binary audio data

// This resumes the waiting workflow
```

#### 9. **Completion Notifications**
**Success:**
```
✅ Done! Your audio has been processed successfully.
```

**Error:**
```
❌ Error: {error_message}
```
Common errors:
- No resume URL found (upload video first)
- Workflow no longer waiting (expired state)
- Audio file not found in message

### Key Features
- **Instant feedback**: Confirms receipt immediately
- **Smart file naming**: Automatically matches audio to video
- **State persistence**: Uses JSONBin for reliable state storage
- **Error recovery**: Provides helpful error messages
- **Webhook resumption**: Seamlessly continues paused workflows

---

## Setup Instructions

### Prerequisites
1. n8n instance (self-hosted or cloud)
2. Google account with Drive API access
3. Telegram bot token
4. Creatomate account and API key
5. Google Gemini API key
6. JSONBin account

### Step 1: Google Drive Setup

1. **Create Google Drive OAuth2 Credentials:**
   - Go to [Google Cloud Console](https://console.cloud.google.com)
   - Create a new project
   - Enable Google Drive API
   - Create OAuth 2.0 credentials
   - Add authorized redirect URI: `{your-n8n-url}/rest/oauth2-credential/callback`

2. **Create Folder Structure:**
   ```
   My Drive/
   └── n8n-test/
       ├── (upload videos here)
       └── (audio files stored here)
   ```

3. **Get Folder ID:**
   - Open the `n8n-test` folder in Google Drive
   - Copy ID from URL: `drive.google.com/drive/folders/{FOLDER_ID}`
   - Use ID: `1htTAPNhr1q_rnNwoZ0HHeTWALgf-rzoc`

### Step 2: Telegram Bot Setup

1. **Create Telegram Bot:**
   - Message [@BotFather](https://t.me/botfather) on Telegram
   - Send `/newbot` command
   - Follow prompts to create bot
   - Save the bot token (e.g., `8521056441:AAEdbe1NCCEFos1yD8kFfZHfCwgAx7XW2O4`)

2. **Get Your Chat ID:**
   - Message your bot
   - Visit: `https://api.telegram.org/bot{TOKEN}/getUpdates`
   - Find your chat ID in the response (e.g., `6024349418`)

### Step 3: Creatomate Setup

1. **Sign up at [Creatomate](https://creatomate.com)**
2. **Get API Key:**
   - Go to Settings → API Keys
   - Copy your API key
3. **Create Video Template:**
   - Create new template in Creatomate
   - Add video layer named `base_video`
   - Add audio layer named `base_audio`
   - Save and copy template ID (e.g., `4da8294a-562b-4ed2-811d-ed3b60973579`)

### Step 4: JSONBin Setup

1. **Sign up at [JSONBin](https://jsonbin.io)**
2. **Create a new bin:**
   - Create empty JSON object: `{}`
   - Copy bin ID (e.g., `694fd909d0ea881f404356fa`)
3. **Get API Key:**
   - Go to API Keys section
   - Copy Master Key (e.g., `$2a$10$8PC0U03GuucFp...`)

### Step 5: Google Gemini Setup

1. **Get API Key:**
   - Visit [Google AI Studio](https://makersuite.google.com/app/apikey)
   - Create new API key
   - Copy the key

### Step 6: Import Workflows

1. **Import each workflow JSON:**
   - In n8n, click "Add workflow"
   - Choose "Import from File"
   - Select each JSON file

2. **Configure credentials for each workflow:**

   **Video to Script Workflow:**
   - Google Drive OAuth2 API (2 nodes)
   - Google Gemini API (2 nodes)
   - Telegram API (1 node)

   **Video Creator Workflow:**
   - Telegram API (2 nodes)
   - Update HTTP Request nodes with Creatomate API key

   **Audio Receiver Workflow:**
   - Telegram API (4 nodes)
   - Google Drive OAuth2 API (1 node)
   - Update HTTP Request nodes with Telegram bot token

### Step 7: Update Configuration Values

**Video to Script:**
```javascript
// Google Drive Trigger1
triggerOn: "specificFolder"
folderToWatch: "1htTAPNhr1q_rnNwoZ0HHeTWALgf-rzoc" // Your folder ID

// Send Script to Telegram
chatId: "6024349418" // Your chat ID

// HTTP Request (JSONBin)
url: "https://api.jsonbin.io/v3/b/694fd909d0ea881f404356fa"
X-Master-Key: "YOUR_JSONBIN_KEY"
```

**Video Creator:**
```javascript
// HTTP Request
Authorization: "Bearer YOUR_CREATOMATE_API_KEY"
template_id: "YOUR_TEMPLATE_ID"

// Send a text message & Send a text message1
chatId: "6024349418" // Your chat ID
```

**Audio Receiver:**
```javascript
// Get File Path & Download Audio nodes
url: "https://api.telegram.org/bot{YOUR_BOT_TOKEN}/..."

// HTTP Request (JSONBin)
url: "https://api.jsonbin.io/v3/b/694fd909d0ea881f404356fa/latest"
X-Master-Key: "YOUR_JSONBIN_KEY"

// Upload file
folderId: "1htTAPNhr1q_rnNwoZ0HHeTWALgf-rzoc" // Your folder ID
```

### Step 8: Activate Workflows

1. Activate all three workflows
2. Ensure they're set to "Active" in n8n

### Step 9: Test the System

1. **Upload test video** to Google Drive `n8n-test` folder
2. **Wait for script** to arrive in Telegram (~1-2 minutes)
3. **Record voiceover** using Telegram voice message or upload audio file
4. **Wait for video** rendering (~15-30 seconds)
5. **Download video** from link in Telegram

---

## Troubleshooting

### Common Issues

#### "No resume URL found"
**Problem:** Audio Receiver can't find paused workflow
**Solution:** 
- Upload a new video to start fresh
- Check JSONBin data is being saved correctly
- Verify HTTP Request node has correct bin ID

#### "Workflow no longer waiting"
**Problem:** Workflow state expired or was reused
**Solution:**
- Upload a new video to create new workflow state
- Each video needs its own audio submission

#### Script doesn't match video length
**Problem:** Word count calculation incorrect
**Solution:**
- Check video duration detection in analysis
- Verify script generation prompt uses correct formula
- Adjust word-per-second rate (currently 2.5)

#### Audio file not uploading
**Problem:** Telegram file download fails
**Solution:**
- Verify Telegram bot token in all HTTP nodes
- Check file size (Telegram has 50MB limit for bots)
- Ensure bot has proper permissions

#### Video rendering stuck
**Problem:** Creatomate render not completing
**Solution:**
- Check Creatomate dashboard for render status
- Verify template ID is correct
- Ensure video/audio URLs are accessible
- Check Creatomate account limits

#### Google Drive trigger not firing
**Problem:** Workflow doesn't detect new videos
**Solution:**
- Verify folder ID is correct
- Check OAuth credentials are valid
- Ensure polling is set to "everyMinute"
- Try re-authorizing Google Drive connection

---

## API Keys Reference

| Service | Key Location | Purpose |
|---------|-------------|---------|
| Google Drive OAuth2 | Cloud Console | Video/audio file access |
| Telegram Bot Token | @BotFather | Send/receive messages |
| Creatomate API Key | Creatomate Settings | Video rendering |
| Google Gemini API | AI Studio | Video analysis & script generation |
| JSONBin Master Key | JSONBin Dashboard | Workflow state persistence |

---

## Architecture Decisions

### Why Three Workflows?

1. **Separation of Concerns:** Each workflow handles one responsibility
2. **Independent Scaling:** Can activate/deactivate parts as needed
3. **Error Isolation:** Failures don't cascade across system
4. **Easier Debugging:** Each workflow can be tested independently

### Why JSONBin for State?

- **Persistent Storage:** Survives n8n restarts
- **Simple API:** Easy to read/write JSON data
- **No Database:** No need for separate database setup
- **Public Access:** Can be accessed from any workflow

### Why Wait Node Instead of Webhook?

- **Polling Simplicity:** Easier than setting up webhook callbacks
- **Status Checking:** Can query render status repeatedly
- **No Timeout:** Can wait indefinitely for completion

---

## Customization Options

### Adjust Script Length
```javascript
// In Generate Script node, modify:
words_per_second = 2.5  // Increase for faster speech
max_words = video_duration * words_per_second
```

### Change Script Style
Edit the prompt in "Generate Script" node to:
- Modify tone (formal, casual, enthusiastic)
- Change language style (more Bengali vs more English)
- Adjust emotional intensity
- Add specific vocabulary

### Modify Video Template
In Creatomate dashboard:
- Add text overlays
- Include background music
- Add transitions
- Apply color grading
- Insert intro/outro

### Add More Validation
In "Check if Audio" node:
- Validate file size
- Check audio duration
- Verify file format
- Add user permissions

---

## Performance Considerations

### Expected Processing Times
- Video analysis: 30-60 seconds (depends on video length)
- Script generation: 10-20 seconds
- Audio upload: 5-10 seconds (depends on file size)
- Video rendering: 15-60 seconds (depends on video length)

**Total:** ~2-3 minutes from video upload to final video

### Optimization Tips
1. **Video Size:** Keep source videos under 100MB for faster processing
2. **Audio Format:** OGG format is optimal for Telegram
3. **Polling Interval:** 10 seconds balances responsiveness with API calls
4. **Cleanup:** Periodically clear old JSONBin data
5. **Caching:** Reuse analyzed videos when possible

---

## Security Best Practices

### API Key Management
- Never commit API keys to version control
- Use n8n's credential system for sensitive data
- Rotate keys periodically
- Use environment variables for deployment

### Access Control
- Restrict Google Drive folder permissions
- Use private Telegram bots
- Limit Creatomate template access
- Set JSONBin privacy to private

### Data Privacy
- Clear old workflow executions regularly
- Delete uploaded videos after processing
- Remove temporary audio files
- Don't log sensitive user data

---

## Scaling Considerations

### For High Volume
1. **Multiple Workers:** Run multiple n8n instances
2. **Queue System:** Add queue management for video processing
3. **CDN Storage:** Move from Google Drive to S3/CDN
4. **Database:** Replace JSONBin with PostgreSQL
5. **Load Balancing:** Distribute webhook requests

### For Multiple Users
1. **User Authentication:** Add user ID tracking
2. **Rate Limiting:** Prevent abuse
3. **Storage Quotas:** Limit per-user storage
4. **Billing Integration:** Track usage for billing

---

## Future Enhancements

### Potential Features
- Multiple language support
- Voice cloning integration
- Automatic subtitle generation
- Background music selection
- Video editing capabilities
- Batch processing
- Web dashboard
- Video preview before rendering
- Template selection
- Custom branding options

### Integration Possibilities
- WhatsApp Business API
- Discord bots
- Slack integration
- Email notifications
- Payment processing
- Analytics dashboard
- Social media auto-posting

---

## Support & Resources

### Documentation Links
- [n8n Documentation](https://docs.n8n.io)
- [Creatomate API Docs](https://creatomate.com/docs/api)
- [Telegram Bot API](https://core.telegram.org/bots/api)
- [Google Drive API](https://developers.google.com/drive)
- [Google Gemini API](https://ai.google.dev/docs)

### Community
- n8n Community Forum
- n8n Discord Server
- GitHub Discussions

---

## License & Credits

This workflow system uses:
- **n8n** - Workflow automation
- **Creatomate** - Video rendering
- **Google Gemini** - AI analysis
- **Telegram** - User interface
- **Google Drive** - File storage
- **JSONBin** - State persistence
