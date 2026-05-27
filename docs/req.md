# Epic: Automated Android Automotive Audio Upgrade Changelog (A14 -> A17)

## Context & Objective
You are an expert Android Automotive OS (AAOS) Systems Architect. Your objective is to generate a highly technical, verified changelog of the Android Audio Framework from Android 14 through Android 17. 

**You will use your terminal execution capabilities to generate the exact git patches locally, read them, and map them to the AAOS Audio Architecture.**

## Architectural Domain Scope
Filter all analysis strictly for the following three architectural layers. Discard any noise outside of this stack:
1. **The Java Service Layer:** `CarAudioService` (Handles Audio Zones, Volume Groups, Audio Routing, and Audio Focus).
2. **The Native Layer:** `AudioControlService` (The C++ proxy service translating Java commands via AIDL).
3. **The HAL Layer:** `AudioControlHAL` (The physical hardware abstraction layer. Pay special attention to the migration from HIDL to Stable AIDL).

---

## User Stories & Execution Steps for Claude

### US 1: Execute Git Commands to Fetch Raw Patches
**As a** system architect, 
**I want** you (Claude) to execute bash commands to generate the diffs directly from the local AOSP repository,
**So that** we have 100% accurate ground-truth data.

* **Acceptance Criteria & Execution Instructions:**
    1. Navigate to your local AOSP `packages/services/Car` directory.
    2. Run `git fetch --tags`.
    3. Execute the following commands to generate the patch files:
       ```bash
       git diff android-14.0.0_r1..android-15.0.0_r1 -- service/src/com/android/car/audio/ cpp/audiocontrol/ > a14_to_a15_audio.patch
       git diff android-15.0.0_r1..android-16.0.0_r1 -- service/src/com/android/car/audio/ cpp/audiocontrol/ > a15_to_a16_audio.patch
       git diff android-16.0.0_r1..main -- service/src/com/android/car/audio/ cpp/audiocontrol/ > a16_to_a17_audio.patch
       ```
    4. Confirm successful generation of these three `.patch` files.

### US 2: Read and Map Patches to the Architecture
**As an** audio framework engineer, 
**I want** you to read the contents of the `.patch` files and map the API bumps to specific architectural layers,
**So that** I understand *where* the upgrade impacts the system (Service, Proxy, or HAL).

* **Acceptance Criteria:**
    * Read `a14_to_a15_audio.patch`, `a15_to_a16_audio.patch`, and `a16_to_a17_audio.patch`.
    * Categorize every major change you find into one of the three layers defined in the Domain Scope.

### US 3: Export Layered Version Changelogs (Markdown)
**As a** documentation manager, 
**I want** the extracted technical details written to isolated Markdown files,
**So that** the changes for each version bump are cleanly documented by architectural layer.

* **Acceptance Criteria:**
    * Create three new files on the file system: `Audio_Framework_A14_to_A15.md`, `Audio_Framework_A15_to_A16.md`, and `Audio_Framework_A16_to_A17.md`.
    * Inside each file, use H2 headers for **CarAudioService Updates**, **AudioControlService Updates**, and **AudioControlHAL Updates**.
    * Include specific class names, AIDL interface changes, and deprecation notices (especially regarding HIDL).

### US 4: Generate Readable HTML Files
**As a** stakeholder, 
**I want** the Markdown files converted into clean HTML files,
**So that** I can easily distribute them to the team for browser viewing.

* **Acceptance Criteria:**
    * Translate the three Markdown files from US 3 into valid HTML5 files on the file system.
    * Include embedded CSS (`<style>`) for high readability (sans-serif fonts, clear headers, distinct styles for inline code).

### US 5: Combine into an Architectural Master Changelog
**As a** system architect, 
**I want** a single, unified document combining all Audio Framework changes from A14 through A17,
**So that** I have a complete lifecycle view of the architecture's evolution.

* **Acceptance Criteria:**
    * Merge the data from US 3 into a single file: `Architectural_Master_Audio_Changelog_A14_to_A17.md`.
    * Include an "Executive Summary" highlighting the macro shifts (e.g., the final death of HIDL, new dynamic routing capabilities).
    * Generate a final HTML version: `Architectural_Master_Audio_Changelog_A14_to_A17.html`.

---
## ⚠️ Instructions to Claude
You have full autonomy. Begin by executing the commands in **US 1** to generate the patches, then proceed sequentially through to **US 5**. Do not stop until all files have been successfully created on the file system.