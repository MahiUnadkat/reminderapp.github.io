# reminderapp.github.io

import tkinter as tk  # Needed for Listbox
import ttkbootstrap as tb
from ttkbootstrap.constants import *
from tkinter import messagebox
from datetime import datetime
import schedule
import threading
import time
import serial
import json
import os

# ----------- Bluetooth Configuration ----------- #
HC05_PORT = "COM4"  # Change this if your HC-05 is on another port
BAUDRATE = 9600

try:
    ser = serial.Serial(HC05_PORT, BAUDRATE, timeout=1)
    time.sleep(2)
    print("✅ Connected to HC-05")
except serial.SerialException as e:
    print("❌ Could not connect to HC-05:", e)
    ser = None

# ----------- Scheduler and Logic ----------- #
reminders = []

def send_reminder(msg):
    now = datetime.now().strftime("%Y-%m-%d %H:%M")
    log(f"🔔 [{now}] Reminder: {msg}")
    if ser:
        try:
            ser.write(f"[REMINDER] {msg}\n".encode())
        except Exception as e:
            log(f"⚠️ Bluetooth error: {e}")
    messagebox.showinfo("Reminder", msg)

def schedule_reminder(reminder):
    dt = datetime.strptime(reminder["datetime"], "%Y-%m-%d %H:%M")
    repetition = reminder["repetition"]
    message = reminder["message"]

    def job():
        send_reminder(message)

    now = datetime.now()
    if repetition == "none":
        if dt > now:
            delay = (dt - now).total_seconds()
            schedule.every(delay).seconds.do(job).tag(message)
    elif repetition == "daily":
        schedule.every().day.at(dt.strftime("%H:%M")).do(job).tag(message)
    elif repetition == "weekly":
        schedule.every().week.at(dt.strftime("%H:%M")).do(job).tag(message)
    elif repetition == "monthly":
        def monthly_job():
            if datetime.now().day == dt.day:
                job()
        schedule.every().day.at(dt.strftime("%H:%M")).do(monthly_job).tag(message)
    elif repetition == "hourly":
        schedule.every().hour.do(job).tag(message)

def add_reminder():
    date_str = date_entry.get().strip()
    time_str = time_entry.get().strip()
    repetition = repetition_box.get().strip().lower()
    message = message_entry.get().strip()

    if not all([date_str, time_str, message]):
        messagebox.showerror("Missing Fields", "Please complete all fields.")
        return

    try:
        dt = datetime.strptime(f"{date_str} {time_str}", "%Y-%m-%d %H:%M")
    except ValueError:
        messagebox.showerror("Format Error", "Date or Time format is invalid.")
        return

    reminder = {
        "datetime": dt.strftime("%Y-%m-%d %H:%M"),
        "repetition": repetition,
        "message": message
    }
    reminders.append(reminder)
    reminder_list.insert(END, f'{reminder["datetime"]} | {repetition} | {message}')
    schedule_reminder(reminder)
    log(f"✅ Reminder added: {message}")
    message_entry.delete(0, END)

def delete_selected():
    selected = reminder_list.curselection()
    if not selected:
        return
    idx = selected[0]
    msg = reminders[idx]["message"]
    reminder_list.delete(idx)
    schedule.clear(msg)
    reminders.pop(idx)
    log(f"🗑️ Reminder deleted: {msg}")

def save_reminders():
    with open("reminders.json", "w") as f:
        json.dump(reminders, f, indent=2)
    log("💾 Reminders saved.")

def load_reminders():
    if not os.path.exists("reminders.json"):
        return
    with open("reminders.json", "r") as f:
        loaded = json.load(f)
    for rem in loaded:
        reminders.append(rem)
        reminder_list.insert(END, f'{rem["datetime"]} | {rem["repetition"]} | {rem["message"]}')
        schedule_reminder(rem)
    log("📂 Reminders loaded.")

def log(text):
    status_text.set(text)
    print(text)

# ----------- Scheduler Thread ----------- #
def run_scheduler():
    while True:
        schedule.run_pending()
        time.sleep(1)

threading.Thread(target=run_scheduler, daemon=True).start()

# ----------- UI Setup ----------- #
app = tb.Window(themename="morph")  # Themes: morph, darkly, flatly, etc.
app.title("Bluetooth Reminder App")
app.geometry("500x550")

frame = tb.Frame(app, padding=10)
frame.pack(fill=BOTH, expand=True)

tb.Label(frame, text="📝 Reminder Message:").pack(anchor="w")
message_entry = tb.Entry(frame, width=50)
message_entry.pack()

tb.Label(frame, text="📅 Date (YYYY-MM-DD):").pack(anchor="w", pady=(5, 0))
date_entry = tb.Entry(frame, width=25)
date_entry.pack()

tb.Label(frame, text="⏰ Time (HH:MM 24-hr):").pack(anchor="w", pady=(5, 0))
time_entry = tb.Entry(frame, width=25)
time_entry.pack()

tb.Label(frame, text="🔁 Repetition:").pack(anchor="w", pady=(5, 0))
repetition_box = tb.Combobox(frame, values=["none", "daily", "weekly", "monthly", "hourly"])
repetition_box.set("none")
repetition_box.pack()

tb.Button(frame, text="➕ Add Reminder", bootstyle=SUCCESS, command=add_reminder).pack(pady=10)
tb.Button(frame, text="🗑 Delete Selected", bootstyle=WARNING, command=delete_selected).pack(pady=5)
tb.Button(frame, text="💾 Save Reminders", bootstyle=INFO, command=save_reminders).pack(pady=5)

tb.Label(frame, text="📋 Scheduled Reminders:").pack(anchor="w", pady=(10, 0))
reminder_list = tk.Listbox(frame, height=10, font=("Consolas", 10))
reminder_list.pack(fill=BOTH, expand=True)

# Status Bar
status_text = tb.StringVar()
status_bar = tb.Label(app, textvariable=status_text, bootstyle=INVERSE, anchor="w")
status_bar.pack(fill=X, side=BOTTOM)
log("Ready.")

load_reminders()
app.mainloop()

if ser:
    ser.close()
