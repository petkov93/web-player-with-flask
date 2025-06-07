Music Player and Downloader Web App
This document outlines the steps and to-dos for building a Flask-based web application that downloads songs using yt-dlp, processes metadata with mutagen, and plays them via a web player interface (HTML/CSS/JS provided by the user). The app will run locally on your PC and serve as a music downloader and player.
Project Overview

Purpose: Download songs from YouTube (or other supported platforms) using yt-dlp, extract/edit metadata with mutagen, and serve a web player to stream or play downloaded songs.
Tech Stack:
Backend: Python, Flask
Libraries: yt-dlp (download songs), mutagen (handle audio metadata)
Frontend: User-provided HTML/CSS/JS for the web player
Storage: Local file system for downloaded songs


Features:
Search and download songs by URL or query
Display song metadata (title, artist, etc.)
Stream/play downloaded songs via the web player
Basic playlist management (optional, depending on your JS)


Assumptions:
You’ll provide the HTML/CSS/JS for the web player.
The app runs locally on your PC (not deployed to a server).
Songs are stored as MP3 files in a local directory.



Prerequisites

Python 3.8+ installed
pip for installing dependencies
FFmpeg installed (required by yt-dlp for audio extraction)
Basic familiarity with Flask, Python, and web development

Steps and To-Dos
1. Set Up the Environment

 Install Python dependencies:
Create a virtual environment:python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate


Install required packages:pip install flask yt-dlp mutagen




 Install FFmpeg:
Download and install FFmpeg from ffmpeg.org or via a package manager:
Windows: Use choco install ffmpeg (with Chocolatey) or add to PATH manually
macOS: brew install ffmpeg
Linux: sudo apt-get install ffmpeg (Ubuntu/Debian) or equivalent


Verify FFmpeg is accessible: ffmpeg -version


 Create project directory structure:music-player-app/
├── app.py                # Flask app
├── static/              # For CSS, JS, and downloaded songs
│   ├── css/             # Your CSS files
│   ├── js/              # Your JS files
│   └── songs/           # Downloaded MP3s
├── templates/           # HTML templates
│   └── index.html       # Your main web player HTML
└── requirements.txt      # List of dependencies


 Save dependencies:pip freeze > requirements.txt



2. Initialize Flask App

 Create app.py with basic Flask setup:from flask import Flask, render_template, request, jsonify, send_file
import os
import yt_dlp
from mutagen.mp3 import MP3
from mutagen.id3 import ID3, TIT2, TPE1

app = Flask(__name__)
SONGS_DIR = os.path.join('static', 'songs')

# Ensure songs directory exists
os.makedirs(SONGS_DIR, exist_ok=True)

@app.route('/')
def index():
    return render_template('index.html')

if __name__ == '__main__':
    app.run(debug=True)


 Test Flask server:
Run python app.py
Visit http://localhost:5000 to ensure Flask serves your index.html (once added).



3. Implement Song Download with yt-dlp

 Add download route in app.py:@app.route('/download', methods=['POST'])
def download_song():
    url = request.json.get('url')  # Expect YouTube URL from frontend
    if not url:
        return jsonify({'error': 'No URL provided'}), 400

    ydl_opts = {
        'format': 'bestaudio/best',
        'outtmpl': os.path.join(SONGS_DIR, '%(title)s.%(ext)s'),
        'postprocessors': [{
            'key': 'FFmpegExtractAudio',
            'preferredcodec': 'mp3',
            'preferredquality': '192',
        }],
    }

    try:
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=True)
            filename = ydl.prepare_filename(info).replace('.webm', '.mp3').replace('.m4a', '.mp3')
            return jsonify({'filename': os.path.basename(filename), 'title': info.get('title', 'Unknown')})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


 Test download:
Use a tool like Postman or your frontend to send a POST request to /download with a JSON body like {"url": "https://youtube.com/watch?v=..."}.
Check that an MP3 file appears in static/songs/.



4. Handle Metadata with mutagen

 Add metadata processing after download in the /download route:def update_metadata(filepath, title, artist=None):
    audio = MP3(filepath, ID3=ID3)
    audio.add_tags() if audio.tags is None else audio.tags
    audio.tags['TIT2'] = TIT2(encoding=3, text=title)
    if artist:
        audio.tags['TPE1'] = TPE1(encoding=3, text=artist)
    audio.save()


Call update_metadata(filename, info.get('title'), info.get('uploader')) after download in the /download route.


 Create route to list songs and metadata:@app.route('/songs')
def get_songs():
    songs = []
    for file in os.listdir(SONGS_DIR):
        if file.endswith('.mp3'):
            audio = MP3(os.path.join(SONGS_DIR, file), ID3=ID3)
            title = audio.get('TIT2', file).text[0] if audio.get('TIT2') else file
            artist = audio.get('TPE1', 'Unknown').text[0] if audio.get('TPE1') else 'Unknown'
            songs.append({'filename': file, 'title': title, 'artist': artist})
    return jsonify(songs)



5. Serve Songs for Playback

 Add route to stream songs:@app.route('/play/<filename>')
def play_song(filename):
    filepath = os.path.join(SONGS_DIR, filename)
    if os.path.exists(filepath):
        return send_file(filepath, mimetype='audio/mpeg')
    return jsonify({'error': 'File not found'}), 404


 Ensure frontend can access songs:
Your web player (in index.html) should fetch the song list from /songs and stream songs via /play/<filename>.



6. Integrate Frontend

 Add your HTML/CSS/JS:
Place your index.html in templates/.
Place CSS and JS files in static/css/ and static/js/.


 Update frontend to interact with backend:
Use JavaScript (e.g., Fetch API) to:
Send POST requests to /download with a YouTube URL.
Fetch song list from /songs.
Stream songs by setting an <audio> element’s src to /play/<filename>.


Example JS for downloading:async function downloadSong(url) {
    const response = await fetch('/download', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url })
    });
    const data = await response.json();
    if (data.error) {
        console.error(data.error);
    } else {
        console.log('Downloaded:', data.title);
    }
}




 Test frontend-backend integration:
Ensure your web player can list and play songs.



7. Test and Debug

 Run the app:
python app.py
Open http://localhost:5000 in your browser.


 Test key features:
Download a song via your frontend or Postman.
Verify metadata (title, artist) in the downloaded MP3.
Play a song through the web player.


 Check for errors:
Handle invalid URLs or failed downloads gracefully.
Ensure FFmpeg is correctly configured for yt-dlp.



8. Optional Enhancements (To-Dos if Desired)

 Add search functionality:
Use yt-dlp to search YouTube (e.g., ytsearch:query) and return results.


 Support playlists:
Allow users to create/save playlists in a JSON file or database.


 Improve metadata:
Extract more metadata (album, genre) or allow manual editing via a form.


 Add authentication:
Restrict access to the app with Flask-Login or basic auth.


 Optimize performance:
Cache song metadata to avoid repeated mutagen reads.
Implement async downloads with threading or asyncio.



9. Deployment (Optional)

 Run locally:
Use python app.py for local use on your PC.


 Consider deployment (if needed later):
Deploy to Heroku, Render, or a local server (e.g., Gunicorn + Nginx).
Update SONGS_DIR to a persistent storage path.


 Secure static files:
Ensure /play/<filename> checks for valid files to prevent unauthorized access.



Notes

Frontend Integration: Since you’re providing the HTML/CSS/JS, ensure your web player can handle dynamic song lists and streaming. I can help debug or tweak the JS if needed.
Legal Note: Downloading copyrighted content via yt-dlp may violate platform terms or local laws. Use responsibly for non-copyrighted or permitted content.
Next Steps: Share your HTML/CSS/JS files or describe their functionality (e.g., how the player handles song lists) if you need specific integration help.

Sample Workflow

User enters a YouTube URL in the web interface.
Frontend sends URL to /download.
Backend downloads the song as MP3, updates metadata, and saves it to static/songs/.
Frontend fetches song list from /songs and displays titles/artists.
User clicks a song, and the web player streams it via /play/<filename>.

Troubleshooting

FFmpeg errors: Ensure FFmpeg is in your system PATH.
Download failures: Check yt-dlp logs for URL or network issues.
Playback issues: Verify your <audio> element supports MP3 and the /play route works.
Metadata issues: Ensure mutagen correctly reads/writes ID3 tags.

