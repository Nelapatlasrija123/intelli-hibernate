# intelli-hibernate
Smart Sleep &amp; Auto-Hibernate Scheduler
import tkinter as tk
from tkinter import messagebox
import time
import threading
import os
import json
from datetime import datetime

# ---------------- Configuration ----------------
config = {
    "inactivity_minutes": 2,     # Inactivity time in minutes
    "scheduled_time": "23:59",   # Scheduled hibernate time
    "auto_hibernate_enabled": True,
    "use_hibernate": True
}

# Load config if exists
if os.path.exists("config.json"):
    with open("config.json", "r") as f:
        config.update(json.load(f))

last_activity = time.time()
script_start = time.time()
grace_period = 10  # seconds after startup before checking inactivity
has_hibernated = False  # ensures only one hibernate per session

# ---------------- Reset Timer on Activity ----------------
def reset_timer(event=None):
    global last_activity
    last_activity = time.time()

# ---------------- Hibernate Function ----------------
def hibernate_now():
    global has_hibernated
    has_hibernated = True
    if config["use_hibernate"]:
        os.system("shutdown /h")
    else:
        os.system("rundll32.exe powrprof.dll,SetSuspendState 0,1,0")

# ---------------- GUI Countdown Popup ----------------
def hibernate_with_popup(countdown=5):
    global has_hibernated
    if has_hibernated:
        return

    popup = tk.Toplevel(root)
    popup.title("Hibernate Warning")
    popup.geometry("300x120")
    popup.resizable(False, False)

    label = tk.Label(popup, text=f"PC will hibernate in {countdown} seconds.", font=("Arial", 12))
    label.pack(pady=20)

    canceled = {"value": False}

    def cancel():
        canceled["value"] = True
        popup.destroy()

    cancel_button = tk.Button(popup, text="Cancel", command=cancel, width=10)
    cancel_button.pack(pady=5)

    def countdown_tick():
        nonlocal countdown
        if canceled["value"]:
            messagebox.showinfo("Cancelled", "Hibernate cancelled due to activity.")
            return
        if time.time() - last_activity < 1:
            messagebox.showinfo("Cancelled", "Hibernate cancelled due to activity.")
            popup.destroy()
            return
        if countdown <= 0:
            popup.destroy()
            hibernate_now()
            return
        label.config(text=f"PC will hibernate in {countdown} seconds.")
        countdown -= 1
        popup.after(1000, countdown_tick)

    countdown_tick()
    popup.transient(root)
    popup.grab_set()
    popup.focus_force()

# ---------------- Monitor Inactivity ----------------
def monitor_inactivity():
    global has_hibernated
    time.sleep(5)  # wait after script start/resume
    while True:
        time.sleep(5)

        # Skip monitoring during grace period
        if time.time() - script_start < grace_period:
            continue

        if config["auto_hibernate_enabled"] and not has_hibernated:
            elapsed = time.time() - last_activity
            if elapsed >= config["inactivity_minutes"] * 60:
                root.after(0, lambda: hibernate_with_popup(5))  # trigger popup in main thread

            # Check scheduled time
            now = datetime.now().strftime("%H:%M")
            if now == config["scheduled_time"]:
                root.after(0, lambda: hibernate_with_popup(5))

# ---------------- Save Config ----------------
def save_config():
    with open("config.json", "w") as f:
        json.dump(config, f)
    messagebox.showinfo("Saved", "Settings saved!")

# ---------------- GUI Setup ----------------
root = tk.Tk()
root.title("Intelli-Hibernate")

tk.Label(root, text="Inactivity Minutes:").grid(row=0, column=0)
inactivity_entry = tk.Entry(root)
inactivity_entry.insert(0, str(config["inactivity_minutes"]))
inactivity_entry.grid(row=0, column=1)

tk.Label(root, text="Scheduled Time (HH:MM):").grid(row=1, column=0)
schedule_entry = tk.Entry(root)
schedule_entry.insert(0, config["scheduled_time"])
schedule_entry.grid(row=1, column=1)

auto_var = tk.BooleanVar(value=config["auto_hibernate_enabled"])
tk.Checkbutton(root, text="Enable Auto Hibernate", variable=auto_var).grid(row=2, column=0, columnspan=2)

hibernate_var = tk.BooleanVar(value=config["use_hibernate"])
tk.Checkbutton(root, text="Use Hibernate (else Sleep)", variable=hibernate_var).grid(row=3, column=0, columnspan=2)

def update_config():
    config["inactivity_minutes"] = int(inactivity_entry.get())
    config["scheduled_time"] = schedule_entry.get()
    config["auto_hibernate_enabled"] = auto_var.get()
    config["use_hibernate"] = hibernate_var.get()
    save_config()

tk.Button(root, text="Save Settings", command=update_config).grid(row=4, column=0, columnspan=2)

# ---------------- Bind User Activity ----------------
root.bind_all("<Any-KeyPress>", reset_timer)
root.bind_all("<Any-Button>", reset_timer)
root.bind_all("<Motion>", reset_timer)

# ---------------- Start Monitoring Thread ----------------
threading.Thread(target=monitor_inactivity, daemon=True).start()

root.mainloop() 

