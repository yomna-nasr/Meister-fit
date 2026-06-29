# The Machine Behind the Mirror: AI Documentation — Meister Fit

> Extracted from the Meister Fit Graduation Project Thesis  
> El Shorouk Academy · Higher Institute of Computers and Information Technology  
> Business Information Systems Department · Academic Year 2025–2026

---

## Overview

The AI-based fitness trainer in Meister Fit relies on computer vision to detect human body posture in real-time, analyze exercise form, count repetitions, and provide feedback. The system requires several software components, each responsible for a specific stage of the processing chain.

**Core stack:**

| Component | Technology | Role |
|---|---|---|
| Pose Estimation | MediaPipe Pose | 33-landmark skeleton detection |
| Image Processing | OpenCV | Format conversion + visual rendering |
| Mathematics | NumPy | Joint angle calculation + statistics |
| Dev Testing UI | Gradio | Webcam interface during development only |
| Production API | FastAPI | REST backend exposed to Flutter |
| Cloud Hosting | Hugging Face Spaces | Free Docker-based deployment |
| Development Env | Google Colab | Collaborative notebook environment |

---

## Part I — MediaPipe Pose

### Definition

MediaPipe is an open-source, cross-platform framework developed by Google for building real-time machine learning pipelines. The specific component used is **MediaPipe Pose**, which takes a single RGB image as input and outputs the positions of **33 body landmarks** — key anatomical points including shoulders, elbows, wrists, hips, knees, and ankles. Each landmark is returned as normalized coordinates (values between 0 and 1) along with a visibility confidence score.

### Role in the System

Pose estimation is the foundation of the entire system. Without knowing where the user's joints are, it is impossible to:
- Calculate joint angles
- Determine exercise stages
- Detect form errors
- Count repetitions

The system specifically extracts the **shoulder, elbow, wrist, and hip positions** for both arms on every frame, passing them to the angle calculation and form analysis stages.

> The pose detection must run fast enough to handle at least 20–30 frames per second without noticeable delay, otherwise the feedback would lag behind the user's actual movements.

**Performance:** ~20–30 ms per frame on CPU — no GPU required.

### Alternatives Rejected

| Alternative | Reason for Rejection |
|---|---|
| OpenPose (Carnegie Mellon) | Requires GPU; too slow on CPU-only consumer devices |
| YOLOv8 Pose | Object detection framework — adds unnecessary complexity for pure pose estimation |
| Custom TensorFlow/PyTorch model | Requires large annotated dataset, significant training time, deep expertise — not feasible for graduation project timeline |
| PoseNet | Fewer landmarks than MediaPipe |

### Advantages
- Real-time performance on CPU (~20–30 ms per frame), no GPU required
- 33 body landmarks covering all joints needed for exercise analysis
- Cross-platform support (Windows, macOS, Linux, Android, iOS)
- Normalized coordinates making the system resolution-independent
- Simple Python API with minimal setup
- Open-source, actively maintained by Google

### Limitations
- Designed for single-person detection; multiple people may cause unreliable results
- Requires reasonably good lighting
- Primarily 2D landmark positions; z-coordinate is less reliable
- Fixed pre-trained model; cannot be easily fine-tuned for specific exercises

---

## Part II — OpenCV

### Definition

OpenCV (Open Source Computer Vision Library) is an open-source library specialized in computer vision and image/video processing. Originally developed by Intel in 2000, it provides over 2,500 algorithms and is the most widely used library in the field. Written in C/C++ for performance, it exposes a Python interface via the `cv2` package.

### Two Roles in the System

**1. Image format conversion**  
MediaPipe expects RGB input, but cameras and OpenCV work in BGR format. OpenCV handles the conversion on every frame. Without this, pose detection receives incorrectly formatted data and produces wrong results.

**2. Visual feedback rendering**  
After detecting the pose and analyzing form, OpenCV draws all visual elements on the video frame:
- Skeleton overlay
- Information panels
- Text labels (rep count, exercise stage)
- Progress bars for each arm
- Color-coded form feedback (green = good, yellow = warning, red = error)
- Semi-transparent overlay panel using image blending

**Performance:** ~5–10 ms per frame for drawing operations.

### Alternatives Rejected

| Alternative | Reason for Rejection |
|---|---|
| Pillow (PIL) | Designed for static image editing; too slow for real-time frame-by-frame work |
| Matplotlib | Designed for static plot generation; rendering a figure per frame is prohibitively slow |

### Advantages
- High performance (C++ core); drawing operations ~5–10 ms per frame
- Comprehensive functionality: color conversion, image blending, text rendering, shape drawing — all in one library
- Native compatibility with MediaPipe (both use NumPy arrays as image format)
- 20+ years of development; extensive community and documentation
- Cross-platform including mobile
- Open-source and free

### Limitations
- BGR default color order requires conversion when working with RGB-expecting libraries
- Limited text rendering (few built-in fonts, no Unicode or custom font support)
- No built-in high-level UI components (buttons, sliders) — which is why Gradio was needed separately

---

## Part III — NumPy

### Definition

NumPy (Numerical Python) is an advanced Python library for scientific computing. Released in 2005, it is the foundation of the scientific Python ecosystem. Its core strength is representing data as typed arrays in contiguous memory blocks, enabling operations to execute in optimized C code — far faster than Python's built-in lists.

### Two Critical Tasks

**1. Joint angle calculation**  
The core of exercise analysis depends on measuring the angle at a joint — for example, the elbow angle formed by the shoulder, elbow, and wrist. This requires the `arctan2` trigonometric function and radian-to-degree conversion. This runs on every frame for both arms: approximately **60 angle calculations per second**.

- Angle ~180° → arm straight (extended)
- Angle ~30–50° → arm bent (curled)

This number drives rep counting and form detection.

**2. Statistical analysis**  
When generating workout summaries, the system calculates the average range of motion across all repetitions using `numpy.mean()`, checking for balanced performance between both arms.

> Additionally, NumPy is the underlying data format for images in both OpenCV and MediaPipe — both represent images as NumPy arrays.

### Alternatives Rejected

| Alternative | Reason for Rejection |
|---|---|
| Python `math` module | Works only on scalar values, not arrays; more code required, slower over thousands of frames |
| SciPy | More advanced than needed; would increase dependencies without practical benefit |

### Advantages
- High-speed vectorized operations on arrays
- Memory efficient compared to Python lists
- Built-in trigonometric functions (arctan2, degrees, abs) required for angle calculation
- Already a dependency of OpenCV and MediaPipe — no additional installation needed
- Easy integration with Pandas, Matplotlib, Scikit-learn

### Limitations
- Not designed for deep learning (no GPU acceleration) — but MediaPipe handles the ML part, so this is not relevant here
- Advanced features like broadcasting and indexing have a learning curve for beginners

---

## Part IV — Gradio (Development Tool)

### Definition

Gradio is an open-source Python library that allows developers to create web-based user interfaces for machine learning models. Acquired by Hugging Face in 2021, it connects input components (text boxes, image uploads, webcam feeds) to Python functions and renders a web page for interaction.

### Role: Development and Testing Only

> **Important:** Gradio was not used in the final production product. It was used strictly as a testing and prototyping tool during the development phase.

The final product uses Flutter (mobile frontend) connected to a FastAPI backend deployed on Hugging Face Spaces.

**Why Gradio was used during development:**
- Access the webcam through a browser and stream frames to the Python backend
- Display processed frames (with visual overlays) back to the developer in real-time
- Interact with the system through buttons (saving sessions, viewing statistics, viewing history)

This allowed the AI and computer vision logic to be developed and validated **independently from the mobile frontend**. Once the backend was verified through Gradio, it was integrated into Flutter with confidence.

### Transition to Flutter for Production

Gradio is web-based and cannot produce native mobile applications. The production version uses **Flutter** because:
- Produces native applications for both Android and iOS from a single codebase
- Provides full control over UI design
- Better performance than web-based interfaces for real-time camera applications on mobile

---

## Part V — FastAPI

### Definition

FastAPI is a modern, high-performance Python web framework for building REST APIs, released in 2018. Built on Starlette (async request handling) and Pydantic (automatic data validation), it generates interactive API documentation automatically via Swagger UI.

### Role in the System

FastAPI serves as the **production backend**. After the core AI logic was validated through Gradio during development, it was wrapped in a FastAPI application and deployed on Hugging Face Spaces.

**Communication flow:**
1. Flutter mobile frontend captures a video frame
2. Frame sent as HTTP POST request to FastAPI endpoint
3. Backend processes the frame through MediaPipe + OpenCV pipeline
4. Returns structured JSON response:

```json
{
  "reps": 5,
  "stage": "down",
  "left_angle": 42.3,
  "right_angle": 44.1,
  "feedback": "Good form",
  "frame": 127
}
```

This architecture **fully decouples** the AI backend from the mobile frontend.

### Alternatives Rejected

| Alternative | Reason for Rejection |
|---|---|
| Flask | Lacks native async support; no automatic request validation |
| Django | Full-stack framework; too heavy for a simple API backend |

### Advantages
- Among the fastest Python web frameworks (built on async I/O)
- Automatic request/response validation via Pydantic
- Auto-generated Swagger UI for easy endpoint testing
- Clean separation between AI logic and mobile frontend
- Straightforward deployment on Hugging Face Spaces

---

## Part VI — Hugging Face Spaces

### Definition

Hugging Face Spaces is a free cloud hosting platform that allows developers to deploy machine learning applications directly from a repository. It supports Gradio, Streamlit, and Docker runtimes — the Docker option enables deploying any Python application, including FastAPI servers.

### Role in the System

The FastAPI backend was deployed via Docker container, making it accessible at a **public HTTPS URL** that the Flutter app uses to send frames and receive results. Hugging Face Spaces eliminated the need to set up or pay for a separate cloud server.

### Advantages
- Free hosting with a public HTTPS endpoint
- Docker support for full control over runtime environment
- Simple deployment: push code, Space rebuilds automatically
- Integrated with the Hugging Face ecosystem

### Limitations
- Free tier has limited CPU and RAM, which affects response time under load
- Spaces may sleep after inactivity, causing a cold start delay on first request
- No persistent storage by default
- Session state (rep count, angle history) resets if load balancing switches replicas

---

## Part VII — Google Colab (Development Tool)

Google Colab is a free cloud-based platform that lets users write and run Python code in the browser on Google's cloud servers. Built on Jupyter Notebooks, it was used during development because:
- Libraries like MediaPipe, OpenCV, and NumPy are pre-installed
- Supports real-time collaboration between team members
- Saves work automatically to Google Drive
- Provides free GPU access when needed

**Note:** Colab was a development tool only. The final application does not depend on it.

---

## Part VIII — The Skeleton Problem: A Technical Post-Mortem

This section documents a critical bug discovered during testing that shaped the final architecture.

### The Problem

| State | Response Size | Round-Trip | Result |
|---|---|---|---|
| Before skeleton | ~1 KB | < 800 ms | Smooth |
| After skeleton added | ~80–150 KB | > 1000 ms | Frames dropped |
| After skeleton removed | ~1 KB | < 800 ms | Smooth |

**Root cause:**  
After adding a skeleton overlay to the backend (drawing the 33-landmark pose on each frame, encoding it as JPEG, base64-encoding it, and attaching it to the JSON response), the payload size increased from ~1 KB to ~80–150 KB per frame.

On Hugging Face's CPU-only free tier, the round-trip exceeded 1000 ms. Flutter's frame loop contained a guard:

```dart
if (_isProcessing) return;
```

When the response took longer than 1000 ms, the timer fired while `_isProcessing` was still `true`. That frame was dropped entirely — this was the "cutting" behavior.

**An additional discovery:** Flutter was never reading the `annotated_frame` field at all. The backend was encoding and sending 80–150 KB per frame that was silently discarded on the Flutter side. The user saw only the raw camera preview with no skeleton.

### What Was Tried Before the Fix

| Approach | Outcome |
|---|---|
| `model_complexity=0` | Faster inference, minimal accuracy difference for arm detection |
| Resize to 640×480 before inference | Smaller input tensor, same normalized coordinates |
| Arms-only drawing (4 connections + 6 joints) | Reduced drawing cost, but encoding still dominated |
| JPEG quality 35 + draw on resized frame | ~15–20 KB payload — still occasional drops on weak connections |
| Time-based velocity feedback | Replaced frame-delta threshold with degrees/second — stayed accurate regardless of frame rate |

None of the above fully eliminated cutting under real network conditions.

### The Fix

Remove the skeleton from the backend entirely. The backend returns JSON only:

```json
{ "reps": 5, "stage": "down", "left_angle": 42.3, "right_angle": 44.1, "feedback": "Good form", "frame": 127 }
```

No `draw_landmarks()`. No `cv2.imencode()`. No `base64.b64encode()`. No `annotated_frame` field. Response back to ~1 KB. Cutting eliminated.

### Network Sensitivity Note

Because the app depends on a remote API, every frame travels over the public internet twice. The following causes frame dropping regardless of backend optimizations:
- 4G/5G signal fluctuation (round-trip can spike from 300 ms to 2000+ ms)
- Hugging Face free tier cold starts (10–20 sec container wake-up after inactivity)
- Load balancing across replicas (session state resets when a different replica handles the request)
- Larger payloads hit network jitter harder (150 KB over a weak connection can take 3–5 seconds)

---

## Processing Pipeline Summary

```
Camera Frame
    ↓
OpenCV captures frame (BGR)
    ↓
OpenCV converts BGR → RGB
    ↓
MediaPipe Pose detects 33 landmarks
    ↓
NumPy calculates joint angles (arctan2)
    ↓
Form logic evaluates stage (up/down) and rep count
    ↓
FastAPI returns JSON { reps, stage, left_angle, right_angle, feedback }
    ↓
Flutter renders real-time feedback to user
```

---

## Tool Comparison Summary

| Library | Role | Developer | Rejected Alternative | Reason for Rejection |
|---|---|---|---|---|
| MediaPipe Pose | Body landmark detection | Google | OpenPose | Requires GPU, complex setup |
| OpenCV | Image processing + visual output | Intel / OpenCV community | Pillow | Too slow for real-time video |
| NumPy | Angle calculation + statistics | NumPy community | Python `math` module | No array support |
| Gradio | Testing interface (dev only) | Hugging Face | Streamlit | Poor real-time streaming support |
| FastAPI | Production API backend | Tiangolo | Flask | No async, no auto-validation |
| Hugging Face Spaces | Backend hosting | Hugging Face | Custom cloud server | Cost + server management overhead |

---

*Source: Meister Fit Graduation Project Thesis — El Shorouk Academy, 2026*
