from datetime import datetime
import tkinter as tk
from tkinter import messagebox, ttk
import json

# ================= VERİ =================
try:
    with open("bakim_gecmisi.json", "r", encoding="utf-8") as f:
        bakim_gecmisi = json.load(f)
except:
    bakim_gecmisi = []

try:
    with open("bakim_takip.json", "r", encoding="utf-8") as f:
        araclar = json.load(f)
except:
    araclar = []


# ================= HESAP =================
def bakima_kalan_km(arac):
    return arac["bakim_araligi"] - (arac["mevcut_km"] - arac["son_bakim_km"])


# ================= ÇIKIŞ =================
def cikis():
    with open("bakim_takip.json", "w", encoding="utf-8") as f:
        json.dump(araclar, f, ensure_ascii=False, indent=4)

    with open("bakim_gecmisi.json", "w", encoding="utf-8") as f:
        json.dump(bakim_gecmisi, f, ensure_ascii=False, indent=4)

    root.destroy()


# ================= HOVER =================
son_hover = {"item": None}

def hover_hareket(event):
    row_id = tablo.identify_row(event.y)
    if son_hover["item"] != row_id:
        if son_hover["item"]:
            tablo.item(son_hover["item"], tags=())
        if row_id:
            tablo.item(row_id, tags=("hover",))
        son_hover["item"] = row_id

def hover_cikis(event):
    if son_hover["item"]:
        tablo.item(son_hover["item"], tags=())
        son_hover["item"] = None


# ================= TABLO =================
def tabloyu_yenile():
    tablo.delete(*tablo.get_children())
    kritik, riskli = "", ""

    for arac in araclar:
        kalan = bakima_kalan_km(arac)

        if kalan <= 0:
            durum, tag = "🔴 GECİKTİ", "kirmizi"
            kritik += f"{arac['plaka']} → {abs(kalan)} km gecikmiş\n"
        elif kalan <= 1000:
            durum, tag = "🟠 YAKLAŞTI", "turuncu"
            riskli += f"{arac['plaka']} → {kalan} km kaldı\n"
        else:
            durum, tag = "🟢 OK", "yesil"

        tablo.insert(
            "", "end",
            values=(arac["plaka"], arac["mevcut_km"], arac["son_bakim_km"], kalan, durum),
            tags=(tag,)
        )

    if kritik or riskli:
        messagebox.showwarning("Bakım Uyarısı", kritik + riskli)


# ================= ARAÇ EKLE =================
def arac_ekle():
    p = tk.Toplevel(root)
    p.title("Araç Ekle")

    for text in ["Plaka", "Mevcut KM", "Bakım Aralığı"]:
        tk.Label(p, text=text).pack()

    e_plaka = tk.Entry(p); e_plaka.pack()
    e_km = tk.Entry(p); e_km.pack()
    e_aralik = tk.Entry(p); e_aralik.pack()

    def kaydet():
        if not e_plaka.get() or not e_km.get() or not e_aralik.get():
            messagebox.showerror("Hata", "Alanları doldur")
            return

        araclar.append({
            "plaka": e_plaka.get().upper(),
            "mevcut_km": int(e_km.get()),
            "son_bakim_km": int(e_km.get()),
            "bakim_araligi": int(e_aralik.get())
        })
        tabloyu_yenile()
        p.destroy()

    tk.Button(p, text="Kaydet", command=kaydet).pack(pady=5)


# ================= KM GÜNCELLE =================
def km_guncelle():
    p = tk.Toplevel(root)
    p.title("KM Güncelle")

    tk.Label(p, text="Plaka").pack()
    e_plaka = tk.Entry(p); e_plaka.pack()
    tk.Label(p, text="Yeni KM").pack()
    e_km = tk.Entry(p); e_km.pack()

    def guncelle():
        for a in araclar:
            if a["plaka"] == e_plaka.get().upper():
                a["mevcut_km"] = int(e_km.get())
                tabloyu_yenile()
                p.destroy()
                return
        messagebox.showerror("Hata", "Araç yok")

    tk.Button(p, text="Güncelle", command=guncelle).pack(pady=5)


# ================= BAKIM YAPILDI =================
def bakim_yapildi():
    p = tk.Toplevel(root)
    p.title("Bakım Yapıldı")

    tk.Label(p, text="Plaka").pack()
    e_plaka = tk.Entry(p); e_plaka.pack()

    def yap():
        for a in araclar:
            if a["plaka"] == e_plaka.get().upper():
                a["son_bakim_km"] = a["mevcut_km"]
                bakim_gecmisi.append({
                    "plaka": a["plaka"],
                    "km": a["mevcut_km"],
                    "tarih": datetime.now().strftime("%d.%m.%Y %H:%M")
                })
                tabloyu_yenile()
                p.destroy()
                return
        messagebox.showerror("Hata", "Araç yok")

    tk.Button(p, text="Onayla", command=yap).pack(pady=5)


# ================= BAKIM GEÇMİŞİ =================
def bakim_gecmisi_ekrani():
    p = tk.Toplevel(root)
    p.title("Bakım Geçmişi")
    p.geometry("600x400")

    ust = tk.Frame(p)
    ust.pack(fill="x", pady=5)

    tk.Label(ust, text="Plaka Ara").pack(side="left")
    e = tk.Entry(ust)
    e.pack(side="left", padx=5)

    tablo2 = ttk.Treeview(p, columns=("plaka","km","tarih"), show="headings")
    for s in ("plaka","km","tarih"):
        tablo2.heading(s, text=s.upper())
        tablo2.column(s, anchor="center", width=180)
    tablo2.pack(expand=True, fill="both")

    def doldur(liste):
        tablo2.delete(*tablo2.get_children())
        for k in liste:
            tablo2.insert("", "end", values=(k["plaka"], k["km"], k["tarih"]))

    tk.Button(ust, text="Ara", command=lambda: doldur(
        [k for k in bakim_gecmisi if k["plaka"] == e.get().upper()]
    )).pack(side="left", padx=5)

    tk.Button(ust, text="Tümü", command=lambda: doldur(bakim_gecmisi))\
        .pack(side="left")

    doldur(bakim_gecmisi)


# ================= ANA =================
root = tk.Tk()
root.title("Bakım Takip Sistemi")
root.geometry("900x550")

style = ttk.Style()
style.configure("Treeview", rowheight=30, font=("Arial", 11))
style.map("Treeview", background=[("selected", "#2c3e50")])

tablo = ttk.Treeview(root, columns=("p","m","s","k","d"), show="headings")
for col, text in zip(("p","m","s","k","d"),
                     ("Plaka","Mevcut","Son","Kalan","Durum")):
    tablo.heading(col, text=text)
    tablo.column(col, anchor="center", width=160)

tablo.tag_configure("hover", background="#ecf0f1")
tablo.tag_configure("kirmizi", background="#ffcccc")
tablo.tag_configure("turuncu", background="#fff0cc")
tablo.tag_configure("yesil", background="#e6ffe6")

tablo.bind("<Motion>", hover_hareket)
tablo.bind("<Leave>", hover_cikis)
tablo.pack(expand=True, fill="both")

buton_frame = tk.Frame(root)
buton_frame.pack(pady=10)

tk.Button(buton_frame, text="Bakım Geçmişi", width=18, command=bakim_gecmisi_ekrani)\
    .grid(row=0, column=0, padx=5)

tk.Button(buton_frame, text="Araç Ekle", width=18, command=arac_ekle)\
    .grid(row=0, column=1, padx=5)

tk.Button(buton_frame, text="KM Güncelle", width=18, command=km_guncelle)\
    .grid(row=0, column=2, padx=5)

tk.Button(buton_frame, text="Bakım Yapıldı", width=18, command=bakim_yapildi)\
    .grid(row=0, column=3, padx=5)

tk.Button(buton_frame, text="Çıkış", width=18, command=cikis)\
    .grid(row=0, column=4, padx=5)

tabloyu_yenile()
root.mainloop()
