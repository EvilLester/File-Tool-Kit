import os
import shutil
import subprocess
import tkinter as tk
from tkinter import ttk, messagebox, scrolledtext
from pathlib import Path
from datetime import datetime
import threading

class SimpleCleanupTool:
    def __init__(self, root):
        self.root = root
        self.root.title("Simple Windows & Game Cleanup Tool")
        self.root.geometry("1000x650")
        self.root.configure(bg="#0d0d0d")
        
        # UI Styling
        style = ttk.Style()
        style.theme_use("clam")
        style.configure(".", background="#0d0d0d", foreground="white", fieldbackground="#1a1a1a")
        style.configure("TButton", background="#222222", foreground="white", borderwidth=1)
        style.map("TButton", background=[("active", "#333333")])
        style.configure("Treeview", background="#1a1a1a", foreground="white", fieldbackground="#1a1a1a", rowheight=24)
        style.configure("Treeview.Heading", background="#111111", foreground="white")

        self.items = []
        self.create_ui()

    def create_ui(self):
        # Title
        title = tk.Label(self.root, text="Simple Windows & Game Cleaner", font=("Segoe UI", 14, "bold"), bg="#0d0d0d", fg="white")
        title.pack(pady=(10, 5))

        # Buttons Frame
        btn_frame = tk.Frame(self.root, bg="#0d0d0d")
        btn_frame.pack(fill="x", padx=10, pady=5)

        ttk.Button(btn_frame, text="Start Scan", command=self.start_scan).pack(side="left", padx=3)
        ttk.Button(btn_frame, text="Open Selected Folder", command=self.open_selected).pack(side="left", padx=3)
        ttk.Button(btn_frame, text="Delete Selected Items", command=self.delete_selected).pack(side="left", padx=3)
        ttk.Button(btn_frame, text="Clear Log", command=self.clear_log).pack(side="right", padx=3)

        # Table (Treeview)
        columns = ("path", "size", "status", "reason")
        self.tree = ttk.Treeview(self.root, columns=columns, show="headings", selectmode="extended")
        
        self.tree.heading("path", text="Folder Path")
        self.tree.heading("size", text="Size")
        self.tree.heading("status", text="Status")
        self.tree.heading("reason", text="Why remove?")

        self.tree.column("path", width=380)
        self.tree.column("size", width=80, anchor="e")
        self.tree.column("status", width=80, anchor="center")
        self.tree.column("reason", width=420)

        # Color tags for status
        self.tree.tag_configure("red", foreground="#ff5555")
        self.tree.tag_configure("yellow", foreground="#ffcc00")

        self.tree.pack(fill="both", expand=True, padx=10, pady=5)
        self.tree.bind("<Double-1>", lambda e: self.open_selected())

        # Log Box
        self.log = scrolledtext.ScrolledText(self.root, height=6, bg="#111111", fg="#dddddd", font=("Consolas", 9), borderwidth=0)
        self.log.pack(fill="x", padx=10, pady=(5, 10))

    def log_msg(self, msg: str):
        timestamp = datetime.now().strftime("%H:%M:%S")
        self.log.insert("end", f"[{timestamp}] {msg}\n")
        self.log.see("end")

    def clear_log(self):
        self.log.delete("1.0", "end")

    def get_size(self, path: Path) -> int:
        total = 0
        try:
            if path.is_file():
                return path.stat().st_size
            for p in path.rglob("*"):
                if p.is_file():
                    try:
                        total += p.stat().st_size
                    except (PermissionError, OSError):
                        continue
        except (PermissionError, OSError):
            pass
        return total

    def format_size(self, size: int) -> str:
        for unit in ["B", "KB", "MB", "GB"]:
            if size < 1024:
                return f"{size:.1f} {unit}"
            size /= 1024
        return f"{size:.1f} TB"

    def add_target(self, path: Path, status: str, reason: str):
        if path.exists():
            size = self.get_size(path)
            if size > 1024: # Only show if larger than 1KB
                self.items.append({
                    "path": str(path),
                    "size": size,
                    "status": status,
                    "reason": reason
                })

    def run_scan(self):
        self.items.clear()
        self.log_msg("Scanning system for junk and caches...")

        home = Path.home()
        local = Path(os.environ.get("LOCALAPPDATA", home / "AppData" / "Local"))
        temp = Path(os.environ.get("TEMP", local / "Temp"))

        # Add target folders to check
        self.add_target(temp, "REMOVE", "Windows temporary files. Safe to delete.")
        self.add_target(local / "Temp", "REMOVE", "Local app temporary files. Safe to delete.")
        self.add_target(local / "CrashDumps", "REMOVE", "System crash reports. Safe to delete.")
        self.add_target(local / "NVIDIA" / "DXCache", "REMOVE", "NVIDIA shader cache. Games will rebuild automatically.")
        self.add_target(local / "AMD" / "DxCache", "REMOVE", "AMD shader cache. Games will rebuild automatically.")
        self.add_target(local / "Rockstar Games" / "Launcher" / "logs", "REMOVE", "Rockstar Launcher logs. Safe to delete.")
        self.add_target(local / "Rockstar Games", "REVIEW", "Main Rockstar folder. Check contents before deleting anything important.")

        # Update UI in main thread
        self.root.after(0, self.update_table)

    def update_table(self):
        self.tree.delete(*self.tree.get_children())
        for item in sorted(self.items, key=lambda x: x["size"], reverse=True):
            self.tree.insert("", "end", values=(
                item["path"],
                self.format_size(item["size"]),
                item["status"],
                item["reason"]
            ), tags=(item["status"].lower(),))

        total_size = sum(i["size"] for i in self.items)
        self.log_msg(f"Scan complete. Found {len(self.items)} items (Total: {self.format_size(total_size)}).")

    def start_scan(self):
        threading.Thread(target=self.run_scan, daemon=True).start()

    def open_selected(self):
        selected = self.tree.selection()
        if not selected:
            messagebox.showinfo("Notice", "Please select an item first.")
            return
        path = self.tree.item(selected[0], "values")[0]
        try:
            subprocess.Popen(f'explorer "{path}"')
            self.log_msg(f"Opened folder: {path}")
        except Exception as e:
            self.log_msg(f"Failed to open: {e}")

    def delete_selected(self):
        selected = self.tree.selection()
        if not selected:
            messagebox.showinfo("Notice", "Please select items to delete.")
            return

        if not messagebox.askyesno("Confirm Delete", "Are you sure you want to delete the selected items?"):
            return

        removed_count = 0
        for item_id in selected:
            values = self.tree.item(item_id, "values")
            path = Path(values[0])
            try:
                if path.is_file():
                    path.unlink()
                elif path.is_dir():
                    shutil.rmtree(path, ignore_errors=True)
                self.tree.delete(item_id)
                self.log_msg(f"Deleted: {path}")
                removed_count += 1
            except Exception as e:
                self.log_msg(f"Could not delete {path}: {e}")

        self.log_msg(f"Finished deletion. Removed {removed_count} item(s).")

if __name__ == "__main__":
    root = tk.Tk()
    app = SimpleCleanupTool(root)
    root.mainloop()
