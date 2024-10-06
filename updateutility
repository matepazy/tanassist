# Ez egy személyreszabott változata az UpdateUtility-nek
# Itt megtalálod az eredetit: https://github.com/matepazy/updateutility

import os
import requests
import customtkinter as ctk
from tkinter import messagebox
from threading import Thread
import subprocess
import sys
import psutil
import ctypes
import logging

logging.basicConfig(
    level=logging.INFO,
    filename='update_log.txt',
    filemode='a',
    format='%(asctime)s - %(levelname)s - %(message)s'
)

ctk.set_appearance_mode("System")
ctk.set_default_color_theme("blue")

def is_admin():
    try:
        return ctypes.windll.shell32.IsUserAnAdmin()
    except:
        return False

def run_as_admin():
    try:
        executable = sys.executable
        script = os.path.abspath(sys.argv[0])
        params = ' '.join([f'"{arg}"' for arg in sys.argv[1:]])
        ctypes.windll.shell32.ShellExecuteW(
            None, "runas", executable, f'"{script}" {params}', None, 1
        )
    except Exception as e:
        logging.error(f"Failed to elevate privileges: {e}")
        sys.exit(1)

def is_application_running(exe_name):
    for proc in psutil.process_iter(['name']):
        try:
            if proc.info['name'] and proc.info['name'].lower() == exe_name.lower():
                logging.info(f"Detected running process: {proc.info['name']}")
                return True
        except (psutil.NoSuchProcess, psutil.AccessDenied, psutil.ZombieProcess):
            pass
    return False

def get_remote_version(url):
    try:
        response = requests.get(url, timeout=10)
        response.raise_for_status()
        return response.text.strip()
    except requests.RequestException as e:
        logging.error(f"Error fetching the remote version: {e}")
        return None

def get_local_version(file_path="ver.tanassist"):
    try:
        if os.path.exists(file_path):
            with open(file_path, 'r') as file:
                content = file.read().strip()
                if content:
                    return content
                else:
                    logging.warning("Local version file is empty.")
                    return None
        else:
            logging.warning(f"Local version file '{file_path}' does not exist.")
            return None
    except Exception as e:
        logging.error(f"Error reading the local version file: {e}")
        return None

def delete_old_executable(exe_path):
    try:
        if os.path.exists(exe_path):
            os.remove(exe_path)
            logging.info(f"Deleted old executable: {exe_path}")
            return True
        else:
            logging.warning(f"Executable {exe_path} does not exist.")
            return True
    except Exception as e:
        logging.error(f"Failed to delete {exe_path}: {e}")
        return False


def download_new_version(download_url, exe_path):
    try:
        logging.info(f"Starting download from {download_url} to {exe_path}")

        with requests.get(download_url, stream=True, timeout=60) as response:
            response.raise_for_status()
            with open(exe_path, 'wb') as f:
                for chunk in response.iter_content(chunk_size=8192):
                    f.write(chunk)

        logging.info(f"Downloaded new version to {exe_path}")
        return True

    except requests.RequestException as e:
        logging.error(f"Error downloading the new version: {e}")
        messagebox.showerror("Hiba történt", f"A letöltés közben hiba történt:\n{e}")
        return False

def restart_application(exe_path):
    try:
        if sys.platform.startswith('win'):
            os.startfile(exe_path)
        elif sys.platform.startswith('darwin'):
            subprocess.Popen(['open', exe_path])
        else:
            subprocess.Popen([exe_path])
        logging.info("Application restarted successfully.")
    except Exception as e:
        logging.error(f"Error restarting application: {e}")

def upgrade_version(old_version, new_version, exe_path, download_url, version_file):
    def proceed_upgrade():
        confirm_button.configure(state="disabled")
        cancel_button.configure(state="disabled")

        download_thread = Thread(target=perform_upgrade)
        download_thread.start()

    def perform_upgrade():
        temp_download_path = exe_path + ".new"
        if not download_new_version(download_url, temp_download_path):
            confirm_button.configure(state="normal")
            cancel_button.configure(state="normal")
            return

        if not delete_old_executable(exe_path):
            logging.error("Failed to delete the old executable after download.")
            messagebox.showerror("Frissítési hiba", "Az UpdateUtility hibát észlelt. A program nem került frissítésre.")
            confirm_button.configure(state="normal")
            cancel_button.configure(state="normal")
            return

        try:
            os.rename(temp_download_path, exe_path)
            logging.info(f"Renamed new executable to: {exe_path}")
        except Exception as e:
            logging.error(f"Failed to rename the new executable: {e}")
            messagebox.showerror("Frissítési hiba", "Az UpdateUtility hibát észlelt. A program nem került frissítésre.")
            confirm_button.configure(state="normal")
            cancel_button.configure(state="normal")
            return

        with open(version_file, 'w') as f:
            f.write(new_version)
        logging.info(f"Version file updated: {new_version}")

        messagebox.showinfo("Frissítés sikeres", f"A program sikeresen frissült a {new_version} verzióra.")

        restart_application(exe_path)

        root.quit()

    root = ctk.CTk()
    root.title(f"TanAssist Frissítés Elérhető | {new_version}")
    root.geometry("500x350")

    upgrade_msg = f"Frissítés:\n\nJelenlegi verzió: {old_version}\nLegújabb verzió: {new_version}\n\n"
    label = ctk.CTkLabel(root, text=upgrade_msg, justify="center", wraplength=480, font=("Arial", 24))
    label.pack(padx=20, pady=20)

    progress_var = ctk.DoubleVar()
    progress_bar = ctk.CTkProgressBar(root, variable=progress_var, width=400)
    progress_bar.set(0.0)
    progress_bar.pack(padx=20, pady=10)

    confirm_button = ctk.CTkButton(root, text="Frissítés", command=proceed_upgrade, font=("Arial", 20))
    confirm_button.pack(padx=20, pady=10)

    cancel_button = ctk.CTkButton(root, text="Mégsem", command=root.quit, font=("Arial", 20))
    cancel_button.pack(padx=20, pady=10)

    root.mainloop()

def check_for_update(version_url, exe_base_url, exe_path="TanAssist.exe", version_file="ver.tanassist"):
    if not is_admin():
        logging.info("Script is not running with admin privileges. Attempting to elevate privileges.")
        run_as_admin()
        sys.exit(0)

    if is_application_running(os.path.basename(exe_path)):
        messagebox.showerror("TanAssist még fut!", "Kérlek állítsd le a programot a frissítés előtt.")
        sys.exit(1)

    remote_version = get_remote_version(version_url)
    local_version = get_local_version(version_file)

    if remote_version and local_version:
        if remote_version > local_version:
            upgrade_version(local_version, remote_version, exe_path, exe_base_url, version_file)
        else:
            logging.info(f"No update available. Local version: {local_version}, Remote version: {remote_version}")
            messagebox.showinfo("Nincs új frissítés", f"Jó hírem van, a legújabb verziót használod. :) ({local_version})")
    else:
        messagebox.showerror("Ellenőrzési hiba", "Nem sikerült ellenőrizni a verziót.")

if __name__ == "__main__":
    version_url = "https://raw.githubusercontent.com/matepazy/tanassist/refs/heads/main/latest.txt"
    NEW_VER = get_remote_version(version_url)
    exe_base_url = f"https://github.com/matepazy/tanassist/releases/download/{NEW_VER}/TanAssistRAWEXE-{NEW_VER}.exe"

    check_for_update(version_url, exe_base_url)
