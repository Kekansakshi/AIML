# AIML
#Higher Performance Weather Forecasting App


from datetime import datetime
import requests
import matplotlib.pyplot as plt
from tkinter import *
from tkinter import messagebox
import pyttsx3
import threading
import speech_recognition as sr
import ttkbootstrap as tb
from ttkbootstrap.constants import *
import pickle
import csv

API_KEY = 'c44ad79124f4592b02c46c61572f6183'

def fetch_weather(city):
    url = f"https://api.openweathermap.org/data/2.5/weather?q={city}&appid={API_KEY}&units=metric"
    res = requests.get(url)
    data = res.json()
    if res.status_code != 200 or data.get("cod") != 200:
        raise ValueError(f"City '{city}' not found or API error: {data.get('message', 'Unknown error')}")
    weather = {
        "city": city,
        "temperature": data["main"]["temp"],
        "humidity": data["main"]["humidity"],
        "wind_speed": data["wind"]["speed"],
        "condition": data["weather"][0]["description"]
    }
    return weather

def speak_weather(weather):
    engine = pyttsx3.init()
    engine.say(f"The current temperature in {weather['city']} is {weather['temperature']} degrees Celsius with {weather['condition']}.")
    engine.runAndWait()

def speak_alert(message):
    engine = pyttsx3.init()
    engine.say(message)
    engine.runAndWait()

def plot_weather(weather):
    mode = plot_mode.get()
    labels = ['Temperature', 'Humidity', 'Wind Speed']
    values = [weather['temperature'], weather['humidity'], weather['wind_speed']]
    fig, ax = plt.subplots(figsize=(5, 4))
    if mode == 'Bar':
        ax.bar(labels, values, color=['#1f77b4', '#ff7f0e', '#2ca02c'])
    elif mode == 'Line':
        ax.plot(labels, values, marker='o', color='green')
    elif mode == 'Scatter':
        ax.scatter(labels, values, color='purple', s=100)
    ax.set_title(f"Weather Stats for {weather['city']}")
    ax.set_ylabel("Values")
    plt.tight_layout()
    plt.show()

def recognize_city():
    recognizer = sr.Recognizer()
    with sr.Microphone() as source:
        try:
            status_label.config(text="Listening for city name...", foreground='blue')
            audio = recognizer.listen(source, timeout=5)
            city = recognizer.recognize_google(audio)
            city_entry.delete(0, END)
            city_entry.insert(0, city)
            status_label.config(text="City recognized successfully!", foreground='green')
        except sr.UnknownValueError:
            messagebox.showerror("Error", "Could not understand audio")
            status_label.config(text="Failed to recognize city", foreground='red')
        except sr.RequestError:
            messagebox.showerror("Error", "Could not request results from Google")
            status_label.config(text="Recognition service error", foreground='red')

def on_fetch():
    try:
        city = city_entry.get()
        weather = fetch_weather(city)
        temperature_label.config(text=f"Temperature: {weather['temperature']}°C")
        humidity_label.config(text=f"Humidity: {weather['humidity']}%")
        wind_label.config(text=f"Wind Speed: {weather['wind_speed']} m/s")
        condition_label.config(text=f"Condition: {weather['condition']}")
        suggestion = suggest_activity(weather)
        activity_label.config(text=f"Suggestion: {suggestion}")
        speak_weather(weather)
        if city not in user_profile['cities']:
            user_profile['cities'].append(city)
            save_profile()
        global last_weather
        last_weather = weather
        check_weather_alert(weather)
    except Exception as e:
        messagebox.showerror("Error", str(e))

def suggest_activity(weather):
    if "rain" in weather["condition"]:
        return "Stay indoors and watch a movie!"
    elif weather["temperature"] > 30:
        return "It's sunny! Go for a hike!"
    else:
        return "Perfect weather for a walk in the park!"

def check_weather_alert(weather):
    alert_message = None
    if weather['temperature'] > 35:
        alert_message = f"High temperature warning for {weather['city']}! It is {weather['temperature']} degrees Celsius."
    elif "rain" in weather['condition']:
        alert_message = f"Rain expected in {weather['city']}. Please carry an umbrella!"
    if alert_message:
        messagebox.showwarning("Weather Alert", alert_message)
        speak_alert(alert_message)

def load_profile():
    try:
        with open('user_profile.pkl', 'rb') as file:
            return pickle.load(file)
    except FileNotFoundError:
        return {'cities': [], 'plot_mode': 'Bar'}

def save_profile():
    with open('user_profile.pkl', 'wb') as file:
        pickle.dump(user_profile, file)

def export_weather_data():
    if 'last_weather' in globals():
        weather = last_weather
        with open('weather_data.csv', mode='w', newline='') as file:
            writer = csv.writer(file)
            writer.writerow(["City", "Temperature", "Humidity", "Wind Speed", "Condition"])
            writer.writerow([weather['city'], weather['temperature'], weather['humidity'], weather['wind_speed'], weather['condition']])
        messagebox.showinfo("Info", "Weather data exported to weather_data.csv")
    else:
        messagebox.showwarning("Warning", "No weather data to export!")

def import_weather_data():
    try:
        with open('weather_data.csv', mode='r') as file:
            reader = csv.reader(file)
            next(reader)
            for row in reader:
                city, temp, humidity, wind_speed, condition = row
                weather = {
                    "city": city,
                    "temperature": float(temp),
                    "humidity": float(humidity),
                    "wind_speed": float(wind_speed),
                    "condition": condition
                }
                global last_weather
                last_weather = weather
                update_weather_display(weather)
        messagebox.showinfo("Info", "Weather data imported successfully!")
    except FileNotFoundError:
        messagebox.showwarning("Warning", "No weather data file found!")

def update_weather_display(weather):
    temperature_label.config(text=f"Temperature: {weather['temperature']}°C")
    humidity_label.config(text=f"Humidity: {weather['humidity']}%")
    wind_label.config(text=f"Wind Speed: {weather['wind_speed']} m/s")
    condition_label.config(text=f"Condition: {weather['condition']}")
    suggestion = suggest_activity(weather)
    activity_label.config(text=f"Suggestion: {suggestion}")

def auto_refresh():
    if 'last_weather' in globals():
        update_weather_display(last_weather)
    root.after(600000, auto_refresh)

root = tb.Window(themename="litera")
root.title("Weather Forecasting App")
root.geometry("600x650")
root.configure(background="#eef3f9")

user_profile = load_profile()

# Title and Date
title_label = tb.Label(root, text="Weather Forecasting App", font=("Helvetica", 20, "bold"), bootstyle="primary")
title_label.pack(pady=10)

current_date = datetime.now().strftime("%A, %d %B %Y")
date_label = tb.Label(root, text=current_date, font=("Helvetica", 12, "italic"), bootstyle="secondary")
date_label.pack(pady=5)

city_frame = Frame(root, bg="#eef3f9")
city_frame.pack(pady=10)
city_label = tb.Label(city_frame, text="Enter City:", font=("Helvetica", 12))
city_label.grid(row=0, column=0, padx=5)
city_entry = tb.Entry(city_frame, font=("Helvetica", 12))
city_entry.grid(row=0, column=1, padx=5)
mic_button = tb.Button(city_frame, text="\U0001F3A4 Speak", command=lambda: threading.Thread(target=recognize_city).start(), bootstyle="info")
mic_button.grid(row=0, column=2, padx=5)

fetch_button = tb.Button(root, text="Fetch Weather", command=lambda: threading.Thread(target=on_fetch).start(), bootstyle="success")
fetch_button.pack(pady=10)

weather_frame = Frame(root, bg="#eef3f9")
weather_frame.pack(pady=10)
temperature_label = tb.Label(weather_frame, text="Temperature:", font=("Helvetica", 12))
temperature_label.pack(pady=2)
humidity_label = tb.Label(weather_frame, text="Humidity:", font=("Helvetica", 12))
humidity_label.pack(pady=2)
wind_label = tb.Label(weather_frame, text="Wind Speed:", font=("Helvetica", 12))
wind_label.pack(pady=2)
condition_label = tb.Label(weather_frame, text="Condition:", font=("Helvetica", 12))
condition_label.pack(pady=2)

activity_label = tb.Label(root, text="Suggestion:", font=("Helvetica", 12, "italic"), bootstyle="warning")
activity_label.pack(pady=10)

plot_mode_frame = Frame(root, bg="#eef3f9")
plot_mode_frame.pack(pady=5)
plot_mode_label = tb.Label(plot_mode_frame, text="Select Plot Type:", font=("Helvetica", 12))
plot_mode_label.grid(row=0, column=0, padx=5)
plot_mode = StringVar(value=user_profile['plot_mode'])
bar_radio = tb.Radiobutton(plot_mode_frame, text="Bar", variable=plot_mode, value="Bar")
line_radio = tb.Radiobutton(plot_mode_frame, text="Line", variable=plot_mode, value="Line")
scatter_radio = tb.Radiobutton(plot_mode_frame, text="Scatter", variable=plot_mode, value="Scatter")
bar_radio.grid(row=0, column=1, padx=5)
line_radio.grid(row=0, column=2, padx=5)
scatter_radio.grid(row=0, column=3, padx=5)

plot_button = tb.Button(root, text="Plot Data", command=lambda: plot_weather(last_weather) if 'last_weather' in globals() else messagebox.showinfo("Info", "No weather data available yet."), bootstyle="secondary")
plot_button.pack(pady=5)

export_button = tb.Button(root, text="Export Weather Data", command=export_weather_data, bootstyle="info")
export_button.pack(pady=5)

import_button = tb.Button(root, text="Import Weather Data", command=import_weather_data, bootstyle="info")
import_button.pack(pady=5)

status_label = tb.Label(root, text="", font=("Helvetica", 10))
status_label.pack(pady=5)

root.after(600000, auto_refresh)
root.mainloop()
