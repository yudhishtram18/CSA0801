"""
Employee Payroll Management System
-----------------------------------
A desktop GUI application built with Python's Tkinter + SQLite,
styled with a clean, card-based, green-accent UI.

Modules (matching the 5 capstone modules):
  1. Employee Management
  2. Attendance & Leave
  3. Payroll Processing
  4. Tax & Deductions
  5. Reports & Payslips

Run with:
    python payroll_app.py

Requirements:
    - Python 3.8+
    - tkinter (usually built-in; on Linux: sudo apt install python3-tk)
    - reportlab (optional, for PDF payslips): pip install reportlab
"""

import sqlite3
import datetime
import os
import tkinter as tk
from tkinter import ttk, messagebox

DB_FILE = "payroll.db"

# Try to import reportlab for PDF payslip generation. If it isn't installed,
# the app still works -- payslips are just written as plain text files.
try:
    from reportlab.lib.pagesizes import A4
    from reportlab.lib.units import mm
    from reportlab.pdfgen import canvas as pdf_canvas
    HAS_REPORTLAB = True
except ImportError:
    HAS_REPORTLAB = False


# ======================================================================
# THEME / UI HELPERS
# ======================================================================

PRIMARY = "#22C55E"        # main green
PRIMARY_DARK = "#16A34A"   # hover / darker green
PRIMARY_SOFT = "#DCFCE7"   # light green (chips / stat cards)
DARK = "#111827"           # near-black text
GRAY = "#6B7280"           # secondary text
BG = "#F5F7FA"             # app background
CARD_BG = "#FFFFFF"        # card background
BORDER = "#E5E7EB"         # hairline border
INPUT_BG = "#F3F4F6"       # input field background
SIDEBAR_BG = "#FFFFFF"
DANGER = "#EF4444"
FONT = "Segoe UI"


def darken(hex_color, factor=0.85):
    hex_color = hex_color.lstrip("#")
    r, g, b = (int(hex_color[i:i + 2], 16) for i in (0, 2, 4))
    r, g, b = (max(0, int(c * factor)) for c in (r, g, b))
    return f"#{r:02x}{g:02x}{b:02x}"


def rounded_rect(canvas_widget, x1, y1, x2, y2, radius=18, **kwargs):
    points = [
        x1 + radius, y1, x2 - radius, y1, x2, y1, x2, y1 + radius,
        x2, y2 - radius, x2, y2, x2 - radius, y2, x1 + radius, y2,
        x1, y2, x1, y2 - radius, x1, y1 + radius, x1, y1,
    ]
    return canvas_widget.create_polygon(points, smooth=True, **kwargs)


class RoundButton(tk.Canvas):
    """A flat, pill-shaped button drawn on a Canvas (Tkinter has no native
    rounded buttons)."""

    def __init__(self, parent, text, command=None, bg=PRIMARY, fg="white",
                 width=200, height=42, radius=None, font_size=11, bold=True):
        parent_bg = parent["bg"] if "bg" in parent.keys() else BG
        super().__init__(parent, width=width, height=height, bg=parent_bg,
                          highlightthickness=0, bd=0)
        self.command = command
        self.bg_color = bg
        radius = radius if radius is not None else height // 2
        weight = "bold" if bold else "normal"
        self.rect = rounded_rect(self, 1, 1, width - 1, height - 1, radius,
                                  fill=bg, outline=bg)
        self.text_id = self.create_text(width / 2, height / 2, text=text,
                                         fill=fg, font=(FONT, font_size, weight))
        self.bind("<Button-1>", self._click)
        self.bind("<Enter>", lambda e: self.itemconfig(self.rect, fill=darken(self.bg_color)))
        self.bind("<Leave>", lambda e: self.itemconfig(self.rect, fill=self.bg_color))
        self.configure(cursor="hand2")

    def _click(self, event):
        if self.command:
            self.command()

    def set_text(self, text):
        self.itemconfig(self.text_id, text=text)


def styled_entry(parent, width=24, show=None):
    e = tk.Entry(parent, width=width, show=show, bg=INPUT_BG, fg=DARK,
                 relief="flat", highlightthickness=1, highlightbackground=BORDER,
                 highlightcolor=PRIMARY, font=(FONT, 10), insertbackground=DARK)
    return e


def field_label(parent, text):
    return tk.Label(parent, text=text, bg=parent["bg"], fg=GRAY, font=(FONT, 9, "bold"))


def card(parent, **kwargs):
    f = tk.Frame(parent, bg=CARD_BG, highlightthickness=1,
                 highlightbackground=BORDER, highlightcolor=BORDER)
    for k, v in kwargs.items():
        f.configure(**{k: v})
    return f


class RoundedScrollbar(tk.Canvas):
    """A slim, pill-shaped vertical scrollbar drawn on a Canvas and styled
    to match the app theme (ttk's built-in scrollbar can't get fully
    rounded edges). Supports click-to-page and drag-to-scroll, and hides
    itself automatically when there's nothing to scroll."""

    def __init__(self, parent, command=None, width=10, bg=None,
                 trough=INPUT_BG, thumb=PRIMARY, thumb_hover=PRIMARY_DARK):
        parent_bg = bg if bg is not None else (parent["bg"] if "bg" in parent.keys() else BG)
        super().__init__(parent, width=width, bg=parent_bg,
                          highlightthickness=0, bd=0)
        self.command = command
        self.trough_color = trough
        self.thumb_color = thumb
        self.thumb_hover = thumb_hover
        self._lo, self._hi = 0.0, 1.0
        self._dragging = False
        self._drag_offset = 0
        self._min_thumb = 28

        self.bind("<Configure>", lambda e: self._redraw())
        self.bind("<Button-1>", self._on_click)
        self.bind("<B1-Motion>", self._on_drag)
        self.bind("<ButtonRelease-1>", self._on_release)
        self.configure(cursor="arrow")

    def set(self, lo, hi):
        """Called by the canvas's yscrollcommand."""
        self._lo, self._hi = float(lo), float(hi)
        self._redraw()

    def _thumb_geometry(self):
        w, h = self.winfo_width(), self.winfo_height()
        if h <= 1:
            return None
        thumb_h = max(self._min_thumb, int((self._hi - self._lo) * h))
        thumb_y = min(int(self._lo * h), h - thumb_h)
        return w, h, max(0, thumb_y), thumb_h

    def _redraw(self):
        self.delete("all")
        geo = self._thumb_geometry()
        if not geo:
            return
        w, h, thumb_y, thumb_h = geo
        # Everything already fits on screen -> no need to show a bar.
        if self._hi - self._lo >= 0.999:
            return
        rounded_rect(self, 2, 2, w - 2, h - 2, radius=w // 2,
                      fill=self.trough_color, outline=self.trough_color)
        rounded_rect(self, 2, thumb_y + 2, w - 2, thumb_y + thumb_h - 2,
                      radius=(w - 4) // 2, fill=self.thumb_color,
                      outline=self.thumb_color, tags=("thumb",))
        self.tag_bind("thumb", "<Enter>", lambda e: self._paint_thumb(self.thumb_hover))
        self.tag_bind("thumb", "<Leave>", lambda e: self._paint_thumb(self.thumb_color)
                      if not self._dragging else None)

    def _paint_thumb(self, color):
        self.itemconfig("thumb", fill=color, outline=color)

    def _on_click(self, event):
        geo = self._thumb_geometry()
        if not geo:
            return
        w, h, thumb_y, thumb_h = geo
        if thumb_y <= event.y <= thumb_y + thumb_h:
            self._dragging = True
            self._drag_offset = event.y - thumb_y
            self._paint_thumb(self.thumb_hover)
        elif self.command:
            self.command("moveto", max(0.0, min(1.0, event.y / h)))

    def _on_drag(self, event):
        if not self._dragging or not self.command:
            return
        geo = self._thumb_geometry()
        if not geo:
            return
        w, h, _, thumb_h = geo
        span = h - thumb_h
        new_y = max(0, min(span, event.y - self._drag_offset))
        self.command("moveto", (new_y / span) if span else 0)

    def _on_release(self, event):
        self._dragging = False
        self._paint_thumb(self.thumb_color)


class ScrollableFrame(tk.Frame):
    """A frame with a vertical scrollbar for content that's taller than the
    visible area. Put widgets inside `.body` (not inside this frame
    directly) and they'll scroll with the mouse wheel or the scrollbar.
    """

    def __init__(self, parent, bg=CARD_BG, **kwargs):
        super().__init__(parent, bg=bg, **kwargs)

        self.canvas = tk.Canvas(self, bg=bg, highlightthickness=0, bd=0)
        # A little breathing room between the content and the scrollbar
        # so the layout doesn't feel cramped.
        gutter = tk.Frame(self, bg=bg, width=10)
        self.vscroll = RoundedScrollbar(self, command=self.canvas.yview, bg=bg)
        self.canvas.configure(yscrollcommand=self.vscroll.set)

        self.canvas.pack(side="left", fill="both", expand=True)
        gutter.pack(side="left", fill="y")
        self.vscroll.pack(side="right", fill="y")

        self.body = tk.Frame(self.canvas, bg=bg)
        self._window = self.canvas.create_window((0, 0), window=self.body, anchor="nw")

        self.body.bind("<Configure>", self._on_body_configure)
        self.canvas.bind("<Configure>", self._on_canvas_configure)

        # Mouse-wheel support: Windows/Mac fire <MouseWheel>, Linux fires
        # <Button-4>/<Button-5>. Only active while the pointer is over the
        # canvas so it doesn't hijack scrolling elsewhere in the app.
        self.canvas.bind("<Enter>", self._bind_mousewheel)
        self.canvas.bind("<Leave>", self._unbind_mousewheel)

    def _on_body_configure(self, event):
        self.canvas.configure(scrollregion=self.canvas.bbox("all"))

    def _on_canvas_configure(self, event):
        # Keep the inner frame the same width as the visible canvas so
        # child widgets (entries, buttons) stretch/wrap correctly.
        self.canvas.itemconfig(self._window, width=event.width)

    def _bind_mousewheel(self, event):
        self.canvas.bind_all("<MouseWheel>", self._on_mousewheel)
        self.canvas.bind_all("<Button-4>", self._on_mousewheel)
        self.canvas.bind_all("<Button-5>", self._on_mousewheel)

    def _unbind_mousewheel(self, event):
        self.canvas.unbind_all("<MouseWheel>")
        self.canvas.unbind_all("<Button-4>")
        self.canvas.unbind_all("<Button-5>")

    def _on_mousewheel(self, event):
        if event.num == 4:
            self.canvas.yview_scroll(-1, "units")
        elif event.num == 5:
            self.canvas.yview_scroll(1, "units")
        else:
            self.canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")


def section_title(parent, text, size=13):
    return tk.Label(parent, text=text, bg=parent["bg"], fg=DARK,
                     font=(FONT, size, "bold"))


def stat_box(parent, value, label, bg=PRIMARY_SOFT, fg=PRIMARY_DARK):
    f = tk.Frame(parent, bg=bg, padx=16, pady=12)
    tk.Label(f, text=value, bg=bg, fg=DARK, font=(FONT, 16, "bold")).pack(anchor="w")
    tk.Label(f, text=label, bg=bg, fg=GRAY, font=(FONT, 9)).pack(anchor="w")
    return f


def apply_global_style():
    style = ttk.Style()
    try:
        style.theme_use("clam")
    except tk.TclError:
        pass

    style.configure("Treeview", background=CARD_BG, fieldbackground=CARD_BG,
                     foreground=DARK, rowheight=28, borderwidth=0, font=(FONT, 10))
    style.configure("Treeview.Heading", background="#EEF7F1", foreground=DARK,
                     font=(FONT, 10, "bold"), relief="flat", padding=6)
    style.map("Treeview.Heading", background=[("active", "#E3F3E8")])
    style.map("Treeview", background=[("selected", PRIMARY_SOFT)],
              foreground=[("selected", DARK)])

    style.configure("TNotebook", background=BG, borderwidth=0)
    style.configure("TNotebook.Tab", padding=(16, 10), font=(FONT, 10, "bold"),
                     background="#EDEFF3", foreground=GRAY, borderwidth=0)
    style.map("TNotebook.Tab", background=[("selected", PRIMARY)],
              foreground=[("selected", "white")])

    style.configure("TCombobox", fieldbackground=INPUT_BG, background=INPUT_BG,
                     foreground=DARK, arrowcolor=PRIMARY_DARK, borderwidth=0)
    style.map("TCombobox", fieldbackground=[("readonly", INPUT_BG)])


# ======================================================================
# DATABASE LAYER
# ======================================================================

class Database:
    """Handles all SQLite access for the app."""

    def __init__(self, path=DB_FILE):
        self.conn = sqlite3.connect(path)
        self.conn.execute("PRAGMA foreign_keys = ON")
        self.create_tables()
        self.seed_admin()

    def create_tables(self):
        c = self.conn.cursor()
        c.executescript(
            """
            CREATE TABLE IF NOT EXISTS employees (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                phone TEXT,
                department TEXT,
                designation TEXT,
                join_date TEXT,
                role TEXT NOT NULL DEFAULT 'Employee',   -- Admin / HR / Employee
                status TEXT NOT NULL DEFAULT 'Active',   -- Active / Resigned / On Hold
                basic REAL DEFAULT 0,
                hra REAL DEFAULT 0,
                da REAL DEFAULT 0,
                other_allowance REAL DEFAULT 0,
                username TEXT UNIQUE,
                password TEXT
            );

            CREATE TABLE IF NOT EXISTS attendance (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                emp_id INTEGER NOT NULL,
                date TEXT NOT NULL,
                status TEXT NOT NULL,       -- Present / Absent / Half-day / Late
                overtime_hours REAL DEFAULT 0,
                FOREIGN KEY (emp_id) REFERENCES employees(id) ON DELETE CASCADE,
                UNIQUE(emp_id, date)
            );

            CREATE TABLE IF NOT EXISTS leaves (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                emp_id INTEGER NOT NULL,
                leave_type TEXT NOT NULL,   -- Casual / Sick / Earned
                from_date TEXT NOT NULL,
                to_date TEXT NOT NULL,
                reason TEXT,
                status TEXT NOT NULL DEFAULT 'Pending',  -- Pending / Approved / Rejected
                FOREIGN KEY (emp_id) REFERENCES employees(id) ON DELETE CASCADE
            );

            CREATE TABLE IF NOT EXISTS payroll (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                emp_id INTEGER NOT NULL,
                month INTEGER NOT NULL,
                year INTEGER NOT NULL,
                present_days INTEGER,
                gross REAL,
                pf REAL,
                esi REAL,
                prof_tax REAL,
                tds REAL,
                total_deductions REAL,
                net_pay REAL,
                status TEXT NOT NULL DEFAULT 'Processed',  -- Pending / Processed / Paid
                FOREIGN KEY (emp_id) REFERENCES employees(id) ON DELETE CASCADE,
                UNIQUE(emp_id, month, year)
            );
            """
        )
        self.conn.commit()

    def seed_admin(self):
        """Create a default admin login if no employees exist yet."""
        c = self.conn.cursor()
        c.execute("SELECT COUNT(*) FROM employees")
        if c.fetchone()[0] == 0:
            c.execute(
                """INSERT INTO employees
                   (name, phone, department, designation, join_date, role, status,
                    basic, hra, da, other_allowance, username, password)
                   VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?)""",
                ("System Admin", "", "Administration", "Administrator",
                 datetime.date.today().isoformat(), "Admin", "Active",
                 0, 0, 0, 0, "admin", "admin123"),
            )
            self.conn.commit()

    # -- generic helpers --
    def run(self, query, params=()):
        c = self.conn.cursor()
        c.execute(query, params)
        self.conn.commit()
        return c

    def fetch_all(self, query, params=()):
        c = self.conn.cursor()
        c.execute(query, params)
        return c.fetchall()

    def fetch_one(self, query, params=()):
        c = self.conn.cursor()
        c.execute(query, params)
        return c.fetchone()


# ======================================================================
# PAYROLL / TAX CALCULATION LOGIC
# ======================================================================

def calc_working_days(year, month):
    """Total calendar days in the month (kept simple for a capstone project)."""
    if month == 12:
        nxt = datetime.date(year + 1, 1, 1)
    else:
        nxt = datetime.date(year, month + 1, 1)
    first = datetime.date(year, month, 1)
    return (nxt - first).days


def calc_tds(annual_taxable_income):
    """Very simplified slab-based income tax (illustrative only, not legal advice)."""
    slabs = [
        (250000, 0.0),
        (500000, 0.05),
        (1000000, 0.20),
        (float("inf"), 0.30),
    ]
    tax = 0.0
    prev_limit = 0
    for limit, rate in slabs:
        if annual_taxable_income > prev_limit:
            taxable_in_slab = min(annual_taxable_income, limit) - prev_limit
            tax += taxable_in_slab * rate
            prev_limit = limit
        else:
            break
    return round(tax / 12, 2)  # monthly TDS


def calc_payroll_for_employee(db: Database, emp, month, year):
    """Compute gross, deductions and net pay for one employee for a given month."""
    emp_id = emp["id"]
    total_days = calc_working_days(year, month)

    rows = db.fetch_all(
        "SELECT status, overtime_hours FROM attendance WHERE emp_id=? AND "
        "strftime('%m', date)=? AND strftime('%Y', date)=?",
        (emp_id, f"{month:02d}", str(year)),
    )
    present_days = 0
    overtime_hours = 0.0
    for status, ot in rows:
        overtime_hours += ot or 0
        if status == "Present":
            present_days += 1
        elif status == "Half-day":
            present_days += 0.5
        elif status == "Late":
            present_days += 1  # still counted as full attendance

    if not rows:
        present_days = total_days

    per_day_basic = emp["basic"] / total_days if total_days else 0
    per_day_hra = emp["hra"] / total_days if total_days else 0
    per_day_da = emp["da"] / total_days if total_days else 0

    earned_basic = round(per_day_basic * present_days, 2)
    earned_hra = round(per_day_hra * present_days, 2)
    earned_da = round(per_day_da * present_days, 2)
    overtime_pay = round(overtime_hours * (per_day_basic / 8 if per_day_basic else 0) * 1.5, 2)

    gross = round(earned_basic + earned_hra + earned_da + emp["other_allowance"] + overtime_pay, 2)

    pf = round(earned_basic * 0.12, 2)
    esi = round(gross * 0.0075, 2) if gross <= 21000 else 0.0
    prof_tax = 200.0 if gross > 15000 else 0.0
    annual_gross = gross * 12
    tds = calc_tds(max(annual_gross - (pf * 12), 0))

    total_deductions = round(pf + esi + prof_tax + tds, 2)
    net_pay = round(gross - total_deductions, 2)

    return {
        "present_days": present_days,
        "gross": gross,
        "pf": pf,
        "esi": esi,
        "prof_tax": prof_tax,
        "tds": tds,
        "total_deductions": total_deductions,
        "net_pay": net_pay,
    }


MONTH_NAMES = ["January", "February", "March", "April", "May", "June", "July",
               "August", "September", "October", "November", "December"]


# ======================================================================
# GUI APPLICATION
# ======================================================================

class PayrollApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Payroll+  |  Employee Payroll Management System")
        self.geometry("1150x700")
        self.minsize(1000, 640)
        self.configure(bg=BG)
        apply_global_style()

        self.db = Database()
        self.current_user = None

        self.container = tk.Frame(self, bg=BG)
        self.container.pack(fill="both", expand=True)

        self.show_login()

    # ------------------------------------------------------------------
    # LOGIN  (styled like a clean auth card)
    # ------------------------------------------------------------------
    def show_login(self):
        for w in self.container.winfo_children():
            w.destroy()
        self.container.configure(bg=BG)

        outer = tk.Frame(self.container, bg=BG)
        outer.place(relx=0.5, rely=0.5, anchor="center")

        # Brand header
        brand = tk.Frame(outer, bg=BG)
        brand.pack(pady=(0, 18))
        logo = tk.Label(brand, text="\U0001F6E1", bg=BG, fg=PRIMARY, font=(FONT, 26))
        logo.pack(side="left", padx=(0, 8))
        tk.Label(brand, text="Payroll+", bg=BG, fg=DARK, font=(FONT, 22, "bold")).pack(side="left")

        auth_card = card(outer, padx=40, pady=36)
        auth_card.pack()

        tk.Label(auth_card, text="Sign in to Payroll+", bg=CARD_BG, fg=DARK,
                 font=(FONT, 16, "bold")).grid(row=0, column=0, columnspan=2, pady=(0, 4), sticky="w")
        tk.Label(auth_card, text="Manage employees, attendance & salaries in one place.",
                 bg=CARD_BG, fg=GRAY, font=(FONT, 9)).grid(row=1, column=0, columnspan=2, pady=(0, 22), sticky="w")

        field_label(auth_card, "Username").grid(row=2, column=0, sticky="w", pady=(0, 4))
        user_entry = styled_entry(auth_card, width=28)
        user_entry.grid(row=3, column=0, columnspan=2, ipady=6, pady=(0, 16), sticky="ew")
        user_entry.insert(0, "admin")

        field_label(auth_card, "Password").grid(row=4, column=0, sticky="w", pady=(0, 4))
        pass_entry = styled_entry(auth_card, width=28, show="*")
        pass_entry.grid(row=5, column=0, columnspan=2, ipady=6, pady=(0, 6), sticky="ew")
        pass_entry.insert(0, "admin123")

        msg = tk.Label(auth_card, text="", fg=DANGER, bg=CARD_BG, font=(FONT, 9))
        msg.grid(row=6, column=0, columnspan=2, sticky="w", pady=(0, 14))

        def do_login():
            u, p = user_entry.get().strip(), pass_entry.get().strip()
            row = self.db.fetch_one(
                "SELECT * FROM employees WHERE username=? AND password=?", (u, p)
            )
            if row:
                self.current_user = self.row_to_dict(row)
                self.show_main_app()
            else:
                msg.config(text="Invalid username or password.")

        btn = RoundButton(auth_card, "Sign in to Payroll+", command=do_login,
                           width=280, height=42, bg=PRIMARY)
        btn.grid(row=7, column=0, columnspan=2, pady=(4, 6))

        tk.Label(auth_card, text="Default admin login:  admin / admin123",
                 bg=CARD_BG, fg=GRAY, font=(FONT, 8)).grid(row=8, column=0, columnspan=2, pady=(10, 0))

        self.bind("<Return>", lambda e: do_login())

    def row_to_dict(self, row):
        cols = ["id", "name", "phone", "department", "designation", "join_date",
                "role", "status", "basic", "hra", "da", "other_allowance",
                "username", "password"]
        return dict(zip(cols, row))

    def logout(self):
        self.current_user = None
        self.show_login()

    # ------------------------------------------------------------------
    # MAIN APP SHELL — left sidebar navigation + content area
    # ------------------------------------------------------------------
    def show_main_app(self):
        for w in self.container.winfo_children():
            w.destroy()
        self.container.configure(bg=BG)

        is_admin_or_hr = self.current_user["role"] in ("Admin", "HR")

        # ---- Sidebar ----
        sidebar = tk.Frame(self.container, bg=SIDEBAR_BG, width=230,
                            highlightthickness=1, highlightbackground=BORDER)
        sidebar.pack(side="left", fill="y")
        sidebar.pack_propagate(False)

        brand = tk.Frame(sidebar, bg=SIDEBAR_BG)
        brand.pack(fill="x", pady=(22, 10), padx=20)
        tk.Label(brand, text="\U0001F6E1", bg=SIDEBAR_BG, fg=PRIMARY, font=(FONT, 20)).pack(side="left")
        tk.Label(brand, text="Payroll+", bg=SIDEBAR_BG, fg=DARK,
                 font=(FONT, 15, "bold")).pack(side="left", padx=6)

        user_box = tk.Frame(sidebar, bg=PRIMARY_SOFT)
        user_box.pack(fill="x", padx=16, pady=(6, 18))
        tk.Label(user_box, text=self.current_user["name"], bg=PRIMARY_SOFT, fg=DARK,
                 font=(FONT, 10, "bold"), anchor="w").pack(fill="x", padx=10, pady=(8, 0))
        tk.Label(user_box, text=self.current_user["role"], bg=PRIMARY_SOFT, fg=PRIMARY_DARK,
                 font=(FONT, 9), anchor="w").pack(fill="x", padx=10, pady=(0, 8))

        self.content = tk.Frame(self.container, bg=BG)
        self.content.pack(side="left", fill="both", expand=True)

        nav_items = [("\U0001F3E0", "Home", lambda: self.load_tab("home"))]
        if is_admin_or_hr:
            nav_items.append(("\U0001F465", "Employees", lambda: self.load_tab("employees")))
        nav_items.append(("\U0001F4C5", "Attendance & Leave", lambda: self.load_tab("attendance")))
        if is_admin_or_hr:
            nav_items.append(("\U0001F4B0", "Payroll", lambda: self.load_tab("payroll")))
            nav_items.append(("\U0001F9FE", "Tax & Deductions", lambda: self.load_tab("tax")))
        nav_items.append(("\U0001F4C4", "Reports & Payslips", lambda: self.load_tab("reports")))

        self.nav_buttons = {}
        for icon, label, action in nav_items:
            btn = tk.Button(sidebar, text=f"  {icon}   {label}", anchor="w",
                             bg=SIDEBAR_BG, fg=GRAY, activebackground=PRIMARY_SOFT,
                             activeforeground=PRIMARY_DARK, relief="flat", bd=0,
                             font=(FONT, 10), padx=10, pady=10, cursor="hand2",
                             command=action)
            btn.pack(fill="x", padx=12, pady=2)
            self.nav_buttons[label] = btn

        tk.Frame(sidebar, bg=SIDEBAR_BG).pack(fill="both", expand=True)  # spacer
        logout_btn = tk.Button(sidebar, text="  \u21A9  Logout", anchor="w", bg=SIDEBAR_BG,
                                fg=DANGER, relief="flat", bd=0, font=(FONT, 10),
                                padx=10, pady=10, cursor="hand2", command=self.logout)
        logout_btn.pack(fill="x", padx=12, pady=16, side="bottom")

        self.is_admin_or_hr = is_admin_or_hr
        self.load_tab("home")

    def highlight_nav(self, label):
        for lbl, btn in self.nav_buttons.items():
            if lbl == label:
                btn.configure(bg=PRIMARY_SOFT, fg=PRIMARY_DARK, font=(FONT, 10, "bold"))
            else:
                btn.configure(bg=SIDEBAR_BG, fg=GRAY, font=(FONT, 10))

    def load_tab(self, name):
        for w in self.content.winfo_children():
            w.destroy()

        if name == "home":
            self.highlight_nav("Home")
            DashboardTab(self.content, self.db, self.current_user, self.is_admin_or_hr).pack(fill="both", expand=True)
        elif name == "employees":
            self.highlight_nav("Employees")
            EmployeeTab(self.content, self.db).pack(fill="both", expand=True)
        elif name == "attendance":
            self.highlight_nav("Attendance & Leave")
            AttendanceLeaveTab(self.content, self.db, self.current_user, self.is_admin_or_hr).pack(fill="both", expand=True)
        elif name == "payroll":
            self.highlight_nav("Payroll")
            PayrollTab(self.content, self.db).pack(fill="both", expand=True)
        elif name == "tax":
            self.highlight_nav("Tax & Deductions")
            TaxTab(self.content, self.db).pack(fill="both", expand=True)
        elif name == "reports":
            self.highlight_nav("Reports & Payslips")
            ReportsTab(self.content, self.db, self.current_user, self.is_admin_or_hr).pack(fill="both", expand=True)


# ----------------------------------------------------------------------
# HOME / DASHBOARD
# ----------------------------------------------------------------------
class DashboardTab(tk.Frame):
    def __init__(self, parent, db: Database, user, is_admin_or_hr):
        super().__init__(parent, bg=BG)
        self.db = db
        self.user = user
        self.is_admin_or_hr = is_admin_or_hr
        self.build_ui()

    def build_ui(self):
        pad = tk.Frame(self, bg=BG, padx=28, pady=24)
        pad.pack(fill="both", expand=True)

        tk.Label(pad, text=f"Hello, {self.user['name'].split()[0]}!", bg=BG, fg=DARK,
                 font=(FONT, 20, "bold")).pack(anchor="w")
        tk.Label(pad, text="Here's your payroll dashboard at a glance.", bg=BG, fg=GRAY,
                 font=(FONT, 10)).pack(anchor="w", pady=(0, 20))

        stats = tk.Frame(pad, bg=BG)
        stats.pack(fill="x", pady=(0, 24))

        today = datetime.date.today()
        month, year = today.month, today.year

        if self.is_admin_or_hr:
            total_emp = self.db.fetch_one("SELECT COUNT(*) FROM employees WHERE status='Active'")[0]
            present_today = self.db.fetch_one(
                "SELECT COUNT(*) FROM attendance WHERE date=? AND status='Present'",
                (today.isoformat(),))[0]
            pending_leaves = self.db.fetch_one(
                "SELECT COUNT(*) FROM leaves WHERE status='Pending'")[0]
            net_sum = self.db.fetch_one(
                "SELECT COALESCE(SUM(net_pay),0) FROM payroll WHERE month=? AND year=?",
                (month, year))[0]

            boxes = [
                (str(total_emp), "Active Employees", PRIMARY_SOFT, PRIMARY_DARK),
                (str(present_today), "Present Today", "#DBEAFE", "#1D4ED8"),
                (str(pending_leaves), "Pending Leave Requests", "#FEF3C7", "#B45309"),
                (f"Rs. {net_sum:,.0f}", f"{MONTH_NAMES[month-1]} Net Payroll", "#FCE7F3", "#BE185D"),
            ]
        else:
            my_leaves = self.db.fetch_one(
                "SELECT COUNT(*) FROM leaves WHERE emp_id=? AND status='Pending'",
                (self.user["id"],))[0]
            my_pay = self.db.fetch_one(
                "SELECT net_pay FROM payroll WHERE emp_id=? AND month=? AND year=?",
                (self.user["id"], month, year))
            my_present = self.db.fetch_one(
                "SELECT COUNT(*) FROM attendance WHERE emp_id=? AND date LIKE ? AND status='Present'",
                (self.user["id"], f"{year}-{month:02d}-%"))[0]

            boxes = [
                (self.user["department"] or "-", "Department", PRIMARY_SOFT, PRIMARY_DARK),
                (str(my_present), "Days Present This Month", "#DBEAFE", "#1D4ED8"),
                (str(my_leaves), "Pending Leave Requests", "#FEF3C7", "#B45309"),
                (f"Rs. {my_pay[0]:,.0f}" if my_pay else "N/A", f"{MONTH_NAMES[month-1]} Net Pay",
                 "#FCE7F3", "#BE185D"),
            ]

        for value, label, bg, fg in boxes:
            box = stat_box(stats, value, label, bg=bg, fg=fg)
            box.pack(side="left", padx=(0, 14), fill="x", expand=True)

        # Recent activity card
        recent = card(pad, padx=18, pady=16)
        recent.pack(fill="both", expand=True)
        section_title(recent, "Recently Added Employees" if self.is_admin_or_hr
                      else "My Recent Attendance").pack(anchor="w", pady=(0, 10))

        columns = ("id", "name", "department", "status") if self.is_admin_or_hr else ("date", "status", "overtime_hours")
        tree = ttk.Treeview(recent, columns=columns, show="headings", height=8)
        for c in columns:
            tree.heading(c, text=c.replace("_", " ").title())
            tree.column(c, width=140)
        tree.pack(fill="both", expand=True)

        if self.is_admin_or_hr:
            rows = self.db.fetch_all(
                "SELECT id, name, department, status FROM employees ORDER BY id DESC LIMIT 8")
        else:
            rows = self.db.fetch_all(
                "SELECT date, status, overtime_hours FROM attendance WHERE emp_id=? "
                "ORDER BY date DESC LIMIT 8", (self.user["id"],))
        for row in rows:
            tree.insert("", "end", values=row)


# ----------------------------------------------------------------------
# TAB: EMPLOYEE MANAGEMENT
# ----------------------------------------------------------------------
class EmployeeTab(tk.Frame):
    def __init__(self, parent, db: Database):
        super().__init__(parent, bg=BG)
        self.db = db
        self.selected_id = None
        self.build_ui()
        self.refresh_list()

    def build_ui(self):
        wrap = tk.Frame(self, bg=BG, padx=24, pady=20)
        wrap.pack(fill="both", expand=True)

        header = tk.Frame(wrap, bg=BG)
        header.pack(fill="x", pady=(0, 14))
        tk.Label(header, text="Employee Management", bg=BG, fg=DARK,
                 font=(FONT, 16, "bold")).pack(side="left")

        body = tk.Frame(wrap, bg=BG)
        body.pack(fill="both", expand=True)

        # -- left: list card --
        left = card(body, padx=16, pady=16)
        left.pack(side="left", fill="both", expand=True, padx=(0, 16))

        search_frame = tk.Frame(left, bg=CARD_BG)
        search_frame.pack(fill="x", pady=(0, 10))
        self.search_var = tk.StringVar()
        se = styled_entry(search_frame, width=28)
        se.configure(textvariable=self.search_var)
        se.pack(side="left", ipady=5, fill="x", expand=True)
        tk.Button(search_frame, text="Search", command=self.refresh_list, relief="flat",
                  bg=PRIMARY_SOFT, fg=PRIMARY_DARK, font=(FONT, 9, "bold"), cursor="hand2"
                  ).pack(side="left", padx=6, ipady=4, ipadx=8)
        tk.Button(search_frame, text="Clear", command=self.clear_search, relief="flat",
                  bg=INPUT_BG, fg=GRAY, font=(FONT, 9), cursor="hand2"
                  ).pack(side="left", ipady=4, ipadx=8)

        columns = ("id", "name", "department", "designation", "role", "status")
        self.tree = ttk.Treeview(left, columns=columns, show="headings", height=20)
        for col in columns:
            self.tree.heading(col, text=col.capitalize())
            self.tree.column(col, width=105)
        self.tree.pack(fill="both", expand=True)
        self.tree.bind("<<TreeviewSelect>>", self.on_select)

        # -- right: form card (scrollable so every field and the action
        # buttons stay reachable even when the window/screen is short) --
        right = card(body, width=340)
        right.pack(side="right", fill="y")
        right.pack_propagate(False)

        right_scroll = ScrollableFrame(right, bg=CARD_BG)
        right_scroll.pack(fill="both", expand=True, padx=18, pady=18)
        form_root = right_scroll.body

        section_title(form_root, "Employee Details", size=12).pack(anchor="w", pady=(0, 12))

        form = tk.Frame(form_root, bg=CARD_BG)
        form.pack(fill="x")

        self.fields = {}
        labels = [
            ("name", "Full Name"), ("phone", "Phone"), ("department", "Department"),
            ("designation", "Designation"), ("join_date", "Join Date (YYYY-MM-DD)"),
            ("basic", "Basic Pay"), ("hra", "HRA"), ("da", "DA"),
            ("other_allowance", "Other Allowance"), ("username", "Username"),
            ("password", "Password"),
        ]
        for key, label in labels:
            field_label(form, label).pack(anchor="w", pady=(6, 2))
            e = styled_entry(form, width=30)
            e.pack(fill="x", ipady=4)
            self.fields[key] = e

        field_label(form, "Role").pack(anchor="w", pady=(6, 2))
        self.role_var = tk.StringVar(value="Employee")
        ttk.Combobox(form, textvariable=self.role_var, values=["Admin", "HR", "Employee"],
                     state="readonly").pack(fill="x")

        field_label(form, "Status").pack(anchor="w", pady=(6, 2))
        self.status_var = tk.StringVar(value="Active")
        ttk.Combobox(form, textvariable=self.status_var,
                     values=["Active", "Resigned", "On Hold"], state="readonly").pack(fill="x")

        btn_divider = tk.Frame(form_root, bg=BORDER, height=1)
        btn_divider.pack(fill="x", pady=(20, 16))

        btns = tk.Frame(form_root, bg=CARD_BG)
        btns.pack(fill="x", pady=(0, 4))
        RoundButton(btns, "Add New", command=self.add_employee, width=130, height=36,
                    bg=PRIMARY, font_size=10).grid(row=0, column=0, padx=(0, 6), pady=4)
        RoundButton(btns, "Update", command=self.update_employee, width=130, height=36,
                    bg="#3B82F6", font_size=10).grid(row=0, column=1, pady=4)
        RoundButton(btns, "Delete", command=self.delete_employee, width=130, height=36,
                    bg=DANGER, font_size=10).grid(row=1, column=0, padx=(0, 6), pady=4)
        RoundButton(btns, "Clear Form", command=self.clear_form, width=130, height=36,
                    bg="#9CA3AF", font_size=10).grid(row=1, column=1, pady=4)

    def clear_search(self):
        self.search_var.set("")
        self.refresh_list()

    def refresh_list(self):
        for r in self.tree.get_children():
            self.tree.delete(r)
        term = self.search_var.get().strip()
        if term:
            rows = self.db.fetch_all(
                "SELECT id, name, department, designation, role, status FROM employees "
                "WHERE name LIKE ? OR CAST(id AS TEXT) = ? OR department LIKE ? "
                "ORDER BY id",
                (f"%{term}%", term, f"%{term}%"),
            )
        else:
            rows = self.db.fetch_all(
                "SELECT id, name, department, designation, role, status FROM employees ORDER BY id"
            )
        for row in rows:
            self.tree.insert("", "end", values=row)

    def on_select(self, event):
        sel = self.tree.selection()
        if not sel:
            return
        emp_id = self.tree.item(sel[0])["values"][0]
        row = self.db.fetch_one("SELECT * FROM employees WHERE id=?", (emp_id,))
        if not row:
            return
        cols = ["id", "name", "phone", "department", "designation", "join_date",
                "role", "status", "basic", "hra", "da", "other_allowance",
                "username", "password"]
        data = dict(zip(cols, row))
        self.selected_id = data["id"]
        for key, entry in self.fields.items():
            entry.delete(0, tk.END)
            entry.insert(0, data.get(key) or "")
        self.role_var.set(data["role"])
        self.status_var.set(data["status"])

    def collect_form(self):
        try:
            return {
                "name": self.fields["name"].get().strip(),
                "phone": self.fields["phone"].get().strip(),
                "department": self.fields["department"].get().strip(),
                "designation": self.fields["designation"].get().strip(),
                "join_date": self.fields["join_date"].get().strip(),
                "basic": float(self.fields["basic"].get() or 0),
                "hra": float(self.fields["hra"].get() or 0),
                "da": float(self.fields["da"].get() or 0),
                "other_allowance": float(self.fields["other_allowance"].get() or 0),
                "username": self.fields["username"].get().strip(),
                "password": self.fields["password"].get().strip(),
                "role": self.role_var.get(),
                "status": self.status_var.get(),
            }
        except ValueError:
            messagebox.showerror("Error", "Salary fields must be numbers.")
            return None

    def add_employee(self):
        data = self.collect_form()
        if not data:
            return
        if not data["name"]:
            messagebox.showerror("Error", "Name is required.")
            return
        try:
            self.db.run(
                """INSERT INTO employees
                   (name, phone, department, designation, join_date, role, status,
                    basic, hra, da, other_allowance, username, password)
                   VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?)""",
                (data["name"], data["phone"], data["department"], data["designation"],
                 data["join_date"], data["role"], data["status"], data["basic"],
                 data["hra"], data["da"], data["other_allowance"],
                 data["username"] or None, data["password"] or None),
            )
            messagebox.showinfo("Success", "Employee added.")
            self.clear_form()
            self.refresh_list()
        except sqlite3.IntegrityError as e:
            messagebox.showerror("Error", f"Could not add employee: {e}")

    def update_employee(self):
        if not self.selected_id:
            messagebox.showwarning("No selection", "Select an employee from the list first.")
            return
        data = self.collect_form()
        if not data:
            return
        try:
            self.db.run(
                """UPDATE employees SET name=?, phone=?, department=?, designation=?,
                   join_date=?, role=?, status=?, basic=?, hra=?, da=?, other_allowance=?,
                   username=?, password=? WHERE id=?""",
                (data["name"], data["phone"], data["department"], data["designation"],
                 data["join_date"], data["role"], data["status"], data["basic"],
                 data["hra"], data["da"], data["other_allowance"],
                 data["username"] or None, data["password"] or None, self.selected_id),
            )
            messagebox.showinfo("Success", "Employee updated.")
            self.refresh_list()
        except sqlite3.IntegrityError as e:
            messagebox.showerror("Error", f"Could not update employee: {e}")

    def delete_employee(self):
        if not self.selected_id:
            messagebox.showwarning("No selection", "Select an employee from the list first.")
            return
        if messagebox.askyesno("Confirm", "Delete this employee? This also removes their "
                                           "attendance, leave and payroll records."):
            self.db.run("DELETE FROM employees WHERE id=?", (self.selected_id,))
            self.clear_form()
            self.refresh_list()

    def clear_form(self):
        self.selected_id = None
        for e in self.fields.values():
            e.delete(0, tk.END)
        self.role_var.set("Employee")
        self.status_var.set("Active")


# ----------------------------------------------------------------------
# TAB: ATTENDANCE & LEAVE
# ----------------------------------------------------------------------
class AttendanceLeaveTab(tk.Frame):
    def __init__(self, parent, db: Database, user, is_admin_or_hr):
        super().__init__(parent, bg=BG)
        self.db = db
        self.user = user
        self.is_admin_or_hr = is_admin_or_hr
        self.build_ui()

    def build_ui(self):
        wrap = tk.Frame(self, bg=BG, padx=24, pady=20)
        wrap.pack(fill="both", expand=True)
        tk.Label(wrap, text="Attendance & Leave", bg=BG, fg=DARK,
                 font=(FONT, 16, "bold")).pack(anchor="w", pady=(0, 14))

        nb = ttk.Notebook(wrap)
        nb.pack(fill="both", expand=True)

        att_frame = tk.Frame(nb, bg=BG, padx=6, pady=12)
        nb.add(att_frame, text="Mark Attendance")
        self.build_attendance_ui(att_frame)

        leave_frame = tk.Frame(nb, bg=BG, padx=6, pady=12)
        nb.add(leave_frame, text="Leave Requests")
        self.build_leave_ui(leave_frame)

    # -- attendance --
    def build_attendance_ui(self, frame):
        top = card(frame, padx=16, pady=14)
        top.pack(fill="x", pady=(0, 14))

        row = tk.Frame(top, bg=CARD_BG)
        row.pack(fill="x")

        if self.is_admin_or_hr:
            field_label(row, "Employee ID").pack(side="left")
            self.att_emp_id = styled_entry(row, width=6)
            self.att_emp_id.pack(side="left", padx=(6, 16), ipady=4)
        else:
            self.att_emp_id = None

        field_label(row, "Date").pack(side="left")
        self.att_date = styled_entry(row, width=12)
        self.att_date.insert(0, datetime.date.today().isoformat())
        self.att_date.pack(side="left", padx=(6, 16), ipady=4)

        field_label(row, "Status").pack(side="left")
        self.att_status = ttk.Combobox(row, values=["Present", "Absent", "Half-day", "Late"],
                                        width=10, state="readonly")
        self.att_status.set("Present")
        self.att_status.pack(side="left", padx=(6, 16))

        field_label(row, "OT hrs").pack(side="left")
        self.att_ot = styled_entry(row, width=6)
        self.att_ot.insert(0, "0")
        self.att_ot.pack(side="left", padx=(6, 16), ipady=4)

        RoundButton(row, "Mark Attendance", command=self.mark_attendance,
                    width=150, height=34, font_size=9).pack(side="left", padx=6)

        list_card = card(frame, padx=14, pady=14)
        list_card.pack(fill="both", expand=True)
        columns = ("id", "emp_id", "date", "status", "overtime_hours")
        self.att_tree = ttk.Treeview(list_card, columns=columns, show="headings", height=15)
        for c in columns:
            self.att_tree.heading(c, text=c.replace("_", " ").title())
            self.att_tree.column(c, width=110)
        self.att_tree.pack(fill="both", expand=True)

        self.refresh_attendance()

    def mark_attendance(self):
        emp_id = self.att_emp_id.get().strip() if self.att_emp_id else str(self.user["id"])
        date = self.att_date.get().strip()
        status = self.att_status.get()
        try:
            ot = float(self.att_ot.get() or 0)
            emp_id = int(emp_id)
        except ValueError:
            messagebox.showerror("Error", "Employee ID and OT hours must be numbers.")
            return
        try:
            self.db.run(
                "INSERT OR REPLACE INTO attendance (emp_id, date, status, overtime_hours) "
                "VALUES (?,?,?,?)",
                (emp_id, date, status, ot),
            )
            self.refresh_attendance()
        except sqlite3.IntegrityError as e:
            messagebox.showerror("Error", str(e))

    def refresh_attendance(self):
        for r in self.att_tree.get_children():
            self.att_tree.delete(r)
        if self.is_admin_or_hr:
            rows = self.db.fetch_all(
                "SELECT id, emp_id, date, status, overtime_hours FROM attendance "
                "ORDER BY date DESC LIMIT 200"
            )
        else:
            rows = self.db.fetch_all(
                "SELECT id, emp_id, date, status, overtime_hours FROM attendance "
                "WHERE emp_id=? ORDER BY date DESC LIMIT 200", (self.user["id"],)
            )
        for row in rows:
            self.att_tree.insert("", "end", values=row)

    # -- leave --
    def build_leave_ui(self, frame):
        if not self.is_admin_or_hr:
            top = card(frame, padx=16, pady=14)
            top.pack(fill="x", pady=(0, 14))
            row = tk.Frame(top, bg=CARD_BG)
            row.pack(fill="x")

            field_label(row, "Type").pack(side="left")
            self.leave_type = ttk.Combobox(row, values=["Casual", "Sick", "Earned"],
                                            width=9, state="readonly")
            self.leave_type.set("Casual")
            self.leave_type.pack(side="left", padx=(6, 16))

            field_label(row, "From").pack(side="left")
            self.leave_from = styled_entry(row, width=11)
            self.leave_from.pack(side="left", padx=(6, 16), ipady=4)

            field_label(row, "To").pack(side="left")
            self.leave_to = styled_entry(row, width=11)
            self.leave_to.pack(side="left", padx=(6, 16), ipady=4)

            field_label(row, "Reason").pack(side="left")
            self.leave_reason = styled_entry(row, width=18)
            self.leave_reason.pack(side="left", padx=(6, 16), ipady=4)

            RoundButton(row, "Apply for Leave", command=self.apply_leave,
                        width=150, height=34, font_size=9).pack(side="left")

        list_card = card(frame, padx=14, pady=14)
        list_card.pack(fill="both", expand=True)
        columns = ("id", "emp_id", "leave_type", "from_date", "to_date", "reason", "status")
        self.leave_tree = ttk.Treeview(list_card, columns=columns, show="headings", height=13)
        for c in columns:
            self.leave_tree.heading(c, text=c.replace("_", " ").title())
            self.leave_tree.column(c, width=95)
        self.leave_tree.pack(fill="both", expand=True, pady=(0, 10))

        if self.is_admin_or_hr:
            btns = tk.Frame(list_card, bg=CARD_BG)
            btns.pack()
            RoundButton(btns, "Approve Selected", bg=PRIMARY,
                        command=lambda: self.set_leave_status("Approved"),
                        width=160, height=34, font_size=9).pack(side="left", padx=5)
            RoundButton(btns, "Reject Selected", bg=DANGER,
                        command=lambda: self.set_leave_status("Rejected"),
                        width=160, height=34, font_size=9).pack(side="left", padx=5)

        self.refresh_leaves()

    def apply_leave(self):
        ltype = self.leave_type.get()
        f, t = self.leave_from.get().strip(), self.leave_to.get().strip()
        reason = self.leave_reason.get().strip()
        if not f or not t:
            messagebox.showerror("Error", "Please enter both from and to dates.")
            return
        self.db.run(
            "INSERT INTO leaves (emp_id, leave_type, from_date, to_date, reason, status) "
            "VALUES (?,?,?,?,?, 'Pending')",
            (self.user["id"], ltype, f, t, reason),
        )
        messagebox.showinfo("Submitted", "Leave request submitted.")
        self.refresh_leaves()

    def set_leave_status(self, status):
        sel = self.leave_tree.selection()
        if not sel:
            messagebox.showwarning("No selection", "Select a leave request first.")
            return
        leave_id = self.leave_tree.item(sel[0])["values"][0]
        self.db.run("UPDATE leaves SET status=? WHERE id=?", (status, leave_id))
        self.refresh_leaves()

    def refresh_leaves(self):
        for r in self.leave_tree.get_children():
            self.leave_tree.delete(r)
        if self.is_admin_or_hr:
            rows = self.db.fetch_all("SELECT * FROM leaves ORDER BY id DESC")
        else:
            rows = self.db.fetch_all(
                "SELECT * FROM leaves WHERE emp_id=? ORDER BY id DESC", (self.user["id"],)
            )
        for row in rows:
            self.leave_tree.insert("", "end", values=row)


# ----------------------------------------------------------------------
# TAB: PAYROLL PROCESSING
# ----------------------------------------------------------------------
class PayrollTab(tk.Frame):
    def __init__(self, parent, db: Database):
        super().__init__(parent, bg=BG)
        self.db = db
        self.build_ui()

    def build_ui(self):
        wrap = tk.Frame(self, bg=BG, padx=24, pady=20)
        wrap.pack(fill="both", expand=True)
        tk.Label(wrap, text="Payroll Processing", bg=BG, fg=DARK,
                 font=(FONT, 16, "bold")).pack(anchor="w", pady=(0, 14))

        top = card(wrap, padx=16, pady=14)
        top.pack(fill="x", pady=(0, 14))
        row = tk.Frame(top, bg=CARD_BG)
        row.pack(fill="x")

        field_label(row, "Month").pack(side="left")
        self.month_cb = ttk.Combobox(row, values=MONTH_NAMES, width=12, state="readonly")
        self.month_cb.current(datetime.date.today().month - 1)
        self.month_cb.pack(side="left", padx=(6, 16))

        field_label(row, "Year").pack(side="left")
        self.year_entry = styled_entry(row, width=6)
        self.year_entry.insert(0, str(datetime.date.today().year))
        self.year_entry.pack(side="left", padx=(6, 16), ipady=4)

        RoundButton(row, "Run Payroll for Active Employees", command=self.run_payroll,
                    width=260, height=36, font_size=9).pack(side="left", padx=6)

        list_card = card(wrap, padx=14, pady=14)
        list_card.pack(fill="both", expand=True)
        columns = ("id", "emp_id", "month", "year", "present_days", "gross",
                   "total_deductions", "net_pay", "status")
        self.tree = ttk.Treeview(list_card, columns=columns, show="headings", height=17)
        for c in columns:
            self.tree.heading(c, text=c.replace("_", " ").title())
            self.tree.column(c, width=90)
        self.tree.pack(fill="both", expand=True, pady=(0, 10))

        RoundButton(list_card, "Mark Selected as Paid", command=self.mark_paid,
                    width=200, height=34, bg="#3B82F6", font_size=9).pack()

        self.refresh()

    def run_payroll(self):
        month = self.month_cb.current() + 1
        try:
            year = int(self.year_entry.get())
        except ValueError:
            messagebox.showerror("Error", "Year must be a number.")
            return

        employees = self.db.fetch_all("SELECT * FROM employees WHERE status='Active'")
        cols = ["id", "name", "phone", "department", "designation", "join_date",
                "role", "status", "basic", "hra", "da", "other_allowance",
                "username", "password"]
        count = 0
        for row in employees:
            emp = dict(zip(cols, row))
            result = calc_payroll_for_employee(self.db, emp, month, year)
            self.db.run(
                """INSERT OR REPLACE INTO payroll
                   (emp_id, month, year, present_days, gross, pf, esi, prof_tax, tds,
                    total_deductions, net_pay, status)
                   VALUES (?,?,?,?,?,?,?,?,?,?,?, 'Processed')""",
                (emp["id"], month, year, result["present_days"], result["gross"],
                 result["pf"], result["esi"], result["prof_tax"], result["tds"],
                 result["total_deductions"], result["net_pay"]),
            )
            count += 1
        messagebox.showinfo("Payroll Run Complete",
                             f"Payroll processed for {count} active employee(s) for "
                             f"{MONTH_NAMES[month-1]} {year}.")
        self.refresh()

    def mark_paid(self):
        sel = self.tree.selection()
        if not sel:
            messagebox.showwarning("No selection", "Select a payroll row first.")
            return
        pid = self.tree.item(sel[0])["values"][0]
        self.db.run("UPDATE payroll SET status='Paid' WHERE id=?", (pid,))
        self.refresh()

    def refresh(self):
        for r in self.tree.get_children():
            self.tree.delete(r)
        rows = self.db.fetch_all(
            "SELECT id, emp_id, month, year, present_days, gross, total_deductions, "
            "net_pay, status FROM payroll ORDER BY year DESC, month DESC, emp_id"
        )
        for row in rows:
            self.tree.insert("", "end", values=row)


# ----------------------------------------------------------------------
# TAB: TAX & DEDUCTIONS
# ----------------------------------------------------------------------
class TaxTab(tk.Frame):
    def __init__(self, parent, db: Database):
        super().__init__(parent, bg=BG)
        self.db = db
        self.build_ui()

    def build_ui(self):
        wrap = tk.Frame(self, bg=BG, padx=24, pady=20)
        wrap.pack(fill="both", expand=True)
        tk.Label(wrap, text="Tax & Deductions", bg=BG, fg=DARK,
                 font=(FONT, 16, "bold")).pack(anchor="w", pady=(0, 14))

        info = card(wrap, padx=18, pady=16)
        info.pack(fill="x", pady=(0, 14))
        tk.Label(info, justify="left", anchor="w", bg=CARD_BG, fg=DARK, font=(FONT, 10),
                 text=(
                     "Deduction rules used by Payroll Processing:\n\n"
                     "  •  Provident Fund (PF): 12% of earned Basic pay\n"
                     "  •  ESI: 0.75% of gross pay, only if gross <= Rs. 21,000\n"
                     "  •  Professional Tax: flat Rs. 200 if gross > Rs. 15,000\n"
                     "  •  Income Tax (TDS): simplified annual slabs, divided by 12:\n"
                     "        Up to Rs. 2,50,000          -> 0%\n"
                     "        Rs. 2,50,001 - Rs. 5,00,000  -> 5%\n"
                     "        Rs. 5,00,001 - Rs. 10,00,000 -> 20%\n"
                     "        Above Rs. 10,00,000          -> 30%\n\n"
                     "This is a simplified model for demonstration purposes only and is "
                     "not a substitute for official tax rules."
                 )).pack(anchor="w", fill="x")

        list_card = card(wrap, padx=14, pady=14)
        list_card.pack(fill="both", expand=True)
        section_title(list_card, "Deductions Breakdown by Processed Payroll", size=11).pack(
            anchor="w", pady=(0, 8))

        columns = ("emp_id", "month", "year", "pf", "esi", "prof_tax", "tds", "total_deductions")
        self.tree = ttk.Treeview(list_card, columns=columns, show="headings", height=13)
        for c in columns:
            self.tree.heading(c, text=c.replace("_", " ").title())
            self.tree.column(c, width=100)
        self.tree.pack(fill="both", expand=True)

        self.refresh()

    def refresh(self):
        for r in self.tree.get_children():
            self.tree.delete(r)
        rows = self.db.fetch_all(
            "SELECT emp_id, month, year, pf, esi, prof_tax, tds, total_deductions "
            "FROM payroll ORDER BY year DESC, month DESC, emp_id"
        )
        for row in rows:
            self.tree.insert("", "end", values=row)


# ----------------------------------------------------------------------
# TAB: REPORTS & PAYSLIPS
# ----------------------------------------------------------------------
class ReportsTab(tk.Frame):
    def __init__(self, parent, db: Database, user, is_admin_or_hr):
        super().__init__(parent, bg=BG)
        self.db = db
        self.user = user
        self.is_admin_or_hr = is_admin_or_hr
        self.build_ui()

    def build_ui(self):
        wrap = tk.Frame(self, bg=BG, padx=24, pady=20)
        wrap.pack(fill="both", expand=True)
        tk.Label(wrap, text="Reports & Payslips", bg=BG, fg=DARK,
                 font=(FONT, 16, "bold")).pack(anchor="w", pady=(0, 14))

        top = card(wrap, padx=16, pady=14)
        top.pack(fill="x", pady=(0, 14))
        row = tk.Frame(top, bg=CARD_BG)
        row.pack(fill="x")

        if self.is_admin_or_hr:
            field_label(row, "Employee ID").pack(side="left")
            self.emp_id_entry = styled_entry(row, width=6)
            self.emp_id_entry.pack(side="left", padx=(6, 16), ipady=4)
        else:
            self.emp_id_entry = None

        field_label(row, "Month").pack(side="left")
        self.month_cb = ttk.Combobox(row, values=MONTH_NAMES, width=12, state="readonly")
        self.month_cb.current(datetime.date.today().month - 1)
        self.month_cb.pack(side="left", padx=(6, 16))

        field_label(row, "Year").pack(side="left")
        self.year_entry = styled_entry(row, width=6)
        self.year_entry.insert(0, str(datetime.date.today().year))
        self.year_entry.pack(side="left", padx=(6, 16), ipady=4)

        RoundButton(row, "Generate Payslip", command=self.generate_payslip,
                    width=170, height=34, font_size=9).pack(side="left", padx=6)

        if self.is_admin_or_hr:
            RoundButton(row, "Department Report", command=self.dept_report, bg="#3B82F6",
                        width=170, height=34, font_size=9).pack(side="left", padx=6)

        out_card = card(wrap, padx=16, pady=16)
        out_card.pack(fill="both", expand=True)
        self.output = tk.Text(out_card, height=24, wrap="word", font=("Consolas", 10),
                               bg=CARD_BG, fg=DARK, relief="flat", highlightthickness=0)
        self.output.pack(fill="both", expand=True)

    def generate_payslip(self):
        emp_id = self.emp_id_entry.get().strip() if self.emp_id_entry else str(self.user["id"])
        month = self.month_cb.current() + 1
        try:
            year = int(self.year_entry.get())
            emp_id = int(emp_id)
        except ValueError:
            messagebox.showerror("Error", "Employee ID and year must be numbers.")
            return

        emp = self.db.fetch_one("SELECT name, department, designation FROM employees WHERE id=?", (emp_id,))
        pay = self.db.fetch_one(
            "SELECT present_days, gross, pf, esi, prof_tax, tds, total_deductions, net_pay, status "
            "FROM payroll WHERE emp_id=? AND month=? AND year=?", (emp_id, month, year)
        )
        if not emp or not pay:
            messagebox.showwarning("Not found",
                                    "No processed payroll found for this employee/month/year. "
                                    "Run Payroll Processing first.")
            return

        name, dept, desig = emp
        (present_days, gross, pf, esi, prof_tax, tds, total_ded, net_pay, status) = pay

        text = f"""
==================================================
              PAYSLIP - {MONTH_NAMES[month-1]} {year}
==================================================
Employee ID   : {emp_id}
Name          : {name}
Department    : {dept}
Designation   : {desig}
Present Days  : {present_days}
Status        : {status}
--------------------------------------------------
EARNINGS
  Gross Pay          : {gross:>12,.2f}
--------------------------------------------------
DEDUCTIONS
  Provident Fund (PF): {pf:>12,.2f}
  ESI                : {esi:>12,.2f}
  Professional Tax   : {prof_tax:>12,.2f}
  Income Tax (TDS)   : {tds:>12,.2f}
  ------------------------------------
  Total Deductions   : {total_ded:>12,.2f}
--------------------------------------------------
NET PAY             : {net_pay:>12,.2f}
==================================================
"""
        self.output.delete("1.0", tk.END)
        self.output.insert(tk.END, text)

        if HAS_REPORTLAB:
            self.save_pdf(emp_id, name, month, year, dept, desig, present_days, status,
                           gross, pf, esi, prof_tax, tds, total_ded, net_pay)
        else:
            self.save_txt(emp_id, month, year, text)

    def save_pdf(self, emp_id, name, month, year, dept, desig, present_days, status,
                 gross, pf, esi, prof_tax, tds, total_ded, net_pay):
        os.makedirs("payslips", exist_ok=True)
        fname = f"payslips/payslip_{emp_id}_{year}_{month:02d}.pdf"
        c = pdf_canvas.Canvas(fname, pagesize=A4)
        width, height = A4
        y = height - 30 * mm

        c.setFont("Helvetica-Bold", 14)
        c.drawCentredString(width / 2, y, f"PAYSLIP - {MONTH_NAMES[month-1]} {year}")
        y -= 12 * mm

        c.setFont("Helvetica", 10)
        lines = [
            f"Employee ID: {emp_id}      Name: {name}",
            f"Department: {dept}      Designation: {desig}",
            f"Present Days: {present_days}      Status: {status}",
            "",
            f"Gross Pay: Rs. {gross:,.2f}",
            "",
            "Deductions:",
            f"  Provident Fund (PF): Rs. {pf:,.2f}",
            f"  ESI: Rs. {esi:,.2f}",
            f"  Professional Tax: Rs. {prof_tax:,.2f}",
            f"  Income Tax (TDS): Rs. {tds:,.2f}",
            f"  Total Deductions: Rs. {total_ded:,.2f}",
            "",
            f"NET PAY: Rs. {net_pay:,.2f}",
        ]
        for line in lines:
            c.drawString(25 * mm, y, line)
            y -= 7 * mm
        c.save()
        self.output.insert(tk.END, f"\n[Saved PDF payslip to: {os.path.abspath(fname)}]\n")

    def save_txt(self, emp_id, month, year, text):
        os.makedirs("payslips", exist_ok=True)
        fname = f"payslips/payslip_{emp_id}_{year}_{month:02d}.txt"
        with open(fname, "w") as f:
            f.write(text)
        self.output.insert(tk.END, f"\n[reportlab not installed - saved plain text payslip to: "
                                    f"{os.path.abspath(fname)}]\n[Run: pip install reportlab  "
                                    f"for real PDF payslips]\n")

    def dept_report(self):
        month = self.month_cb.current() + 1
        try:
            year = int(self.year_entry.get())
        except ValueError:
            messagebox.showerror("Error", "Year must be a number.")
            return

        rows = self.db.fetch_all(
            """SELECT e.department, COUNT(*), SUM(p.gross), SUM(p.total_deductions), SUM(p.net_pay)
               FROM payroll p JOIN employees e ON p.emp_id = e.id
               WHERE p.month=? AND p.year=?
               GROUP BY e.department ORDER BY e.department""",
            (month, year),
        )
        if not rows:
            messagebox.showinfo("No data", "No payroll data for this month/year yet.")
            return

        text = f"\nDEPARTMENT-WISE SALARY REPORT - {MONTH_NAMES[month-1]} {year}\n"
        text += "=" * 60 + "\n"
        text += f"{'Department':<20}{'Employees':<12}{'Gross':<15}{'Net Pay':<15}\n"
        text += "-" * 60 + "\n"
        for dept, cnt, gross_sum, ded_sum, net_sum in rows:
            text += f"{dept or 'N/A':<20}{cnt:<12}{gross_sum:<15,.2f}{net_sum:<15,.2f}\n"
        self.output.delete("1.0", tk.END)
        self.output.insert(tk.END, text)


# ======================================================================
# ENTRY POINT
# ======================================================================
if __name__ == "__main__":
    app = PayrollApp()
    app.mainloop()
