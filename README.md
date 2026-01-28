import sys
import os
import threading
import time
from pathlib import Path
import ctypes
import tkinter as tk
from tkinter import ttk, filedialog, messagebox
from PIL import Image, ImageTk

# ==========================================================
# Força o uso da DLL do MediaInfo (local, mesma pasta do exe)
# ==========================================================
try:
    # Garante compatibilidade com .exe, .py e Jupyter
    base_dir = None
    if getattr(sys, "frozen", False):
        base_dir = Path(sys.executable).parent
    else:
        try:
            base_dir = Path(__file__).parent
        except NameError:
            base_dir = Path(os.getcwd())

    dll_path = base_dir / "MediaInfo.dll"
    if dll_path.exists():
        ctypes.CDLL(str(dll_path))
    else:
        print("⚠ MediaInfo.dll não encontrada — os codecs não serão exibidos.")
except Exception as e:
    print("Aviso: falha ao carregar MediaInfo.dll:", e)

# Importa pymediainfo
try:
    from pymediainfo import MediaInfo
    HAS_MEDIAINFO = True
except Exception as e:
    HAS_MEDIAINFO = False
    print("⚠ pymediainfo não está instalado:", e)

# ==========================================================
# Drag & Drop
# ==========================================================
try:
    from tkinterdnd2 import DND_FILES, TkinterDnD
    TKDND_OK = True
except Exception:
    TKDND_OK = False

# ==========================================================
# VLC
# ==========================================================
try:
    os.add_dll_directory(r"C:\Program Files\VideoLAN\VLC")
except Exception:
    pass
import vlc

# ==========================================================
# CONFIG
# ==========================================================
APP_TITLE = "Compara Vídeo"
SOBRE_TEXTO = "Sergio Costa\n\nNovembro 2025\nVersão 1.5.7"

def app_dir() -> Path:
    if getattr(sys, 'frozen', False):
        return Path(sys.executable).parent
    try:
        return Path(__file__).parent
    except NameError:
        return Path(os.getcwd())

ICON_PATH = app_dir() / "comparavideo.ico"
LOGO_PATH = app_dir() / "comparavideo.png"

# ==========================================================
# MAIN CLASS
# ==========================================================
BaseTk = TkinterDnD.Tk if TKDND_OK else tk.Tk

class ComparaVideoApp(BaseTk):
    def __init__(self):
        super().__init__()
        self.title(APP_TITLE)
        try:
            if ICON_PATH.exists():
                self.iconbitmap(str(ICON_PATH))
        except Exception:
            pass

        self.vlc_instance = vlc.Instance()
        self.player_left = self.vlc_instance.media_player_new()
        self.player_right = self.vlc_instance.media_player_new()

        self._build_ui()
        self.after(200, self._attach_render_targets)
        self.protocol("WM_DELETE_WINDOW", self.on_close)
        self.bind("<space>", lambda e: self.toggle_pause())

    # ---------------------------------
    def _build_ui(self):
        style = ttk.Style(self)
        try:
            style.theme_use("clam")
        except Exception:
            pass
        self.configure(bg=style.lookup("TFrame", "background", default="#d9d9d9"))

        top_bar = ttk.Frame(self)
        top_bar.pack(side=tk.TOP, fill=tk.X, padx=8, pady=4)
        ttk.Button(top_bar, text="Sobre", command=self._sobre).pack(side=tk.RIGHT)

        videos_frame = ttk.Frame(self)
        videos_frame.pack(side=tk.TOP, fill=tk.BOTH, expand=True, padx=8, pady=8)

        # Esquerda
        left_frame = ttk.Frame(videos_frame)
        left_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=(0,4))
        self.left_canvas = tk.Canvas(left_frame, bg="black", highlightthickness=0)
        self.left_canvas.pack(fill=tk.BOTH, expand=True)
        self.left_info = ttk.Label(left_frame, text="", justify="left", anchor="w", font=("Segoe UI", 9))
        self.left_info.pack(fill=tk.X, pady=4)

        # Direita
        right_frame = ttk.Frame(videos_frame)
        right_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=(4,0))
        self.right_canvas = tk.Canvas(right_frame, bg="black", highlightthickness=0)
        self.right_canvas.pack(fill=tk.BOTH, expand=True)
        self.right_info = ttk.Label(right_frame, text="", justify="left", anchor="w", font=("Segoe UI", 9))
        self.right_info.pack(fill=tk.X, pady=4)

        # Botões
        controls = ttk.Frame(self)
        controls.pack(side=tk.TOP, pady=10)
        ttk.Button(controls, text="Abrir Vídeo(s)", command=self._abrir_videos).grid(row=0, column=0, padx=8)
        ttk.Button(controls, text="Play", command=self.play).grid(row=0, column=1, padx=8)
        ttk.Button(controls, text="Pause", command=self.pause).grid(row=0, column=2, padx=8)
        ttk.Button(controls, text="Stop", command=self.stop).grid(row=0, column=3, padx=8)

    # ---------------------------------
    def _attach_render_targets(self):
        self.update_idletasks()
        try:
            self.player_left.set_hwnd(self.left_canvas.winfo_id())
            self.player_right.set_hwnd(self.right_canvas.winfo_id())
        except Exception:
            self.after(300, self._attach_render_targets)

    # ---------------------------------
    def _abrir_videos(self):
        paths = filedialog.askopenfilenames(
            title="Selecione 1 ou 2 vídeos",
            filetypes=[("Vídeos", "*.mp4;*.mov;*.mkv;*.avi;*.wmv;*.m4v;*.webm"), ("Todos", "*.*")]
        )
        if not paths:
            return
        if len(paths) == 1:
            self._carregar_video_left(paths[0])
        else:
            self._carregar_video_left(paths[0])
            self._carregar_video_right(paths[1])

    def _carregar_video_left(self, path):
        media = self.vlc_instance.media_new(path)
        self.player_left.set_media(media)
        self._previsualizar(self.player_left)
        self._mostrar_info(path, self.left_info)

    def _carregar_video_right(self, path):
        media = self.vlc_instance.media_new(path)
        self.player_right.set_media(media)
        self._previsualizar(self.player_right)
        self._mostrar_info(path, self.right_info)

    def _previsualizar(self, player):
        def preview():
            player.play()
            time.sleep(0.3)
            player.set_pause(1)
        threading.Thread(target=preview, daemon=True).start()

    # ---------------------------------
    def _mostrar_info(self, path, label_widget):
        def worker():
            vcodec, acodec, fps, width, height, duration = "—", "—", "—", "—", "—", "—"
            try:
                if HAS_MEDIAINFO:
                    mi = MediaInfo.parse(path)
                    for track in mi.tracks:
                        if track.track_type == "Video":
                            vcodec = track.format or track.codec or "—"
                            width = track.width or "—"
                            height = track.height or "—"
                            fps = track.frame_rate or "—"
                            duration = round(float(track.duration) / 1000, 1) if track.duration else "—"
                        elif track.track_type == "Audio":
                            acodec = track.format or track.codec or "não possui camada de áudio"
            except Exception as e:
                print("Erro ao ler MediaInfo:", e)

            texto = (
                f"Duração: {duration}s\n"
                f"Dimensões: {width}x{height}\n"
                f"Codec de vídeo: {vcodec}\n"
                f"Codec de áudio: {acodec}\n"
                f"FPS: {fps}\n"
                f"Extensão: {Path(path).suffix.lower()}"
            )
            label_widget.config(text=texto)
        threading.Thread(target=worker, daemon=True).start()

    # ---------------------------------
    def play(self):
        self.player_left.play()
        self.player_right.play()

    def pause(self):
        self.player_left.pause()
        self.player_right.pause()

    def toggle_pause(self):
        for p in [self.player_left, self.player_right]:
            if p.is_playing():
                p.pause()
            else:
                p.play()

    def stop(self):
        self.player_left.stop()
        self.player_right.stop()

    # ---------------------------------
    def _sobre(self):
        top = tk.Toplevel(self)
        top.title("Sobre")
        if LOGO_PATH.exists():
            img = Image.open(LOGO_PATH)
            w, h = img.size
            new_size = (int(w * 0.7), int(h * 0.7))
            img = img.resize(new_size, Image.LANCZOS)
            img_tk = ImageTk.PhotoImage(img)
            lbl = tk.Label(top, image=img_tk)
            lbl.image = img_tk
            lbl.pack(pady=10)
        tk.Label(top, text="Sergio Costa", font=("Segoe UI", 11, "bold")).pack(pady=2)
        tk.Label(top, text="Novembro 2025", font=("Segoe UI", 10)).pack(pady=1)
        tk.Label(top, text="Versão 1.5.7", font=("Segoe UI", 9)).pack(pady=5)

    def on_close(self):
        self.player_left.stop()
        self.player_right.stop()
        self.vlc_instance.release()
        self.destroy()

# ==========================================================
def main():
    app = ComparaVideoApp()
    app.geometry("1200x700")
    app.minsize(800, 500)
    app.mainloop()

if __name__ == "__main__":
    main()
