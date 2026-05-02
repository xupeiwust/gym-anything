# Interactive Testing Guide

## CRITICAL: All Testing Must Be Interactive

**DO NOT write Python test scripts or use mock data.** All testing must be done by:
1. Starting the actual environment
2. Interacting with it LIVE using xdotool plus screenshot-based UI grounding (Claude: `visual_grounding` MCP tool; Codex: native screenshot understanding or `ask_cua.py`)
3. Observing real results and fixing issues iteratively

This applies to:
- Environment installation testing
- Task setup testing
- Task start state verification
- End-to-end testing

---

## The Interactive Testing Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│                    INTERACTIVE TESTING LOOP                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Start environment                                            │
│     └── env.reset() → VM boots, hooks run                        │
│                                                                  │
│  2. Take screenshot                                              │
│     └── DISPLAY=:1 scrot /tmp/screen.png                         │
│                                                                  │
│  3. Analyze screenshot (Claude: visual_grounding; Codex: ask_cua)             │
│     └── python ask_cua.py --question \"...\" --screenshot_path ...            │
│                                                                  │
│  4. Perform action via xdotool                                   │
│     └── DISPLAY=:1 xdotool mousemove X Y click 1                 │
│     └── DISPLAY=:1 xdotool type "text"                           │
│                                                                  │
│  5. Take screenshot, observe result                              │
│                                                                  │
│  6. Repeat steps 2-5 to gather sufficient evidence task is doable│
│                                                                  │
│  7. Run export script and check verification                     │
│                                                                  │
│  8. If issues found → fix scripts → env.close() → restart        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Start the Environment

```python
from gym_anything.api import from_config

# Start environment with task
env = from_config("benchmarks/cua_world/environments/<env_name>", task_id="<task_name>")
obs = env.reset(seed=42, use_cache=False)

# Get connection info for SSH
ssh_port = env._runner.ssh_port
print(f"SSH Port: {ssh_port}")
```

Connect via SSH for running commands:

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect('localhost', port=ssh_port, username='ga', password='password123', timeout=10)
```

---

## Step 2: Take Screenshots

Always take a screenshot before deciding what to do:

```python
# Take screenshot inside the VM
ssh.exec_command('DISPLAY=:1 scrot /tmp/screen.png')

# Download to local machine
sftp = ssh.open_sftp()
sftp.get('/tmp/screen.png', '/tmp/local_screen.png')
sftp.close()
```

---

## Step 3: Get UI Guidance (Claude vs Codex)

Use screenshot-based UI grounding to decide what action to take next:

- Claude: use the `visual_grounding` MCP tool with `question` + `screenshot_path`
- Codex: either inspect the screenshot directly, or run `ask_cua.py`

```bash
python ask_cua.py \
    --question "I need to click on the File menu. Where should I click?" \
    --screenshot_path /tmp/local_screen.png
```

**Example questions to ask:**
- "I need to click on the File menu. What are the coordinates?"
- "How do I open the Import dialog in this application?"
- "Is there an error dialog visible? If so, what does it say?"
- "Has the data loaded successfully? What do I see on screen?"
- "Where is the Save button located?"

**IMPORTANT:** Both `visual_grounding` and `ask_cua.py` return coordinates normalized to 1280x720. You must scale them to the actual screen resolution (usually 1920x1080):

```python
# CUA-style tools return coordinates for 1280x720
cua_x, cua_y = 640, 360

# Scale to actual resolution (1920x1080)
actual_x = int(cua_x * 1920 / 1280)
actual_y = int(cua_y * 1080 / 720)
```

---

## Step 4: Perform Actions with xdotool

Execute the actions via SSH:

```python
# Mouse click at coordinates
ssh.exec_command(f'DISPLAY=:1 xdotool mousemove {actual_x} {actual_y} click 1')

# Type text
ssh.exec_command('DISPLAY=:1 xdotool type "Hello World"')

# Press keys
ssh.exec_command('DISPLAY=:1 xdotool key Return')
ssh.exec_command('DISPLAY=:1 xdotool key ctrl+s')  # Save
ssh.exec_command('DISPLAY=:1 xdotool key alt+F4')  # Close window

# Wait for UI to update
import time
time.sleep(1)
```

### Common xdotool Commands

| Action | Command |
|--------|---------|
| Click at position | `xdotool mousemove X Y click 1` |
| Right-click | `xdotool mousemove X Y click 3` |
| Double-click | `xdotool mousemove X Y click --repeat 2 1` |
| Type text | `xdotool type "text here"` |
| Press Enter | `xdotool key Return` |
| Press Escape | `xdotool key Escape` |
| Keyboard shortcut | `xdotool key ctrl+s` |
| Alt menu access | `xdotool key alt+f` (File menu) |

---

## Step 5: Iterate Until Task Complete

Keep taking screenshots and performing actions until the task is done:

```python
# Interactive loop
while True:
    # Take screenshot
    ssh.exec_command('DISPLAY=:1 scrot /tmp/screen.png')
    sftp = ssh.open_sftp()
    sftp.get('/tmp/screen.png', '/tmp/local_screen.png')
    sftp.close()

    # View screenshot (opens in default viewer)
    import subprocess
    subprocess.run(['xdg-open', '/tmp/local_screen.png'])

    # Decide next action - either manually or via screenshot grounding (Claude: visual_grounding; Codex: ask_cua)
    action = input("Enter action (or 'done' to finish): ")
    if action == 'done':
        break

    # Execute action
    ssh.exec_command(f'DISPLAY=:1 {action}')
    time.sleep(1)
```

---

## Step 6: Verify Task Start State

After setting up the task, verify the start state is correct:

1. Take a screenshot and use screenshot grounding (Claude: `visual_grounding`; Codex: native/`ask_cua.py`) to confirm:
   - Application is visible and in the correct screen/view
   - Real data is loaded and visible
   - The agent would know what to do from the screenshot alone

2. Check that the setup_task.sh ran without errors:
```python
stdout = ssh.exec_command('cat /home/ga/env_setup_pre_task.log')[1]
print(stdout.read().decode())
```

Then run the actual verifier to confirm it passes/fails correctly. There is
no standalone `env.verify()` — finalize the episode through the normal
`mark_done` path and read the result off `info["verifier"]`:

```python
# Trigger post_task hook + verifier and write summary.json
_, _, _, info = env.step([], mark_done=True)
result = info["verifier"]
print(f"Passed: {result['passed']}, Score: {result['score']}")
print(f"Feedback: {result['feedback']}")
```

Equivalent CLI invocation (boots, finalizes, runs verifier, reports):

```bash
gym-anything verify task <env_dir> --task <task_name>
```

---

## Step 7: Fix Issues and Restart

When you find issues:

1. **Identify the problem** from screenshots or logs
2. **Edit the relevant script** (install, setup, export, etc.)
3. **Close the environment**: `env.close()`
4. **Restart fresh** and test again

```python
# Close current environment
env.close()

# After fixing scripts, restart
env = from_config("benchmarks/cua_world/environments/<env_name>", task_id="<task_name>")
obs = env.reset(seed=42, use_cache=False)
```

---

## Debugging: Check Hook Logs

If something isn't working, check the hook logs:

```python
# Pre-start hook log (installation)
stdin, stdout, stderr = ssh.exec_command('cat /home/ga/env_setup_pre_start.log')
print("=== PRE-START LOG ===")
print(stdout.read().decode())

# Post-start hook log (setup)
stdin, stdout, stderr = ssh.exec_command('cat /home/ga/env_setup_post_start.log')
print("=== POST-START LOG ===")
print(stdout.read().decode())
```

---

## Debugging: Check Process and Window State

```python
# Check if application is running
stdin, stdout, stderr = ssh.exec_command('ps aux | grep -i <app-name> | grep -v grep')
print(stdout.read().decode())

# List all windows
stdin, stdout, stderr = ssh.exec_command('DISPLAY=:1 wmctrl -l')
print(stdout.read().decode())

# Focus a specific window
ssh.exec_command('DISPLAY=:1 wmctrl -a "Window Title"')
```

---

## Common Issues and Solutions

| Issue | Solution |
|-------|----------|
| GNOME Activities triggers on click | Avoid top-left corner; use keyboard shortcuts instead |
| Menu clicks not registering | Use `alt+<key>` keyboard shortcuts for menus |
| Application not visible | Check DISPLAY=:1 is set; add sleep after launch |
| xdotool coordinates wrong | Scale normalized 1280x720 coords to actual resolution |
| Window not focused | Use `wmctrl -a "Title"` before xdotool commands |
| Dialog blocking interaction | Use Escape key or click specific dismiss button |

---

## Anti-Pattern: DO NOT Do This

**WRONG - Writing test scripts:**
```python
# DO NOT DO THIS
def test_task():
    env = from_config(...)
    obs = env.reset()
    # ... automated test logic ...
    assert result['passed']
```

**WRONG - Using mock data:**
```python
# DO NOT DO THIS
mock_data = {"found": True, "value": "test"}
```

**RIGHT - Interactive testing:**
```
1. Start environment
2. Take screenshot
3. Use screenshot grounding: "Where do I click to open File menu?"
4. Execute: xdotool mousemove X Y click 1
5. Take screenshot
6. Observe result
7. Repeat until done
8. Verify task start state is correct via screenshot grounding
```

---

## Cleanup

```python
# Close environment when done
env.close()
```

Or manually kill lingering processes:
```bash
pkill -f "ga_qemu_"
```

---

## Android Interactive Testing

For Android environments, use ADB commands instead of xdotool:

```bash
# Set up ADB path
export ADB=~/.cache/gym-anything/android-sdk/platform-tools/adb

# Take screenshot
$ADB -s emulator-5556 exec-out screencap -p > /tmp/screenshot.png

# Tap at coordinates
$ADB -s emulator-5556 shell input tap 540 1200

# Swipe
$ADB -s emulator-5556 shell input swipe 540 1600 540 1000 150

# Type text
$ADB -s emulator-5556 shell input text "hello"

# Press key
$ADB -s emulator-5556 shell input keyevent KEYCODE_HOME
```

The same interactive loop applies: take screenshot → analyze screenshot → perform action → repeat.
