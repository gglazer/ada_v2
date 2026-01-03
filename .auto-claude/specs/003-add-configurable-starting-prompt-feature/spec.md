# Specification: Add Configurable Starting Prompt Feature

## Overview

This feature enables users to configure a custom starting/system prompt for the AI assistant through the frontend settings UI. The prompt defines the AI's identity ("who am I") and general behavior guidelines. Currently, the system prompt is hardcoded in the backend (`ada.py`). This change will make it user-configurable, persisted in `settings.json`, and editable via the existing Settings Window.

## Workflow Type

**Type**: feature

**Rationale**: This is a new user-facing feature that requires changes across the frontend (UI), backend (settings handling), and core AI configuration (system instruction). It follows the existing settings pattern already established in the codebase.

## Task Scope

### Services Involved
- **Frontend (React/Electron)** (primary) - Add UI for editing the starting prompt in SettingsWindow
- **Backend (Python/FastAPI)** (primary) - Handle storage and application of the starting prompt

### This Task Will:
- [ ] Add a `starting_prompt` field to settings.json schema
- [ ] Add a textarea UI component in SettingsWindow.jsx for editing the prompt
- [ ] Modify backend to read the custom prompt from settings
- [ ] Pass the custom prompt to the Gemini API session configuration
- [ ] Persist and load the prompt via the existing settings mechanism

### Out of Scope:
- Multiple prompt profiles/presets
- Prompt templates with placeholders
- Real-time prompt switching (requires session restart)
- Prompt validation or sanitization (user has full control)

## Service Context

### Frontend (React + Electron + Vite)

**Tech Stack:**
- Language: JavaScript/JSX
- Framework: React 18 with Vite bundler
- Styling: Tailwind CSS
- Communication: Socket.IO client
- Key directories: `src/`, `src/components/`

**Entry Point:** `src/App.jsx`

**How to Run:**
```bash
npm run dev
```

**Port:** 5173 (Vite dev server)

### Backend (Python + FastAPI)

**Tech Stack:**
- Language: Python 3.11
- Framework: FastAPI + Socket.IO (python-socketio)
- AI: Google Gemini 2.5 Live API
- Key directories: `backend/`

**Entry Point:** `backend/server.py`

**How to Run:**
```bash
conda activate ada_v2 && python backend/server.py
```

**Port:** 8000

## Files to Modify

| File | Service | What to Change |
|------|---------|---------------|
| `backend/settings.json` | Backend | Add `starting_prompt` field with default value |
| `backend/server.py` | Backend | Include `starting_prompt` in DEFAULT_SETTINGS, pass to AudioLoop |
| `backend/ada.py` | Backend | Accept and use custom system_instruction instead of hardcoded value |
| `src/components/SettingsWindow.jsx` | Frontend | Add textarea for editing starting prompt, save on change |

## Files to Reference

These files show patterns to follow:

| File | Pattern to Copy |
|------|----------------|
| `src/components/SettingsWindow.jsx` | Existing settings UI patterns (toggles, dropdowns, Socket.IO integration) |
| `backend/server.py` | Settings load/save pattern, Socket.IO event handlers (`get_settings`, `update_settings`) |
| `backend/ada.py` | How `config` is structured with `system_instruction` |

## Patterns to Follow

### Settings Toggle Pattern (SettingsWindow.jsx)

From `src/components/SettingsWindow.jsx`:

```jsx
const toggleFaceAuth = () => {
    const newVal = !faceAuthEnabled;
    setFaceAuthEnabled(newVal); // Optimistic Update
    localStorage.setItem('face_auth_enabled', newVal);
    socket.emit('update_settings', { face_auth_enabled: newVal });
};
```

**Key Points:**
- Use optimistic UI update for responsiveness
- Emit `update_settings` event to backend
- Backend broadcasts updated settings back to all clients

### Settings Persistence (server.py)

From `backend/server.py`:

```python
DEFAULT_SETTINGS = {
    "face_auth_enabled": False,
    "tool_permissions": {...},
    "printers": [],
    "kasa_devices": [],
    "camera_flipped": False
}

@sio.event
async def update_settings(sid, data):
    # Handle specific keys
    if "camera_flipped" in data:
        SETTINGS["camera_flipped"] = data["camera_flipped"]
    save_settings()
    await sio.emit('settings', SETTINGS)
```

**Key Points:**
- DEFAULT_SETTINGS provides schema and defaults
- update_settings handler processes incoming changes
- save_settings() persists to settings.json
- Broadcast updated settings to all connected clients

### System Instruction Configuration (ada.py)

From `backend/ada.py`:

```python
config = types.LiveConnectConfig(
    response_modalities=["AUDIO"],
    output_audio_transcription={},
    input_audio_transcription={},
    system_instruction="Your name is Ada, which stands for Advanced Design Assistant. "
        "You have a witty and charming personality. "
        "Your creator is Gábor, and you address him as 'Sir'. ...",
    tools=tools,
    speech_config=types.SpeechConfig(...)
)
```

**Key Points:**
- `system_instruction` is a string passed to LiveConnectConfig
- This is what needs to be made dynamic
- The config is used when connecting: `client.aio.live.connect(model=MODEL, config=config)`

## Requirements

### Functional Requirements

1. **Starting Prompt Storage**
   - Description: Store the custom starting prompt in settings.json
   - Acceptance: The `starting_prompt` field exists in settings.json and persists across restarts

2. **Starting Prompt UI**
   - Description: Add a textarea in SettingsWindow for editing the prompt
   - Acceptance: User can view, edit, and save the starting prompt via the Settings panel

3. **Prompt Application**
   - Description: The custom prompt is used as the Gemini system_instruction
   - Acceptance: AI behavior reflects the custom prompt content when session starts

4. **Default Prompt**
   - Description: Provide a sensible default prompt if none is configured
   - Acceptance: First-time users get the current default ADA personality

### Edge Cases

1. **Empty Prompt** - Use default prompt if user clears the field completely
2. **Very Long Prompt** - No hard limit, but display scrollable textarea (Gemini has its own limits)
3. **Session Already Running** - Show info that changes take effect on next session start
4. **Special Characters** - Prompt is plain text, no escaping needed

## Implementation Notes

### DO
- Follow the existing pattern in SettingsWindow.jsx for the textarea styling
- Use the same Socket.IO event flow (`update_settings` / `settings`) as other settings
- Add a debounce on the textarea to avoid sending every keystroke
- Provide a "Reset to Default" button for convenience
- Show a note that changes apply on next session restart

### DON'T
- Create a new Socket.IO event - reuse existing `update_settings` pattern
- Store the prompt in localStorage (keep it server-side in settings.json)
- Try to hot-reload the prompt into an active Gemini session (requires restart)
- Add complex validation - let the user write whatever they want

## Development Environment

### Start Services

```bash
# Single command (starts backend automatically)
conda activate ada_v2 && npm run dev

# Two-terminal setup (recommended for debugging)
# Terminal 1 - Backend:
conda activate ada_v2 && python backend/server.py
# Terminal 2 - Frontend:
npm run dev
```

### Service URLs
- Frontend: http://localhost:5173
- Backend API: http://localhost:8000
- Backend Docs: http://localhost:8000/docs

### Required Environment Variables
- `GEMINI_API_KEY`: Google Gemini API key (in `.env` file)

## Success Criteria

The task is complete when:

1. [ ] `starting_prompt` field exists in settings.json with a default value
2. [ ] SettingsWindow displays a textarea for editing the starting prompt
3. [ ] Changes to the prompt are persisted via the existing settings mechanism
4. [ ] The backend uses the custom prompt when initializing the Gemini session
5. [ ] Default prompt matches the current hardcoded ADA personality
6. [ ] No console errors in frontend or backend
7. [ ] Existing settings functionality remains unaffected

## QA Acceptance Criteria

**CRITICAL**: These criteria must be verified by the QA Agent before sign-off.

### Unit Tests
| Test | File | What to Verify |
|------|------|----------------|
| Settings Load | `tests/test_settings.py` | Default starting_prompt is loaded when not present |
| Settings Save | `tests/test_settings.py` | starting_prompt is persisted correctly |

### Integration Tests
| Test | Services | What to Verify |
|------|----------|----------------|
| Settings Round-trip | Frontend <-> Backend | Update starting_prompt via Socket.IO, verify persistence |
| Prompt Application | Backend <-> Gemini | Custom prompt is passed to LiveConnectConfig |

### End-to-End Tests
| Flow | Steps | Expected Outcome |
|------|-------|------------------|
| Edit Prompt | 1. Open Settings 2. Edit prompt 3. Close Settings 4. Restart session | AI uses new prompt behavior |
| Reset Default | 1. Edit prompt 2. Click Reset 3. Restart session | AI reverts to default personality |

### Browser Verification (if frontend)
| Page/Component | URL | Checks |
|----------------|-----|--------|
| Settings Window | Open via Settings icon | Textarea visible, editable, saves on blur/change |
| Prompt Section | Settings > Starting Prompt | Label, textarea, reset button present |

### Database Verification (if applicable)
| Check | Query/Command | Expected |
|-------|---------------|----------|
| Settings File | `cat backend/settings.json` | Contains `starting_prompt` field |

### QA Sign-off Requirements
- [ ] All unit tests pass
- [ ] All integration tests pass
- [ ] All E2E tests pass
- [ ] Browser verification complete
- [ ] Settings file contains starting_prompt
- [ ] No regressions in existing functionality
- [ ] Code follows established patterns
- [ ] No security vulnerabilities introduced
