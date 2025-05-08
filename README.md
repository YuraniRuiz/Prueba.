import tkinter as tk
from tkinter import filedialog, messagebox, ttk
import os
import sqlite3
import csv
import platform
import subprocess

# =================== CONEXIÓN A BASE DE DATOS =====================
conn = sqlite3.connect('pdf_manager.db')
cursor = conn.cursor()

cursor.execute('''
CREATE TABLE IF NOT EXISTS pdfs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    path TEXT NOT NULL,
    category TEXT
)
''')
conn.commit()

# =================== VENTANA PRINCIPAL =====================
root = tk.Tk()
root.title("Gestor de PDFs - Alcaldía")
root.geometry("900x550")
root.configure(bg="white")

# =================== COLORES Y FUENTES =====================
AZUL = "#0033A0"
BLANCO = "white"
AMARILLO = "#FFD700"
FUENTE_GENERAL = ("Georgia", 11)
FUENTE_TITULO = ("Georgia", 16, "bold")

root.option_add("*Font", FUENTE_GENERAL)

# =================== LOGO =====================
logo_frame = tk.Frame(root, bg=BLANCO)
logo_frame.pack(pady=(10, 0))

try:
    logo_image = tk.PhotoImage(file="alcaldia_logo.gif")
    logo_label = tk.Label(logo_frame, image=logo_image, bg=BLANCO)
    logo_label.image = logo_image  
    logo_label.pack()
except Exception:
    logo_label = tk.Label(logo_frame, text="", bg=BLANCO, fg=AZUL, font=FUENTE_TITULO)
    logo_label.pack()

# =================== FUNCIONES =====================
def add_pdf():
    file_path = filedialog.askopenfilename(filetypes=[("PDF Files", "*.pdf")])
    if file_path:
        file_name = os.path.basename(file_path)
        category = category_entry.get().strip()
        if not category:
            messagebox.showwarning("Categoría requerida", "Debes ingresar una categoría.")
            return
        try:
            cursor.execute("INSERT INTO pdfs (name, path, category) VALUES (?, ?, ?)", (file_name, file_path, category))
            conn.commit()
            messagebox.showinfo("Éxito", "PDF agregado correctamente.")
            load_pdfs()
        except Exception as e:
            messagebox.showerror("Error", f"Ocurrió un error al agregar el PDF: {e}")

def delete_pdf():
    selected = tree.selection()
    if selected:
        for item in selected:
            pdf_id = tree.item(item, "values")[0]
            cursor.execute("DELETE FROM pdfs WHERE id=?", (pdf_id,))
        conn.commit()
        load_pdfs()
    else:
        messagebox.showwarning("Seleccionar PDF", "Debes seleccionar al menos un PDF para eliminar.")

def load_pdfs():
    for item in tree.get_children():
        tree.delete(item)
    # Ordenar por año en la categoría (si empieza con un año de 4 dígitos)
    cursor.execute("""
        SELECT id, name, category, path FROM pdfs
        ORDER BY 
            CASE 
                WHEN category GLOB '[0-9][0-9][0-9][0-9]*' THEN CAST(SUBSTR(category, 1, 4) AS INTEGER)
                ELSE 0
            END DESC
    """)
    for row in cursor.fetchall():
        tree.insert("", "end", values=row)

def export_list():
    file_path = filedialog.asksaveasfilename(defaultextension=".csv", filetypes=[("CSV files", "*.csv")])
    if file_path:
        cursor.execute("SELECT name, category, path FROM pdfs")
        with open(file_path, "w", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            writer.writerow(["Nombre", "Categoría", "Ruta"])
            writer.writerows(cursor.fetchall())
        messagebox.showinfo("Éxito", "Lista exportada correctamente.")

def open_pdf(event):
    selected_item = tree.focus()
    if selected_item:
        values = tree.item(selected_item, "values")
        pdf_path = values[3]
        try:
            if platform.system() == 'Windows':
                os.startfile(pdf_path)
            elif platform.system() == 'Darwin':
                subprocess.call(('open', pdf_path))
            else:
                subprocess.call(('xdg-open', pdf_path))
        except Exception as e:
            messagebox.showerror("Error", f"No se pudo abrir el archivo PDF:\n{e}")

# =================== INTERFAZ =====================
frame = tk.Frame(root, bg=BLANCO)
frame.pack(pady=20)

tk.Label(frame, text="Categoría:", bg=BLANCO, fg=AZUL, font=FUENTE_GENERAL).grid(row=0, column=0, padx=5)
category_entry = tk.Entry(frame, font=FUENTE_GENERAL)
category_entry.grid(row=0, column=1, padx=5)

add_button = tk.Button(frame, text="Agregar PDF", bg=AMARILLO, fg="black", font=FUENTE_GENERAL, command=add_pdf)
add_button.grid(row=0, column=2, padx=5)

delete_button = tk.Button(frame, text="Eliminar PDF", bg=AZUL, fg=BLANCO, font=FUENTE_GENERAL, command=delete_pdf)
delete_button.grid(row=0, column=3, padx=5)

export_button = tk.Button(frame, text="Exportar Lista", bg=AZUL, fg=BLANCO, font=FUENTE_GENERAL, command=export_list)
export_button.grid(row=0, column=4, padx=5)

# =================== TABLA =====================
columns = ("ID", "Nombre", "Categoría", "Ruta")
tree = ttk.Treeview(root, columns=columns, show="headings", height=15)

style = ttk.Style()
style.theme_use("clam")
style.configure("Treeview",
                background=BLANCO,
                foreground="black",
                rowheight=25,
                fieldbackground=BLANCO,
                font=FUENTE_GENERAL)
style.configure("Treeview.Heading", font=("Georgia", 11, "bold"))
style.map("Treeview", background=[("selected", AMARILLO)])

for col in columns:
    tree.heading(col, text=col)
    tree.column(col, width=200 if col != "ID" else 50)

tree.pack(padx=10, pady=10, fill="both", expand=True)
tree.bind("<Double-1>", open_pdf)

# =================== CARGAR DATOS =====================
load_pdfs()

# =================== EJECUTAR =====================
root.mainloop()
