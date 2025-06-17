# 7
File
import os
import requests
from pydub.generators import Sine
from pydub import AudioSegment

MUBERT_API_KEY = os.getenv("MUBERT_API_KEY")
MUBERT_EMAIL = os.getenv("MUBERT_EMAIL")

def generate_music_with_pydub(path="assets/generated_music.mp3", duration=10):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    tone1 = Sine(440).to_audio_segment(duration=500)
    tone2 = Sine(554).to_audio_segment(duration=500)
    tone3 = Sine(660).to_audio_segment(duration=500)
    silence = AudioSegment.silent(duration=250)
    sequence = tone1 + silence + tone2 + silence + tone3 + silence
    full_music = sequence * (duration * 1000 // len(sequence))
    full_music.export(path, format="mp3")
    return path

def generate_music_with_ai(style="ambient", duration=30, path="assets/generated_music.mp3"):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    if not MUBERT_API_KEY or not MUBERT_EMAIL:
        raise ValueError("Missing Mubert credentials")
    payload = {
        "email": MUBERT_EMAIL,
        "token": MUBERT_API_KEY,
        "mode": "track",
        "genre": style,
        "duration": duration,
        "format": "mp3"
    }
    r = requests.post("https://api.mubert.com/v2/Generate", json=payload)
    r.raise_for_status()
    audio_url = r.json()["data"]["track_url"]
    content = requests.get(audio_url).content
    with open(path, "wb") as f:
        f.write(content)
    return path

def generate_background_music(duration=30, style="ambient", use_ai=True, path="assets/generated_music.mp3"):
    try:
        if use_ai:
            return generate_music_with_ai(style=style, duration=duration, path=path)
        raise ValueError("AI disabled")
    except Exception as e:
        print("⚠️ Fallback:", e)
        return generate_music_with_pydub(path=path, duration=duration)from moviepy.editor import VideoFileClip, AudioFileClip, CompositeAudioClip, vfx
from app.utils.music_generator import generate_background_music

def apply_edits(input_path: str, output_path: str, instructions: dict):
    clip = VideoFileClip(input_path)
    if "speed" in instructions:
        clip = clip.fx(vfx.speedx, instructions["speed"])
    if instructions.get("filter") == "grayscale":
        clip = clip.fx(vfx.blackwhite)
    mood = instructions.get("mood", "ambient")
    use_ai = instructions.get("use_ai_music", True)
    music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=use_ai)
    if music_path:
        audio = AudioFileClip(music_path).set_duration(clip.duration)
        new_audio = CompositeAudioClip([clip.audio, audio]) if clip.audio else audio
        clip = clip.set_audio(new_audio)
    clip.write_videofile(output_path, codec="libx264", audio_codec="aac")from app.prompt_parser import parse_prompt
from app.video_generator import generate_video_with_pixverse
from app.video_editor import apply_edits

def process_prompt_and_generate_video(prompt: str, resolution: str, output_path: str, use_ai_music: bool = True):
    instructions = parse_prompt(prompt)
    instructions["use_ai_music"] = use_ai_music
    video_path = generate_video_with_pixverse(prompt, resolution)
    apply_edits(video_path, output_path, instructions)
    return output_pathimport os
import uuid
import asyncio
from fastapi import FastAPI, HTTPException
from fastapi.staticfiles import StaticFiles
from pydantic import BaseModel
from app.services.video_service import process_prompt_and_generate_video

app = FastAPI()
os.makedirs("output", exist_ok=True)

class VideoRequest(BaseModel):
    prompt: str
    resolution: str
    use_ai_music: bool = True

running_jobs = {}

@app.post("/generate")
async def generate_video_endpoint(data: VideoRequest):
    job_id = str(uuid.uuid4())
    output_path = f"output/final_video_{job_id}.mp4"
    task = asyncio.create_task(
        generate_video_job(data, output_path, job_id)
    )
    running_jobs[job_id] = task
    return {"job_id": job_id}

async def generate_video_job(data, output_path, job_id):
    try:
        await asyncio.to_thread(
            process_prompt_and_generate_video,
            data.prompt,
            data.resolution,
            output_path,
            data.use_ai_music
        )
    except asyncio.CancelledError:
        print(f"🛑 Job {job_id} cancelled")
    finally:
        running_jobs.pop(job_id, None)

@app.post("/cancel/{job_id}")
async def cancel_video_job(job_id: str):
    task = running_jobs.get(job_id)
    if task and not task.done():
        task.cancel()
        return {"status": "cancelled"}
    raise HTTPException(status_code=404, detail="Job not found")

@app.get("/status/{job_id}")
async def job_status(job_id: str):
    exists = job_id in running_jobs
    return {"status": "running" if exists else "done"}

app.mount("/static", StaticFiles(directory="output"), name="static")uvicorn app.main:app --host 0.0.0.0 --port 10000http.post(Uri.parse('http://192.168.x.x:10000/generate'))PYTHONPATH=. uvicorn app.main:app --host 0.0.0.0 --port 10000video_ai_backend/ ├── app/ │   ├── main.py  ← contains `app = FastAPI()`flutter emulators --launch android flutter runflutter build apk --releasestart: uvicorn app.main:app --host 0.0.0.0 --port 10000Column(   children: [     TextField(controller: _promptController, decoration: InputDecoration(labelText: "Describe your video")),     SwitchListTile(       title: Text("Use AI-Generated Music"),       value: _useAIMusic,       onChanged: (val) => setState(() => _useAIMusic = val),     ),     ElevatedButton(       onPressed: _isLoading ? null : _startGeneration,       child: Text("Generate Video"),     ),     if (_isLoading) ...[       CircularProgressIndicator(),       ElevatedButton(onPressed: _cancelGeneration, child: Text("Cancel")),     ],     if (_videoUrl != null) ...[       Text("Your video is ready!"),       ElevatedButton(         onPressed: () => launchUrl(Uri.parse(_videoUrl!)),         child: Text("Watch Video"),       )     ]   ], )bool _isLoading = false; bool _useAIMusic = true; String? _jobId; String? _videoUrl; final TextEditingController _promptController = TextEditingController();  void _startGeneration() async {   setState(() => _isLoading = true);   final response = await http.post(     Uri.parse('https://your-server.com/generate'),     headers: {'Content-Type': 'application/json'},     body: jsonEncode({       "prompt": _promptController.text,       "resolution": "720p",       "use_ai_music": _useAIMusic,     }),   );    final data = jsonDecode(response.body);   _jobId = data["job_id"];   _pollForCompletion(_jobId!); }  void _pollForCompletion(String jobId) async {   while (true) {     await Future.delayed(Duration(seconds: 3));     final statusRes = await http.get(Uri.parse('https://your-server.com/status/$jobId'));     if (jsonDecode(statusRes.body)['status'] == "done") {       setState(() {         _videoUrl = "https://your-server.com/static/final_video_$jobId.mp4";         _isLoading = false;       });       break;     }   } }  void _cancelGeneration() async {   if (_jobId != null) {     await http.post(Uri.parse('https://your-server.com/cancel/$_jobId'));     setState(() {       _isLoading = false;       _jobId = null;     });   } }import os, requests from pydub import AudioSegment from pydub.generators import Sine  MUBERT_API_KEY = os.getenv("MUBERT_API_KEY") MUBERT_EMAIL = os.getenv("MUBERT_EMAIL")  def generate_music_with_pydub(path="assets/generated_music.mp3", duration=10):     os.makedirs(os.path.dirname(path), exist_ok=True)     tone = (Sine(440).to_audio_segment(500) + AudioSegment.silent(250)) * (duration * 1000 // 750)     tone.export(path, format="mp3")     return path  def generate_music_with_ai(style="ambient", duration=30, path="assets/generated_music.mp3"):     os.makedirs(os.path.dirname(path), exist_ok=True)     if not MUBERT_API_KEY or not MUBERT_EMAIL:         raise ValueError("Missing Mubert credentials")      payload = {         "email": MUBERT_EMAIL,         "token": MUBERT_API_KEY,         "mode": "track",         "genre": style,         "duration": duration,         "format": "mp3"     }      r = requests.post("https://api.mubert.com/v2/Generate", json=payload)     r.raise_for_status()     audio_url = r.json()["data"]["track_url"]     content = requests.get(audio_url).content      with open(path, "wb") as f:         f.write(content)      return path  def generate_background_music(duration=30, style="ambient", use_ai=True, path="assets/generated_music.mp3"):     try:         if use_ai:             return generate_music_with_ai(style=style, duration=duration, path=path)         raise ValueError("AI disabled")     except Exception as e:         print("⚠️ Fallback:", e)         return generate_music_with_pydub(path=path, duration=duration)from moviepy.editor import VideoFileClip, AudioFileClip, CompositeAudioClip, vfx from app.utils.music_generator import generate_background_music  def apply_edits(input_path: str, output_path: str, instructions: dict):     clip = VideoFileClip(input_path)      if "speed" in instructions:         clip = clip.fx(vfx.speedx, instructions["speed"])      if instructions.get("filter") == "grayscale":         clip = clip.fx(vfx.blackwhite)      mood = instructions.get("mood", "ambient")     use_ai = instructions.get("use_ai_music", True)     music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=use_ai)      if music_path:         audio = AudioFileClip(music_path).set_duration(clip.duration)         new_audio = CompositeAudioClip([clip.audio, audio]) if clip.audio else audio         clip = clip.set_audio(new_audio)      clip.write_videofile(output_path, codec="libx264", audio_codec="aac")from app.prompt_parser import parse_prompt from app.video_generator import generate_video_with_pixverse from app.video_editor import apply_edits  def process_prompt_and_generate_video(prompt: str, resolution: str, output_path: str, use_ai_music=True):     instructions = parse_prompt(prompt)     instructions["use_ai_music"] = use_ai_music      video_path = generate_video_with_pixverse(prompt, resolution)     apply_edits(video_path, output_path, instructions)      return output_pathfrom fastapi import FastAPI, HTTPException from fastapi.staticfiles import StaticFiles from pydantic import BaseModel from app.services.video_service import process_prompt_and_generate_video import uuid import os import asyncio  app = FastAPI() os.makedirs("output", exist_ok=True)  class VideoRequest(BaseModel):     prompt: str     resolution: str     use_ai_music: bool = True  running_jobs = {}  @app.post("/generate") async def generate_video_endpoint(data: VideoRequest):     job_id = str(uuid.uuid4())     output_path = f"output/final_video_{job_id}.mp4"      task = asyncio.create_task(         generate_video_job(data, output_path, job_id)     )     running_jobs[job_id] = task      return {"job_id": job_id}  async def generate_video_job(data, output_path, job_id):     try:         await asyncio.to_thread(             process_prompt_and_generate_video,             data.prompt,             data.resolution,             output_path,             data.use_ai_music         )     except asyncio.CancelledError:         print(f"🛑 Job {job_id} cancelled")     finally:         running_jobs.pop(job_id, None)  @app.post("/cancel/{job_id}") async def cancel_video_job(job_id: str):     task = running_jobs.get(job_id)     if task and not task.done():         task.cancel()         return {"status": "cancelled"}     raise HTTPException(status_code=404, detail="Job not found")  @app.get("/status/{job_id}") async def job_status(job_id: str):     exists = job_id in running_jobs     return {"status": "running" if exists else "done"}  app.mount("/static", StaticFiles(directory="output"), name="static")video_ai_backend/ ├── app/ │   ├── main.py                    ✅ FastAPI app + endpoints │   ├── services/ │   │   ├── video_service.py       ✅ Generation pipeline │   │   ├── video_editor.py        ✅ MoviePy-based edits │   ├── utils/ │   │   └── music_generator.py     ✅ AI & Pydub fallback │   ├── prompt_parser.py           ✅ Extracts instructions from natural prompt │   └── video_generator.py         ✅ PixVerse API integration ├── output/                        ✅ Final MP4sfinal response = await http.post( ... ); final jobId = jsonDecode(response.body)["job_id"]; setState(() => _jobId = jobId);String? _jobId;  ElevatedButton(   onPressed: _isLoading ? _cancelGeneration : null,   child: Text("Cancel"), ),  void _cancelGeneration() async {   if (_jobId != null) {     final response = await http.post(       Uri.parse('https://your-server.com/cancel/$_jobId'),     );     print(response.body);      setState(() {       _isLoading = false;       _jobId = null;     });   } }from fastapi import HTTPException  @app.post("/cancel/{job_id}") async def cancel_video_job(job_id: str):     task = running_jobs.get(job_id)     if task and not task.done():         task.cancel()         return {"status": "cancelled"}     raise HTTPException(status_code=404, detail="Job not found or already finished")async def generate_video_job(data, output_path, job_id):     try:         final_path = await asyncio.to_thread(             process_prompt_and_generate_video,             data.prompt,             data.resolution,             output_path,             data.use_ai_music         )     except asyncio.CancelledError:         print(f"🛑 Job {job_id} was cancelled")         return     finally:         running_jobs.pop(job_id, None)from fastapi import BackgroundTasks import uuid  @app.post("/generate") async def generate_video_endpoint(data: VideoRequest):     job_id = str(uuid.uuid4())     output_path = f"output/final_video_{job_id}.mp4"      task = asyncio.create_task(         generate_video_job(data, output_path, job_id)     )     running_jobs[job_id] = task      return {"job_id": job_id}import asyncio  # Maps job_id -> asyncio.Task running_jobs = {}bool _isLoading = false; String? _videoUrl;use_ai = instructions.get("use_ai_music", True) music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=use_ai)def process_prompt_and_generate_video(prompt: str, resolution: str, output_path: str, use_ai_music: bool = True):     instructions = parse_prompt(prompt)     instructions["use_ai_music"] = use_ai_music  # pass this to editor      video_path = generate_video_with_pixverse(prompt, resolution)     apply_edits(video_path, output_path, instructions)      return output_path@app.post("/generate") async def generate_video_endpoint(data: VideoRequest):     os.makedirs("output", exist_ok=True)     output_path = f"output/final_video_{uuid.uuid4()}.mp4"      final_path = process_prompt_and_generate_video(         data.prompt,         data.resolution,         output_path,         use_ai_music=data.use_ai_music     )      video_url = f"http://your-server.com/static/{os.path.basename(final_path)}"     return {"video_url": video_url}class VideoRequest(BaseModel):     prompt: str     resolution: str     use_ai_music: bool = True  # default to AIstatic Future<String> generateAndEditVideo(String prompt, String resolution, bool useAIMusic) async {   final response = await http.post(     Uri.parse('https://your-server.com/generate'),     headers: {'Content-Type': 'application/json'},     body: jsonEncode({       "prompt": prompt,       "resolution": resolution,       "use_ai_music": useAIMusic,     }),   );    if (response.statusCode == 200) {     return jsonDecode(response.body)["video_url"];   } else {     throw Exception("Video generation failed: ${response.body}");   } }bool useAIMusic = true;  SwitchListTile(   title: Text('Use AI-Generated Music'),   value: useAIMusic,   onChanged: (val) {     setState(() {       useAIMusic = val;     });   }, ),use_ai = os.getenv("USE_AI_MUSIC", "true").lower() == "true"USE_AI_MUSIC=truefrom app.utils.music_generator import generate_background_music  def apply_edits(input_path: str, output_path: str, instructions: dict):     clip = VideoFileClip(input_path)      # Speed change     if "speed" in instructions:         clip = clip.fx(vfx.speedx, instructions["speed"])      # Grayscale filter     if instructions.get("filter") == "grayscale":         clip = clip.fx(vfx.blackwhite)      # Music     music_path = instructions.get("music")     if not music_path:         mood = instructions.get("mood", "ambient")         music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=True)      if music_path:         audio = AudioFileClip(music_path).set_duration(clip.duration)         new_audio = CompositeAudioClip([clip.audio, audio]) if clip.audio else audio         clip = clip.set_audio(new_audio)      clip.write_videofile(output_path, codec="libx264", audio_codec="aac")from app.utils.music_generator import generate_background_music  def apply_edits(input_path: str, output_path: str, instructions: dict):     clip = VideoFileClip(input_path)      # Speed change     if "speed" in instructions:         clip = clip.fx(vfx.speedx, instructions["speed"])      # Grayscale filter     if instructions.get("filter") == "grayscale":         clip = clip.fx(vfx.blackwhite)      # Music     music_path = instructions.get("music")     if not music_path:         mood = instructions.get("mood", "ambient")         music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=True)      if music_path:         audio = AudioFileClip(music_path).set_durationimport os import requests from pydub.generators import Sine from pydub import AudioSegment  MUBERT_API_KEY = os.getenv("MUBERT_API_KEY") MUBERT_EMAIL = os.getenv("MUBERT_EMAIL")  def generate_music_with_pydub(path="assets/generated_music.mp3", duration=10):     os.makedirs(os.path.dirname(path), exist_ok=True)     tone1 = Sine(440).to_audio_segment(duration=500)     tone2 = Sine(554).to_audio_segment(duration=500)     tone3 = Sine(660).to_audio_segment(duration=500)     silence = AudioSegment.silent(duration=250)     sequence = tone1 + silence + tone2 + silence + tone3 + silence     full_music = sequence * (duration * 1000 // len(sequence))     full_music.export(path, format="mp3")     return path  def generate_music_with_ai(style="ambient", duration=30, path="assets/generated_music.mp3"):     os.makedirs(os.path.dirname(path), exist_ok=True)      if not MUBERT_API_KEY or not MUBERT_EMAIL:         raise ValueError("MUBERT credentials not found")      payload = {         "email": MUBERT_EMAIL,         "token": MUBERT_API_KEY,         "mode": "track",         "genre": style,         "duration": duration,         "format": "mp3"     }      response = requests.post("https://api.mubert.com/v2/Generate", json=payload)     response.raise_for_status()     music_url = response.json()["data"]["track_url"]     music_data = requests.get(music_url)      with open(path, "wb") as f:         f.write(music_data.content)      return path  def generate_background_music(duration=30, style="ambient", use_ai=True, path="assets/generated_music.mp3"):     try:         if use_ai:             print("🎼 Generating music via Mubert AI...")             return generate_music_with_ai(style=style, duration=duration, path=path)         else:             raise ValueError("AI music generation disabled or unavailable.")     except Exception as e:         print("⚠️ AI music generation failed, falling back to simple tones:", e)         return generate_music_with_pydub(path=path, duration=duration)import os import requests from pydub.generators import Sine from pydub import AudioSegment  MUBERT_API_KEY = os.getenv("MUBERT_API_KEY") MUBERT_EMAIL = os.getenv("MUBERT_EMAIL")  def generate_music_with_pydub(path="assets/generated_music.mp3", duration=10):     os.makedirs(os.path.dirname(path), existgenre = instructions.get("mood", "ambient") music_path = generate_music_with_ai(genre, duration=int(clip.duration)){   "filter": "grayscale",   "speed": 0.5,   "mood": "lofi" }genre = instructions.get("mood", "ambient")  # extract from GPT maybe music_path = generate_music_with_ai(genre, duration=int(clip.duration))# Auto-generate music if not specified if "music" in instructions and not music_path:     music_path = generate_music_with_ai("ambient", duration=int(clip.duration)) elif "music" not in instructions:     music_path = generate_music_with_ai("ambient", duration=int(clip.duration))MUBERT_API_KEY=your-api-key MUBERT_EMAIL=your@email.comimport requests import os from pydub import AudioSegment from pathlib import Path  MUBERT_API_KEY = os.getenv("MUBERT_API_KEY") MUBERT_EMAIL = os.getenv("MUBERT_EMAIL")  def generate_music_with_ai(style: str = "ambient", duration: int = 30, path: str = "assets/generated_music.mp3") -> str:     os.makedirs(os.path.dirname(path), exist_ok=True)      payload = {         "email": MUBERT_EMAIL,         "token": MUBERT_API_KEY,         "mode": "track",         "genre": style,         "duration": duration,         "format": "mp3"     }      response = requests.post("https://api.mubert.com/v2/Generate", json=payload)     response.raise_for_status()      music_url = response.json()["data"]["track_url"]     music_data = requests.get(music_url)      with open(path, "wb") as f:         f.write(music_data.content)      return pathimport requests import os from pydub import AudioSegment from pathlib import Path  MUBERT_API_KEY = os.getenv("MUBERT_API_KEY") MUBERT_EMAIL = os{   "data": {     "track_url": "https://cdn.mubert.com/path/to/track.mp3"   } }flutter emulators --launch android flutter runflutter build apk --releasestart: uvicorn app.main:app --host 0.0.0.0 --port 10000Column(   children: [     TextField(controller: _promptController, decoration: InputDecoration(labelText: "Describe your video")),     SwitchListTile(       title: Text("Use AI-Generated Music"),       value: _useAIMusic,       onChanged: (val) => setState(() => _useAIMusic = val),     ),     ElevatedButton(       onPressed: _isLoading ? null : _startGeneration,       child: Text("Generate Video"),     ),     if (_isLoading) ...[       CircularProgressIndicator(),       ElevatedButton(onPressed: _cancelGeneration, child: Text("Cancel")),     ],     if (_videoUrl != null) ...[       Text("Your video is ready!"),       ElevatedButton(         onPressed: () => launchUrl(Uri.parse(_videoUrl!)),         child: Text("Watch Video"),       )     ]   ], )bool _isLoading = false; bool _useAIMusic = true; String? _jobId; String? _videoUrl; final TextEditingController _promptController = TextEditingController();  void _startGeneration() async {   setState(() => _isLoading = true);   final response = await http.post(     Uri.parse('https://your-server.com/generate'),     headers: {'Content-Type': 'application/json'},     body: jsonEncode({       "prompt": _promptController.text,       "resolution": "720p",       "use_ai_music": _useAIMusic,     }),   );    final data = jsonDecode(response.body);   _jobId = data["job_id"];   _pollForCompletion(_jobId!); }  void _pollForCompletion(String jobId) async {   while (true) {     await Future.delayed(Duration(seconds: 3));     final statusRes = await http.get(Uri.parse('https://your-server.com/status/$jobId'));     if (jsonDecode(statusRes.body)['status'] == "done") {       setState(() {         _videoUrl = "https://your-server.com/static/final_video_$jobId.mp4";         _isLoading = false;       });       break;     }   } }  void _cancelGeneration() async {   if (_jobId != null) {     await http.post(Uri.parse('https://your-server.com/cancel/$_jobId'));     setState(() {       _isLoading = false;       _jobId = null;     });   } }import os, requests from pydub import AudioSegment from pydub.generators import Sine  MUBERT_API_KEY = os.getenv("MUBERT_API_KEY") MUBERT_EMAIL = os.getenv("MUBERT_EMAIL")  def generate_music_with_pydub(path="assets/generated_music.mp3", duration=10):     os.makedirs(os.path.dirname(path), exist_ok=True)     tone = (Sine(440).to_audio_segment(500) + AudioSegment.silent(250)) * (duration * 1000 // 750)     tone.export(path, format="mp3")     return path  def generate_music_with_ai(style="ambient", duration=30, path="assets/generated_music.mp3"):     os.makedirs(os.path.dirname(path), exist_ok=True)     if not MUBERT_API_KEY or not MUBERT_EMAIL:         raise ValueError("Missing Mubert credentials")      payload = {         "email": MUBERT_EMAIL,         "token": MUBERT_API_KEY,         "mode": "track",         "genre": style,         "duration": duration,         "format": "mp3"     }      r = requests.post("https://api.mubert.com/v2/Generate", json=payload)     r.raise_for_status()     audio_url = r.json()["data"]["track_url"]     content = requests.get(audio_url).content      with open(path, "wb") as f:         f.write(content)      return path  def generate_background_music(duration=30, style="ambient", use_ai=True, path="assets/generated_music.mp3"):     try:         if use_ai:             return generate_music_with_ai(style=style, duration=duration, path=path)         raise ValueError("AI disabled")     except Exception as e:         print("⚠️ Fallback:", e)         return generate_music_with_pydub(path=path, duration=duration)from moviepy.editor import VideoFileClip, AudioFileClip, CompositeAudioClip, vfx from app.utils.music_generator import generate_background_music  def apply_edits(input_path: str, output_path: str, instructions: dict):     clip = VideoFileClip(input_path)      if "speed" in instructions:         clip = clip.fx(vfx.speedx, instructions["speed"])      if instructions.get("filter") == "grayscale":         clip = clip.fx(vfx.blackwhite)      mood = instructions.get("mood", "ambient")     use_ai = instructions.get("use_ai_music", True)     music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=use_ai)      if music_path:         audio = AudioFileClip(music_path).set_duration(clip.duration)         new_audio = CompositeAudioClip([clip.audio, audio]) if clip.audio else audio         clip = clip.set_audio(new_audio)      clip.write_videofile(output_path, codec="libx264", audio_codec="aac")from app.prompt_parser import parse_prompt from app.video_generator import generate_video_with_pixverse from app.video_editor import apply_edits  def process_prompt_and_generate_video(prompt: str, resolution: str, output_path: str, use_ai_music=True):     instructions = parse_prompt(prompt)     instructions["use_ai_music"] = use_ai_music      video_path = generate_video_with_pixverse(prompt, resolution)     apply_edits(video_path, output_path, instructions)      return output_pathfrom fastapi import FastAPI, HTTPException from fastapi.staticfiles import StaticFiles from pydantic import BaseModel from app.services.video_service import process_prompt_and_generate_video import uuid import os import asyncio  app = FastAPI() os.makedirs("output", exist_ok=True)  class VideoRequest(BaseModel):     prompt: str     resolution: str     use_ai_music: bool = True  running_jobs = {}  @app.post("/generate") async def generate_video_endpoint(data: VideoRequest):     job_id = str(uuid.uuid4())     output_path = f"output/final_video_{job_id}.mp4"      task = asyncio.create_task(         generate_video_job(data, output_path, job_id)     )     running_jobs[job_id] = task      return {"job_id": job_id}  async def generate_video_job(data, output_path, job_id):     try:         await asyncio.to_thread(             process_prompt_and_generate_video,             data.prompt,             data.resolution,             output_path,             data.use_ai_music         )     except asyncio.CancelledError:         print(f"🛑 Job {job_id} cancelled")     finally:         running_jobs.pop(job_id, None)  @app.post("/cancel/{job_id}") async def cancel_video_job(job_id: str):     task = running_jobs.get(job_id)     if task and not task.done():         task.cancel()         return {"status": "cancelled"}     raise HTTPException(status_code=404, detail="Job not found")  @app.get("/status/{job_id}") async def job_status(job_id: str):     exists = job_id in running_jobs     return {"status": "running" if exists else "done"}  app.mount("/static", StaticFiles(directory="output"), name="static")video_ai_backend/ ├── app/ │   ├── main.py                    ✅ FastAPI app + endpoints │   ├── services/ │   │   ├── video_service.py       ✅ Generation pipeline │   │   ├── video_editor.py        ✅ MoviePy-based edits │   ├── utils/ │   │   └── music_generator.py     ✅ AI & Pydub fallback │   ├── prompt_parser.py           ✅ Extracts instructions from natural prompt │   └── video_generator.py         ✅ PixVerse API integration ├── output/                        ✅ Final MP4sfinal response = await http.post( ... ); final jobId = jsonDecode(response.body)["job_id"]; setState(() => _jobId = jobId);String? _jobId;  ElevatedButton(   onPressed: _isLoading ? _cancelGeneration : null,   child: Text("Cancel"), ),  void _cancelGeneration() async {   if (_jobId != null) {     final response = await http.post(       Uri.parse('https://your-server.com/cancel/$_jobId'),     );     print(response.body);      setState(() {       _isLoading = false;       _jobId = null;     });   } }from fastapi import HTTPException  @app.post("/cancel/{job_id}") async def cancel_video_job(job_id: str):     task = running_jobs.get(job_id)     if task and not task.done():         task.cancel()         return {"status": "cancelled"}     raise HTTPException(status_code=404, detail="Job not found or already finished")async def generate_video_job(data, output_path, job_id):     try:         final_path = await asyncio.to_thread(             process_prompt_and_generate_video,             data.prompt,             data.resolution,             output_path,             data.use_ai_music         )     except asyncio.CancelledError:         print(f"🛑 Job {job_id} was cancelled")         return     finally:         running_jobs.pop(job_id, None)from fastapi import BackgroundTasks import uuid  @app.post("/generate") async def generate_video_endpoint(data: VideoRequest):     job_id = str(uuid.uuid4())     output_path = f"output/final_video_{job_id}.mp4"      task = asyncio.create_task(         generate_video_job(data, output_path, job_id)     )     running_jobs[job_id] = task      return {"job_id": job_id}import asyncio  # Maps job_id -> asyncio.Task running_jobs = {}bool _isLoading = false; String? _videoUrl;use_ai = instructions.get("use_ai_music", True) music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=use_ai)def process_prompt_and_generate_video(prompt: str, resolution: str, output_path: str, use_ai_music: bool = True):     instructions = parse_prompt(prompt)     instructions["use_ai_music"] = use_ai_music  # pass this to editor      video_path = generate_video_with_pixverse(prompt, resolution)     apply_edits(video_path, output_path, instructions)      return output_path@app.post("/generate") async def generate_video_endpoint(data: VideoRequest):     os.makedirs("output", exist_ok=True)     output_path = f"output/final_video_{uuid.uuid4()}.mp4"      final_path = process_prompt_and_generate_video(         data.prompt,         data.resolution,         output_path,         use_ai_music=data.use_ai_music     )      video_url = f"http://your-server.com/static/{os.path.basename(final_path)}"     return {"video_url": video_url}class VideoRequest(BaseModel):     prompt: str     resolution: str     use_ai_music: bool = True  # default to AIstatic Future<String> generateAndEditVideo(String prompt, String resolution, bool useAIMusic) async {   final response = await http.post(     Uri.parse('https://your-server.com/generate'),     headers: {'Content-Type': 'application/json'},     body: jsonEncode({       "prompt": prompt,       "resolution": resolution,       "use_ai_music": useAIMusic,     }),   );    if (response.statusCode == 200) {     return jsonDecode(response.body)["video_url"];   } else {     throw Exception("Video generation failed: ${response.body}");   } }bool useAIMusic = true;  SwitchListTile(   title: Text('Use AI-Generated Music'),   value: useAIMusic,   onChanged: (val) {     setState(() {       useAIMusic = val;     });   }, ),use_ai = os.getenv("USE_AI_MUSIC", "true").lower() == "true"USE_AI_MUSIC=truefrom app.utils.music_generator import generate_background_music  def apply_edits(input_path: str, output_path: str, instructions: dict):     clip = VideoFileClip(input_path)      # Speed change     if "speed" in instructions:         clip = clip.fx(vfx.speedx, instructions["speed"])      # Grayscale filter     if instructions.get("filter") == "grayscale":         clip = clip.fx(vfx.blackwhite)      # Music     music_path = instructions.get("music")     if not music_path:         mood = instructions.get("mood", "ambient")         music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=True)      if music_path:         audio = AudioFileClip(music_path).set_duration(clip.duration)         new_audio = CompositeAudioClip([clip.audio, audio]) if clip.audio else audio         clip = clip.set_audio(new_audio)      clip.write_videofile(output_path, codec="libx264", audio_codec="aac")from app.utils.music_generator import generate_background_music  def apply_edits(input_path: str, output_path: str, instructions: dict):     clip = VideoFileClip(input_path)      # Speed change     if "speed" in instructions:         clip = clip.fx(vfx.speedx, instructions["speed"])      # Grayscale filter     if instructions.get("filter") == "grayscale":         clip = clip.fx(vfx.blackwhite)      # Music     music_path = instructions.get("music")     if not music_path:         mood = instructions.get("mood", "ambient")         music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=True)      if music_path:         audio = AudioFileClip(music_path).set_durationimport os import requests from pydub.generators import Sine from pydub import AudioSegment  MUBERT_API_KEY = os.getenv("MUBERT_API_KEY") MUBERT_EMAIL = os.getenv("MUBERT_EMAIL")  def generate_music_with_pydub(path="assets/generated_music.mp3", duration=10):     os.makedirs(os.path.dirname(path), exist_ok=True)     tone1 = Sine(440).to_audio_segment(duration=500)     tone2 = Sine(554).to_audio_segment(duration=500)     tone3 = Sine(660).to_audio_segment(duration=500)     silence = AudioSegment.silent(duration=250)     sequence = tone1 + silence + tone2 + silence + tone3 + silence     full_music = sequence * (duration * 1000 // len(sequence))     full_music.export(path, format="mp3")     return path  def generate_music_with_ai(style="ambient", duration=30, path="assets/generated_music.mp3"):     os.makedirs(os.path.dirname(path), exist_ok=True)      if not MUBERT_API_KEY or not MUBERT_EMAIL:         raise ValueError("MUBERT credentials not found")      payload = {         "email": MUBERT_EMAIL,         "token": MUBERT_API_KEY,         "mode": "track",         "genre": style,         "duration": duration,         "format": "mp3"     }      response = requests.post("https://api.mubert.com/v2/Generate", json=payload)     response.raise_for_status()     music_url = response.json()["data"]["track_url"]     music_data = requests.get(music_url)      with open(path, "wb") as f:         f.write(music_data.content)      return path  def generate_background_music(duration=30, style="ambient", use_ai=True, path="assets/generated_music.mp3"):     try:         if use_ai:             print("🎼 Generating music via Mubert AI...")             return generate_music_with_ai(style=style, duration=duration, path=path)         else:             raise ValueError("AI music generation disabled or unavailable.")     except Exception as e:         print("⚠️ AI music generation failed, falling back to simple tones:", e)         return generate_music_with_pydub(path=path, duration=duration)import os
import uuid
import asyncio
from fastapi import FastAPI, HTTPException
from fastapi.staticfiles import StaticFiles
from pydantic import BaseModel
from app.services.video_service import process_prompt_and_generate_video

app = FastAPI()
os.makedirs("output", exist_ok=True)

class VideoRequest(BaseModel):
    prompt: str
    resolution: str
    use_ai_music: bool = True

running_jobs = {}

@app.post("/generate")
async def generate_video_endpoint(data: VideoRequest):
    job_id = str(uuid.uuid4())
    output_path = f"output/final_video_{job_id}.mp4"
    task = asyncio.create_task(
        generate_video_job(data, output_path, job_id)
    )
    running_jobs[job_id] = task
    return {"job_id": job_id}

async def generate_video_job(data, output_path, job_id):
    try:
        await asyncio.to_thread(
            process_prompt_and_generate_video,
            data.prompt,
            data.resolution,
            output_path,
            data.use_ai_music
        )
    except asyncio.CancelledError:
        print(f"🛑 Job {job_id} cancelled")
    finally:
        running_jobs.pop(job_id, None)

@app.post("/cancel/{job_id}")
async def cancel_video_job(job_id: str):
    task = running_jobs.get(job_id)
    if task and not task.done():
        task.cancel()
        return {"status": "cancelled"}
    raise HTTPException(status_code=404, detail="Job not found")

@app.get("/status/{job_id}")
async def job_status(job_id: str):
    exists = job_id in running_jobs
    return {"status": "running" if exists else "done"}

app.mount("/static", StaticFiles(directory="output"), name="static")
video_ai_backend/
├── app/
│   ├── main.py                     # Entry point for FastAPI (app = FastAPI())
│   ├── services/
│   │   └── video_service.py        # Core logic to process prompts & generate video
│   ├── models/
│   │   └── request_models.py       # Pydantic models for request/response
│   ├── routes/
│   │   └── video_routes.py         # Route handlers for /generate, etc.
│   ├── utils/
│   │   └── helpers.py              # Optional: helper functions (file paths, video utils)
│   └── static/                     # For serving generated video files (if any)
│       └── output/                 # Output directory for generated video files
├── requirements.txt                # Python dependencies
├── README.md                       # Project overview
├── .env                            # (Optional) Configs like API keysfrom fastapi import FastAPI
from app.routes import video_routes

app = FastAPI()

app.include_router(video_routes.router)from fastapi import APIRouter
from app.models.request_models import VideoRequest
from app.services.video_service import process_prompt_and_generate_video

router = APIRouter()

@router.post("/generate")
async def generate_video(request: VideoRequest):
    return await process_prompt_and_generate_video(request.prompt, request.resolution)from pydantic import BaseModel

class VideoRequest(BaseModel):
    prompt: str
    resolution: strimport uuid
import os

async def process_prompt_and_generate_video(prompt: str, resolution: str) -> dict:
    output_file = f"app/static/output/video_{uuid.uuid4()}.mp4"
    # Dummy generation logic
    with open(output_file, "w") as f:
        f.write(f"Generated video for prompt: {prompt}, resolution: {resolution}")
    return {"video_url": output_file}3c6795add7f18b90d289d0af325a7b7e40cdcee103a08fa2ec4da5c92b49e835https://six-yx3h.onrender.com52.41.36.82
54.191.253.12
44.226.122.3a74486defce06ec7626708a8c9aab061from app.prompt_parser import parse_prompt
from app.video_generator import generate_video_with_pixverse
from app.video_editor import apply_edits

def process_prompt_and_generate_video(prompt: str, resolution: str, output_path: str, use_ai_music: bool = True):
    instructions = parse_prompt(prompt)
    instructions["use_ai_music"] = use_ai_music
    video_path = generate_video_with_pixverse(prompt, resolution)
    apply_edits(video_path, output_path, instructions)
    return output_pathfrom moviepy.editor import VideoFileClip, AudioFileClip, CompositeAudioClip, vfx
from app.utils.music_generator import generate_background_music

def apply_edits(input_path: str, output_path: str, instructions: dict):
    clip = VideoFileClip(input_path)
    if "speed" in instructions:
        clip = clip.fx(vfx.speedx, instructions["speed"])
    if instructions.get("filter") == "grayscale":
        clip = clip.fx(vfx.blackwhite)
    mood = instructions.get("mood", "ambient")
    use_ai = instructions.get("use_ai_music", True)
    music_path = generate_background_music(duration=int(clip.duration), style=mood, use_ai=use_ai)
    if music_path:
        audio = AudioFileClip(music_path).set_duration(clip.duration)
        new_audio = CompositeAudioClip([clip.audio, audio]) if clip.audio else audio
        clip = clip.set_audio(new_audio)
    clip.write_videofile(output_path, codec="libx264", audio_codec="aac")import os
import requests
from pydub.generators import Sine
from pydub import AudioSegment

MUBERT_API_KEY = os.getenv("MUBERT_API_KEY")
MUBERT_EMAIL = os.getenv("MUBERT_EMAIL")

def generate_music_with_pydub(path="assets/generated_music.mp3", duration=10):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    tone1 = Sine(440).to_audio_segment(duration=500)
    tone2 = Sine(554).to_audio_segment(duration=500)
    tone3 = Sine(660).to_audio_segment(duration=500)
    silence = AudioSegment.silent(duration=250)
    sequence = tone1 + silence + tone2 + silence + tone3 + silence
    full_music = sequence * (duration * 1000 // len(sequence))
    full_music.export(path, format="mp3")
    return path

def generate_music_with_ai(style="ambient", duration=30, path="assets/generated_music.mp3"):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    if not MUBERT_API_KEY or not MUBERT_EMAIL:
        raise ValueError("Missing Mubert credentials")
    payload = {
        "email": MUBERT_EMAIL,
        "token": MUBERT_API_KEY,
        "mode": "track",
        "genre": style,
        "duration": duration,
        "format": "mp3"
    }
    r = requests.post("https://api.mubert.com/v2/Generate", json=payload)
    r.raise_for_status()
    audio_url = r.json()["data"]["track_url"]
    content = requests.get(audio_url).content
    with open(path, "wb") as f:
        f.write(content)
    return path

def generate_background_music(duration=30, style="ambient", use_ai=True, path="assets/generated_music.mp3"):
    try:
        if use_ai:
            return generate_music_with_ai(style=style, duration=duration, path=path)
        raise ValueError("AI disabled")
    except Exception as e:
        print("⚠️ Fallback:", e)
        return generate_music_with_pydub(path=path, duration=duration)
