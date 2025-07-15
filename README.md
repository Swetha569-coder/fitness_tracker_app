# fitness_tracker_app
🎯 A simple desktop fitness tracker built with Python, Tkinter, and SQLite. Log daily activities, track calories and duration, view history, and visualize your progress with charts. Lightweight and easy to use.
import tkinter as tk
from tkinter import ttk, messagebox
import sqlite3
from datetime import datetime, date
from tkinter import simpledialog
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

# ------------------ DATABASE SETUP ------------------
conn = sqlite3.connect("fitness_data.db")
cursor = conn.cursor()

cursor.execute('''
    CREATE TABLE IF NOT EXISTS fitness (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        activity TEXT,
        duration INTEGER,
        calories INTEGER,
        entry_date TEXT
    )
''')
conn.commit()

# ------------------ FUNCTIONS ------------------
def add_entry():
    activity = activity_var.get().strip()
    duration = duration_var.get().strip()
    calories = calories_var.get().strip()

    if not (activity and duration and calories):
        messagebox.showwarning("Input Error", "Please fill in all fields.")
        return

    try:
        duration = int(duration)
        calories = int(calories)
    except ValueError:
        messagebox.showerror("Input Error", "Duration and calories must be numbers.")
        return

    entry_date = datetime.now().strftime("%Y-%m-%d")
    cursor.execute("INSERT INTO fitness (activity, duration, calories, entry_date) VALUES (?, ?, ?, ?)",
                   (activity, duration, calories, entry_date))
    conn.commit()
    messagebox.showinfo("Success", "Activity added successfully.")
    refresh_summary()
    clear_fields()

def refresh_summary():
    today = date.today().strftime("%Y-%m-%d")
    cursor.execute("SELECT SUM(duration), SUM(calories) FROM fitness WHERE entry_date = ?", (today,))
    result = cursor.fetchone()
    total_duration = result[0] if result[0] else 0
    total_calories = result[1] if result[1] else 0

    duration_label.config(text=f"Total Duration Today: {total_duration} min")
    calories_label.config(text=f"Total Calories Today: {total_calories} kcal")
    duration_bar['value'] = min(total_duration / 60 * 100, 100)
    calories_bar['value'] = min(total_calories / 500 * 100, 100)

def clear_fields():
    activity_var.set("")
    duration_var.set("")
    calories_var.set("")

def show_history():
    history_window = tk.Toplevel(root)
    history_window.title("Activity History")
    history_window.geometry("500x400")

    filter_date = simpledialog.askstring("Filter", "Enter date (YYYY-MM-DD) or leave blank for all:")
    if filter_date:
        cursor.execute("SELECT activity, duration, calories, entry_date FROM fitness WHERE entry_date = ? ORDER BY entry_date DESC", (filter_date,))
    else:
        cursor.execute("SELECT activity, duration, calories, entry_date FROM fitness ORDER BY entry_date DESC")

    tree = ttk.Treeview(history_window, columns=("Activity", "Duration", "Calories", "Date"), show='headings')
    for col in ("Activity", "Duration", "Calories", "Date"):
        tree.heading(col, text=col)
        tree.column(col, anchor="center", width=100)
    tree.pack(fill=tk.BOTH, expand=True)

    for row in cursor.fetchall():
        tree.insert('', tk.END, values=row)

def show_charts():
    cursor.execute("SELECT entry_date, SUM(duration), SUM(calories) FROM fitness GROUP BY entry_date ORDER BY entry_date")
    data = cursor.fetchall()

    if not data:
        messagebox.showinfo("No Data", "No activity data to display.")
        return

    dates = [row[0] for row in data]
    durations = [row[1] for row in data]
    calories = [row[2] for row in data]

    chart_window = tk.Toplevel(root)
    chart_window.title("Activity Charts")
    chart_window.geometry("600x400")

    fig, ax = plt.subplots(figsize=(6, 3))
    ax.plot(dates, durations, label="Duration (min)", marker='o')
    ax.plot(dates, calories, label="Calories (kcal)", marker='x')
    ax.set_title("Daily Activity Summary")
    ax.set_xlabel("Date")
    ax.set_ylabel("Value")
    ax.legend()
    ax.grid(True)

    canvas = FigureCanvasTkAgg(fig, master=chart_window)
    canvas.draw()
    canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True)

# ------------------ UI SETUP ------------------
root = tk.Tk()
root.title("Fitness Tracker App")
root.geometry("450x600")
root.resizable(False, False)
style = ttk.Style()
style.configure("TButton", padding=6, relief="flat", background="#2c3e50", foreground="white", font=('Arial', 10, 'bold'))

activity_var = tk.StringVar()
duration_var = tk.StringVar()
calories_var = tk.StringVar()

# Entry Frame
entry_frame = tk.LabelFrame(root, text="Log New Activity", padx=15, pady=15, font=('Arial', 10, 'bold'))
entry_frame.pack(fill="x", padx=20, pady=15)

tk.Label(entry_frame, text="Activity:", font=('Arial', 10)).grid(row=0, column=0, sticky="e")
tk.Entry(entry_frame, textvariable=activity_var, width=30).grid(row=0, column=1)

tk.Label(entry_frame, text="Duration (min):", font=('Arial', 10)).grid(row=1, column=0, sticky="e")
tk.Entry(entry_frame, textvariable=duration_var, width=30).grid(row=1, column=1)

tk.Label(entry_frame, text="Calories Burned:", font=('Arial', 10)).grid(row=2, column=0, sticky="e")
tk.Entry(entry_frame, textvariable=calories_var, width=30).grid(row=2, column=1)

tk.Button(entry_frame, text="➕ Add Entry", command=add_entry).grid(row=3, column=0, columnspan=2, pady=12)

# Summary Frame
summary_frame = tk.LabelFrame(root, text="Today's Summary", padx=15, pady=10, font=('Arial', 10, 'bold'))
summary_frame.pack(fill="x", padx=20, pady=5)

duration_label = tk.Label(summary_frame, text="Total Duration Today: 0 min", font=('Arial', 10))
duration_label.pack(anchor="w")
duration_bar = ttk.Progressbar(summary_frame, length=350, maximum=100)
duration_bar.pack(pady=5)

calories_label = tk.Label(summary_frame, text="Total Calories Today: 0 kcal", font=('Arial', 10))
calories_label.pack(anchor="w")
calories_bar = ttk.Progressbar(summary_frame, length=350, maximum=100)
calories_bar.pack(pady=5)

# Buttons Frame
buttons_frame = tk.Frame(root)
buttons_frame.pack(pady=20)

ttk.Button(buttons_frame, text="📜 View History", command=show_history).grid(row=0, column=0, padx=10)
ttk.Button(buttons_frame, text="📈 Show Charts", command=show_charts).grid(row=0, column=1, padx=10)
ttk.Button(buttons_frame, text="🔄 Refresh", command=refresh_summary).grid(row=0, column=2, padx=10)

# Run initial summary
refresh_summary()

# Start app
root.mainloop()

# Close DB connection
conn.close()
