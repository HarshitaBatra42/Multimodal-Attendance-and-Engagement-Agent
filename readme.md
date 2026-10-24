# Multimodal Attendance & Engagement Agent


**Demo video:** [Watch on YouTube](https://youtu.be/m9piJEPX8Ts)

A classroom assistant that takes attendance from a camera by recognising enrolled students, estimates how engaged the class is from video and audio signals, saves every session to a database, and suggests teaching improvements.

## Features
- Teacher registration and login
- Classes with a section (for example 10-A) and an attendance type: once a day (school) or every lecture (college)
- Student roster per class: roll number, name and face enrolment from the camera or from photos
- Live dashboard: faces are tracked and each student is identified by name and roll number; a student is marked present only after sustained presence (not a single frame)
- Absent list with names, saved for every session
- Engagement analysis from head pose (facing the front), movement/inactivity and speech activity, with a 0-100 class score every 5 seconds
- Attendance history in SQLite: date and class filters, per-student session details, CSV export, delete to retake
- Teaching suggestions from Google Gemini using a numbers-only summary, with a built-in offline fallback

## Tech stack
Python 3.12, Flask, SQLite, OpenCV (YuNet face detection, SFace face recognition), MediaPipe, sounddevice + NumPy, Google Gemini API (`google-genai`), HTML/CSS/JavaScript.

## How it works
1. **Tracking:** MediaPipe detects faces and a tracker follows each face between frames.
2. **Identification:** for each tracked face, OpenCV SFace turns the face into a 128-number fingerprint and compares it with the enrolled students of the selected class. A student is accepted after several consistent matches. Unmatched faces are shown as Unknown and are not counted.
3. **Attendance:** an identified student is marked present after 2 seconds of continuous presence. Everyone not seen is saved as absent.
4. **Attention:** head yaw and pitch come from MediaPipe face landmarks; facing away or down lowers the score.
5. **Movement:** a person who stays completely still for 30 seconds is flagged inactive.
6. **Audio:** the microphone level is compared with the room's background noise to detect speech.
7. **Engagement score:** 60 points for facing the front + 40 points for not being inactive, averaged over the class every 5 seconds.
8. **Suggestions:** saved numbers (never video, faces or names) go to Gemini when the teacher clicks the button. If Gemini is unavailable, rule-based suggestions are shown.

## Setup
Tested on Windows with Python 3.12.

```bash
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
```

Create a `.env` file (copy `.env.example`) and fill in:
- `SECRET_KEY`: any long random text
- `GEMINI_API_KEY`: free key from https://aistudio.google.com/apikey (optional; without it the built-in suggestions are used)
- `GEMINI_MODEL`: for example `gemini-flash-latest`

Download the two face-recognition models (they are not stored in the repository because of their size):

```bash
curl -L -o models\face_detector_yunet.onnx https://github.com/opencv/opencv_zoo/raw/main/models/face_detection_yunet/face_detection_yunet_2023mar.onnx
curl -L -o models\face_recognizer_sface.onnx https://github.com/opencv/opencv_zoo/raw/main/models/face_recognition_sface/face_recognition_sface_2021dec.onnx
```

## Run
```bash
python app.py
```
Open http://127.0.0.1:5000 and follow these steps:
1. Register a teacher account and log in.
2. Click **+ Add class** (class, section, attendance type).
3. Click **Manage students** and add each student with a roll number, name and several face photos (or use the camera).
4. Back on the dashboard, choose the class and press **Start**. Press **Stop** to save the session.
5. Open **Records** for history, per-student details and CSV downloads.

## Project structure
```
app.py                  Flask routes
modules/
  vision.py             face detection (tracking)
  tracker.py            tracking + sustained-presence rule
  recognition.py        face fingerprints and matching (OpenCV SFace)
  identity.py           who is each tracked face, who is present
  head_pose.py          head yaw/pitch
  attention.py          attentive / looking-away rules
  movement.py           inactivity detection
  audio.py              speech detection
  engagement.py         engagement score + 5-second timeline
  pipeline.py           runs everything in a background thread
  db.py                 SQLite storage
  genai_utils.py        Gemini + offline suggestions
templates/, static/     HTML, CSS, JavaScript
test_audio.py, test_engagement.py   manual test scripts
```

## Privacy
Video frames are processed in memory and never saved. Student photos are not stored: only a numeric face fingerprint is kept, in the local `attendance.db` (never uploaded; the file is excluded from Git). Only a numeric engagement summary is sent to Gemini, and only when the teacher clicks the suggestions button. Using face recognition on students needs the consent of the students or their guardians.

## Limitations
- Recognition accuracy depends on lighting, camera angle, distance and the quality of the enrolment photos. Faces far from the camera are harder to recognise. The match strictness can be tuned with `MATCH_THRESHOLD` in `modules/recognition.py`.
- There is no liveness check, so a photo of a student held up to the camera could be accepted.
- Head-pose thresholds were calibrated on one webcam and may need adjusting (`modules/attention.py`).
- Engagement is a heuristic score, not a validated measure.
- Gemini suggestions need an internet connection; everything else runs locally.

## Future work
Recorded-video upload, liveness check, per-student engagement reports, FastAPI backend, Docker.