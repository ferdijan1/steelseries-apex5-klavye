Herkese Merhabalar;
SteelSeries apex 5 Oled ekranlı bir klavye aldım ancak uygulamasında oled i özelleştirebileceğim çok bir seçenek olmadığını fark ettim bunun üzerine claude ai ile birlikte bir uygulama oluşturdum test ettim şimdilik aklıma gelen çoğu özelliği koydum gerekli gereksiz :D 
Yapmışken de paylaşmak istedim
Ancak şimdilik fikirlerim tükendi daha da geliştirmek isterim ama şuan ki haliyle bile büyük fark var diyebilirim.
Kullanım ve yorumlarınızı 4 gözle bekliyorum Teşekkürler...


Kullanım için ApexOLED zip dosyasını masaüstüne klasör olarak oluşturup klasörün içinde "dist" klasörü görüceksiniz içine girdiğinizde exe halini görebilirsiniz. Çift tıklayarak kullanıma başlayabilirsiniz.

Claude un oluşturduğu bütün yazı kodları buraya aktarıyorum kontrol edebilirsiniz!








"""
Apex OLED Tool v8
SteelSeries Apex 5 OLED ekranı için: uygulamaya göre yazı/görsel, arka planda çalan medya,
öncelik sistemi, kayan yazı, temalar, Windows ile otomatik başlatma.

Gereksinim: Windows 10/11, SteelSeries GG açık
  pip install customtkinter psutil requests pillow icoextract pystray pycaw comtypes
  pip install winrt-runtime winrt-Windows.Media.Control winrt-Windows.Foundation winrt-Windows.Foundation.Collections winrt-Windows.Storage.Streams
"""
import json
import math
import os
import re
import copy
import queue
import shutil
import subprocess
import sys
import threading
import time
import ctypes
import tkinter as tk
from ctypes import wintypes
from tkinter import filedialog, messagebox

import customtkinter as ctk
import psutil
import requests
from PIL import Image, ImageDraw, ImageFont, ImageOps, ImageTk

try:
    from icoextract import IconExtractor
except Exception:
    IconExtractor = None

try:
    import winreg
except Exception:
    winreg = None

try:
    import pystray
except Exception:
    pystray = None

APP = "ApexOLED"
GAME = "APEX_OLED_TOOL"
CORE_PROPS = r"C:\ProgramData\SteelSeries\SteelSeries Engine 3\coreProps.json"
DATA_DIR = os.path.join(os.environ.get("APPDATA", "."), APP)
IMG_DIR = os.path.join(DATA_DIR, "images")
ICON_DIR = os.path.join(DATA_DIR, "icons")
CFG_PATH = os.path.join(DATA_DIR, "config.json")
HEARTBEAT = 3
OWN_NAMES = {"apexoled.exe", "python.exe", "pythonw.exe"}
RUN_KEY = r"Software\Microsoft\Windows\CurrentVersion\Run"

CATEGORIES = {
    "Oyun": "#ef4444",
    "Tarayıcı": "#3b82f6",
    "Sosyal": "#a855f7",
    "Müzik": "#22c55e",
    "Çalışma": "#f59e0b",
    "Sistem": "#06b6d4",
    "Diğer": "#64748b",
}
DEFAULT_CATEGORY = "Diğer"

MODE_LIST = [("Yazı", "text"), ("Medya", "media"), ("Başlık", "title"), ("Sistem", "sys")]
MODE_DESC = {
    "text": "Sabit bir yazı gösterir. Örn: CHROME",
    "media": "Çalan şarkı / video adını gösterir (Windows medya bilgisi).",
    "title": "Pencere ya da sekme başlığını gösterir. Anime adı gibi bilgiler için.",
    "sys": "CPU, GPU, RAM gibi sistem bilgilerini gösterir (Sistem sekmesinden ayarlanır).",
}
MODE_SHORT = {"text": "sabit metin", "media": "çalan medya", "title": "pencere başlığı", "sys": "sistem bilgisi"}

SIZES = {"Küçük": 16, "Orta": 22, "Büyük": 28}
WEIGHTS = {"Normal": 0, "Kalın": 1, "Çok kalın": 2}
STYLES = {"Sürekli akış": "loop", "Duraklamalı akış": "pause", "Git-gel": "pingpong"}

PRIORITIES = [
    ("media", "Medya öncelikli",
     "Müzik ya da film çalıyorsa her zaman onu göster. Çalmıyorsa uygulama kuralını göster."),
    ("app", "Öndeki uygulama öncelikli",
     "Kuralı olan bir programdaysan onu göster. Kuralsız bir programdaysan çalan medyayı göster."),
    ("rotate", "Dönüşümlü",
     "İkisi de varsa belirlediğin sürede sırayla göster."),
    ("category", "Kategori öncelikli",
     "Öndeki program seçtiğin kategorideyse (örneğin Oyun) onu göster, değilse çalan medyayı göster."),
]

IDLE_MODES = {"Hiçbir şey": "none", "Saat": "clock"}
SYS_WHEN = {"Sadece kurallarda": "rules", "Hiçbir şey yokken": "idle", "Belirli aralıklarla": "interval"}
SYS_WHEN_DESC = {
    "rules": "Sadece Kurallar sekmesinde 'Sistem' modunu seçtiğin programlarda gösterilir.",
    "idle": "Ekranda başka bir şey yokken (kural da müzik de yoksa) sürekli gösterilir.",
    "interval": "Seçtiğin aralıkla kısa süre görünüp kaybolur. Oyun ya da müzik yazısının üstünde çıkar.",
}
SYS_EVERY = {"1 dakikada bir": 1, "2 dakikada bir": 2, "5 dakikada bir": 5, "10 dakikada bir": 10,
             "15 dakikada bir": 15, "30 dakikada bir": 30}
VIEW_MODES = ["Kartlar", "Liste", "Gruplu"]
SYS_MODES = {"Dönüşümlü (tek tek)": "rotate", "Tek satırda birleştir": "joined"}
VOL_STYLES = ["Yazı", "Çubuk", "Yazı + çubuk"]

# (anahtar, ad, varsayılan şablon, varsayılan açık mı)
SYS_DEFS = [
    ("cpu", "CPU kullanımı", "CPU %{v}", True),
    ("cpu_temp", "CPU sıcaklığı", "CPU {v}°C", True),
    ("cpu_clock", "CPU hızı", "{v} GHz", False),
    ("ram", "RAM kullanımı (%)", "RAM %{v}", True),
    ("ram_used", "RAM kullanılan (GB)", "RAM {v} GB", False),
    ("gpu", "GPU kullanımı", "GPU %{v}", True),
    ("gpu_temp", "GPU sıcaklığı", "GPU {v}°C", True),
    ("gpu_mem", "VRAM kullanımı (%)", "VRAM %{v}", False),
    ("gpu_power", "GPU gücü (W)", "GPU {v} W", False),
    ("disk", "Disk doluluğu", "Disk %{v}", False),
    ("net_down", "İndirme hızı", "↓ {v}", False),
    ("net_up", "Yükleme hızı", "↑ {v}", False),
    ("battery", "Pil", "Pil %{v}", False),
    ("clock", "Saat", "{v}", False),
    ("uptime", "Açık kalma süresi", "Açık {v}", False),
]

# profil / kategori ayar alanları: ad, tür, seçenekler, varsayılan
FIELDS = {
    "size": ("Yazı boyutu", "seg", list(SIZES), "Orta"),
    "weight": ("Yazı kalınlığı", "seg", list(WEIGHTS), "Kalın"),
    "scroll": ("Uzun yazıyı kaydır", "sw", None, True),
    "style": ("Kaydırma şekli", "opt", list(STYLES), "Sürekli akış"),
    "speed": ("Kaydırma hızı", "sl", (10, 120, 110, "{:.0f} px/sn", int), 40),
    "pause": ("Duraklama süresi", "sl", (0, 4, 40, "{:.1f} sn", float), 1.5),
    "fps": ("Kare sayısı", "sl", (5, 20, 15, "{:.0f} kare/sn", int), 10),
    "prefix": ("Yazının başına ekle", "entry", None, ""),
    "upper": ("BÜYÜK HARFLE yaz", "sw", None, False),
    "priority": ("Öncelik modu", "map", {t: k for k, t, _ in PRIORITIES}, "app"),
    "bg_media": ("Arka plandaki medyayı göster", "sw", None, True),
    "vol_on": ("Ses göstergesi", "sw", None, True),
    "idle_mode": ("Boşta ne gösterilsin", "map", IDLE_MODES, "none"),
    "invert": ("Görselleri ters çevir", "sw", None, False),
    "media_pop": ("Yeni şarkıyı kısa süre göster", "sw", None, True),
    "sys_when": ("Sistem bilgisi ne zaman", "map", SYS_WHEN, "idle"),
    "sys_every": ("Aralıklı gösterim sıklığı", "map", SYS_EVERY, 5),
}
CAT_FIELDS = ["size", "weight", "scroll", "style", "speed", "pause", "prefix", "upper"]
PROFILE_FIELDS = ["size", "weight", "scroll", "style", "speed", "pause", "fps", "priority", "bg_media",
                  "vol_on", "idle_mode", "invert", "media_pop", "sys_when", "sys_every"]

DEFAULT_THEME = "Neon Gece"
THEMES = {
    "Neon Gece": {"mode": "dark", "bg": "#0b0b17", "side": "#11111f", "card": "#171728", "card2": "#23233a",
                  "text": "#eceaff", "muted": "#8f8bb3", "accent": "#8b5cf6", "accent_hover": "#7c3aed",
                  "accent2": "#22d3ee", "ok": "#22c55e", "warn": "#f59e0b", "err": "#ef4444"},
    "Konsol": {"mode": "dark", "bg": "#0d0f13", "side": "#12151a", "card": "#181c23", "card2": "#252a34",
               "text": "#f1f5f9", "muted": "#8d97a8", "accent": "#2dbd4e", "accent_hover": "#22a041",
               "accent2": "#38bdf8", "ok": "#22c55e", "warn": "#f59e0b", "err": "#ef4444"},
    "Okyanus": {"mode": "dark", "bg": "#07131f", "side": "#0b1c2c", "card": "#10263a", "card2": "#18364f",
                "text": "#e6f4ff", "muted": "#7fa3bf", "accent": "#0ea5e9", "accent_hover": "#0284c7",
                "accent2": "#2dd4bf", "ok": "#22c55e", "warn": "#f59e0b", "err": "#ef4444"},
    "Gün Batımı": {"mode": "dark", "bg": "#180d12", "side": "#20111a", "card": "#2b1822", "card2": "#3b2230",
                   "text": "#ffeef2", "muted": "#c08a9a", "accent": "#fb7185", "accent_hover": "#f43f5e",
                   "accent2": "#fbbf24", "ok": "#22c55e", "warn": "#f59e0b", "err": "#ef4444"},
    "Orman": {"mode": "dark", "bg": "#09140e", "side": "#0e1c14", "card": "#14281c", "card2": "#1d3a29",
              "text": "#e9fbe9", "muted": "#7fae8c", "accent": "#22c55e", "accent_hover": "#16a34a",
              "accent2": "#a3e635", "ok": "#22c55e", "warn": "#f59e0b", "err": "#ef4444"},
    "Oyuncu Kırmızısı": {"mode": "dark", "bg": "#120808", "side": "#1a0b0b", "card": "#241010", "card2": "#341818",
                         "text": "#ffecec", "muted": "#b98080", "accent": "#ef4444", "accent_hover": "#dc2626",
                         "accent2": "#f97316", "ok": "#22c55e", "warn": "#f59e0b", "err": "#ef4444"},
    "Klasik Karanlık": {"mode": "dark", "bg": "#111418", "side": "#161a20", "card": "#1b2029", "card2": "#2b3340",
                        "text": "#e5e9f0", "muted": "#8b94a3", "accent": "#3b82f6", "accent_hover": "#2563eb",
                        "accent2": "#94a3b8", "ok": "#22c55e", "warn": "#f59e0b", "err": "#ef4444"},
    "Pamuk Şeker": {"mode": "light", "bg": "#fdf2f8", "side": "#fce7f3", "card": "#ffffff", "card2": "#fbcfe8",
                    "text": "#3b0a2a", "muted": "#9d5b82", "accent": "#ec4899", "accent_hover": "#db2777",
                    "accent2": "#8b5cf6", "ok": "#16a34a", "warn": "#d97706", "err": "#dc2626"},
}

DEFAULTS = {
    "rules": [],
    "default_text": "",
    "invert": False,
    "size": "Orta",
    "weight": "Kalın",
    "scroll": True,
    "style": "Sürekli akış",
    "speed": 40,
    "pause": 1.5,
    "fps": 10,
    "native_text": False,
    "bg_media": True,
    "priority": "app",
    "priority_category": "Oyun",
    "rotate_secs": 8,
    "theme": DEFAULT_THEME,
    "auto_start": False,
    "start_minimized": False,
    "preview_text": "Örnek Şarkı Adı - Sanatçı Adı",
    "side_open": True,
    "idle_clock": False,
    "shutdown_at": 0.0,
    "shutdown_minutes": 60,
    "profiles": [],
    "active_profile": "",
    "cat_settings": {},
    "sys_metrics": {k: {"on": on, "fmt": f} for k, _l, f, on in SYS_DEFS},
    "sys_mode": "rotate",
    "sys_secs": 4,
    "sys_sep": "  |  ",
    "lhm_port": 8085,
    "alerts_on": False,
    "alert_cpu": 85,
    "alert_gpu": 85,
    "vol_on": True,
    "vol_secs": 2.0,
    "vol_style": "Yazı + çubuk",
    "vol_text": "Ses: {v}",
    "vol_mute": "Sessiz",
    "idle_mode": "none",
    "close_to_tray": True,
    "sys_when": "idle",
    "sys_every": 5,
    "sys_show_secs": 12,
    "media_pop": True,
    "media_pop_secs": 5,
    "rule_secs": 8,
    "media_secs": 8,
    "prio_cats": [],
    "sys_over": True,
    "view_mode": "Kartlar",
}


# ---------------------------------------------------------------- ayarlar
def load_config():
    os.makedirs(IMG_DIR, exist_ok=True)
    os.makedirs(ICON_DIR, exist_ok=True)
    try:
        with open(CFG_PATH, "r", encoding="utf-8") as f:
            cfg = json.load(f)
    except Exception:
        cfg = {}
    for k, v in DEFAULTS.items():
        cfg.setdefault(k, copy.deepcopy(v))
    for r in cfg["rules"]:
        r.setdefault("text", "")
        r.setdefault("image", "")
        r.setdefault("path", "")
        r.setdefault("category", DEFAULT_CATEGORY)
        r.setdefault("mode", "text")
        r.setdefault("filter", "")
        r.setdefault("remove", "")
    for p in cfg["profiles"]:
        p.setdefault("name", "Profil")
        p.setdefault("exes", "")
        p.setdefault("cats", [])
        p.setdefault("settings", {})
    for k, _l, f, on in SYS_DEFS:
        m = cfg["sys_metrics"].setdefault(k, {"on": on, "fmt": f})
        m.setdefault("on", on)
        m.setdefault("fmt", f)
    if cfg.pop("idle_clock", False) and cfg.get("idle_mode", "none") == "none":
        cfg["idle_mode"] = "clock"
    if cfg.get("idle_mode") == "sys":
        cfg["idle_mode"] = "none"
        cfg["sys_when"] = "idle"
    for p in cfg["profiles"]:
        st = p.get("settings", {})
        if st.get("idle_mode") == "sys":
            st["idle_mode"] = "none"
            st["sys_when"] = "idle"
    if cfg.get("priority") == "category" and not cfg.get("prio_cats") and cfg.get("priority_category"):
        cfg["prio_cats"] = [cfg["priority_category"]]
    if cfg["theme"] not in THEMES:
        cfg["theme"] = DEFAULT_THEME
    return cfg


def save_config(cfg):
    os.makedirs(DATA_DIR, exist_ok=True)
    with open(CFG_PATH, "w", encoding="utf-8") as f:
        json.dump(cfg, f, ensure_ascii=False, indent=2)


def text_settings(cfg):
    return (
        SIZES.get(cfg["size"], 22),
        WEIGHTS.get(cfg["weight"], 1),
        bool(cfg["scroll"]),
        STYLES.get(cfg["style"], "loop"),
        float(cfg["speed"]),
        float(cfg["pause"]),
    )


# ---------------------------------------------------------------- tema
def mix(a, b, t):
    a, b = a.lstrip("#"), b.lstrip("#")
    ar, ag, ab = [int(a[i:i + 2], 16) for i in (0, 2, 4)]
    br, bg, bb = [int(b[i:i + 2], 16) for i in (0, 2, 4)]
    return "#%02x%02x%02x" % (round(ar + (br - ar) * t), round(ag + (bg - ag) * t), round(ab + (bb - ab) * t))


TH = {}


def set_theme_colors(name):
    TH.clear()
    TH.update(THEMES.get(name, THEMES[DEFAULT_THEME]))
    TH["soft_hover"] = mix(TH["card2"], TH["accent"], 0.35)
    TH["border"] = mix(TH["card"], TH["text"], 0.14)
    TH["err_hover"] = mix(TH["err"], "#000000", 0.25)
    TH["ok_hover"] = mix(TH["ok"], "#000000", 0.25)
    TH["g_left"] = mix(TH["bg"], TH["accent2"], 0.10)
    TH["g_mid"] = mix(TH["bg"], TH["accent"], 0.45)
    TH["g_right"] = mix(mix(TH["accent"], TH["accent2"], 0.5), TH["bg"], 0.15)


set_theme_colors(DEFAULT_THEME)

HDR_H = 124
SIDE_OPEN, SIDE_CLOSED = 210, 78


def rr_pts(x1, y1, x2, y2, r):
    """Canvas'ta smooth=True ile yuvarlak köşeli dikdörtgen çizmek için noktalar."""
    return [x1 + r, y1, x1 + r, y1, x2 - r, y1, x2 - r, y1, x2, y1, x2, y1 + r, x2, y1 + r, x2, y2 - r,
            x2, y2 - r, x2, y2, x2 - r, y2, x2 - r, y2, x1 + r, y2, x1 + r, y2, x1, y2, x1, y2 - r,
            x1, y2 - r, x1, y1 + r, x1, y1 + r, x1, y1]


def hex2rgb(h):
    h = h.lstrip("#")
    return tuple(int(h[i:i + 2], 16) for i in (0, 2, 4))


def banner_hex(xn, yn):
    """Üst başlık gradyanının (x, y) noktasındaki rengi (0-1 arası koordinat)."""
    t = max(0.0, min(1.0, xn * 0.85 + (1 - yn) * 0.15))
    if t < 0.5:
        c = mix(TH["g_left"], TH["g_mid"], t * 2)
    else:
        c = mix(TH["g_mid"], TH["g_right"], (t - 0.5) * 2)
    k = max(0.0, (yn - 0.25) / 0.75)
    k = k * k * (3 - 2 * k)
    return mix(c, TH["bg"], k)


def make_banner(w, h):
    sw, sh = 64, 16
    small = Image.new("RGB", (sw, sh))
    for x in range(sw):
        for y in range(sh):
            small.putpixel((x, y), hex2rgb(banner_hex(x / (sw - 1), y / (sh - 1))))
    return small.resize((max(1, w), h), Image.BICUBIC)


def font(size=13, bold=False):
    return ctk.CTkFont(size=size, weight="bold" if bold else "normal")


def lbl(parent, text, size=13, bold=False, muted=False, **kw):
    return ctk.CTkLabel(parent, text=text, text_color=TH["muted"] if muted else TH["text"],
                        font=font(size, bold), **kw)


def btn(parent, text, command, kind="accent", width=100, height=34, radius=12, **kw):
    colors = {
        "accent": (TH["accent"], TH["accent_hover"], "#ffffff"),
        "soft": (TH["card2"], TH["soft_hover"], TH["text"]),
        "danger": (TH["err"], TH["err_hover"], "#ffffff"),
        "ok": (TH["ok"], TH["ok_hover"], "#ffffff"),
    }[kind]
    return ctk.CTkButton(parent, text=text, command=command, width=width, height=height, corner_radius=radius,
                         fg_color=colors[0], hover_color=colors[1], text_color=colors[2],
                         font=font(13, True), **kw)


def entry(parent, width=300, placeholder="", radius=10, height=30):
    return ctk.CTkEntry(parent, width=width, height=height, corner_radius=radius, placeholder_text=placeholder, fg_color=TH["card2"],
                        border_color=TH["border"], text_color=TH["text"],
                        placeholder_text_color=TH["muted"])


def switch(parent, text, command):
    return ctk.CTkSwitch(parent, text=text, command=command, fg_color=TH["card2"], progress_color=TH["accent"],
                         button_color=TH["text"], button_hover_color=TH["accent2"], text_color=TH["text"])


def segmented(parent, values, command):
    return ctk.CTkSegmentedButton(parent, values=values, command=command, fg_color=TH["card2"],
                                  selected_color=TH["accent"], selected_hover_color=TH["accent_hover"],
                                  unselected_color=TH["card2"], unselected_hover_color=TH["soft_hover"],
                                  text_color=TH["text"])


def option(parent, values, width=200, command=None):
    return ctk.CTkOptionMenu(parent, values=values, width=width, command=command, fg_color=TH["card2"],
                             button_color=TH["accent"], button_hover_color=TH["accent_hover"],
                             text_color=TH["text"], dropdown_fg_color=TH["card"],
                             dropdown_hover_color=TH["card2"], dropdown_text_color=TH["text"])


def combo(parent, values, width=300, command=None):
    return ctk.CTkComboBox(parent, values=values, width=width, command=command, fg_color=TH["card2"],
                           border_color=TH["border"], button_color=TH["accent"],
                           button_hover_color=TH["accent_hover"], text_color=TH["text"],
                           dropdown_fg_color=TH["card"], dropdown_hover_color=TH["card2"],
                           dropdown_text_color=TH["text"])


def slider(parent, lo, hi, steps, command, width=240):
    return ctk.CTkSlider(parent, from_=lo, to=hi, number_of_steps=steps, width=width, command=command,
                         fg_color=TH["card2"], progress_color=TH["accent"], button_color=TH["accent2"],
                         button_hover_color=TH["accent"])


def scroll_frame(parent):
    return ctk.CTkScrollableFrame(parent, fg_color="transparent", scrollbar_button_color=TH["card2"],
                                  scrollbar_button_hover_color=TH["accent"])


def card(parent, **kw):
    return ctk.CTkFrame(parent, corner_radius=20, fg_color=TH["card"], border_width=1,
                        border_color=TH["border"], **kw)


# ---------------------------------------------------------------- kapanış zamanlayıcısı (Windows'un kendi komutu)
def run_shutdown(secs):
    """secs=None: bekleyen kapanışı iptal et. Sayı: o kadar saniye sonra kapat."""
    flags = 0x08000000  # pencere açma
    try:
        subprocess.run(["shutdown", "/a"], capture_output=True, creationflags=flags)
        if secs is None:
            return True
        r = subprocess.run(["shutdown", "/s", "/t", str(int(secs)), "/c", "Apex OLED kapanış zamanlayıcısı"],
                           capture_output=True, creationflags=flags)
        return r.returncode == 0
    except Exception:
        return False


# ---------------------------------------------------------------- Windows ile başlatma
def startup_command():
    if getattr(sys, "frozen", False):
        return f'"{sys.executable}" --startup'
    py = sys.executable
    if py.lower().endswith("python.exe"):
        py = py[:-10] + "pythonw.exe"
    return f'"{py}" "{os.path.abspath(__file__)}" --startup'


def is_startup_enabled():
    if winreg is None:
        return False
    try:
        with winreg.OpenKey(winreg.HKEY_CURRENT_USER, RUN_KEY, 0, winreg.KEY_READ) as k:
            winreg.QueryValueEx(k, APP)
            return True
    except OSError:
        return False


def set_startup(enabled):
    if winreg is None:
        return
    with winreg.OpenKey(winreg.HKEY_CURRENT_USER, RUN_KEY, 0, winreg.KEY_SET_VALUE) as k:
        if enabled:
            winreg.SetValueEx(k, APP, 0, winreg.REG_SZ, startup_command())
        else:
            try:
                winreg.DeleteValue(k, APP)
            except OSError:
                pass


# ---------------------------------------------------------------- program ikonları
_no_icon = set()


def find_running_path(exe):
    for p in psutil.process_iter(["name", "exe"]):
        try:
            if (p.info.get("name") or "").lower() == exe.lower():
                return p.info.get("exe") or ""
        except Exception:
            pass
    return ""


def get_icon_image(exe, path="", cache_only=False):
    if not exe:
        return None
    cache = os.path.join(ICON_DIR, exe.lower() + ".png")
    if os.path.exists(cache):
        try:
            return Image.open(cache).convert("RGBA")
        except Exception:
            pass
    if cache_only or IconExtractor is None or exe.lower() in _no_icon:
        return None
    path = path or find_running_path(exe)
    if not path or not os.path.exists(path):
        return None
    try:
        data = IconExtractor(path).get_icon()
        img = Image.open(data).convert("RGBA")
        img.thumbnail((64, 64))
        img.save(cache)
        return img
    except Exception:
        _no_icon.add(exe.lower())
        return None


def render_icon(exe, path, color, size, bg, extract=True):
    """İkonu, arka plan rengine oturan yuvarlak köşeli bir plakanın üstüne tek parça (şeffaflıksız) çizer."""
    S = size * 3
    img = Image.new("RGBA", (S, S), hex2rgb(bg) + (255,))
    d = ImageDraw.Draw(img)
    pad = int(S * 0.03)
    box = (pad, pad, S - pad, S - pad)
    radius = int(S * 0.24)
    ico = get_icon_image(exe, path, cache_only=not extract)
    if ico is not None:
        plate = mix(bg, "#ffffff" if TH["mode"] == "dark" else "#000000", 0.10)
        d.rounded_rectangle(box, radius=radius, fill=hex2rgb(plate) + (255,))
        inner = int(S * 0.64)
        ico = ImageOps.contain(ico.convert("RGBA"), (inner, inner), Image.LANCZOS)
        img.alpha_composite(ico, ((S - ico.width) // 2, (S - ico.height) // 2))
    else:
        d.rounded_rectangle(box, radius=radius, fill=hex2rgb(color) + (255,))
        letter = (exe[:1] or "?").upper()
        f = load_font(int(S * 0.5), True)
        bb = d.textbbox((0, 0), letter, font=f)
        d.text(((S - (bb[2] - bb[0])) / 2 - bb[0], (S - (bb[3] - bb[1])) / 2 - bb[1]), letter, font=f,
               fill=(255, 255, 255, 255))
    return img.convert("RGB")


def make_icon(parent, exe, path, color, keep, size=40, bg=None, extract=True):
    bg = bg or TH["card"]
    img = render_icon(exe, path, color, size, bg, extract)
    ci = ctk.CTkImage(light_image=img, dark_image=img, size=(size, size))
    keep.append(ci)
    return ctk.CTkLabel(parent, text="", image=ci, width=size, height=size, fg_color=bg)


# ---------------------------------------------------------------- görsel / yazı çizimi
def image_to_mono(path, invert=False):
    """Görseli 128x40 tek renkli (1 bit) görüntüye çevirir."""
    img = Image.open(path).convert("RGBA")
    bg = Image.new("RGBA", img.size, (0, 0, 0, 255))
    bg.alpha_composite(img)
    gray = ImageOps.contain(bg.convert("L"), (128, 40))
    canvas = Image.new("L", (128, 40), 0)
    canvas.paste(gray, ((128 - gray.width) // 2, (40 - gray.height) // 2))
    if invert:
        canvas = ImageOps.invert(canvas)
    return canvas.convert("1")


def image_to_bytes(path, invert=False):
    """128x40 tek renkli bitmap (640 bayt, satır sırası, MSB solda)."""
    return list(image_to_mono(path, invert).tobytes())


FONT_DIR = os.path.join(os.environ.get("WINDIR", r"C:\Windows"), "Fonts")
_font_cache = {}


def load_font(size, bold):
    key = (size, bold)
    if key in _font_cache:
        return _font_cache[key]
    names = ["segoeuib.ttf", "arialbd.ttf", "calibrib.ttf"] if bold else ["segoeui.ttf", "arial.ttf", "calibri.ttf"]
    f = None
    for n in names:
        try:
            f = ImageFont.truetype(os.path.join(FONT_DIR, n), size)
            break
        except Exception:
            continue
    if f is None:
        f = ImageFont.load_default()
    _font_cache[key] = f
    return f


class Scroller:
    """Metni 128x40 tek renkli kareler olarak üretir; sığmıyorsa kaydırır."""
    W, H, GAP = 128, 40, 48

    def __init__(self, text, size, weight, scroll, style, speed, pause):
        self.style = style
        self.speed = max(1.0, float(speed))
        self.pause = max(0.0, float(pause))
        fnt = load_font(size, weight >= 1)
        stroke = 1 if weight >= 2 else 0
        text = (text or "").replace("\n", " ").strip() or " "
        probe = ImageDraw.Draw(Image.new("1", (1, 1)))
        try:
            asc, desc = fnt.getmetrics()
        except Exception:
            asc, desc = size, 0
        bbox = probe.textbbox((0, 0), text, font=fnt, stroke_width=stroke)
        width = int(math.ceil(max(fnt.getlength(text), bbox[2]))) + stroke * 2 + 1
        y = (self.H - (asc + desc)) // 2
        strip = Image.new("1", (width, self.H), 0)
        ImageDraw.Draw(strip).text((stroke, y), text, font=fnt, fill=1, stroke_width=stroke)
        self.w = width
        self.fits = width <= self.W
        self.animated = (not self.fits) and scroll
        self.static = None
        self.wide = None
        if self.fits:
            canvas = Image.new("1", (self.W, self.H), 0)
            canvas.paste(strip, ((self.W - width) // 2, 0))
            self.static = canvas
        elif not scroll:
            canvas = Image.new("1", (self.W, self.H), 0)
            canvas.paste(strip.crop((0, 0, self.W, self.H)), (0, 0))
            self.static = canvas
        elif style == "pingpong":
            self.wide = strip
        else:
            wide = Image.new("1", (width + self.GAP + self.W, self.H), 0)
            wide.paste(strip, (0, 0))
            wide.paste(strip, (width + self.GAP, 0))
            self.wide = wide

    def offset(self, t):
        if not self.animated:
            return 0
        sp, p = self.speed, self.pause
        if self.style == "pingpong":
            dist = self.w - self.W
            run = dist / sp
            cyc = 2 * (run + p)
            tt = t % cyc
            if tt < p:
                o = 0
            elif tt < p + run:
                o = (tt - p) * sp
            elif tt < 2 * p + run:
                o = dist
            else:
                o = dist - (tt - 2 * p - run) * sp
            return int(max(0, min(dist, o)))
        loop = self.w + self.GAP
        if self.style == "pause":
            cyc = loop / sp + p
            tt = t % cyc
            return 0 if tt < p else int((tt - p) * sp) % loop
        return int(t * sp) % loop

    def frame(self, t):
        if not self.animated:
            return self.static
        o = self.offset(t)
        return self.wide.crop((o, 0, o + self.W, self.H))


# ---------------------------------------------------------------- pencere / başlık
def foreground_info():
    user32 = ctypes.windll.user32
    hwnd = user32.GetForegroundWindow()
    pid = wintypes.DWORD()
    user32.GetWindowThreadProcessId(hwnd, ctypes.byref(pid))
    try:
        exe = psutil.Process(pid.value).name().lower()
    except Exception:
        exe = None
    length = user32.GetWindowTextLengthW(hwnd)
    buf = ctypes.create_unicode_buffer(length + 1)
    user32.GetWindowTextW(hwnd, buf, length + 1)
    return exe, buf.value


BROWSER_SUFFIX = re.compile(
    r"\s+[-–—]\s+(Google Chrome|Opera GX|Opera|Microsoft Edge|Mozilla Firefox|Brave)(\s+[-–—]\s+.*)?$", re.I)


def split_list(s):
    return [x.strip() for x in (s or "").split(",") if x.strip()]


def clean_text(text, remove):
    t = BROWSER_SUFFIX.sub("", text or "")
    t = re.sub(r"^\(\d+\)\s*", "", t)
    for phrase in remove:
        t = re.sub(re.escape(phrase), "", t, flags=re.I)
    t = re.sub(r"\s{2,}", " ", t).strip(" -–—|·")
    return t


# ---------------------------------------------------------------- Windows medya bilgisi
class MediaWatcher(threading.Thread):
    """Windows'un 'şu an çalıyor' bilgisini okur (tarayıcıdaki müzik/video dahil, arka planda da)."""

    def __init__(self):
        super().__init__(daemon=True)
        self.stop_flag = threading.Event()
        self.items = []
        self.error = ""
        self.ready = False

    def run(self):
        try:
            import asyncio
            from winrt.windows.media.control import (
                GlobalSystemMediaTransportControlsSessionManager as Manager)
        except Exception as e:
            self.error = f"kütüphane eksik ({e})"
            return
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        state = {"manager": None}

        async def poll():
            if state["manager"] is None:
                state["manager"] = await Manager.request_async()
            items = []
            for s in state["manager"].get_sessions():
                try:
                    props = await s.try_get_media_properties_async()
                    info = s.get_playback_info()
                    items.append({
                        "app": (s.source_app_user_model_id or "").lower(),
                        "title": props.title or "",
                        "artist": props.artist or "",
                        "playing": int(info.playback_status) == 4,
                    })
                except Exception:
                    pass
            return items

        while not self.stop_flag.is_set():
            try:
                self.items = loop.run_until_complete(poll())
                self.ready = True
                self.error = ""
            except Exception as e:
                self.error = str(e)
                state["manager"] = None
            self.stop_flag.wait(1.5)

    def pick(self, exe):
        playing = [i for i in self.items if i["playing"] and i["title"]]
        stem = os.path.splitext(exe or "")[0].lower()
        if stem:
            for i in playing:
                if stem in i["app"]:
                    return i
        return playing[0] if playing else None

    @staticmethod
    def text_of(item):
        if not item:
            return ""
        return f"{item['artist']} - {item['title']}" if item["artist"] else item["title"]


# ---------------------------------------------------------------- metin / sistem yardımcıları
def tr_upper(t):
    return t.replace("i", "İ").replace("ı", "I").upper()


def decorate_text(cfg, text, cat):
    """Kategorinin 'ön ek' ve 'BÜYÜK HARF' ayarlarını yazıya uygular."""
    if not text:
        return text
    cs = cfg.get("cat_settings", {}).get(cat, {})
    if cs.get("upper"):
        text = tr_upper(text)
    return cs.get("prefix", "") + text


def human_rate(bps):
    if bps < 1024:
        return f"{bps:.0f} B/s"
    if bps < 1024 ** 2:
        return f"{bps / 1024:.0f} KB/s"
    return f"{bps / 1024 ** 2:.1f} MB/s"


def human_uptime(sec):
    sec = int(sec)
    d, r = divmod(sec, 86400)
    h, r = divmod(r, 3600)
    m = r // 60
    return f"{d}g {h}sa" if d else f"{h}sa {m}dk"


def metric_text(key, raw):
    if raw is None:
        return None
    if isinstance(raw, str):
        return raw
    if key in ("cpu_clock", "ram_used"):
        return f"{raw:.1f}"
    return f"{raw:.0f}"


def sys_lines(cfg, data):
    out = []
    for key, _label, dfmt, don in SYS_DEFS:
        m = cfg.get("sys_metrics", {}).get(key, {})
        if not m.get("on", don):
            continue
        v = metric_text(key, data.get(key))
        if v is None:
            continue
        out.append((m.get("fmt") or dfmt).replace("{v}", v))
    return out


def sys_compose(cfg, data, now):
    lines = sys_lines(cfg, data)
    if not lines:
        return ""
    if cfg.get("sys_mode") == "joined":
        return (cfg.get("sys_sep") or "  |  ").join(lines)
    period = max(1.0, float(cfg.get("sys_secs", 4)))
    return lines[int(now / period) % len(lines)]


_vol_cache = {}


def render_volume(level, muted, style, tmpl, mute_txt):
    """Ses göstergesini 128x40 tek renkli görüntü olarak çizer."""
    key = (level, muted, style, tmpl, mute_txt)
    if key in _vol_cache:
        return _vol_cache[key]
    W, H = 128, 40
    img = Image.new("1", (W, H), 0)
    d = ImageDraw.Draw(img)
    text = mute_txt if muted else (tmpl or "Ses: {v}").replace("{v}", str(level))
    show_text = style in ("Yazı", "Yazı + çubuk")
    show_bar = style in ("Çubuk", "Yazı + çubuk")
    if not show_text and not show_bar:
        show_text = True
    if show_text:
        size = 16 if show_bar else 24
        f = load_font(size, True)
        while size > 10 and f.getlength(text) > W - 6:
            size -= 2
            f = load_font(size, True)
        tw = f.getlength(text)
        asc, desc = f.getmetrics()
        y = 2 if show_bar else (H - (asc + desc)) // 2
        d.text(((W - tw) // 2, y), text, font=f, fill=1)
    if show_bar:
        top = 27 if show_text else 14
        h = 10 if show_text else 12
        d.rectangle((4, top, W - 5, top + h), outline=1)
        fill_w = int((W - 14) * (0 if muted else level) / 100)
        if fill_w > 0:
            d.rectangle((6, top + 2, 6 + fill_w, top + h - 2), fill=1)
    if len(_vol_cache) > 400:
        _vol_cache.clear()
    _vol_cache[key] = img
    return img


def list_programs():
    """(açık pencereli programlar, arka plan işlemleri): [(exe, yol, pencere başlığı)]"""
    user32 = ctypes.windll.user32
    found = {}
    EnumProc = ctypes.WINFUNCTYPE(ctypes.c_bool, wintypes.HWND, wintypes.LPARAM)

    def cb(hwnd, _lp):
        try:
            if user32.IsWindowVisible(hwnd):
                n = user32.GetWindowTextLengthW(hwnd)
                if n > 0:
                    buf = ctypes.create_unicode_buffer(n + 1)
                    user32.GetWindowTextW(hwnd, buf, n + 1)
                    pid = wintypes.DWORD()
                    user32.GetWindowThreadProcessId(hwnd, ctypes.byref(pid))
                    pr = psutil.Process(pid.value)
                    name = pr.name()
                    if name.lower() not in OWN_NAMES and name.lower() not in found:
                        found[name.lower()] = (name, pr.exe(), buf.value)
        except Exception:
            pass
        return True

    user32.EnumWindows(EnumProc(cb), 0)
    visible = sorted(found.values(), key=lambda x: x[0].lower())
    others = {}
    for pr in psutil.process_iter(["name", "exe"]):
        try:
            n = pr.info.get("name") or ""
            if n.lower().endswith(".exe") and n.lower() not in found and n.lower() not in OWN_NAMES:
                others.setdefault(n.lower(), (n, pr.info.get("exe") or "", ""))
        except Exception:
            pass
    return visible, sorted(others.values(), key=lambda x: x[0].lower())


def bind_click(widgets, fn):
    for w in widgets:
        try:
            w.bind("<Button-1>", fn)
        except Exception:
            pass


# ---------------------------------------------------------------- sistem bilgisi (CPU, GPU, RAM, sıcaklık...)
NVSMI_PATHS = [r"C:\Windows\System32\nvidia-smi.exe", r"C:\Program Files\NVIDIA Corporation\NVSMI\nvidia-smi.exe"]


def find_nvsmi():
    p = shutil.which("nvidia-smi")
    if p:
        return p
    for c in NVSMI_PATHS:
        if os.path.exists(c):
            return c
    return None


def to_float(x):
    try:
        return float(str(x).strip())
    except Exception:
        return None


def read_nvidia(path):
    r = subprocess.run([path, "--query-gpu=utilization.gpu,temperature.gpu,memory.used,memory.total,power.draw",
                        "--format=csv,noheader,nounits"], capture_output=True, text=True, timeout=3,
                       creationflags=0x08000000)
    line = (r.stdout or "").strip().splitlines()[0]
    parts = [x.strip() for x in line.split(",")]
    used, total = to_float(parts[2]), to_float(parts[3])
    return {
        "gpu": to_float(parts[0]),
        "gpu_temp": to_float(parts[1]),
        "gpu_mem": (used / total * 100) if used is not None and total else None,
        "gpu_power": to_float(parts[4]),
    }


def _num(v):
    m = re.search(r"-?\d+(?:[.,]\d+)?", v or "")
    return float(m.group().replace(",", ".")) if m else None


def parse_lhm(tree):
    """LibreHardwareMonitor'un data.json ağacından CPU/GPU sıcaklık ve GPU yükünü çıkarır."""
    leaves = []

    def walk(node, path):
        here = path + [(node.get("Text", ""), (node.get("ImageURL") or "").lower())]
        kids = node.get("Children") or []
        if kids:
            for k in kids:
                walk(k, here)
        elif node.get("Value"):
            leaves.append((here, node.get("Text", ""), node["Value"]))

    walk(tree, [])

    def kind_of(path):
        for _t, img in path:
            if "cpu" in img:
                return "cpu"
        for _t, img in path:
            if any(k in img for k in ("nvidia", "ati", "amd", "intel", "gpu")):
                return "gpu"
        return None

    def cpu_score(low):
        for i, key in enumerate(("package", "tctl", "tdie", "core max", "cpu", "core")):
            if key in low:
                return i
        return 99

    cpu_t, gpu_t = [], []
    out = {}
    for path, name, val in leaves:
        k, n = kind_of(path), _num(val)
        if k is None or n is None:
            continue
        low = name.lower()
        vl = val.strip().lower()
        if vl.endswith("c"):
            (cpu_t if k == "cpu" else gpu_t).append((cpu_score(low) if k == "cpu" else (0 if "core" in low else 1), n))
        elif vl.endswith("%") and k == "gpu" and low == "gpu core":
            out["gpu"] = n
    if cpu_t:
        out["cpu_temp"] = min(cpu_t)[1]
    if gpu_t:
        out["gpu_temp"] = min(gpu_t)[1]
    return out


class SysStats(threading.Thread):
    """Saniyede bir CPU/RAM/ağ/GPU/sıcaklık değerlerini toplar."""

    def __init__(self, cfg):
        super().__init__(daemon=True)
        self.cfg = cfg
        self.stop_flag = threading.Event()
        self.data = {}
        self.nvsmi = None
        self.nv_checked = False
        self.nv_ok = False
        self.lhm_ok = False
        self._net = None
        self._slow_t = 0.0
        self._slow = {}

    def slow(self):
        out = {}
        if not self.nv_checked:
            self.nv_checked = True
            self.nvsmi = find_nvsmi()
        if self.nvsmi:
            try:
                out.update(read_nvidia(self.nvsmi))
                self.nv_ok = True
            except Exception:
                self.nv_ok = False
        try:
            port = int(self.cfg.get("lhm_port", 8085))
            tree = requests.get(f"http://127.0.0.1:{port}/data.json", timeout=0.8).json()
            for k, v in parse_lhm(tree).items():
                if out.get(k) is None:
                    out[k] = v
            self.lhm_ok = True
        except Exception:
            self.lhm_ok = False
        return out

    def run(self):
        try:
            psutil.cpu_percent(None)
        except Exception:
            pass
        while not self.stop_flag.is_set():
            d = {}
            now = time.time()
            try:
                d["cpu"] = psutil.cpu_percent(None)
                vm = psutil.virtual_memory()
                d["ram"] = vm.percent
                d["ram_used"] = vm.used / 1024 ** 3
                try:
                    f = psutil.cpu_freq()
                    d["cpu_clock"] = f.current / 1000 if f else None
                except Exception:
                    d["cpu_clock"] = None
                try:
                    d["disk"] = psutil.disk_usage(os.environ.get("SystemDrive", "C:") + "\\").percent
                except Exception:
                    d["disk"] = None
                io = psutil.net_io_counters()
                if self._net:
                    dt = max(0.2, now - self._net[0])
                    d["net_down"] = human_rate((io.bytes_recv - self._net[1]) / dt)
                    d["net_up"] = human_rate((io.bytes_sent - self._net[2]) / dt)
                self._net = (now, io.bytes_recv, io.bytes_sent)
                try:
                    b = psutil.sensors_battery()
                    d["battery"] = b.percent if b else None
                except Exception:
                    d["battery"] = None
                d["clock"] = time.strftime("%H:%M")
                d["uptime"] = human_uptime(now - psutil.boot_time())
                if now - self._slow_t >= 2.0:
                    self._slow_t = now
                    self._slow = self.slow()
                d.update({k: v for k, v in self._slow.items() if v is not None})
                self.data = d
            except Exception:
                pass
            self.stop_flag.wait(1.0)


class VolumeWatcher(threading.Thread):
    """Windows ana ses seviyesini izler; değişince 'changed_at' güncellenir."""

    def __init__(self):
        super().__init__(daemon=True)
        self.stop_flag = threading.Event()
        self.level = None
        self.muted = False
        self.changed_at = 0.0
        self.error = ""

    def run(self):
        try:
            import comtypes
            comtypes.CoInitialize()
            from ctypes import cast, POINTER
            from comtypes import CLSCTX_ALL
            from pycaw.pycaw import AudioUtilities, IAudioEndpointVolume
        except Exception as e:
            self.error = f"kütüphane eksik ({e})"
            return

        def endpoint():
            dev = AudioUtilities.GetSpeakers()
            ep = getattr(dev, "EndpointVolume", None)
            if ep is None:
                iface = dev.Activate(IAudioEndpointVolume._iid_, CLSCTX_ALL, None)
                ep = cast(iface, POINTER(IAudioEndpointVolume))
            return ep

        ep, ep_t, prev = None, 0.0, None
        while not self.stop_flag.is_set():
            try:
                now = time.time()
                if ep is None or now - ep_t > 20:
                    ep, ep_t, prev = endpoint(), now, None
                lv = int(round(ep.GetMasterVolumeLevelScalar() * 100))
                mu = bool(ep.GetMute())
                if prev is not None and (lv, mu) != prev:
                    self.changed_at = time.time()
                prev = (lv, mu)
                self.level, self.muted, self.error = lv, mu, ""
            except Exception as e:
                ep, prev = None, None
                self.error = str(e)
            self.stop_flag.wait(0.15)


# ---------------------------------------------------------------- ortak arayüz parçaları
class OledPreview(ctk.CTkFrame):
    """Klavye ekranının yaklaşık canlı önizlemesi. get_params: (yazı, ayarlar) ya da 128x40 görüntü döndürür."""

    def __init__(self, master, get_params, scale=3):
        super().__init__(master, fg_color="transparent")
        self.get_params = get_params
        self.scale = scale
        self.sc = None
        self.text = None
        self.ts = None
        self.t0 = time.time()
        self._img = None
        self.lbl = ctk.CTkLabel(self, text="", fg_color="#080a0e", corner_radius=8,
                                width=128 * scale, height=40 * scale)
        self.lbl.pack()
        self._loop()

    def _loop(self):
        try:
            if not self.winfo_exists():
                return
            res = self.get_params()
            if isinstance(res, Image.Image):
                fr = res
            else:
                text, ts = res
                if ts != self.ts:
                    self.ts = ts
                    self.t0 = time.time()
                    self.sc = None
                if text != self.text or self.sc is None:
                    self.text = text
                    self.sc = Scroller(text or " ", *ts)
                fr = self.sc.frame(time.time() - self.t0)
            img = ImageOps.colorize(fr.convert("L"), black=(8, 10, 14), white=(170, 220, 255))
            img = img.resize((128 * self.scale, 40 * self.scale), Image.NEAREST)
            ci = ctk.CTkImage(light_image=img, dark_image=img, size=img.size)
            self.lbl.configure(image=ci)
            self._img = ci
        except Exception:
            pass
        try:
            self.after(100, self._loop)
        except Exception:
            pass


def field_row(parent, key, data, base, fire):
    """Bir ayar satırı: kutucuk işaretliyse bu değer 'data' içinde geçersiz kılınır, değilse genel ayar kullanılır."""
    label, kind, spec, dflt = FIELDS[key]
    row = ctk.CTkFrame(parent, fg_color="transparent")
    row.pack(fill="x", padx=14, pady=4)
    var = tk.BooleanVar(value=key in data)
    cur = data.get(key, base.get(key, dflt))
    holder = ctk.CTkFrame(row, fg_color="transparent")
    box = {}

    def read():
        w = box["w"]
        if kind in ("seg", "opt"):
            return w.get()
        if kind == "map":
            return spec.get(w.get(), dflt)
        if kind == "sw":
            return bool(w.get())
        if kind == "sl":
            v = w.get()
            return float(round(v, 1)) if spec[4] is float else int(round(v))
        return w.get()

    def commit(*_):
        if var.get():
            data[key] = read()
        else:
            data.pop(key, None)
        if kind == "sl":
            try:
                box["val"].configure(text=spec[3].format(read()))
            except Exception:
                pass
        fire()

    def set_enabled():
        try:
            box["w"].configure(state="normal" if var.get() else "disabled")
        except Exception:
            pass

    def toggled():
        set_enabled()
        commit()

    ctk.CTkCheckBox(row, text="", width=26, checkbox_width=20, checkbox_height=20, variable=var, command=toggled,
                    fg_color=TH["accent"], hover_color=TH["accent_hover"], border_color=TH["muted"]).pack(side="left")
    lbl(row, label, 13, width=190, anchor="w").pack(side="left")
    holder.pack(side="left")
    if kind == "seg":
        w = segmented(holder, spec, commit)
        w.set(cur if cur in spec else spec[0])
    elif kind == "opt":
        w = option(holder, spec, 190, commit)
        w.set(cur if cur in spec else spec[0])
    elif kind == "map":
        w = option(holder, list(spec), 230, commit)
        rev = {v: k for k, v in spec.items()}
        w.set(rev.get(cur, next(iter(spec))))
    elif kind == "sw":
        w = switch(holder, "", commit)
        if cur:
            w.select()
        else:
            w.deselect()
    elif kind == "sl":
        w = slider(holder, spec[0], spec[1], spec[2], commit, 200)
        w.set(cur)
        box["val"] = lbl(holder, spec[3].format(cur), 12, True, width=90, anchor="w")
    else:
        w = entry(holder, 230, "")
        w.insert(0, str(cur or ""))
        w.bind("<KeyRelease>", commit)
    box["w"] = w
    w.pack(side="left")
    if kind == "sl":
        box["val"].pack(side="left", padx=(10, 0))
    set_enabled()


def build_fields(parent, keys, data, base, on_change=None):
    for key in keys:
        field_row(parent, key, data, base, on_change or (lambda: None))


# ---------------------------------------------------------------- GameSense
class GameSense:
    def __init__(self):
        self.base = None
        self.counter = 0
        self.http = requests.Session()

    def _post(self, path, payload):
        r = self.http.post(self.base + path, json=payload, timeout=3)
        r.raise_for_status()

    def connect(self):
        with open(CORE_PROPS, "r", encoding="utf-8") as f:
            addr = json.load(f)["address"]
        self.base = "http://" + addr
        self._post("/game_metadata", {
            "game": GAME,
            "game_display_name": "Apex OLED Tool",
            "developer": "Apex OLED Tool",
        })
        self._post("/bind_game_event", {
            "game": GAME, "event": "SHOW_TEXT",
            "min_value": 0, "max_value": 100, "icon_id": 0,
            "handlers": [{
                "device-type": "screened-128x40", "zone": "one", "mode": "screen",
                "datas": [{
                    "icon-id": 0,
                    "lines": [{"has-text": True, "context-frame-key": "line1"}],
                }],
            }],
        })
        self._post("/bind_game_event", {
            "game": GAME, "event": "SHOW_IMAGE",
            "min_value": 0, "max_value": 100, "icon_id": 0,
            "handlers": [{
                "device-type": "screened-128x40", "zone": "one", "mode": "screen",
                "datas": [{"has-text": False, "image-data": [0] * 640}],
            }],
        })

    def _event(self, event, frame):
        self.counter = (self.counter + 1) % 100
        self._post("/game_event", {
            "game": GAME, "event": event,
            "data": {"value": self.counter, "frame": frame},
        })

    def show_text(self, text):
        self._event("SHOW_TEXT", {"line1": text})

    def show_image(self, data):
        self._event("SHOW_IMAGE", {"image-data-128x40": data})


# ---------------------------------------------------------------- arka plan işçisi
class Worker(threading.Thread):
    def __init__(self, cfg, media, stats, vol, set_status):
        super().__init__(daemon=True)
        self.cfg = cfg
        self.media = media
        self.stats = stats
        self.vol = vol
        self.set_status = set_status
        self.stop_flag = threading.Event()
        self.gs = GameSense()
        self.cache = {}
        self.last_fg = None
        self.connected = False
        self.eff = dict(cfg)
        self.cur_profile = ""
        self._sys_cache = (0.0, "")
        self.t_start = time.time()
        self.force_sys_until = 0.0
        self._media_last = None
        self._pop_until = 0.0
        self.source = "—"

    def _image_data(self, name, invert):
        key = (name, invert)
        if key not in self.cache:
            self.cache[key] = image_to_bytes(os.path.join(IMG_DIR, name), invert)
        return self.cache[key]

    def match(self, exe, title):
        if not exe:
            return None
        cands = [r for r in self.cfg["rules"] if r["exe"].lower() == exe and r.get("enabled", True)]
        cands.sort(key=lambda r: 0 if (r.get("filter") or "").strip() else 1)
        low = (title or "").lower()
        for r in cands:
            f = (r.get("filter") or "").strip().lower()
            if not f or f in low:
                return r
        return None

    def pick_profile(self, exe, rule):
        profiles = self.cfg.get("profiles", [])
        cat = rule.get("category") if rule else None
        for p in profiles:
            if exe and exe in [x.lower() for x in split_list(p.get("exes"))]:
                return p
            if cat and cat in p.get("cats", []):
                return p
        name = self.cfg.get("active_profile")
        if name:
            for p in profiles:
                if p.get("name") == name:
                    return p
        return None

    def decorate(self, text, cat):
        return decorate_text(self.cfg, text, cat)

    def sys_text(self, eff):
        now = time.time()
        if eff.get("sys_mode") == "joined":
            if now - self._sys_cache[0] >= 2.0:
                self._sys_cache = (now, sys_compose(eff, self.stats.data, now))
            return self._sys_cache[1]
        return sys_compose(eff, self.stats.data, now)

    def rule_content(self, rule, exe, title, eff):
        cat = rule.get("category", DEFAULT_CATEGORY)
        if rule.get("image"):
            return ("image", rule["image"], bool(eff["invert"]))
        mode = rule.get("mode", "text")
        fallback = rule.get("text") or os.path.splitext(rule["exe"])[0]
        remove = split_list(rule.get("remove"))
        text = ""
        if mode == "media":
            text = clean_text(MediaWatcher.text_of(self.media.pick(exe)), remove)
        elif mode == "title":
            text = clean_text(title, remove)
        elif mode == "sys":
            text = self.sys_text(eff)
        return ("text", self.decorate(text or fallback, cat), cat, "sys" if mode == "sys" else None)

    def volume_content(self, eff):
        v = self.vol
        if not eff.get("vol_on") or v.level is None:
            return None
        if time.time() - v.changed_at >= float(eff.get("vol_secs", 2.0)):
            return None
        if eff.get("native_text"):
            txt = eff.get("vol_mute", "Sessiz") if v.muted else eff.get("vol_text", "Ses: {v}").replace("{v}", str(v.level))
            return ("text", txt, "Sistem", "vol")
        return ("vol", v.level, v.muted)

    def alert_content(self, eff):
        if not eff.get("alerts_on"):
            return None
        d = self.stats.data
        msgs = []
        ct, gt = d.get("cpu_temp"), d.get("gpu_temp")
        if ct is not None and ct >= float(eff.get("alert_cpu", 85)):
            msgs.append(f"CPU {ct:.0f}°C")
        if gt is not None and gt >= float(eff.get("alert_gpu", 85)):
            msgs.append(f"GPU {gt:.0f}°C")
        if msgs:
            return ("text", "! SICAK  " + "  ".join(msgs), "Sistem", "alert")
        return None

    def sys_interval(self, eff):
        """Belirli aralıklarla kısa süre görünen sistem bilgisi."""
        if eff.get("sys_when") != "interval":
            return None
        period = max(1, int(eff.get("sys_every", 5))) * 60
        show = float(eff.get("sys_show_secs", 12))
        data = self.stats.data
        if eff.get("sys_mode") != "joined":
            n = len(sys_lines(eff, data))
            show = max(show, n * max(1.0, float(eff.get("sys_secs", 4))))
        phase = (time.time() - self.t_start) % period
        if phase >= show:
            return None
        t = sys_compose(eff, data, phase)
        if not t:
            return None
        return ("text", self.decorate(t, "Sistem"), "Sistem", "sys")

    def resolve(self):
        cfg = self.cfg
        now = time.time()
        exe, title = foreground_info()
        if exe in OWN_NAMES and self.last_fg:
            exe, title = self.last_fg
        else:
            self.last_fg = (exe, title)
        rule = self.match(exe, title)
        prof = self.pick_profile(exe, rule)
        eff = dict(cfg)
        if prof:
            eff.update(prof.get("settings", {}))
        self.eff = eff
        self.cur_profile = prof.get("name", "") if prof else ""

        rc = self.rule_content(rule, exe, title, eff) if rule else None
        media_txt = MediaWatcher.text_of(self.media.pick(exe)) if eff["bg_media"] else ""
        mc = ("text", self.decorate(media_txt, "Müzik"), "Müzik", None) if media_txt else None

        # yeni şarkı bildirimi
        cur = media_txt or ""
        if self._media_last is None:
            self._media_last = cur
        elif cur and cur != self._media_last:
            self._media_last = cur
            if eff.get("media_pop", True):
                self._pop_until = now + float(eff.get("media_pop_secs", 5))
        elif not cur:
            self._media_last = ""
        pop = mc if (mc and now < self._pop_until) else None

        # boşta içerik
        fb = None
        if cfg["default_text"]:
            fb = ("text", self.decorate(cfg["default_text"], "Diğer"), "Diğer", None)
        else:
            if eff.get("sys_when", "idle") == "idle":
                t = self.sys_text(eff)
                if t:
                    fb = ("text", self.decorate(t, "Sistem"), "Sistem", "sys")
            if fb is None and eff.get("idle_mode") == "clock":
                fb = ("text", time.strftime("%H:%M"), "Sistem", "clock")
        st = self.sys_interval(eff)

        pr = eff["priority"]
        cats = eff.get("prio_cats", [])
        rcat = rule.get("category") if rule else None
        if pr == "media":
            order = [mc, rc]
        elif pr == "rotate":
            if rc and mc:
                rs, ms = max(2.0, float(eff.get("rule_secs", 8))), max(2.0, float(eff.get("media_secs", 8)))
                order = [rc, mc] if (now % (rs + ms)) < rs else [mc, rc]
            else:
                order = [rc, mc]
        elif pr == "category":
            order = [rc, mc] if (rcat and rcat in cats) else [mc, rc]
        else:
            order = [rc, mc]
        if st and eff.get("sys_over", True):
            order = [st] + order
        else:
            order = order + [st]
        order.append(fb)
        picked = next((c for c in order if c), None)

        vc = self.volume_content(eff)
        ac = self.alert_content(eff)
        forced = None
        if now < self.force_sys_until:
            t = sys_compose(eff, self.stats.data, now) or "Sistem verisi yok"
            forced = ("text", self.decorate(t, "Sistem"), "Sistem", "sys")
        final = vc or ac or forced or pop or picked
        names = [("Ses göstergesi", vc), ("Sıcaklık uyarısı", ac), ("Sistem bilgisi (test)", forced),
                 ("Yeni şarkı bildirimi", pop), ("Kural", rc), ("Çalan medya", mc),
                 ("Sistem bilgisi (aralıklı)", st), ("Boşta içerik", fb)]
        self.source = next((n for n, c in names if c is not None and c is final), "Hiçbir şey")
        return final

    def text_cfg(self, content):
        eff = self.eff
        m = dict(eff)
        if content and content[0] == "text" and len(content) > 2:
            m.update(self.cfg.get("cat_settings", {}).get(content[2], {}))
        return text_settings(m)

    @staticmethod
    def describe(content):
        if content is None:
            return "Çalışıyor · eşleşme yok"
        if content[0] == "image":
            return "Çalışıyor · görsel"
        if content[0] == "vol":
            return f"Çalışıyor · ses {content[1]}"
        return "Çalışıyor · " + content[1][:40]

    def run(self):
        content = None
        scroller = None
        last_sig = None
        t0 = 0.0
        last_off = None
        last_send = 0.0
        last_eval = 0.0
        while not self.stop_flag.is_set():
            tick = time.time()
            try:
                if not self.connected:
                    try:
                        self.gs.connect()
                        self.connected = True
                        content, last_sig, last_eval = None, None, 0.0
                        self.set_status("Çalışıyor")
                    except Exception:
                        self.set_status("Bekliyor · SteelSeries GG hazır değil, tekrar denenecek")
                        self.stop_flag.wait(4)
                        continue

                now = time.time()
                changed = False
                keep = False
                if now - last_eval >= 0.4:
                    last_eval = now
                    new = self.resolve()
                    if new != content:
                        keep = bool(content and new and content[0] == "text" and new[0] == "text"
                                    and content[3] is not None and content[3] == new[3])
                        content = new
                        changed = True
                eff = self.eff
                ts = self.text_cfg(content)
                sig = ts + (bool(eff["native_text"]),)
                if sig != last_sig:
                    last_sig = sig
                    changed = True
                    keep = False
                if changed:
                    scroller, last_off, last_send = None, None, 0.0
                    if not keep:
                        t0 = now
                    if content and content[0] == "text" and not eff["native_text"]:
                        scroller = Scroller(content[1], *ts)
                    self.set_status(self.describe(content))

                if content is None:
                    pass
                elif content[0] == "vol":
                    if now - last_send >= HEARTBEAT:
                        img = render_volume(content[1], content[2], eff.get("vol_style", "Yazı + çubuk"),
                                            eff.get("vol_text", "Ses: {v}"), eff.get("vol_mute", "Sessiz"))
                        self.gs.show_image(list(img.tobytes()))
                        last_send = now
                elif content[0] == "image":
                    if now - last_send >= HEARTBEAT:
                        self.gs.show_image(self._image_data(content[1], content[2]))
                        last_send = now
                elif eff["native_text"]:
                    if last_off is None or now - last_send >= HEARTBEAT:
                        self.gs.show_text(content[1])
                        last_off, last_send = 0, now
                elif scroller is not None and scroller.animated:
                    off = scroller.offset(now - t0)
                    if off != last_off or now - last_send >= HEARTBEAT:
                        self.gs.show_image(list(scroller.frame(now - t0).tobytes()))
                        last_off, last_send = off, now
                elif scroller is not None:
                    if last_off is None or now - last_send >= HEARTBEAT:
                        self.gs.show_image(list(scroller.frame(0).tobytes()))
                        last_off, last_send = 0, now
            except Exception as e:
                self.connected = False
                self.set_status(f"Hata: {e}")
                self.stop_flag.wait(2)
                continue
            fps = max(3, int(self.eff.get("fps", 10)))
            self.stop_flag.wait(max(0.01, 1.0 / fps - (time.time() - tick)))
        self.set_status("Durduruldu")


# ---------------------------------------------------------------- profil penceresi
class ProfileDialog(ctk.CTkToplevel):
    def __init__(self, master, profile=None):
        super().__init__(master, fg_color=TH["bg"])
        self.app = master
        self.title("Profil düzenle" if profile else "Yeni profil")
        self.geometry("720x760")
        self.minsize(680, 600)
        self.result = None
        p = profile or {}
        self.settings = dict(p.get("settings", {}))
        self.cat_vars = {}

        bottom = ctk.CTkFrame(self, fg_color="transparent")
        bottom.pack(side="bottom", fill="x", padx=24, pady=14)
        btn(bottom, "Kaydet", self.ok, width=130, height=40, radius=20).pack(side="right")
        btn(bottom, "İptal", self.destroy, kind="soft", width=90, height=40, radius=20).pack(side="right", padx=8)

        body = scroll_frame(self)
        body.pack(fill="both", expand=True, padx=12, pady=(12, 0))

        c1 = card(body)
        c1.pack(fill="x", padx=6, pady=(0, 10))
        lbl(c1, "Profil adı", 13, True).pack(anchor="w", padx=16, pady=(14, 4))
        self.name = entry(c1, 340, "Örn: Oyun modu", radius=14, height=34)
        self.name.pack(anchor="w", padx=16)
        self.name.insert(0, p.get("name", ""))

        lbl(c1, "Otomatik geçiş: şu programlar öndeyken (virgülle)", 13, True).pack(anchor="w", padx=16, pady=(14, 4))
        self.exes = entry(c1, 440, "Örn: cs2.exe, valorant.exe", radius=14, height=34)
        self.exes.pack(anchor="w", padx=16)
        self.exes.insert(0, p.get("exes", ""))

        lbl(c1, "Otomatik geçiş: şu kategorideki bir kural eşleşince", 13, True).pack(anchor="w", padx=16, pady=(14, 4))
        grid = ctk.CTkFrame(c1, fg_color="transparent")
        grid.pack(anchor="w", padx=16)
        for i, name in enumerate(CATEGORIES):
            v = tk.BooleanVar(value=name in p.get("cats", []))
            self.cat_vars[name] = v
            ctk.CTkCheckBox(grid, text=name, variable=v, width=110, fg_color=CATEGORIES[name],
                            hover_color=CATEGORIES[name], border_color=TH["muted"],
                            text_color=TH["text"]).grid(row=i // 4, column=i % 4, padx=6, pady=4, sticky="w")
        lbl(c1, "Hiçbirini seçmezsen profil sadece sağ üstteki menüden elle seçilince çalışır.", 12, muted=True,
            wraplength=620, justify="left").pack(anchor="w", padx=16, pady=(8, 14))

        c2 = card(body)
        c2.pack(fill="x", padx=6, pady=(0, 10))
        lbl(c2, "Bu profilde değişecek ayarlar", 15, True).pack(anchor="w", padx=16, pady=(14, 0))
        lbl(c2, "Kutucuğu işaretlediğin ayar bu profilde senin değerinle çalışır. İşaretsiz ayarlar genel ayarlardan gelir.",
            12, muted=True, wraplength=620, justify="left").pack(anchor="w", padx=16, pady=(2, 8))
        build_fields(c2, PROFILE_FIELDS, self.settings, master.cfg)
        ctk.CTkFrame(c2, height=10, fg_color="transparent").pack()

        self.transient(master)
        self.lift()
        self.after(200, self.grab_set)

    def ok(self):
        name = self.name.get().strip()
        if not name:
            messagebox.showwarning("Eksik", "Profil adı gerekli.", parent=self)
            return
        self.result = {
            "name": name, "exes": self.exes.get().strip(),
            "cats": [n for n, v in self.cat_vars.items() if v.get()],
            "settings": dict(self.settings),
        }
        self.destroy()


# ---------------------------------------------------------------- kural penceresi
class RuleDialog(ctk.CTkToplevel):
    def __init__(self, master, rule=None):
        super().__init__(master, fg_color=TH["bg"])
        self.app = master
        self.title("Kural düzenle" if rule else "Yeni kural")
        self.geometry("960x740")
        self.minsize(900, 660)
        self.result = None
        self.rule = rule or {}
        self.image_src = ""
        self.image_name = self.rule.get("image", "")
        self.keep = []
        self.keep_list = []
        self.sel_exe = self.rule.get("exe", "")
        self.sel_path = self.rule.get("path", "")
        self.cat = self.rule.get("category", DEFAULT_CATEGORY)
        self.mode = self.rule.get("mode", "text")
        self.show_bg = False
        self.visible, self.others = [], []
        self._img_key = None
        self._img_val = None
        try:
            self.visible, self.others = list_programs()
        except Exception:
            pass

        bottom = ctk.CTkFrame(self, fg_color="transparent")
        bottom.pack(side="bottom", fill="x", padx=24, pady=14)
        btn(bottom, "Kaydet", self.ok, width=130, height=40, radius=20).pack(side="right")
        btn(bottom, "İptal", self.destroy, kind="soft", width=90, height=40, radius=20).pack(side="right", padx=8)

        body = ctk.CTkFrame(self, fg_color="transparent")
        body.pack(fill="both", expand=True, padx=20, pady=(16, 0))
        body.grid_columnconfigure(0, weight=0)
        body.grid_columnconfigure(1, weight=1)
        body.grid_rowconfigure(0, weight=1)

        # ---- sol: program seçimi
        left = card(body, width=320)
        left.grid(row=0, column=0, sticky="ns", padx=(0, 12))
        left.pack_propagate(False)
        lbl(left, "1 · Program seç", 15, True).pack(anchor="w", padx=16, pady=(14, 2))
        lbl(left, "Açık pencerelerden birine tıkla ya da arama kutusuna program adını yaz.", 12, muted=True,
            wraplength=280, justify="left").pack(anchor="w", padx=16)
        self.search = entry(left, 288, "Ara ya da yaz (örn: chrome.exe)", radius=16, height=34)
        self.search.pack(padx=16, pady=(10, 6))
        self.search.bind("<KeyRelease>", lambda _e: self.fill_list())
        self.sw_bg = switch(left, "Arka plan işlemlerini de göster", self.toggle_bg)
        self.sw_bg.pack(anchor="w", padx=16, pady=(0, 4))
        self.plist = scroll_frame(left)
        self.plist.pack(fill="both", expand=True, padx=6, pady=(4, 10))

        # ---- sağ: ayarlar
        right = scroll_frame(body)
        right.grid(row=0, column=1, sticky="nsew")

        sc = card(right)
        sc.pack(fill="x", padx=4, pady=(0, 8))
        srow = ctk.CTkFrame(sc, fg_color="transparent")
        srow.pack(fill="x", padx=14, pady=12)
        self.sel_holder = ctk.CTkFrame(srow, fg_color="transparent", width=56, height=56)
        self.sel_holder.pack(side="left")
        self.sel_holder.pack_propagate(False)
        sinfo = ctk.CTkFrame(srow, fg_color="transparent")
        sinfo.pack(side="left", padx=12)
        lbl(sinfo, "Seçili program", 12, muted=True).pack(anchor="w")
        self.sel_lbl = lbl(sinfo, "—", 17, True)
        self.sel_lbl.pack(anchor="w")

        pc = card(right)
        pc.pack(fill="x", padx=4, pady=8)
        lbl(pc, "Ekranda böyle görünür", 12, muted=True).pack(anchor="w", padx=16, pady=(12, 6))
        OledPreview(pc, self.preview_params).pack(anchor="w", padx=16, pady=(0, 14))

        mc = card(right)
        mc.pack(fill="x", padx=4, pady=8)
        lbl(mc, "2 · Ekranda ne gösterilsin?", 15, True).pack(anchor="w", padx=16, pady=(14, 8))
        self.mode_seg = segmented(mc, [l for l, _ in MODE_LIST], self.on_mode)
        self.mode_seg.pack(fill="x", padx=16)
        self.mode_seg.set(next((l for l, v in MODE_LIST if v == self.mode), "Yazı"))
        self.mode_desc = lbl(mc, "", 12, muted=True, wraplength=520, justify="left")
        self.mode_desc.pack(anchor="w", padx=16, pady=(8, 8))
        self.text_lbl = lbl(mc, "", 13, True)
        self.text_lbl.pack(anchor="w", padx=16, pady=(4, 4))
        self.text = entry(mc, 420, "Örn: CHROME", radius=14, height=34)
        self.text.pack(anchor="w", padx=16, pady=(0, 14))
        if self.rule.get("text"):
            self.text.insert(0, self.rule["text"])

        cc = card(right)
        cc.pack(fill="x", padx=4, pady=8)
        lbl(cc, "3 · Kategori", 15, True).pack(anchor="w", padx=16, pady=(14, 2))
        lbl(cc, "Kategorinin kendi yazı ayarları (hız, boyut...) bu kurala uygulanır.", 12, muted=True).pack(
            anchor="w", padx=16, pady=(0, 6))
        chips = ctk.CTkFrame(cc, fg_color="transparent")
        chips.pack(anchor="w", padx=12, pady=(0, 14))
        self.chip_btns = {}
        for i, (name, col) in enumerate(CATEGORIES.items()):
            b = ctk.CTkButton(chips, text=name, width=92, height=30, corner_radius=15, border_width=2,
                              border_color=col, fg_color="transparent", text_color=TH["text"],
                              hover_color=mix(TH["card"], col, 0.35), font=font(12, True),
                              command=lambda n=name: self.set_cat(n))
            b.grid(row=i // 4, column=i % 4, padx=4, pady=4)
            self.chip_btns[name] = b
        self.apply_chips()

        ic = card(right)
        ic.pack(fill="x", padx=4, pady=8)
        lbl(ic, "4 · Görsel (isteğe bağlı)", 15, True).pack(anchor="w", padx=16, pady=(14, 6))
        irow = ctk.CTkFrame(ic, fg_color="transparent")
        irow.pack(fill="x", padx=16)
        btn(irow, "Görsel seç...", self.pick, width=110, height=34, radius=17).pack(side="left")
        btn(irow, "Kaldır", self.clear_img, kind="soft", width=76, height=34, radius=17).pack(side="left", padx=8)
        self.img_lbl = lbl(irow, "", 12, muted=True)
        self.img_lbl.pack(side="left", padx=6)
        lbl(ic, "Görsel seçersen yazı yerine görsel gösterilir (128x40, tek renge çevrilir).", 12, muted=True,
            wraplength=520, justify="left").pack(anchor="w", padx=16, pady=(6, 14))

        self.adv_open = bool(self.rule.get("filter") or self.rule.get("remove"))
        ac = card(right)
        ac.pack(fill="x", padx=4, pady=8)
        self.adv_btn = btn(ac, "", self.toggle_adv, kind="soft", width=200, height=32, radius=16)
        self.adv_btn.pack(anchor="w", padx=16, pady=14)
        self.adv = ctk.CTkFrame(ac, fg_color="transparent")
        lbl(self.adv, "Sadece başlıkta şu ifade geçerse", 13, True).pack(anchor="w", padx=16, pady=(0, 4))
        self.filter = entry(self.adv, 420, "Örn: Anizium  (aynı tarayıcıda farklı siteleri ayırır)", radius=14, height=34)
        self.filter.pack(anchor="w", padx=16)
        if self.rule.get("filter"):
            self.filter.insert(0, self.rule["filter"])
        lbl(self.adv, "Başlıktan silinecek ifadeler (virgülle)", 13, True).pack(anchor="w", padx=16, pady=(12, 4))
        self.remove = entry(self.adv, 420, "Örn: Anizium, İzle, YouTube Music", radius=14, height=34)
        self.remove.pack(anchor="w", padx=16, pady=(0, 14))
        if self.rule.get("remove"):
            self.remove.insert(0, self.rule["remove"])
        self.apply_adv()

        self.on_mode(self.mode_seg.get())
        self.refresh_img_label()
        self.refresh_selected()
        self.fill_list()
        self.transient(master)
        self.lift()
        self.after(200, self.grab_set)

    # ---- program listesi
    def toggle_bg(self):
        self.show_bg = bool(self.sw_bg.get())
        self.fill_list()

    def fill_list(self):
        for w in self.plist.winfo_children():
            w.destroy()
        self.keep_list.clear()
        q = self.search.get().strip().lower()
        groups = [("AÇIK PENCERELER", self.visible, True)]
        if self.show_bg:
            groups.append(("ARKA PLAN İŞLEMLERİ", self.others, False))
        shown = 0
        exact = False
        for title, items, extract in groups:
            rows = [it for it in items if not q or q in it[0].lower() or q in it[2].lower()]
            if not rows:
                continue
            lbl(self.plist, title, 11, True, muted=True).pack(anchor="w", padx=10, pady=(8, 2))
            for exe, path, wt in rows[:80]:
                self.make_prog_row(exe, path, wt, extract)
                shown += 1
                if exe.lower() == q:
                    exact = True
        if q and not exact:
            name = q if q.endswith(".exe") else q + ".exe"
            row = ctk.CTkFrame(self.plist, corner_radius=12, fg_color=TH["card2"])
            row.pack(fill="x", padx=4, pady=(10, 3))
            t = lbl(row, f"“{name}” adını kullan", 13, True)
            t.pack(padx=12, pady=10)
            bind_click([row, t], lambda _e, n=name: self.pick_prog(n, ""))
        elif shown == 0:
            lbl(self.plist, "Açık pencere bulunamadı.\nArka plan işlemlerini aç ya da adı yaz.", 12, muted=True,
                justify="left").pack(padx=12, pady=20)

    def make_prog_row(self, exe, path, wt, extract):
        sel = exe.lower() == self.sel_exe.lower()
        bg = TH["card2"] if sel else TH["card"]
        row = ctk.CTkFrame(self.plist, corner_radius=12, fg_color=bg, border_width=2 if sel else 0,
                           border_color=TH["accent"])
        row.pack(fill="x", padx=4, pady=3)
        ic = make_icon(row, exe, path, CATEGORIES[DEFAULT_CATEGORY], self.keep_list, size=34, bg=bg, extract=extract)
        ic.pack(side="left", padx=(8, 8), pady=6)
        info = ctk.CTkFrame(row, fg_color="transparent")
        info.pack(side="left", fill="x", expand=True)
        n = lbl(info, exe, 13, True, anchor="w")
        n.pack(anchor="w")
        ws = [row, ic, info, n]
        if wt:
            t = lbl(info, wt if len(wt) <= 30 else wt[:29] + "…", 11, muted=True, anchor="w")
            t.pack(anchor="w")
            ws.append(t)
        bind_click(ws, lambda _e, e=exe, p=path: self.pick_prog(e, p))

    def pick_prog(self, exe, path):
        self.sel_exe = exe
        self.sel_path = path
        self.refresh_selected()
        self.after(1, self.fill_list)

    def refresh_selected(self):
        for w in self.sel_holder.winfo_children():
            w.destroy()
        self.keep.clear()
        color = CATEGORIES.get(self.cat, CATEGORIES[DEFAULT_CATEGORY])
        if self.sel_exe:
            make_icon(self.sel_holder, self.sel_exe, self.sel_path, color, self.keep, size=52,
                      bg=TH["card"]).pack(expand=True)
        self.sel_lbl.configure(text=self.sel_exe or "Henüz seçilmedi")

    # ---- kategori / mod
    def set_cat(self, name):
        self.cat = name
        self.apply_chips()
        self.refresh_selected()

    def apply_chips(self):
        for n, b in self.chip_btns.items():
            if n == self.cat:
                b.configure(fg_color=CATEGORIES[n], text_color="#ffffff")
            else:
                b.configure(fg_color="transparent", text_color=TH["text"])

    def on_mode(self, label):
        self.mode = dict(MODE_LIST).get(label, "text")
        self.mode_desc.configure(text=MODE_DESC[self.mode])
        self.text_lbl.configure(text="Gösterilecek yazı" if self.mode == "text"
                                else "Yedek yazı (veri bulunamazsa gösterilir)")

    def toggle_adv(self):
        self.adv_open = not self.adv_open
        self.apply_adv()

    def apply_adv(self):
        self.adv_btn.configure(text=("Gelişmiş ayarlar ▾" if self.adv_open else "Gelişmiş ayarlar ▸"))
        if self.adv_open:
            self.adv.pack(fill="x")
        else:
            self.adv.pack_forget()

    # ---- önizleme
    def preview_params(self):
        app = self.app
        cfg = app.cfg
        if self.image_src or self.image_name:
            path = self.image_src or os.path.join(IMG_DIR, self.image_name)
            key = (path, bool(cfg["invert"]))
            if key != self._img_key:
                self._img_key = key
                try:
                    self._img_val = image_to_mono(path, key[1])
                except Exception:
                    self._img_val = Image.new("1", (128, 40), 0)
            return self._img_val
        txt = self.text.get().strip()
        if self.mode == "media":
            sample = "Sanatçı - Şarkı Adı"
        elif self.mode == "title":
            sample = "Anime Adı - 5. Bölüm"
        elif self.mode == "sys":
            sample = sys_compose(cfg, app.stats.data, time.time()) or "CPU %34"
        else:
            sample = txt or os.path.splitext(self.sel_exe)[0].upper() or "ÖRNEK"
        m = dict(cfg)
        m.update(cfg.get("cat_settings", {}).get(self.cat, {}))
        return (decorate_text(cfg, sample, self.cat), text_settings(m))

    # ---- görsel
    def refresh_img_label(self):
        name = self.image_src and os.path.basename(self.image_src) or (
            self.image_name.split("_", 1)[-1] if self.image_name else "")
        self.img_lbl.configure(text=name or "(yok)")

    def pick(self):
        p = filedialog.askopenfilename(
            parent=self, filetypes=[("Görseller", "*.png *.jpg *.jpeg *.bmp *.gif *.ico"), ("Hepsi", "*.*")])
        if p:
            self.image_src = p
            self.refresh_img_label()

    def clear_img(self):
        self.image_src = ""
        self.image_name = ""
        self.refresh_img_label()

    def ok(self):
        exe = (self.sel_exe or self.search.get().strip()).strip()
        if not exe:
            messagebox.showwarning("Eksik", "Soldaki listeden bir program seç ya da adını yaz.", parent=self)
            return
        if not exe.lower().endswith(".exe"):
            exe += ".exe"
        image = self.image_name
        if self.image_src:
            os.makedirs(IMG_DIR, exist_ok=True)
            image = f"{int(time.time())}_{os.path.basename(self.image_src)}"
            shutil.copy(self.image_src, os.path.join(IMG_DIR, image))
        path = self.sel_path or find_running_path(exe) or self.rule.get("path", "")
        _no_icon.discard(exe.lower())
        self.result = {
            "exe": exe, "text": self.text.get().strip(), "image": image,
            "category": self.cat, "path": path, "mode": self.mode,
            "filter": self.filter.get().strip(), "remove": self.remove.get().strip(),
        }
        self.destroy()


# ---------------------------------------------------------------- ana pencere
(PAGE_RULES, PAGE_PROF, PAGE_CAT, PAGE_TEXT, PAGE_SYS, PAGE_PRIO, PAGE_THEME, PAGE_GEN) = (
    "Kurallar", "Profiller", "Kategoriler", "Yazı ve kaydırma", "Sistem", "Öncelik", "Tema", "Genel")
PAGE_ORDER = [PAGE_RULES, PAGE_PROF, PAGE_CAT, PAGE_TEXT, PAGE_SYS, PAGE_PRIO, PAGE_THEME, PAGE_GEN]
NAV_GLYPH = {PAGE_RULES: "☰", PAGE_PROF: "⚑", PAGE_CAT: "▣", PAGE_TEXT: "Aa", PAGE_SYS: "◉",
             PAGE_PRIO: "⇅", PAGE_THEME: "◐", PAGE_GEN: "⚙"}
CAT_NOTES = {
    "Müzik": "Çalan medya (şarkı / video adı) bu kategorinin ayarlarını kullanır.",
    "Sistem": "Sistem bilgisi, boşta saat ve sıcaklık uyarısı bu kategorinin ayarlarını kullanır.",
    "Diğer": "'Hiçbir şey yokken yazı' (Genel sekmesi) bu kategorinin ayarlarını kullanır.",
}
PREVIEW_SCALE = 3


class App(ctk.CTk):
    def __init__(self):
        super().__init__()
        self.cfg = load_config()
        set_theme_colors(self.cfg["theme"])
        ctk.set_appearance_mode(TH["mode"])
        self.title("Apex OLED")
        self.geometry("1120x780")
        self.minsize(980, 680)
        self.worker = None
        self.filter = "Tümü"
        self.keep = []
        self.media = MediaWatcher()
        self.media.start()
        self.stats = SysStats(self.cfg)
        self.stats.start()
        self.vol = VolumeWatcher()
        self.vol.start()
        self.building = False
        self.status_text = "Durduruldu"
        self.status_state = "off"
        self.current_page = PAGE_RULES
        self.pages = {}
        self.nav = {}
        self._prev_sig = None
        self._prev_scroller = None
        self._prev_t0 = time.time()
        self._prev_img = None
        self._tick_n = 0
        self.side_open = bool(self.cfg.get("side_open", True))
        self.q = queue.Queue()
        self.tray = None
        self.sys_rows = {}
        self.cat_sel = "Oyun"

        self.build_ui(PAGE_RULES)
        self.protocol("WM_DELETE_WINDOW", self.on_close)
        self.after(100, self.tick)
        if self.cfg["auto_start"]:
            self.after(1000, self.toggle)
        if "--startup" in sys.argv and self.cfg["start_minimized"]:
            self.after(300, self.hide_to_tray if (pystray and self.cfg.get("close_to_tray", True)) else self.iconify)

    # ---- iskelet
    def build_ui(self, page):
        self.building = True
        for w in self.winfo_children():
            w.destroy()
        self.keep.clear()
        self.pages, self.nav = {}, {}
        self._hdr_w = 0
        self._hdr_job = None
        self._hdr_photo = None
        self.configure(fg_color=TH["bg"])
        self.grid_columnconfigure(0, weight=0)
        self.grid_columnconfigure(1, weight=1)
        self.grid_rowconfigure(0, weight=1)

        # sol şerit (açılır / kapanır)
        self.side = ctk.CTkFrame(self, width=SIDE_OPEN if self.side_open else SIDE_CLOSED, corner_radius=0,
                                 fg_color=TH["side"])
        self.side.grid(row=0, column=0, sticky="nsew")
        self.side.grid_propagate(False)
        self.side.pack_propagate(False)
        side = self.side
        head = ctk.CTkFrame(side, fg_color="transparent")
        head.pack(fill="x", padx=11, pady=(20, 18))
        self.toggle_btn = ctk.CTkButton(head, text="≡", width=56, height=46, corner_radius=16,
                                        fg_color=TH["accent"], hover_color=TH["accent_hover"],
                                        text_color="#ffffff", font=font(22, True), command=self.toggle_side)
        self.toggle_btn.pack(side="left")
        self.brand = lbl(head, "Apex OLED", 17, True)
        for name in PAGE_ORDER:
            b = ctk.CTkButton(side, text=NAV_GLYPH[name], width=56, height=46, corner_radius=16,
                              fg_color="transparent", hover_color=TH["card2"], text_color=TH["muted"],
                              font=font(18, True), command=lambda n=name: self.show_page(n))
            b.pack(padx=11, pady=2)
            self.nav[name] = b

        # kapanış zamanlayıcısı (sol alt)
        self.timer_card = ctk.CTkFrame(side, corner_radius=16, fg_color=TH["card"], border_width=1,
                                       border_color=TH["border"])
        lbl(self.timer_card, "Kapanış zamanlayıcı", 12, True).pack(anchor="w", padx=12, pady=(10, 4))
        trow = ctk.CTkFrame(self.timer_card, fg_color="transparent")
        trow.pack(fill="x", padx=12)
        self.timer_entry = entry(trow, 54, "", radius=10, height=32)
        self.timer_entry.pack(side="left")
        self.timer_entry.insert(0, str(self.cfg.get("shutdown_minutes", 60)))
        lbl(trow, "dk", 12, muted=True).pack(side="left", padx=(6, 0))
        self.timer_btn = btn(trow, "Kur", self.timer_toggle, width=54, height=32, radius=10)
        self.timer_btn.pack(side="right")
        self.timer_lbl = lbl(self.timer_card, "Kapalı", 12, muted=True)
        self.timer_lbl.pack(anchor="w", padx=12, pady=(6, 10))
        self.timer_mini = ctk.CTkButton(side, text="⏻", width=56, height=52, corner_radius=16,
                                        fg_color="transparent", hover_color=TH["card2"], text_color=TH["muted"],
                                        font=font(20, True), command=self.toggle_side)
        self.apply_side()

        main = ctk.CTkFrame(self, fg_color="transparent", corner_radius=0)
        main.grid(row=0, column=1, sticky="nsew")
        main.grid_columnconfigure(0, weight=1)
        main.grid_rowconfigure(1, weight=1)
        self.build_header(main)

        self.content = ctk.CTkFrame(main, fg_color="transparent")
        self.content.grid(row=1, column=0, sticky="nsew", padx=24, pady=(4, 18))
        self.content.grid_columnconfigure(0, weight=1)
        self.content.grid_rowconfigure(0, weight=1)

        builders = {
            PAGE_RULES: self.build_rules_page, PAGE_PROF: self.build_prof_page,
            PAGE_CAT: self.build_cat_page, PAGE_TEXT: self.build_text_page,
            PAGE_SYS: self.build_sys_page, PAGE_PRIO: self.build_prio_page,
            PAGE_THEME: self.build_theme_page, PAGE_GEN: self.build_general_page,
        }
        for name in PAGE_ORDER:
            p = ctk.CTkFrame(self.content, fg_color="transparent")
            p.grid(row=0, column=0, sticky="nsew")
            self.pages[name] = p
            builders[name](p)
        self._prev_sig = None
        self.apply_status_visuals()
        self.refresh_timer_ui()
        self.building = False
        self.show_page(page)

    def build_header(self, parent):
        cv = tk.Canvas(parent, height=HDR_H, highlightthickness=0, bd=0, bg=TH["bg"])
        self.hdr = cv
        cv.grid(row=0, column=0, sticky="ew")
        self.hdr_bg = cv.create_image(0, 0, anchor="nw")
        self.hdr_title = cv.create_text(32, 20, anchor="nw", text="", fill=TH["text"],
                                        font=("Segoe UI Semibold", 26))
        self.hdr_sub = cv.create_text(34, 66, anchor="nw", text="", fill=mix(TH["text"], TH["bg"], 0.3),
                                      font=("Segoe UI", 11))
        # butonlar doğrudan canvas'a çizilir: arkadaki gradyanla köşelerde kare görünmez
        self.hb = {}
        self.hbtn("start", TH["ok"], TH["ok_hover"], "Başlat", self.toggle, "#ffffff", 13, True)
        self.hbtn("pill", TH["card"], TH["card"], "", None, mix(TH["text"], TH["bg"], 0.15), 11, False,
                  outline=TH["border"])
        self.hdr_dot = cv.create_text(0, 0, text="●", fill=TH["muted"], font=("Segoe UI", 14), anchor="w")
        self.hbtn("add", TH["accent"], TH["accent_hover"], "+  Kural ekle", self.add_rule, "#ffffff", 11, True)
        self.hbtn("prof", TH["card2"], TH["soft_hover"], "", self.open_prof_menu, TH["text"], 11, False)
        cv.itemconfig(self.hb["pill"]["tx"], anchor="w")
        cv.bind("<Configure>", self.on_hdr_configure)
        self.update_prof_menu()

    def hbtn(self, tag, fill, hover, text, cmd, fg, size, bold, outline=""):
        cv = self.hdr
        bg = cv.create_polygon([0] * 12, smooth=True, fill=fill, outline=outline, width=1 if outline else 0,
                               tags=(tag,))
        tx = cv.create_text(0, 0, text=text, fill=fg, anchor="center", tags=(tag,),
                            font=("Segoe UI Semibold" if bold else "Segoe UI", size))
        self.hb[tag] = {"bg": bg, "tx": tx, "fill": fill, "hover": hover, "cmd": cmd}
        if cmd:
            def enter(_e, t=tag):
                cv.itemconfig(self.hb[t]["bg"], fill=self.hb[t]["hover"])
                cv.config(cursor="hand2")

            def leave(_e, t=tag):
                cv.itemconfig(self.hb[t]["bg"], fill=self.hb[t]["fill"])
                cv.config(cursor="")

            cv.tag_bind(tag, "<Button-1>", lambda _e, t=tag: self.hb[t]["cmd"]())
            cv.tag_bind(tag, "<Enter>", enter)
            cv.tag_bind(tag, "<Leave>", leave)

    def place_h(self, tag, x1, y1, x2, y2):
        h = self.hb[tag]
        self.hdr.coords(h["bg"], *rr_pts(x1, y1, x2, y2, (y2 - y1) / 2))
        self.hdr.coords(h["tx"], (x1 + x2) / 2, (y1 + y2) / 2)

    def layout_header(self):
        w = self._hdr_w
        if w < 50:
            return
        add_on = getattr(self, "current_page", PAGE_RULES) == PAGE_RULES
        self.place_h("start", w - 144, 18, w - 24, 58)
        px1 = w - 144 - 12 - 300
        self.place_h("pill", px1, 18, w - 156, 58)
        self.hdr.coords(self.hdr_dot, px1 + 16, 38)
        self.hdr.coords(self.hb["pill"]["tx"], px1 + 38, 38)
        self.place_h("add", w - 164, 70, w - 24, 108)
        self.hdr.itemconfig(self.hb["add"]["bg"], state="normal" if add_on else "hidden")
        self.hdr.itemconfig(self.hb["add"]["tx"], state="normal" if add_on else "hidden")
        x2 = w - 24 - (152 if add_on else 0)
        self.place_h("prof", x2 - 190, 70, x2, 108)

    def on_hdr_configure(self, e):
        w = e.width
        if w < 50 or w == self._hdr_w:
            return
        self._hdr_w = w
        self.layout_header()
        if self._hdr_job:
            try:
                self.after_cancel(self._hdr_job)
            except Exception:
                pass
        self._hdr_job = self.after(60, self.redraw_header)

    def redraw_header(self):
        self._hdr_job = None
        try:
            photo = ImageTk.PhotoImage(make_banner(self._hdr_w, HDR_H))
            self._hdr_photo = photo
            self.hdr.itemconfig(self.hdr_bg, image=photo)
        except Exception:
            pass

    def open_prof_menu(self):
        m = tk.Menu(self, tearoff=0, bg=TH["card"], fg=TH["text"], activebackground=TH["accent"],
                    activeforeground="#ffffff", bd=0, relief="flat", font=("Segoe UI", 11))
        m.add_command(label="Varsayılan", command=lambda: self.on_profile_pick("Profil: Varsayılan"))
        for p in self.cfg["profiles"]:
            m.add_command(label=p["name"], command=lambda n=p["name"]: self.on_profile_pick("Profil: " + n))
        x1, _y1, _x2, y2 = self.hdr.bbox(self.hb["prof"]["bg"])
        try:
            m.tk_popup(self.hdr.winfo_rootx() + int(x1), self.hdr.winfo_rooty() + int(y2) + 4)
        finally:
            m.grab_release()

    def show_page(self, name):
        self.current_page = name
        for n, p in self.pages.items():
            if n == name:
                p.grid()
            else:
                p.grid_remove()
        for n, b in self.nav.items():
            if n == name:
                b.configure(fg_color=TH["accent"], text_color="#ffffff", hover_color=TH["accent_hover"])
            else:
                b.configure(fg_color="transparent", text_color=TH["muted"], hover_color=TH["card2"])
        try:
            title, sub = getattr(self.pages[name], "hdr_info", ("", ""))
            self.hdr.itemconfig(self.hdr_title, text=title)
            self.hdr.itemconfig(self.hdr_sub, text=sub)
            self.layout_header()
        except Exception:
            pass

    def page_header(self, parent, title, subtitle, right=None):
        parent.hdr_info = (title, subtitle)

    # ---- kurallar sayfası
    def build_rules_page(self, page):
        self.page_header(page, "Kurallar", "Hangi programda ekranda ne yazsın")
        bar = ctk.CTkFrame(page, fg_color="transparent")
        bar.pack(fill="x", pady=(0, 10))
        self.search = entry(bar, 220, "Kural ara...", radius=18, height=36)
        self.search.pack(side="left", padx=(0, 10))
        self.seg = option(bar, ["Tümü"] + list(CATEGORIES), 140, self.set_filter)
        self.seg.set(self.filter)
        self.seg.pack(side="left")
        self.view_seg = segmented(bar, VIEW_MODES, self.on_view)
        self.view_seg.set(self.cfg.get("view_mode") if self.cfg.get("view_mode") in VIEW_MODES else "Kartlar")
        self.view_seg.pack(side="right")
        self.search.bind("<KeyRelease>", lambda _e: self.refresh())
        self.list = scroll_frame(page)
        self.list.pack(fill="both", expand=True)
        for c in range(3):
            self.list.grid_columnconfigure(c, weight=1, uniform="tile")
        self.refresh()

    def set_filter(self, value):
        self.filter = value
        self.refresh()

    def on_view(self, value):
        self.cfg["view_mode"] = value
        save_config(self.cfg)
        self.refresh()

    def refresh(self):
        for w in self.list.winfo_children():
            w.destroy()
        self.keep.clear()
        q = self.search.get().strip().lower()
        view = self.cfg.get("view_mode", "Kartlar")
        items = []
        for idx, rule in enumerate(self.cfg["rules"]):
            if self.filter != "Tümü" and rule.get("category", DEFAULT_CATEGORY) != self.filter:
                continue
            if q and q not in (rule["exe"] + " " + rule.get("text", "")).lower():
                continue
            items.append((idx, rule))
        free = self.filter == "Tümü" and not q
        if view == "Liste":
            for idx, rule in items:
                self.make_row(idx, rule)
            if free:
                btn(self.list, "+  Yeni kural ekle", self.add_rule, kind="soft", width=180, height=38,
                    radius=19).pack(anchor="w", padx=6, pady=12)
        elif view == "Gruplu":
            for cat, color in CATEGORIES.items():
                group = [(i, r) for i, r in items if r.get("category", DEFAULT_CATEGORY) == cat]
                if not group:
                    continue
                head = ctk.CTkFrame(self.list, fg_color="transparent")
                head.pack(fill="x", padx=6, pady=(14, 2))
                ctk.CTkLabel(head, text="●", text_color=color, font=font(16), width=18).pack(side="left")
                lbl(head, cat, 16, True).pack(side="left", padx=(4, 8))
                lbl(head, f"{len(group)} kural", 12, muted=True).pack(side="left")
                sub = ctk.CTkFrame(self.list, fg_color="transparent")
                sub.pack(fill="x")
                for c in range(3):
                    sub.grid_columnconfigure(c, weight=1, uniform="g" + cat)
                for pos, (idx, rule) in enumerate(group):
                    self.make_tile(sub, pos, idx, rule)
        else:
            for pos, (idx, rule) in enumerate(items):
                self.make_tile(self.list, pos, idx, rule)
            if free:
                self.make_add_tile(len(items))
        if not items and not free:
            lbl(self.list, "Sonuç bulunamadı.", 14, muted=True).pack(pady=60)

    def rule_sub(self, rule):
        sub = rule["exe"] + "  ·  " + MODE_SHORT.get(rule.get("mode", "text"), "sabit metin")
        if rule.get("filter"):
            sub += f"  ·  başlıkta: {rule['filter']}"
        if rule.get("image"):
            sub += "  ·  görsel: " + rule["image"].split("_", 1)[-1]
        return sub

    def toggle_rule(self, idx):
        r = self.cfg["rules"][idx]
        r["enabled"] = not r.get("enabled", True)
        save_config(self.cfg)
        self.refresh()

    def dup_rule(self, idx):
        r = copy.deepcopy(self.cfg["rules"][idx])
        self.cfg["rules"].insert(idx + 1, r)
        save_config(self.cfg)
        self.refresh()

    def make_tile(self, parent, pos, idx, rule):
        cat = rule.get("category", DEFAULT_CATEGORY)
        color = CATEGORIES.get(cat, CATEGORIES[DEFAULT_CATEGORY])
        en = rule.get("enabled", True)
        tile = card(parent)
        tile.grid(row=pos // 3, column=pos % 3, padx=7, pady=7, sticky="nsew")

        topc = mix(TH["card"], color, 0.4 if en else 0.12)
        top = ctk.CTkFrame(tile, height=92, corner_radius=14, fg_color=topc)
        top.pack(fill="x", padx=8, pady=(8, 0))
        top.pack_propagate(False)
        make_icon(top, rule["exe"], rule.get("path", ""), color, self.keep, size=56, bg=topc).pack(expand=True)
        ctk.CTkLabel(top, text=cat if en else "KAPALI", fg_color=color if en else TH["card2"], corner_radius=9,
                     text_color="white" if en else TH["muted"], width=64, height=20,
                     font=font(11, True)).place(relx=1.0, x=-8, y=8, anchor="ne")

        title = rule.get("text") or os.path.splitext(rule["exe"])[0]
        lbl(tile, title, 15, True, muted=not en, anchor="w").pack(anchor="w", padx=14, pady=(10, 0))
        lbl(tile, self.rule_sub(rule), 11, muted=True, anchor="w", justify="left", wraplength=210).pack(
            anchor="w", padx=14, pady=(0, 6))

        row = ctk.CTkFrame(tile, fg_color="transparent")
        row.pack(fill="x", padx=8, pady=(0, 12))
        sw = switch(row, "", lambda i=idx: self.toggle_rule(i))
        sw.pack(side="right", padx=(0, 4))
        if en:
            sw.select()
        btn(row, "Düzenle", lambda i=idx: self.edit_rule(i), kind="soft", width=74, height=28).pack(
            side="left", padx=(4, 3))
        btn(row, "Kopya", lambda i=idx: self.dup_rule(i), kind="soft", width=58, height=28).pack(
            side="left", padx=3)
        btn(row, "Sil", lambda i=idx: self.del_rule(i), kind="danger", width=44, height=28).pack(
            side="left", padx=3)

    def make_row(self, idx, rule):
        cat = rule.get("category", DEFAULT_CATEGORY)
        color = CATEGORIES.get(cat, CATEGORIES[DEFAULT_CATEGORY])
        en = rule.get("enabled", True)
        row = card(self.list)
        row.pack(fill="x", padx=4, pady=4)
        inner = ctk.CTkFrame(row, fg_color="transparent")
        inner.pack(fill="x", padx=12, pady=8)
        make_icon(inner, rule["exe"], rule.get("path", ""), color, self.keep, size=44, bg=TH["card"]).pack(
            side="left", padx=(0, 12))
        sw = switch(inner, "", lambda i=idx: self.toggle_rule(i))
        sw.pack(side="right")
        if en:
            sw.select()
        btn(inner, "Sil", lambda i=idx: self.del_rule(i), kind="danger", width=48, height=30).pack(
            side="right", padx=4)
        btn(inner, "Kopya", lambda i=idx: self.dup_rule(i), kind="soft", width=62, height=30).pack(
            side="right", padx=4)
        btn(inner, "Düzenle", lambda i=idx: self.edit_rule(i), kind="soft", width=80, height=30).pack(
            side="right", padx=4)
        ctk.CTkLabel(inner, text=cat, fg_color=color, corner_radius=9, text_color="white", width=70, height=22,
                     font=font(11, True)).pack(side="right", padx=10)
        info = ctk.CTkFrame(inner, fg_color="transparent")
        info.pack(side="left", fill="x", expand=True)
        title = rule.get("text") or os.path.splitext(rule["exe"])[0]
        lbl(info, title, 14, True, muted=not en, anchor="w").pack(anchor="w")
        lbl(info, self.rule_sub(rule), 11, muted=True, anchor="w").pack(anchor="w")

    def make_add_tile(self, pos):
        t = ctk.CTkFrame(self.list, corner_radius=20, fg_color="transparent", border_width=2,
                         border_color=TH["card2"], height=200)
        t.grid(row=pos // 3, column=pos % 3, padx=7, pady=7, sticky="nsew")
        plus = ctk.CTkLabel(t, text="+", font=font(44, True), text_color=TH["accent"])
        plus.place(relx=0.5, rely=0.4, anchor="center")
        txt = lbl(t, "Yeni kural ekle", 13, True, muted=True)
        txt.place(relx=0.5, rely=0.72, anchor="center")
        for w in (t, plus, txt):
            w.bind("<Button-1>", lambda _e: self.add_rule())

    def add_rule(self):
        d = RuleDialog(self)
        self.wait_window(d)
        if d.result:
            self.cfg["rules"].append(d.result)
            save_config(self.cfg)
            self.refresh()

    def edit_rule(self, idx):
        d = RuleDialog(self, self.cfg["rules"][idx])
        self.wait_window(d)
        if d.result:
            self.cfg["rules"][idx] = d.result
            save_config(self.cfg)
            self.refresh()

    def del_rule(self, idx):
        r = self.cfg["rules"][idx]
        if messagebox.askyesno("Sil", f"{r['exe']} kuralı silinsin mi?", parent=self):
            del self.cfg["rules"][idx]
            save_config(self.cfg)
            self.refresh()

    # ---- yazı ve kaydırma sayfası
    def build_text_page(self, page):
        self.page_header(page, "Yazı ve kaydırma", "Ekranda yazı nasıl görünsün, nasıl aksın")
        body = scroll_frame(page)
        body.pack(fill="both", expand=True)
        c = self.cfg

        panel = ctk.CTkFrame(body, corner_radius=18, fg_color=TH["card"], border_width=2, border_color=TH["accent"])
        panel.pack(anchor="w", padx=4, pady=(0, 6))
        lbl(panel, "Canlı önizleme (gerçek ekranın yaklaşık görünümü)", 12, muted=True).pack(anchor="w", padx=16, pady=(12, 6))
        self.preview_lbl = ctk.CTkLabel(panel, text="", fg_color="#080a0e", corner_radius=8,
                                        width=128 * PREVIEW_SCALE, height=40 * PREVIEW_SCALE)
        self.preview_lbl.pack(padx=16)
        self.preview_entry = entry(panel, 384, "Önizleme metni")
        self.preview_entry.pack(padx=16, pady=(10, 14))
        self.preview_entry.insert(0, c["preview_text"])
        self.preview_entry.bind("<KeyRelease>", lambda _e: self.save_opts())

        def row(text):
            f = ctk.CTkFrame(body, fg_color="transparent")
            f.pack(fill="x", padx=4, pady=6)
            lbl(f, text, 13, width=180, anchor="w").pack(side="left")
            return f

        f = row("Yazı boyutu")
        self.seg_size = segmented(f, list(SIZES), self.save_opts)
        self.seg_size.set(c["size"])
        self.seg_size.pack(side="left")

        f = row("Yazı kalınlığı")
        self.seg_weight = segmented(f, list(WEIGHTS), self.save_opts)
        self.seg_weight.set(c["weight"])
        self.seg_weight.pack(side="left")

        f = row("Uzun yazıyı kaydır")
        self.sw_scroll = switch(f, "", self.save_opts)
        self.sw_scroll.pack(side="left")
        if c["scroll"]:
            self.sw_scroll.select()

        f = row("Kaydırma şekli")
        self.opt_style = option(f, list(STYLES), 200, self.save_opts)
        self.opt_style.set(c["style"])
        self.opt_style.pack(side="left")

        self.add_slider(body, "Kaydırma hızı", "speed", 10, 120, 110, "{:.0f} piksel/sn", int)
        self.add_slider(body, "Duraklama süresi", "pause", 0, 4, 40, "{:.1f} sn", float)
        self.add_slider(body, "Kare sayısı (akıcılık)", "fps", 5, 20, 15, "{:.0f} kare/sn", int)
        lbl(body, "Kare sayısını artırmak akıcılığı artırır ama bazı bilgisayarlarda SteelSeries GG'yi yorabilir. "
                  "Takılma olursa düşür.", 12, muted=True, wraplength=620, justify="left").pack(anchor="w", padx=4, pady=(4, 0))

    def add_slider(self, parent, text, key, lo, hi, steps, fmt, cast):
        f = ctk.CTkFrame(parent, fg_color="transparent")
        f.pack(fill="x", padx=4, pady=6)
        lbl(f, text, 13, width=180, anchor="w").pack(side="left")
        value_lbl = lbl(f, fmt.format(self.cfg[key]), 13, True, width=120, anchor="w")

        def on_change(v):
            val = cast(round(v, 1)) if cast is float else cast(round(v))
            self.cfg[key] = val
            value_lbl.configure(text=fmt.format(val))
            save_config(self.cfg)

        s = slider(f, lo, hi, steps, on_change)
        s.set(self.cfg[key])
        s.pack(side="left", padx=(0, 10))
        value_lbl.pack(side="left")

    # ---- öncelik sayfası
    def build_prio_page(self, page):
        self.page_header(page, "Öncelik", "Aynı anda birden fazla şey ekran istediğinde hangisi kazansın")
        body = scroll_frame(page)
        body.pack(fill="both", expand=True)
        c = self.cfg

        live = card(body)
        live.pack(fill="x", padx=4, pady=(0, 10))
        lbl(live, "Şu an klavyede", 12, muted=True).pack(anchor="w", padx=16, pady=(12, 0))
        self.prio_live = lbl(live, "—", 20, True)
        self.prio_live.pack(anchor="w", padx=16)
        self.prio_live2 = lbl(live, "", 12, muted=True, wraplength=640, justify="left")
        self.prio_live2.pack(anchor="w", padx=16, pady=(2, 12))

        oc = card(body)
        oc.pack(fill="x", padx=4, pady=8)
        lbl(oc, "Karar sırası", 15, True).pack(anchor="w", padx=16, pady=(14, 4))
        lbl(oc, "1.  Ses göstergesi (ses değişince)\n2.  Sıcaklık uyarısı (sınır aşılınca)\n"
                "3.  Yeni şarkı bildirimi (şarkı değişince kısa süre)\n"
                "4.  Aşağıda seçtiğin sıralama modu (kural / medya)\n"
                "5.  Sistem bilgisi (aralıklı ise burada ya da en üstte)\n6.  Boşta içerik (yazı, saat, sistem bilgisi)",
            12, muted=True, justify="left").pack(anchor="w", padx=16, pady=(0, 14))

        self.prio_var = tk.StringVar(value=c["priority"])
        for key, title, desc in PRIORITIES:
            box = card(body)
            box.pack(fill="x", padx=4, pady=5)
            ctk.CTkRadioButton(box, text=title, variable=self.prio_var, value=key, command=self.on_priority,
                               fg_color=TH["accent"], hover_color=TH["accent_hover"],
                               border_color=TH["muted"], text_color=TH["text"],
                               font=font(14, True)).pack(anchor="w", padx=16, pady=(14, 2))
            lbl(box, desc, 12, muted=True, wraplength=620, justify="left").pack(anchor="w", padx=44, pady=(0, 12))
            if key == "rotate":
                self.add_slider(box, "Kural kaç saniye", "rule_secs", 3, 30, 27, "{:.0f} sn", int)
                self.add_slider(box, "Medya kaç saniye", "media_secs", 3, 30, 27, "{:.0f} sn", int)
            if key == "category":
                lbl(box, "Bu kategorilerdeki kurallar önde olsun:", 13, True).pack(anchor="w", padx=44, pady=(0, 4))
                grid = ctk.CTkFrame(box, fg_color="transparent")
                grid.pack(anchor="w", padx=40, pady=(0, 12))
                self.prio_cat_vars = {}
                for i, name in enumerate(CATEGORIES):
                    v = tk.BooleanVar(value=name in c.get("prio_cats", []))
                    self.prio_cat_vars[name] = v
                    ctk.CTkCheckBox(grid, text=name, variable=v, width=110, command=self.save_prio_cats,
                                    fg_color=CATEGORIES[name], hover_color=CATEGORIES[name],
                                    border_color=TH["muted"], text_color=TH["text"]).grid(
                        row=i // 4, column=i % 4, padx=6, pady=4, sticky="w")

        ec = card(body)
        ec.pack(fill="x", padx=4, pady=8)
        lbl(ec, "Ek davranışlar", 15, True).pack(anchor="w", padx=16, pady=(14, 0))
        self.sw_bg_media = self.make_switch(ec, "Arka plandaki medyayı da göster", None, self.save_opts,
                                            "Başka programa geçsen bile tarayıcıda, Spotify'da ya da VLC'de çalan "
                                            "şarkı/video bilgisi okunur.")
        if c["bg_media"]:
            self.sw_bg_media.select()
        self.sw_media_pop = self.make_switch(ec, "Şarkı değişince adını kısa süre öne çıkar", "media_pop",
                                             self.save_prio,
                                             "Oyundayken bile yeni şarkının adı birkaç saniye görünür, sonra önceki "
                                             "yazı geri gelir.")
        self.add_slider(ec, "Öne çıkma süresi", "media_pop_secs", 2, 15, 13, "{:.0f} sn", int)
        self.sw_sys_over = self.make_switch(ec, "Aralıklı sistem bilgisi diğerlerinin üstünde çıksın", "sys_over",
                                            self.save_prio,
                                            "Kapatırsan sistem bilgisi sadece kural ve medya yokken görünür.")
        ctk.CTkFrame(ec, height=12, fg_color="transparent").pack()
        self.update_prio_live()

    def on_priority(self):
        self.cfg["priority"] = self.prio_var.get()
        save_config(self.cfg)

    def save_prio_cats(self):
        self.cfg["prio_cats"] = [n for n, v in self.prio_cat_vars.items() if v.get()]
        save_config(self.cfg)

    def save_prio(self):
        try:
            self.cfg["media_pop"] = bool(self.sw_media_pop.get())
            self.cfg["sys_over"] = bool(self.sw_sys_over.get())
            save_config(self.cfg)
        except Exception:
            pass

    def update_prio_live(self):
        try:
            w = self.worker
            if w and w.is_alive():
                fg = w.last_fg[0] if w.last_fg else "—"
                self.prio_live.configure(text=w.source or "—", text_color=TH["accent2"])
                self.prio_live2.configure(text=f"Öndeki program: {fg}  ·  Etkin profil: {w.cur_profile or 'Varsayılan'}")
            else:
                self.prio_live.configure(text="Program çalışmıyor", text_color=TH["muted"])
                self.prio_live2.configure(text="Sağ üstteki Başlat düğmesine basınca burada o an neyin gösterildiği ve nedeni görünür.")
        except Exception:
            pass

    # ---- tema sayfası
    def build_theme_page(self, page):
        self.page_header(page, "Tema", "Uygulamanın renklerini seç, hemen değişir")
        grid = ctk.CTkFrame(page, fg_color="transparent")
        grid.pack(fill="both", expand=True)
        for i, (name, t) in enumerate(THEMES.items()):
            sel = name == self.cfg["theme"]
            c = ctk.CTkFrame(grid, width=300, height=104, corner_radius=16, fg_color=t["card"],
                             border_width=3 if sel else 1, border_color=t["accent"] if sel else t["card2"])
            c.grid(row=i // 2, column=i % 2, padx=8, pady=8, sticky="w")
            c.grid_propagate(False)
            ctk.CTkLabel(c, text=name + ("  ✓" if sel else ""), text_color=t["text"],
                         font=font(15, True)).place(x=16, y=14)
            for j, key in enumerate(("bg", "accent", "accent2", "card2")):
                ctk.CTkFrame(c, width=30, height=30, corner_radius=15, fg_color=t[key],
                             border_width=1, border_color=t["muted"]).place(x=16 + j * 40, y=56)
            for w in [c] + list(c.winfo_children()):
                w.bind("<Button-1>", lambda _e, n=name: self.set_theme(n))

    def set_theme(self, name):
        if name == self.cfg["theme"]:
            return
        self.save_opts()
        self.cfg["theme"] = name
        save_config(self.cfg)
        set_theme_colors(name)
        ctk.set_appearance_mode(TH["mode"])
        self.build_ui(PAGE_THEME)

    # ---- genel sayfa
    def make_switch(self, parent, text, key, command=None, note=None):
        f = ctk.CTkFrame(parent, fg_color="transparent")
        f.pack(fill="x", padx=16, pady=(12, 0))
        s = switch(f, text, command or self.save_opts)
        s.pack(anchor="w")
        if key and self.cfg.get(key):
            s.select()
        if note:
            lbl(f, note, 12, muted=True, wraplength=620, justify="left").pack(anchor="w", padx=44, pady=(2, 0))
        return s

    def build_general_page(self, page):
        self.page_header(page, "Genel", "Başlatma, ekran ve medya algılama ayarları")
        body = scroll_frame(page)
        body.pack(fill="both", expand=True)

        start = card(body)
        start.pack(fill="x", padx=4, pady=(0, 10))
        lbl(start, "Başlatma", 15, True).pack(anchor="w", padx=16, pady=(14, 0))
        self.sw_startup = self.make_switch(start, "Windows açılınca bu programı da aç", None, self.toggle_startup,
                                           "Programı taşırsan ya da yeniden derlersen bu kutuyu bir kez kapatıp aç.")
        if is_startup_enabled():
            self.sw_startup.select()
        self.sw_autostart = self.make_switch(start, "Program açılınca otomatik başlat", "auto_start", None,
                                             "Başlat'a basman gerekmez, program açılır açılmaz ekrana göndermeye başlar.")
        self.sw_minimized = self.make_switch(start, "Windows ile açıldığında küçültülmüş başlat", "start_minimized",
                                             None, "Pencere görev çubuğunda (ya da tepsi açıksa tepside) küçük kalır.")
        self.sw_tray = self.make_switch(start, "Pencereyi kapatınca tepsiye küçült (arka planda çalışsın)",
                                        "close_to_tray", None,
                                        "Saat yanındaki tepsi simgesinden açabilir ya da Çıkış diyebilirsin." +
                                        ("" if pystray else "  (Şu an pystray kurulu değil: python -m pip install pystray)"))
        ctk.CTkFrame(start, height=12, fg_color="transparent").pack()

        scr = card(body)
        scr.pack(fill="x", padx=4, pady=10)
        lbl(scr, "Ekran", 15, True).pack(anchor="w", padx=16, pady=(14, 0))
        f = ctk.CTkFrame(scr, fg_color="transparent")
        f.pack(fill="x", padx=16, pady=(12, 0))
        lbl(f, "Hiçbir şey yokken yazı", 13, width=180, anchor="w").pack(side="left")
        self.default_entry = entry(f, 280, "Boş bırakırsan ekran normale döner")
        self.default_entry.pack(side="left")
        if self.cfg["default_text"]:
            self.default_entry.insert(0, self.cfg["default_text"])
        self.default_entry.bind("<KeyRelease>", lambda _e: self.save_opts())
        self.sw_invert = self.make_switch(scr, "Görselleri ters çevir", "invert", None,
                                          "Kendi eklediğin görseller ters (siyah-beyaz yer değiştirmiş) görünürse aç.")
        f2 = ctk.CTkFrame(scr, fg_color="transparent")
        f2.pack(fill="x", padx=16, pady=(12, 0))
        lbl(f2, "Hiçbir şey yokken", 13, width=180, anchor="w").pack(side="left")
        self.opt_idle = option(f2, list(IDLE_MODES), 220, self.save_opts)
        self.opt_idle.set(next((k for k, v in IDLE_MODES.items() if v == self.cfg.get("idle_mode", "none")),
                               "Hiçbir şey"))
        self.opt_idle.pack(side="left")
        lbl(scr, "Yukarıdaki yazı doluysa o gösterilir; boşsa buradaki seçim (saat ya da sistem bilgisi) kullanılır.",
            12, muted=True, wraplength=620, justify="left").pack(anchor="w", padx=16, pady=(2, 0))
        self.sw_native = self.make_switch(scr, "Uyumluluk modu (SteelSeries'ın kendi yazı tipi)", "native_text", None,
                                          "Yazı ekranda hiç görünmüyorsa dene. Bu modda kayan yazı, boyut ve kalınlık çalışmaz.")
        ctk.CTkFrame(scr, height=12, fg_color="transparent").pack()

        med = card(body)
        med.pack(fill="x", padx=4, pady=10)
        lbl(med, "Medya algılama", 15, True).pack(anchor="w", padx=16, pady=(14, 0))
        self.media_lbl = lbl(med, "...", 13, muted=True, wraplength=640, justify="left")
        self.media_lbl.pack(anchor="w", padx=16, pady=(6, 14))

    # ---- ayarlar
    def save_opts(self, *_):
        try:
            c = self.cfg
            c["size"] = self.seg_size.get()
            c["weight"] = self.seg_weight.get()
            c["scroll"] = bool(self.sw_scroll.get())
            c["style"] = self.opt_style.get()
            c["preview_text"] = self.preview_entry.get()
            c["default_text"] = self.default_entry.get()
            c["invert"] = bool(self.sw_invert.get())
            c["native_text"] = bool(self.sw_native.get())
            c["idle_mode"] = IDLE_MODES.get(self.opt_idle.get(), "none")
            c["close_to_tray"] = bool(self.sw_tray.get())
            c["bg_media"] = bool(self.sw_bg_media.get())
            c["auto_start"] = bool(self.sw_autostart.get())
            c["start_minimized"] = bool(self.sw_minimized.get())
            save_config(c)
        except Exception:
            pass

    def toggle_startup(self):
        try:
            set_startup(bool(self.sw_startup.get()))
        except Exception as e:
            messagebox.showerror("Hata", f"Başlangıç ayarı değiştirilemedi: {e}", parent=self)

    # ---- profil menüsü (üst bar)
    def prof_values(self):
        return ["Profil: Varsayılan"] + ["Profil: " + p["name"] for p in self.cfg["profiles"]]

    def update_prof_menu(self):
        try:
            names = [p["name"] for p in self.cfg["profiles"]]
            act = self.cfg.get("active_profile", "")
            if act not in names:
                act = ""
                self.cfg["active_profile"] = ""
            label = act or "Varsayılan"
            if len(label) > 16:
                label = label[:15] + "…"
            self.hdr.itemconfig(self.hb["prof"]["tx"], text=f"Profil: {label}  ▾")
        except Exception:
            pass

    def on_profile_pick(self, v):
        name = v.replace("Profil: ", "", 1)
        self.cfg["active_profile"] = "" if name == "Varsayılan" else name
        save_config(self.cfg)
        self.refresh_profiles()

    # ---- profiller sayfası
    def build_prof_page(self, page):
        self.page_header(page, "Profiller", "Oyuna ya da kullanıma göre ayar setleri; istersen kendiliğinden geçsin")
        top = ctk.CTkFrame(page, fg_color="transparent")
        top.pack(fill="x", pady=(0, 6))
        btn(top, "+ Yeni profil", self.add_profile, width=130, height=36, radius=18).pack(side="right")
        lbl(top, "Profil, genel ayarların üstüne seçtiğin ayarları değiştirir. Program ya da kategori tanımlarsan o "
                 "öndeyken otomatik devreye girer; sağ üstteki menüden de elle seçebilirsin.", 12, muted=True,
            wraplength=640, justify="left").pack(side="left")
        self.prof_now = lbl(page, "", 12, True)
        self.prof_now.pack(anchor="w", pady=(2, 6))
        self.prof_list = scroll_frame(page)
        self.prof_list.pack(fill="both", expand=True)
        self.refresh_profiles()

    def refresh_profiles(self):
        try:
            for w in self.prof_list.winfo_children():
                w.destroy()
            active = self.cfg.get("active_profile", "")
            self.make_profile_card(None, active == "")
            for i, p in enumerate(self.cfg["profiles"]):
                self.make_profile_card(i, active == p["name"])
            self.update_prof_menu()
            self.update_prof_now()
        except Exception:
            pass

    def update_prof_now(self):
        try:
            if self.worker and self.worker.is_alive():
                name = self.worker.cur_profile or "Varsayılan"
                self.prof_now.configure(text=f"Şu an etkin profil: {name}", text_color=TH["accent2"])
            else:
                self.prof_now.configure(text="Program çalışmıyor (Başlat'a basınca profiller devreye girer)",
                                        text_color=TH["muted"])
        except Exception:
            pass

    def make_profile_card(self, idx, is_active):
        p = None if idx is None else self.cfg["profiles"][idx]
        c = card(self.prof_list)
        c.pack(fill="x", padx=4, pady=5)
        if is_active:
            c.configure(border_color=TH["accent"], border_width=2)
        top = ctk.CTkFrame(c, fg_color="transparent")
        top.pack(fill="x", padx=16, pady=(12, 2))
        lbl(top, "Varsayılan" if p is None else p["name"], 16, True).pack(side="left")
        if is_active:
            ctk.CTkLabel(top, text="ETKİN", fg_color=TH["accent"], corner_radius=9, text_color="#ffffff",
                         width=56, height=20, font=font(11, True)).pack(side="left", padx=10)
        if p is None:
            sub = "Genel ayarlar: Yazı ve kaydırma, Öncelik ve Genel sekmelerindekiler."
        else:
            parts = []
            if p.get("exes"):
                parts.append("Programlar: " + p["exes"])
            if p.get("cats"):
                parts.append("Kategoriler: " + ", ".join(p["cats"]))
            if not parts:
                parts.append("Sadece elle seçilince çalışır")
            parts.append(f"{len(p['settings'])} ayar değişiyor")
            sub = "  ·  ".join(parts)
        lbl(c, sub, 12, muted=True, anchor="w", justify="left", wraplength=600).pack(anchor="w", padx=16)
        row = ctk.CTkFrame(c, fg_color="transparent")
        row.pack(fill="x", padx=12, pady=(8, 12))
        if not is_active:
            btn(row, "Etkinleştir", lambda i=idx: self.activate_profile(i), width=100, height=30,
                radius=15).pack(side="left", padx=4)
        if p is not None:
            btn(row, "Düzenle", lambda i=idx: self.edit_profile(i), kind="soft", width=84, height=30,
                radius=15).pack(side="left", padx=4)
            btn(row, "Kopyala", lambda i=idx: self.dup_profile(i), kind="soft", width=84, height=30,
                radius=15).pack(side="left", padx=4)
            btn(row, "Sil", lambda i=idx: self.del_profile(i), kind="danger", width=56, height=30,
                radius=15).pack(side="left", padx=4)

    def activate_profile(self, idx):
        self.cfg["active_profile"] = "" if idx is None else self.cfg["profiles"][idx]["name"]
        save_config(self.cfg)
        self.refresh_profiles()

    def unique_profile_name(self, name, skip=None):
        names = [p["name"] for i, p in enumerate(self.cfg["profiles"]) if i != skip] + ["Varsayılan"]
        base, n = name, 2
        while name in names:
            name = f"{base} {n}"
            n += 1
        return name

    def add_profile(self):
        d = ProfileDialog(self)
        self.wait_window(d)
        if d.result:
            d.result["name"] = self.unique_profile_name(d.result["name"])
            self.cfg["profiles"].append(d.result)
            save_config(self.cfg)
            self.refresh_profiles()

    def edit_profile(self, idx):
        old = self.cfg["profiles"][idx]
        d = ProfileDialog(self, old)
        self.wait_window(d)
        if d.result:
            d.result["name"] = self.unique_profile_name(d.result["name"], skip=idx)
            if self.cfg.get("active_profile") == old["name"]:
                self.cfg["active_profile"] = d.result["name"]
            self.cfg["profiles"][idx] = d.result
            save_config(self.cfg)
            self.refresh_profiles()

    def dup_profile(self, idx):
        p = copy.deepcopy(self.cfg["profiles"][idx])
        p["name"] = self.unique_profile_name(p["name"] + " (kopya)")
        self.cfg["profiles"].insert(idx + 1, p)
        save_config(self.cfg)
        self.refresh_profiles()

    def del_profile(self, idx):
        p = self.cfg["profiles"][idx]
        if messagebox.askyesno("Sil", f"'{p['name']}' profili silinsin mi?", parent=self):
            if self.cfg.get("active_profile") == p["name"]:
                self.cfg["active_profile"] = ""
            del self.cfg["profiles"][idx]
            save_config(self.cfg)
            self.refresh_profiles()

    # ---- kategoriler sayfası
    def build_cat_page(self, page):
        self.page_header(page, "Kategoriler", "Her kategori için ayrı hız, boyut ve stil")
        if self.cat_sel not in CATEGORIES:
            self.cat_sel = "Oyun"
        self.cat_seg = segmented(page, list(CATEGORIES), self.on_cat_pick)
        self.cat_seg.set(self.cat_sel)
        self.cat_seg.pack(fill="x", pady=(0, 8))
        self.cat_body = scroll_frame(page)
        self.cat_body.pack(fill="both", expand=True)
        self.render_cat()

    def on_cat_pick(self, v):
        self.cat_sel = v
        self.render_cat()

    def cat_preview(self, cat):
        cfg = self.cfg
        m = dict(cfg)
        m.update(cfg.get("cat_settings", {}).get(cat, {}))
        return (decorate_text(cfg, "Bu uzun bir örnek yazıdır, kaydırmayı dene", cat), text_settings(m))

    def render_cat(self):
        for w in self.cat_body.winfo_children():
            w.destroy()
        cat = self.cat_sel
        data = self.cfg["cat_settings"].setdefault(cat, {})
        pv = card(self.cat_body)
        pv.pack(fill="x", padx=4, pady=(0, 10))
        lbl(pv, f"{cat} kategorisi · canlı önizleme", 12, muted=True).pack(anchor="w", padx=16, pady=(12, 6))
        OledPreview(pv, lambda c=cat: self.cat_preview(c)).pack(anchor="w", padx=16, pady=(0, 10))
        note = CAT_NOTES.get(cat, "Kuralı bu kategoride olan programlar bu kategorinin ayarlarını kullanır.")
        lbl(pv, note, 12, muted=True, wraplength=620, justify="left").pack(anchor="w", padx=16, pady=(0, 14))
        fc = card(self.cat_body)
        fc.pack(fill="x", padx=4, pady=8)
        lbl(fc, "Bu kategoride değişecek ayarlar", 15, True).pack(anchor="w", padx=16, pady=(14, 0))
        lbl(fc, "Kutucuğu işaretlediğin ayar sadece bu kategorideki yazılar için geçerli olur; işaretsizler "
                "genel ayardan gelir.", 12, muted=True, wraplength=620, justify="left").pack(
            anchor="w", padx=16, pady=(2, 8))
        build_fields(fc, CAT_FIELDS, data, self.cfg, self.save_cat)
        btn(fc, "Bu kategoriyi sıfırla", self.reset_cat, kind="soft", width=170, height=32, radius=16).pack(
            anchor="w", padx=16, pady=(8, 14))

    def save_cat(self):
        save_config(self.cfg)

    def reset_cat(self):
        self.cfg["cat_settings"][self.cat_sel] = {}
        save_config(self.cfg)
        self.render_cat()

    # ---- sistem sayfası
    def sys_preview(self):
        cfg = self.cfg
        t = sys_compose(cfg, self.stats.data, time.time()) or "Gösterilecek veri yok"
        m = dict(cfg)
        m.update(cfg.get("cat_settings", {}).get("Sistem", {}))
        return (decorate_text(cfg, t, "Sistem"), text_settings(m))

    def vol_preview(self):
        v = self.vol
        lv = v.level if v.level is not None else 40
        c = self.cfg
        return render_volume(lv, v.muted, c["vol_style"], c["vol_text"], c["vol_mute"])

    def build_sys_page(self, page):
        self.page_header(page, "Sistem", "CPU, GPU, RAM, sıcaklık bilgileri ve ses göstergesi")
        body = scroll_frame(page)
        body.pack(fill="both", expand=True)
        c = self.cfg

        pv = card(body)
        pv.pack(fill="x", padx=4, pady=(0, 10))
        lbl(pv, "Canlı önizleme (seçili bilgilerle)", 12, muted=True).pack(anchor="w", padx=16, pady=(12, 6))
        OledPreview(pv, self.sys_preview).pack(anchor="w", padx=16, pady=(0, 8))
        lbl(pv, "Bu önizleme sadece burada görünür. Klavyede görmek için aşağıdaki 'Ne zaman gösterilsin?' "
                "ayarına bak.", 12, muted=True, wraplength=640, justify="left").pack(anchor="w", padx=16,
                                                                                   pady=(0, 14))

        wc = card(body)
        wc.pack(fill="x", padx=4, pady=8)
        lbl(wc, "Ne zaman gösterilsin?", 15, True).pack(anchor="w", padx=16, pady=(14, 6))
        fw = ctk.CTkFrame(wc, fg_color="transparent")
        fw.pack(fill="x", padx=16, pady=4)
        lbl(fw, "Gösterim zamanı", 13, width=180, anchor="w").pack(side="left")
        self.opt_sys_when = option(fw, list(SYS_WHEN), 230, lambda _v: self.save_sys_when())
        self.opt_sys_when.set(next((k for k, v in SYS_WHEN.items() if v == c["sys_when"]), "Hiçbir şey yokken"))
        self.opt_sys_when.pack(side="left")
        self.when_desc = lbl(wc, SYS_WHEN_DESC.get(c["sys_when"], ""), 12, muted=True, wraplength=640,
                             justify="left")
        self.when_desc.pack(anchor="w", padx=16, pady=(4, 6))
        fe = ctk.CTkFrame(wc, fg_color="transparent")
        fe.pack(fill="x", padx=16, pady=4)
        lbl(fe, "Aralıklı: sıklık", 13, width=180, anchor="w").pack(side="left")
        self.opt_sys_every = option(fe, list(SYS_EVERY), 230, lambda _v: self.save_sys_when())
        self.opt_sys_every.set(next((k for k, v in SYS_EVERY.items() if v == c["sys_every"]), "5 dakikada bir"))
        self.opt_sys_every.pack(side="left")
        self.add_slider(wc, "Aralıklı: ekranda kalma", "sys_show_secs", 3, 60, 57, "{:.0f} sn", int)
        lbl(wc, "Dönüşümlü gösterimde süre, açık olan bilgilerin hepsi görünene kadar kendiliğinden uzar.", 12,
            muted=True, wraplength=640, justify="left").pack(anchor="w", padx=16, pady=(2, 8))
        btn(wc, "Şimdi klavyede göster (10 sn)", self.test_sys, width=230, height=34, radius=17).pack(
            anchor="w", padx=16, pady=(4, 14))

        mc = card(body)
        mc.pack(fill="x", padx=4, pady=8)
        lbl(mc, "Gösterilecek bilgiler", 15, True).pack(anchor="w", padx=16, pady=(14, 0))
        lbl(mc, "Anahtarı aç, yazı şablonunu istediğin gibi düzenle ({v} değerin yerine geçer). En sağdaki sütun şu "
                "anki gerçek değeri gösterir; 'yok' yazıyorsa o bilgi bu bilgisayarda okunamıyor demektir.", 12,
            muted=True, wraplength=640, justify="left").pack(anchor="w", padx=16, pady=(2, 8))
        self.sys_rows = {}
        for key, label, dfmt, _on in SYS_DEFS:
            row = ctk.CTkFrame(mc, fg_color="transparent")
            row.pack(fill="x", padx=16, pady=3)
            sw = switch(row, "", lambda k=key: self.save_sys_metric(k))
            sw.pack(side="left")
            lbl(row, label, 13, width=170, anchor="w").pack(side="left")
            ent = entry(row, 170, dfmt, radius=10, height=30)
            ent.pack(side="left")
            ent.insert(0, c["sys_metrics"][key]["fmt"])
            ent.bind("<KeyRelease>", lambda _e, k=key: self.save_sys_metric(k))
            val = lbl(row, "…", 12, True, width=110, anchor="w")
            val.pack(side="left", padx=(12, 0))
            if c["sys_metrics"][key]["on"]:
                sw.select()
            self.sys_rows[key] = (sw, ent, val)
        ctk.CTkFrame(mc, height=10, fg_color="transparent").pack()

        dc = card(body)
        dc.pack(fill="x", padx=4, pady=8)
        lbl(dc, "Gösterim şekli", 15, True).pack(anchor="w", padx=16, pady=(14, 6))
        f = ctk.CTkFrame(dc, fg_color="transparent")
        f.pack(fill="x", padx=16, pady=4)
        lbl(f, "Şekil", 13, width=180, anchor="w").pack(side="left")
        self.opt_sys_mode = option(f, list(SYS_MODES), 230, lambda _v: self.save_sys_opts())
        self.opt_sys_mode.set(next((k for k, v in SYS_MODES.items() if v == c["sys_mode"]), "Dönüşümlü (tek tek)"))
        self.opt_sys_mode.pack(side="left")
        self.add_slider(dc, "Dönüşümlü: her bilgi", "sys_secs", 1, 15, 14, "{:.0f} sn", int)
        f2 = ctk.CTkFrame(dc, fg_color="transparent")
        f2.pack(fill="x", padx=16, pady=(4, 14))
        lbl(f2, "Birleşik: ayırıcı", 13, width=180, anchor="w").pack(side="left")
        self.sys_sep_entry = entry(f2, 120, "  |  ")
        self.sys_sep_entry.pack(side="left")
        self.sys_sep_entry.insert(0, c["sys_sep"])
        self.sys_sep_entry.bind("<KeyRelease>", lambda _e: self.save_sys_opts())

        ac = card(body)
        ac.pack(fill="x", padx=4, pady=8)
        lbl(ac, "Sıcaklık uyarısı", 15, True).pack(anchor="w", padx=16, pady=(14, 0))
        self.sw_alert = self.make_switch(ac, "Sıcaklık sınırı aşılınca ekranda uyar", "alerts_on", self.save_alert,
                                         "Uyarı, oyun ya da müzik yazısının üstünde gösterilir; sıcaklık düşünce kaybolur.")
        ctk.CTkFrame(ac, height=6, fg_color="transparent").pack()
        self.add_slider(ac, "CPU sınırı", "alert_cpu", 60, 110, 50, "{:.0f} °C", int)
        self.add_slider(ac, "GPU sınırı", "alert_gpu", 60, 110, 50, "{:.0f} °C", int)
        ctk.CTkFrame(ac, height=8, fg_color="transparent").pack()

        sc = card(body)
        sc.pack(fill="x", padx=4, pady=8)
        lbl(sc, "Sıcaklık ve GPU kaynağı", 15, True).pack(anchor="w", padx=16, pady=(14, 4))
        self.src_lbl = lbl(sc, "...", 12, True, justify="left")
        self.src_lbl.pack(anchor="w", padx=16)
        lbl(sc, "NVIDIA ekran kartında GPU kullanımı, sıcaklık, VRAM ve güç kendiliğinden okunur. CPU sıcaklığı "
                "(ve AMD/Intel ekran kartları) için ücretsiz LibreHardwareMonitor programı gerekir: yönetici olarak aç, "
                "Options menüsünden 'Remote Web Server' → 'Run' seçeneğini işaretle. Windows CPU sıcaklığını kendi "
                "başına vermiyor.", 12, muted=True, wraplength=640, justify="left").pack(anchor="w", padx=16, pady=(6, 6))
        f3 = ctk.CTkFrame(sc, fg_color="transparent")
        f3.pack(fill="x", padx=16, pady=(0, 14))
        lbl(f3, "LibreHardwareMonitor portu", 13, width=190, anchor="w").pack(side="left")
        self.lhm_entry = entry(f3, 90, "8085")
        self.lhm_entry.pack(side="left")
        self.lhm_entry.insert(0, str(c["lhm_port"]))
        self.lhm_entry.bind("<KeyRelease>", lambda _e: self.save_src())

        vc = card(body)
        vc.pack(fill="x", padx=4, pady=8)
        lbl(vc, "Ses göstergesi", 15, True).pack(anchor="w", padx=16, pady=(14, 0))
        self.sw_vol = self.make_switch(vc, "Ses değişince ekranda göster", "vol_on", self.save_vol,
                                       "Ses tuşlarına ya da ses kaydırıcısına dokununca kısa süre görünür, sonra önceki yazı geri gelir.")
        ctk.CTkFrame(vc, height=6, fg_color="transparent").pack()
        self.add_slider(vc, "Ekranda kalma süresi", "vol_secs", 1, 6, 10, "{:.1f} sn", float)
        f4 = ctk.CTkFrame(vc, fg_color="transparent")
        f4.pack(fill="x", padx=16, pady=4)
        lbl(f4, "Görünüm", 13, width=180, anchor="w").pack(side="left")
        self.opt_vol_style = option(f4, VOL_STYLES, 200, lambda _v: self.save_vol())
        self.opt_vol_style.set(c["vol_style"] if c["vol_style"] in VOL_STYLES else VOL_STYLES[2])
        self.opt_vol_style.pack(side="left")
        f5 = ctk.CTkFrame(vc, fg_color="transparent")
        f5.pack(fill="x", padx=16, pady=4)
        lbl(f5, "Yazı şablonu", 13, width=180, anchor="w").pack(side="left")
        self.vol_text_entry = entry(f5, 200, "Ses: {v}")
        self.vol_text_entry.pack(side="left")
        self.vol_text_entry.insert(0, c["vol_text"])
        self.vol_text_entry.bind("<KeyRelease>", lambda _e: self.save_vol())
        f6 = ctk.CTkFrame(vc, fg_color="transparent")
        f6.pack(fill="x", padx=16, pady=4)
        lbl(f6, "Sessizdeyken yazı", 13, width=180, anchor="w").pack(side="left")
        self.vol_mute_entry = entry(f6, 200, "Sessiz")
        self.vol_mute_entry.pack(side="left")
        self.vol_mute_entry.insert(0, c["vol_mute"])
        self.vol_mute_entry.bind("<KeyRelease>", lambda _e: self.save_vol())
        OledPreview(vc, self.vol_preview).pack(anchor="w", padx=16, pady=(10, 4))
        self.vol_lbl = lbl(vc, "", 12, muted=True, wraplength=640, justify="left")
        self.vol_lbl.pack(anchor="w", padx=16, pady=(0, 14))
        self.update_sys_live()

    def save_sys_metric(self, key):
        try:
            sw, ent, _v = self.sys_rows[key]
            self.cfg["sys_metrics"][key] = {"on": bool(sw.get()), "fmt": ent.get()}
            save_config(self.cfg)
        except Exception:
            pass

    def save_sys_opts(self):
        try:
            self.cfg["sys_mode"] = SYS_MODES.get(self.opt_sys_mode.get(), "rotate")
            sep = self.sys_sep_entry.get()
            self.cfg["sys_sep"] = sep if sep.strip() else "  |  "
            save_config(self.cfg)
        except Exception:
            pass

    def save_sys_when(self):
        try:
            self.cfg["sys_when"] = SYS_WHEN.get(self.opt_sys_when.get(), "idle")
            self.cfg["sys_every"] = SYS_EVERY.get(self.opt_sys_every.get(), 5)
            self.when_desc.configure(text=SYS_WHEN_DESC.get(self.cfg["sys_when"], ""))
            save_config(self.cfg)
        except Exception:
            pass

    def test_sys(self):
        if self.worker and self.worker.is_alive():
            self.worker.force_sys_until = time.time() + 10
        else:
            messagebox.showinfo("Test", "Önce sağ üstteki Başlat düğmesine bas.", parent=self)

    def save_alert(self):
        try:
            self.cfg["alerts_on"] = bool(self.sw_alert.get())
            save_config(self.cfg)
        except Exception:
            pass

    def save_src(self):
        try:
            self.cfg["lhm_port"] = int(self.lhm_entry.get().strip())
            save_config(self.cfg)
        except Exception:
            pass

    def save_vol(self):
        try:
            c = self.cfg
            c["vol_on"] = bool(self.sw_vol.get())
            c["vol_style"] = self.opt_vol_style.get()
            c["vol_text"] = self.vol_text_entry.get() or "Ses: {v}"
            c["vol_mute"] = self.vol_mute_entry.get() or "Sessiz"
            save_config(c)
        except Exception:
            pass

    def update_sys_live(self):
        try:
            d = self.stats.data
            for key, (_sw, ent, val) in self.sys_rows.items():
                v = metric_text(key, d.get(key))
                val.configure(text=(ent.get().replace("{v}", v) if v is not None else "yok"))
            st = self.stats
            nv = "çalışıyor ✓" if st.nv_ok else ("bulundu ama okunamadı" if st.nvsmi else "bulunamadı")
            lh = "bağlı ✓" if st.lhm_ok else "bağlı değil (CPU sıcaklığı için gerekli)"
            self.src_lbl.configure(text=f"NVIDIA (nvidia-smi): {nv}\nLibreHardwareMonitor: {lh}")
            if self.vol.error and self.vol.level is None:
                self.vol_lbl.configure(text="Ses algılama çalışmıyor: " + self.vol.error +
                                       "  (python -m pip install pycaw comtypes)")
            else:
                self.vol_lbl.configure(text="Ses tuşuna bas: önizleme gerçek ses seviyeni gösterir.")
        except Exception:
            pass

    # ---- sol şerit
    def toggle_side(self):
        self.side_open = not self.side_open
        self.cfg["side_open"] = self.side_open
        save_config(self.cfg)
        self.apply_side()

    def apply_side(self):
        opened = self.side_open
        self.side.configure(width=SIDE_OPEN if opened else SIDE_CLOSED)
        for name, b in self.nav.items():
            if opened:
                b.configure(text=f"{NAV_GLYPH[name]}    {name}", anchor="w", width=SIDE_OPEN - 22, height=42,
                            font=font(14, True))
            else:
                b.configure(text=NAV_GLYPH[name], anchor="center", width=56, height=46, font=font(18, True))
        if opened:
            self.brand.pack(side="left", padx=(12, 0))
            self.timer_mini.pack_forget()
            self.timer_card.pack(side="bottom", fill="x", padx=11, pady=14)
        else:
            self.brand.pack_forget()
            self.timer_card.pack_forget()
            self.timer_mini.pack(side="bottom", pady=14)

    # ---- kapanış zamanlayıcısı
    def timer_toggle(self):
        if float(self.cfg.get("shutdown_at", 0)) > time.time():
            self.timer_cancel()
            return
        try:
            mins = float(self.timer_entry.get().replace(",", ".").strip())
        except ValueError:
            messagebox.showwarning("Zamanlayıcı", "Dakikayı sayı olarak yaz. Örn: 45", parent=self)
            return
        if mins < 1 or mins > 10080:
            messagebox.showwarning("Zamanlayıcı", "1 ile 10080 dakika arasında bir değer yaz.", parent=self)
            return
        secs = int(mins * 60)
        if not run_shutdown(secs):
            messagebox.showerror("Zamanlayıcı", "Kapanış zamanlayıcısı kurulamadı.", parent=self)
            return
        self.cfg["shutdown_at"] = time.time() + secs
        self.cfg["shutdown_minutes"] = int(mins) if mins == int(mins) else mins
        save_config(self.cfg)
        self.refresh_timer_ui()

    def timer_cancel(self):
        run_shutdown(None)
        self.cfg["shutdown_at"] = 0.0
        save_config(self.cfg)
        self.refresh_timer_ui()

    def refresh_timer_ui(self):
        try:
            left = float(self.cfg.get("shutdown_at", 0)) - time.time()
            if left > 0:
                h, r = divmod(int(left), 3600)
                m, sec = divmod(r, 60)
                self.timer_lbl.configure(text=f"Kalan: {h}:{m:02d}:{sec:02d}", text_color=TH["accent2"])
                self.timer_btn.configure(text="İptal", fg_color=TH["err"], hover_color=TH["err_hover"])
                self.timer_mini.configure(text_color=TH["err"])
            else:
                if self.cfg.get("shutdown_at"):
                    self.cfg["shutdown_at"] = 0.0
                    save_config(self.cfg)
                self.timer_lbl.configure(text="Kapalı", text_color=TH["muted"])
                self.timer_btn.configure(text="Kur", fg_color=TH["accent"], hover_color=TH["accent_hover"])
                self.timer_mini.configure(text_color=TH["muted"])
        except Exception:
            pass

    # ---- durum / başlat-durdur
    def apply_status_visuals(self):
        try:
            color = {"ok": TH["ok"], "warn": TH["warn"], "err": TH["err"], "off": TH["muted"]}[self.status_state]
            cv = self.hdr
            cv.itemconfig(self.hdr_dot, fill=color)
            txt = self.status_text if len(self.status_text) <= 36 else self.status_text[:35] + "…"
            cv.itemconfig(self.hb["pill"]["tx"], text=txt)
            h = self.hb["start"]
            if bool(self.worker and self.worker.is_alive()):
                h["fill"], h["hover"] = TH["err"], TH["err_hover"]
                cv.itemconfig(h["tx"], text="Durdur")
            else:
                h["fill"], h["hover"] = TH["ok"], TH["ok_hover"]
                cv.itemconfig(h["tx"], text="Başlat")
            cv.itemconfig(h["bg"], fill=h["fill"])
        except Exception:
            pass

    def set_status(self, text):
        def apply():
            self.status_text = text
            if text.startswith("Hata"):
                self.status_state = "err"
            elif text.startswith("Bekliyor"):
                self.status_state = "warn"
            elif text.startswith("Durduruldu"):
                self.status_state = "off"
            else:
                self.status_state = "ok"
            self.apply_status_visuals()
        try:
            self.after(0, apply)
        except Exception:
            pass

    def toggle(self):
        if self.worker and self.worker.is_alive():
            self.worker.stop_flag.set()
            self.status_text, self.status_state = "Durduruldu", "off"
            self.worker = None
            self.apply_status_visuals()
        else:
            self.save_opts()
            self.worker = Worker(self.cfg, self.media, self.stats, self.vol, self.set_status)
            self.worker.start()
            self.apply_status_visuals()

    # ---- önizleme ve medya etiketi
    def tick(self):
        try:
            if not self.building:
                self._tick_n += 1
                if self._tick_n % 10 == 0:
                    self.refresh_timer_ui()
                    if self.current_page == PAGE_SYS:
                        self.update_sys_live()
                if self._tick_n % 5 == 0 and self.current_page == PAGE_PROF:
                    self.update_prof_now()
                if self._tick_n % 5 == 0 and self.current_page == PAGE_PRIO:
                    self.update_prio_live()
                self.process_queue()
                if self.current_page == PAGE_TEXT:
                    text = self.preview_entry.get() or " "
                    sig = text_settings(self.cfg) + (text,)
                    if sig != self._prev_sig:
                        self._prev_sig = sig
                        self._prev_scroller = Scroller(text, *text_settings(self.cfg))
                        self._prev_t0 = time.time()
                    fr = self._prev_scroller.frame(time.time() - self._prev_t0)
                    img = ImageOps.colorize(fr.convert("L"), black=(8, 10, 14), white=(170, 220, 255))
                    img = img.resize((128 * PREVIEW_SCALE, 40 * PREVIEW_SCALE), Image.NEAREST)
                    ci = ctk.CTkImage(light_image=img, dark_image=img, size=img.size)
                    self.preview_lbl.configure(image=ci)
                    self._prev_img = ci
                if self.current_page == PAGE_GEN and self._tick_n % 10 == 0:
                    if self.media.error and not self.media.ready:
                        txt = "Medya algılama çalışmıyor: " + self.media.error
                    else:
                        m = self.media.pick(None)
                        txt = ("Şu an algılanan: " + MediaWatcher.text_of(m)) if m else \
                            "Şu an çalan medya algılanmadı. Tarayıcıda bir şarkı/video oynat ve burayı tekrar kontrol et."
                    self.media_lbl.configure(text=txt)
        except Exception:
            pass
        try:
            self.after(100, self.tick)
        except Exception:
            pass

    # ---- tepsi simgesi ve kapanış
    def make_tray_image(self):
        img = Image.new("RGBA", (64, 64), (0, 0, 0, 0))
        d = ImageDraw.Draw(img)
        d.rounded_rectangle((2, 2, 62, 62), radius=16, fill=hex2rgb(TH["accent"]) + (255,))
        f = load_font(40, True)
        bb = d.textbbox((0, 0), "A", font=f)
        d.text(((64 - (bb[2] - bb[0])) / 2 - bb[0], (64 - (bb[3] - bb[1])) / 2 - bb[1]), "A", font=f,
               fill=(255, 255, 255, 255))
        return img

    def hide_to_tray(self):
        if pystray is None:
            self.iconify()
            return
        if self.tray is None:
            try:
                menu = pystray.Menu(
                    pystray.MenuItem("Aç", lambda icon, item: self.q.put("show"), default=True),
                    pystray.MenuItem("Başlat / Durdur", lambda icon, item: self.q.put("toggle")),
                    pystray.MenuItem("Çıkış", lambda icon, item: self.q.put("quit")))
                self.tray = pystray.Icon("ApexOLED", self.make_tray_image(), "Apex OLED", menu)
                self.tray.run_detached()
            except Exception:
                self.tray = None
                self.iconify()
                return
        self.withdraw()

    def process_queue(self):
        while True:
            try:
                cmd = self.q.get_nowait()
            except queue.Empty:
                break
            if cmd == "show":
                self.deiconify()
                self.lift()
                self.focus_force()
            elif cmd == "toggle":
                self.toggle()
            elif cmd == "quit":
                self.quit_app()
                return

    def on_close(self):
        if self.cfg.get("close_to_tray", True) and pystray is not None:
            self.hide_to_tray()
        else:
            self.quit_app()

    def quit_app(self):
        if self.worker:
            self.worker.stop_flag.set()
        for t in (self.media, self.stats, self.vol):
            t.stop_flag.set()
        if self.tray is not None:
            try:
                self.tray.stop()
            except Exception:
                pass
        self.destroy()


if __name__ == "__main__":
    if sys.platform != "win32":
        print("Bu program sadece Windows için.")
        sys.exit(1)
    App().mainloop()
