# XR Construction Collaboration Digital Twin Platform - VR 

A multi-platform Extended Reality application for construction site collaboration. Workers on-site use AR on their phones; project managers review models in VR. Both share the same 3D models, annotations, and real-time communication through a common backend.

Built as my MSc thesis at BME (Budapest University of Technology and Economics), Faculty of Electrical Engineering and Informatics, 2024.

---

## The Problem

Construction projects involve many stakeholders — architects, engineers, site workers, project managers — who all need to work from the same information but rarely occupy the same space at the same time. Existing XR tools for construction either:

- Require proprietary cloud storage (Autodesk Construction Cloud, Procore)
- Work on VR headsets only, or phones only — not both
- Lack real-time communication built in
- Have UIs too complex for workers actually on a construction site

This project builds an alternative: server-agnostic, dual-platform (AR phone + VR headset), with voice and text communication built in, and a UI designed for industrial users who are not looking at a screen all day.

---

## Architecture

```
[AR Client — Phone]          [VR Client — Oculus Quest 2]
        |                               |
        └──────── HTTP (REST) ──────────┘
                        |
                 [Node.js Server]
                 [SQLite3 Database]
                        |
                 [AssetBundle]
                 (any URL / Google Drive)

        [Unity Authentication Server] ── UGS Auth
        [Vivox Server] ── voice + text chat
```

The Node.js/SQLite backend can run on any server — no vendor lock-in. Models are loaded from any URL at runtime via AssetBundles.

---

## Features

### Models
- Load 3D building models at runtime from any URL (AssetBundle format)
- Save and restore model states (component colors, cube positions) to the server
- Color-coded status system: **original → green → yellow → red** to indicate component condition or repair status
- Build mode (VR): place and reposition cubes in 3D space for quick spatial sketching
- Occlusion culling configured per-model to keep frame rates acceptable on standalone Android hardware

| Build Mode (DemoCubes) | Color-coded status | Load at runtime (lamppost) |
|---|---|---|
| <img width="338" height="320" alt="image" src="https://github.com/user-attachments/assets/0d3be665-014f-422e-93ce-2beaae281266" /> | <img width="463" height="343" alt="image" src="https://github.com/user-attachments/assets/87f97788-ec39-4cb0-9f23-288a7996f765" />  | <img width="437" height="436" alt="image" src="https://github.com/user-attachments/assets/01f92b85-fce7-4a77-b4eb-8af37747315a" />|


### Annotations
- Create annotations attached to specific model components
- Annotations appear as floating buttons in 3D space at the saved position
- Annotation list sorted by creation time (newest first)
- Accessible from both AR and VR clients

| Attached annotations | List of changes |
|---|---|
| <img width="205" height="183" alt="image" src="https://github.com/user-attachments/assets/1a59e391-eca5-4b30-a3a7-b7ff87c2136d" /> | <img width="205" height="146" alt="image" src="https://github.com/user-attachments/assets/a249e1fc-105b-4981-b71c-8b8e7cd7d1f0" /> | 


### Communication
- **Voice chat** via Unity Vivox SDK — no separate server to maintain
- **Text chat** in the same channel
- Vivox handles cross-platform audio between phone and headset simultaneously

### Coordinate Synchronization (AR)
- AR clients scan a QR code to establish a shared world origin
- All models and annotation markers appear at the correct physical location relative to the QR code
- Model loading is gated behind QR scan — prevents misaligned placement

### Authentication
- Unity Authentication (UGS) manages session tokens and player IDs
- Vivox channel access is tied to the authenticated player ID
- User permissions and project access controlled via the SQLite database

---

## Tech Stack

| Layer | Technology |
|---|---|
| Game engine | Unity |
| AR framework | ARFoundation |
| VR platform | Oculus Quest 2 (Android standalone) |
| Networking | REST HTTP |
| Backend | Node.js + Express 4.19 |
| Database | SQLite3 5.1.7 (via REST API) |
| Voice/chat | Unity Vivox SDK |
| Auth | Unity Game Services (UGS) Authentication |
| 3D assets | Unity AssetBundles (runtime loading) |
| Language | C# |

---

## How Model State Works

Models are never stored in the database as geometry — only their **state** is stored (color per component, position per cube). The geometry lives in AssetBundles hosted at a URL.

When a user saves a model:
1. The app finds all tagged objects in the scene
2. Records type, position, rotation, scale, and color for each
3. Serializes to JSON and POSTs to `/projects` on the Node.js server
4. A timestamp is saved with each entry

When loading:
1. The app fetches the model list for the active project
2. Each entry appears as a button with its save timestamp
3. On tap, the app reconstructs the model from the saved JSON state
4. If the entry is an AssetBundle URL, it downloads and instantiates the prefab first, then applies the saved state

This means multiple saved versions of the same model exist as a timestamped history — you can see how a model's status changed over time.

---

## Platform Differences

### AR (Phone)
- UI rendered on-screen (2D canvas), not in 3D space
- Touch input via `TouchPhase` — raycasting into 3D scene
- No build mode (cube placement disabled)
- QR scan required before model interaction

### VR (Oculus Quest 2)
- UI rendered as a floating 3D panel that follows the user
- Controller input via `XRRayInteractor` and `InputDevices`
- Build mode enabled — place and grab cubes with trigger/grip buttons
- Teleportation for navigating large building models
- Integrated keyboard for text input

---

## Tested On

- **Oculus Quest 2** — VR, Android standalone
- **Samsung Galaxy A54** — AR, Android

---

## Key Decisions

**Why SQLite + Node.js instead of Firebase/MongoDB?**  
SQLite is a single file, runs on any server, free, and requires no cloud account. The entire database can be opened and inspected with free tools. For a construction site deployment, this matters — the server should be controllable and auditable without a vendor relationship.

**Why AssetBundles over IFC/BIM direct import?**  
IFC parsing in Unity requires third-party libraries and the file sizes are large. AssetBundles give a clean import path: the model designer exports once from their tool to a Unity prefab, and from there it can be hosted anywhere and loaded at runtime. The tradeoff is that model prep requires Unity editor access.

**Why Vivox over building custom voice?**  
Voice synchronization across platforms (Android phone + Android headset) with low latency is a solved problem via Vivox. The alternative — maintaining a WebRTC or TURN server — adds operational complexity with no benefit for this use case.

---

## Limitations & Future Work

- Authentication is currently hardcoded/demo mode 
- IFC/BIM direct import would remove the AssetBundle preparation step
- Speech-to-text for annotations would be valuable on construction sites (typing is impractical)
- User testing with actual construction workers is the most important next step
- A web dashboard for viewing project data without a headset would complement the XR clients

