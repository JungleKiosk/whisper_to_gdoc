# YouTube Video Transcription Pipeline

This script allows you to download a YouTube video, transcribe its audio using Whisper, and upload the transcription to Google Docs. Below are the steps and code to achieve this.

## Prerequisites

Make sure you have the following dependencies installed:

```sh
!pip install yt-dlp
!pip install git+https://github.com/openai/whisper.git
!pip install --upgrade google-api-python-client google-auth-httplib2 google-auth-oauthlib
!pip install pydub
!apt-get install ffmpeg -y
```

## Authentication with Google

This script uses Google Colab for authentication:

```python
from google.colab import auth
auth.authenticate_user()

import google.auth
from googleapiclient.discovery import build
from googleapiclient.errors import HttpError
from googleapiclient.http import MediaIoBaseUpload
import io

# Define the necessary scopes.
SCOPES = [
    'https://www.googleapis.com/auth/documents',
    'https://www.googleapis.com/auth/drive'
]

# Get the default credentials.
creds, _ = google.auth.default(scopes=SCOPES)

# Create the services for Google Docs and Google Drive.
docs_service = build('docs', 'v1', credentials=creds)
drive_service = build('drive', 'v3', credentials=creds)
```

## Helper Functions

### Get or Create Folder ID

This function will get the folder ID on Google Drive based on the path, creating it if it doesn't exist:

```python
def get_or_create_folder_id_from_path(path):
    """
    Gets the ID of the folder on Google Drive given a path.
    If the folder does not exist, it is created.
    """
    parts = path.strip('/').split('/')
    current_folder_id = 'root'

    for part in parts:
        query = f"name='{part}' and mimeType='application/vnd.google-apps.folder' and '{current_folder_id}' in parents"
        response = drive_service.files().list(q=query, spaces='drive', fields='files(id, name)').execute()
        files = response.get('files', [])
        if not files:
            # Create the folder
            folder_metadata = {
                'name': part,
                'mimeType': 'application/vnd.google-apps.folder',
                'parents': [current_folder_id]
            }
            folder = drive_service.files().create(body=folder_metadata, fields='id').execute()
            current_folder_id = folder.get('id')
            print(f"Folder created: {part} (ID: {current_folder_id})")
        else:
            current_folder_id = files[0]['id']
    return current_folder_id
```

### Upload Transcription to Google Docs

This function uploads the transcription text as a Google Docs document:

```python
def upload_transcription_as_google_doc(content, folder_id, filename='transcription'):
    """
    Creates a new Google Docs document in the specified folder and inserts the contents.
    """
    try:
        # Document metadata
        doc_metadata = {
            'name': filename,
            'mimeType': 'application/vnd.google-apps.document',
            'parents': [folder_id]
        }

        # Upload content as a text file
        media = MediaIoBaseUpload(io.BytesIO(content.encode('utf-8')),
                                  mimetype='text/plain',
                                  resumable=True)

        # Create Google Docs document
        doc = drive_service.files().create(body=doc_metadata,
                                           media_body=media,
                                           fields='id').execute()
        doc_id = doc.get('id')
        print(f"✅ Google Docs document uploaded with ID: {doc_id}")
        print(f"Link to doc: https://docs.google.com/document/d/{doc_id}/edit")
        return doc_id
    except HttpError as error:
        print(f"❌ error: {error}")
        return None
```

## Downloading the Video from YouTube and Transcription

This script uses yt-dlp to download the audio from a YouTube video and then transcribes it using Whisper:

```python
import os
import whisper
from pydub import AudioSegment

# Enter the URL of the YouTube video you wish to transcribe.
youtube_url = 'https://www.youtube.com/watch?v=4oUxPWnrXNk'  # Replace with your URL

# Define the name of the output audio file.
output_audio = 'audio.mp3'

# Download the audio from the YouTube video using yt-dlp
!yt-dlp -x --audio-format mp3 -o "{output_audio}" "{youtube_url}"

# Verify that the audio file has been downloaded.
if os.path.exists(output_audio):
    print(f"✅ Audio file successfully downloaded: {output_audio}")
else:
    print(f"❌ Error downloading audio file: {output_audio}")

# Load the smallest Whisper model for faster processing
model = whisper.load_model("tiny")  # 'tiny' is faster; consider 'base' if you need higher accuracy
```

## Splitting and Transcribing Audio

If the audio is long, this function splits it into 10-minute segments, and then each segment is transcribed:

```python
def split_audio(audio_path, segment_length_ms=600000):  # Segments of 10 minutes
    audio = AudioSegment.from_mp3(audio_path)
    total_length_ms = len(audio)
    segments = []
    
    for i in range(0, total_length_ms, segment_length_ms):
        segment = audio[i:i+segment_length_ms]
        segment_path = f'segment_{i//segment_length_ms + 1}.mp3'
        segment.export(segment_path, format="mp3")
        segments.append(segment_path)
    
    return segments

# Split audio into 10-minute segments (if the audio is very long)
segments = split_audio(output_audio)
print(f"Audio split into {len(segments)} segments.")

# Transcribe each segment and combine transcriptions
transcript_text = ""

for segment in segments:
    transcription = model.transcribe(segment, language='English')  # Modify language if needed
    transcript_text += transcription['text'] + "\n"
    print(f"✅ Transcription completed for {segment}")

print("✅⚡ Full transcription completed.")
```

## Uploading the Transcription to Google Docs

Finally, upload the transcribed text to Google Docs in the specified folder:

```python
# Define the folder path on Google Drive (relative to "My Drive")
folder_path = 'whisper/transcripts'

# Get or create the folder ID
folder_id = get_or_create_folder_id_from_path(folder_path)
if folder_id:
    print(f"✅ Folder ID: {folder_id}")
else:
    print("❌ Unable to get or create folder ID.")

# Define the document title
doc_title = 'transcription'

# Upload the transcription as a Google Docs document
if folder_id:
    doc_id = upload_transcription_as_google_doc(transcript_text, folder_id, filename=doc_title)
    if doc_id:
        print(f"🌳🦊🌳 Google Docs document successfully created: https://docs.google.com/document/d/{doc_id}/edit")
    else:
        print("❌ Unable to create Google Docs document.")
else:
    print("❌ Unable to upload transcription without a valid folder ID.")
```