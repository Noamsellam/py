import tkinter as tk
from tkinter import simpledialog, messagebox
from tkinter import ttk
import subprocess
import threading
import time

class Device:
    def __init__(self, name, ip, status="לא זמין"):
        self.name = name
        self.ip = ip
        self.status = status

class MainApplication(tk.Tk):
    def __init__(self):
        super().__init__()

        self.title("Network Devices Status")
        self.geometry("600x400")

        self.create_widgets()

        self.start_ping_check()

        self.protocol("WM_DELETE_WINDOW", self.on_closing)

    def create_widgets(self):
        # הוספת לחצן "הוסף עמדה"
        add_device_button = tk.Button(self, text="הוסף עמדה", command=self.add_device)
        add_device_button.grid(row=0, column=0, pady=10)

        # הוספת לחצן "מחק מידע מהקובץ"
        delete_data_button = tk.Button(self, text="מחק מידע מהקובץ", command=self.delete_data_from_file)
        delete_data_button.grid(row=0, column=1, pady=10)

        self.tree = ttk.Treeview(self, columns=("Name", "IP", "Status"), show="headings", height=10, selectmode="browse")
        self.tree.heading("Name", text="שם")
        self.tree.heading("IP", text="כתובת IP")
        self.tree.heading("Status", text="מצב")
        self.tree.grid(row=1, column=0, columnspan=2, pady=20)

        # קריאה לפונקציה load_devices_from_file כדי להציג את המכשירים מהקובץ
        self.load_devices_from_file()

        # הוספת יכולת בחירה לטבלה
        self.tree.bind("<ButtonRelease-1>", self.on_tree_click)

    def add_device(self):
        add_device_window = tk.Toplevel(self)
        add_device_window.title("הוסף עמדה")

        name_label = tk.Label(add_device_window, text="שם:")
        name_label.grid(row=0, column=0)
        name_entry = tk.Entry(add_device_window)
        name_entry.grid(row=0, column=1)

        ip_label = tk.Label(add_device_window, text="כתובת IP:")
        ip_label.grid(row=1, column=0)
        ip_entry = tk.Entry(add_device_window)
        ip_entry.grid(row=1, column=1)

        def confirm():
            name = name_entry.get()
            ip = ip_entry.get()

            if name and ip:
                if self.is_device_exists(name, ip):
                    messagebox.showwarning("שגיאה", "השם או כתובת ה-IP כבר קיימים. נסה שנית.")
                else:
                    new_device = Device(name, ip)
                    self.tree.insert("", tk.END, values=(new_device.name, new_device.ip, new_device.status))
                    self.add_device_to_file(new_device)

            add_device_window.destroy()

        confirm_button = tk.Button(add_device_window, text="אישור", command=confirm)
        confirm_button.grid(row=2, columnspan=2)

    def delete_data_from_file(self):
        try:
            # פתח את הקובץ במצב כתיבה ומחק את כל התוכן
            with open("devices.txt", "w"):
                pass
            # אם נמחק בהצלחה, נעדכן את המכשירים במערך ובטבלה
            self.devices = []
            self.tree.delete(*self.tree.get_children())
            messagebox.showinfo("הודעה", "המידע נמחק בהצלחה.")
        except Exception as e:
            # אם יש שגיאה, הדפס אותה
            messagebox.showerror("שגיאה", f"שגיאה במחיקת המידע: {e}")

    def show_device_details(self, device):
        details_window = tk.Toplevel(self)
        details_window.title(f"פרטי מכשיר - {device.name}")
        details_window.geometry("400x200")

        name_label = tk.Label(details_window, text=f"שם: {device.name}")
        name_label.pack(side=tk.LEFT, padx=10)

        ip_label = tk.Label(details_window, text=f"כתובת IP: {device.ip}")
        ip_label.pack(side=tk.LEFT, padx=10)

        delete_button = tk.Button(details_window, text="מחק",
                                  command=lambda: self.delete_device_and_cancel(device, details_window))
        delete_button.pack(side=tk.LEFT, padx=10)

        confirm_button = tk.Button(details_window, text="שמור",
                                   command=lambda: self.save_and_close_device(device, details_window))
        confirm_button.pack(side=tk.LEFT, padx=10)

    def delete_device_and_cancel(self, device, window):
        # מחיקת העמדה מהטבלה
        item_id = self.tree.get_children()[self.devices.index(device)]
        self.tree.delete(item_id)

        # מחיקת המכשיר מהקובץ
        self.delete_device_from_file(device)

        # סגירת החלון
        window.destroy()

        # לאחר המחיקה, עדכן את המצב בקובץ
        self.save_devices_to_file()

    def save_and_close_device(self, device, window):
        # שמירת העמדה לקובץ
        self.save_devices_to_file()

        # סגירת החלון
        window.destroy()

    def refresh_device(self, device):
        status = self.check_device_status(device)
        device.status = "זמין" if status else "לא זמין"
        item_id = self.devices.index(device)
        self.tree.item(item_id, values=(device.name, device.ip, device.status))

    def delete_device(self, device):
        self.devices.remove(device)
        item_id = self.tree.get_children()[self.devices.index(device)]
        self.tree.delete(item_id)
        self.delete_device_from_file(device)

    def is_device_exists(self, name, ip):
        for device in self.devices:
            if device.name == name or device.ip == ip:
                return True
        return False

    def check_device_status(self, device):
        try:
            subprocess.check_output(["ping", "-c", "1", device.ip])
            return True
        except subprocess.CalledProcessError:
            return False

    def start_ping_check(self):
        def ping_check_thread():
            while True:
                for device in self.devices:
                    status = self.check_device_status(device)
                    device.status = "זמין" if status else "לא זמין"
                    item_id = self.devices.index(device)
                    self.tree.item(item_id, values=(device.name, device.ip, device.status))

                self.save_devices_to_file()

                self.update_idletasks()  # הוספת עדכון גרפי

                time.sleep(60)

        ping_check_thread = threading.Thread(target=ping_check_thread)
        ping_check_thread.daemon = True
        ping_check_thread.start()

    def on_closing(self):
        self.save_devices_to_file()
        self.destroy()

    def load_devices_from_file(self):
        try:
            with open("devices.txt", "r") as file:
                lines = file.readlines()
                devices = [Device(*line.strip().split(',')) for line in lines]

            for device in devices:
                self.tree.insert("", tk.END, values=(device.name, device.ip, device.status))

            self.devices = devices

        except FileNotFoundError:
            return []

    def save_devices_to_file(self):
        with open("devices.txt", "w") as file:
            for device in self.devices:
                file.write(f"{device.name},{device.ip}\n")

    def delete_device_from_file(self, device):
        self.devices.remove(device)
        self.save_devices_to_file()

    def add_device_to_file(self, device):
        self.devices.append(device)
        self.save_devices_to_file()

    def on_tree_click(self, event):
        # פונקציה זו תיקרא כאשר המשתמש לוחץ על שורה בטבלה
        selected_items = self.tree.selection()
        if selected_items:
            for item_id in selected_items:
                try:
                    # נבדוק אם המחרוזת מתחילה ב-I ואז נמיר את החלק המספרי
                    if item_id.startswith("I"):
                        item_number = int(item_id[1:])
                        selected_device = self.devices[item_number - 1]
                        print(f"בחרת במכשיר: {selected_device.name}")
                    else:
                        # אחרת, הדפס שגיאה
                        print(f"Invalid item_id format: {item_id}")
                except ValueError as e:
                    # הדפס את הערך שגרם לשגיאה
                    print(f"Error converting item_id to integer: {e}")


if __name__ == "__main__":
    app = MainApplication()
    app.mainloop()
