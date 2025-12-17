import tkinter as tk
from tkinter import ttk, messagebox
import sqlite3
from datetime import datetime, timedelta

# --- CONFIGURACIÓN DE COLORES Y CONSTANTES ---
# Colores de estado para Treeview (Se mantienen para accesibilidad y visualización)
STATUS_COLORS = {
    "Completada": {"bg": "#D4EDDA", "fg": "#155724"}, # Verde
    "En progreso": {"bg": "#FFF3CD", "fg": "#856404"}, # Amarillo
    "Pendiente": {"bg": "#F8D7DA", "fg": "#721C24"}    # Rojo
}

# Rangos de prioridad numérica
PRIORITY_VALUES = list(range(1, 6)) # De 1 (Baja) a 5 (Alta)

# Configuración de la Base de Datos
DB_FILE = "tasks.db"

# ================= VALIDACIÓN =================

def validar_fecha(f):
    """Valida que la cadena de texto sea una fecha válida en formato YYYY-MM-DD."""
    try:
        datetime.strptime(f, "%Y-%m-%d")
        return True
    except:
        return False

# ================= BASE DE DATOS =================
class TaskDB:
    def __init__(self, path=DB_FILE):
        self.conn = sqlite3.connect(path)
        self.cursor = self.conn.cursor()

        try:
            # Asegurar que el campo 'priority' puede manejar números (TEXT es flexible)
            self.cursor.execute("ALTER TABLE tasks ADD COLUMN category TEXT")
            self.conn.commit()
        except sqlite3.OperationalError:
            pass

        self.cursor.execute("""
        CREATE TABLE IF NOT EXISTS tasks(
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            description TEXT,
            due_date TEXT,
            priority TEXT, 
            status TEXT,
            created_at TEXT,
            category TEXT)
        """)
        self.conn.commit()

    def add_task(self, t, d, du, pr, st, cat):
        # La prioridad ahora es un número guardado como texto
        self.cursor.execute(
            "INSERT INTO tasks(title,description,due_date,priority,status,created_at,category) VALUES(?,?,?,?,?,?,?)",
            (t, d, du, pr, st, datetime.now().strftime("%Y-%m-%d %H:%M:%S"), cat)
        )
        self.conn.commit()

    def update_task(self, id, t, d, du, pr, st, cat):
        self.cursor.execute(
            "UPDATE tasks SET title=?,description=?,due_date=?,priority=?,status=?,category=? WHERE id=?",
            (t, d, du, pr, st, cat, id)
        )
        self.conn.commit()

    def delete_task(self, id):
        self.cursor.execute("DELETE FROM tasks WHERE id=?", (id,))
        self.conn.commit()

    def get_all_tasks(self):
        # Ordenamos por Prioridad (numéricamente, Alto a Bajo) y luego por id
        return self.cursor.execute(
            "SELECT id,title,description,created_at,due_date,priority,status,category FROM tasks ORDER BY CAST(priority AS INTEGER) DESC, id DESC"
        ).fetchall()

    def get_task(self, id):
        return self.cursor.execute(
            "SELECT id,title,description,created_at,due_date,priority,status,category FROM tasks WHERE id=?",
            (id,)
        ).fetchone()

# ================= FORMULARIO (Agregar/Editar) =================
class TaskForm(tk.Toplevel):
    def __init__(self, master, db, on_save, task_id=None):
        super().__init__(master)
        self.db, self.on_save, self.task_id = db, on_save, task_id
        self.title("Agregar Tarea" if not task_id else "Editar Tarea")
        self.resizable(False, False)
        
        self.transient(master)
        self.grab_set()

        pad = 8

        # --- Widgets de entrada ---
        ttk.Label(self, text="Título:*").grid(row=0, column=0, padx=pad, pady=pad, sticky="e")
        ttk.Label(self, text="Descripción:").grid(row=1, column=0, padx=pad, pady=pad, sticky="ne")
        ttk.Label(self, text="Fecha vencimiento: (YYYY-MM-DD)").grid(row=2, column=0, padx=pad, pady=pad, sticky="e")
        ttk.Label(self, text="Prioridad: (1=Baja, 5=Alta)").grid(row=3, column=0, padx=pad, pady=pad, sticky="e") # Nuevo texto
        ttk.Label(self, text="Estado:").grid(row=4, column=0, padx=pad, pady=pad, sticky="e")
        ttk.Label(self, text="Categoría/Etiqueta:").grid(row=5, column=0, padx=pad, pady=pad, sticky="e")

        self.title_var = tk.StringVar()
        ttk.Entry(self, textvariable=self.title_var, width=45).grid(row=0, column=1)

        self.desc_text = tk.Text(self, width=45, height=5)
        self.desc_text.grid(row=1, column=1)

        self.due_var = tk.StringVar()
        if not task_id:
            self.due_var.set((datetime.today() + timedelta(days=1)).strftime("%Y-%m-%d"))
        ttk.Entry(self, textvariable=self.due_var).grid(row=2, column=1)

        # PRIORIDAD NUMÉRICA: Usamos Combobox con valores de 1 a 5
        self.priority_var = tk.StringVar(value="3") # Valor por defecto: 3 (Media)
        ttk.Combobox(self, textvariable=self.priority_var,
                     values=PRIORITY_VALUES,
                     state="readonly").grid(row=3, column=1)

        self.status_var = tk.StringVar(value="Pendiente")
        ttk.Combobox(self, textvariable=self.status_var,
                     values=["Pendiente", "En progreso", "Completada"],
                     state="readonly").grid(row=4, column=1)

        self.category_var = tk.StringVar(value="")
        ttk.Entry(self, textvariable=self.category_var, width=45).grid(row=5, column=1)

        # --- Botones ---
        ttk.Button(self, text="Guardar", command=self.guardar).grid(row=6, column=0, pady=10)
        ttk.Button(self, text="Cancelar", command=self.destroy).grid(row=6, column=1, pady=10)

        if task_id:
            self.cargar()
            
        self.wait_window(self)

    def cargar(self):
        r = self.db.get_task(self.task_id)
        if r:
            _, t, d, _, due, pr, st, cat = r
            self.title_var.set(t)
            self.desc_text.insert("1.0", d or "")
            self.due_var.set(due or "")
            self.priority_var.set(pr or "3") # Asegurar valor por defecto si es nulo
            self.status_var.set(st)
            self.category_var.set(cat or "")

    def guardar(self):
        if not self.title_var.get().strip():
            return messagebox.showwarning("Validación", "El título es obligatorio.", parent=self)

        due_date_str = self.due_var.get().strip()
        if due_date_str and not validar_fecha(due_date_str):
            return messagebox.showwarning("Validación", "Fecha inválida. Use formato YYYY-MM-DD.", parent=self)

        data = (
            self.title_var.get().strip(),
            self.desc_text.get("1.0", tk.END).strip(),
            due_date_str,
            self.priority_var.get(),
            self.status_var.get(),
            self.category_var.get().strip()
        )

        if self.task_id:
            self.db.update_task(self.task_id, *data)
        else:
            self.db.add_task(*data)

        self.on_save()
        self.destroy()

# ================= APP PRINCIPAL =================
class TaskManagerApp(tk.Tk): 
    def __init__(self):
        super().__init__()
        self.title("Gestor de Tareas Personales")
        self.geometry("1050x580") 
        self.db = TaskDB()
        self.visual_to_real_id = {}
        
        # Eliminar lógica de tema Claro/Oscuro, usar estilos ttk básicos
        self.style = ttk.Style(self)
        self.configure(bg='#f0f0f0') 
        self.style.configure('TFrame', background='#f0f0f0')
        self.style.configure('TLabel', background='#f0f0f0', foreground='black')

        self.crear_widgets()
        self.crear_estilos_colores()
        self.cargar_tareas()
        self.verificar_recordatorios()

    # --- Estilos ---
    def crear_estilos_colores(self):
        """Define los estilos de color para las filas del Treeview."""
        for status, colors in STATUS_COLORS.items():
            tag_name = status.lower().replace(" ", "_")
            self.tree.tag_configure(tag_name, background=colors["bg"], foreground=colors["fg"])
        
        # Estilo adicional para tareas vencidas
        self.tree.tag_configure("vencida", background="#FFC0CB", foreground="#9E0000") # Rosa claro/Rojo oscuro

    # --- Interfaz de Usuario ---
    def crear_widgets(self):
        # Frame superior para controles y botones
        top = ttk.Frame(self)
        top.pack(fill="x", padx=8, pady=6)

        # Controles de Filtrado
        ttk.Label(top, text="Estado:").pack(side="left")
        self.filter_status = tk.StringVar(value="Todos")
        ttk.Combobox(top, textvariable=self.filter_status,
                     values=["Todos", "Pendiente", "En progreso", "Completada"],
                     width=14, state="readonly").pack(side="left", padx=5)

        # FILTRO DE PLAZO DINÁMICO (Reemplaza filtro de Prioridad)
        ttk.Label(top, text="Plazo:").pack(side="left")
        self.filter_deadline = tk.StringVar(value="Todos")
        ttk.Combobox(top, textvariable=self.filter_deadline,
                     values=["Todos", "Vencidas", "Vence Hoy", "Próximos 7 Días", "Próximo Mes"],
                     width=16, state="readonly").pack(side="left", padx=5)

        ttk.Button(top, text="Aplicar Filtros", command=self.cargar_tareas).pack(side="left", padx=6)

        # Botones de Acción
        # El botón de modo oscuro/claro ha sido ELIMINADO
        ttk.Button(top, text="Stats", command=self.ver_stats).pack(side="right", padx=4)
        ttk.Button(top, text="Eliminar", command=self.eliminar).pack(side="right", padx=4)
        ttk.Button(top, text="Editar", command=self.editar).pack(side="right", padx=4)
        ttk.Button(top, text="Agregar", command=self.agregar).pack(side="right", padx=4)

        # Treeview (Tabla)
        # La columna 'priority' ahora muestra números
        cols = ("vid", "title", "description", "due", "priority", "status", "category")
        headers = ["Id", "Título", "Descripción", "Vencimiento", "Prioridad", "Estado", "Categoría"]
        widths = [50, 150, 290, 110, 90, 110, 130] 

        self.tree = ttk.Treeview(self, columns=cols, show="headings")
        for c, h, w in zip(cols, headers, widths):
            self.tree.heading(c, text=h)
            self.tree.column(c, width=w, anchor="w")
        self.tree.pack(fill="both", expand=True, padx=8)

        # Área de información y Contador de Tareas
        bottom_frame = ttk.Frame(self)
        bottom_frame.pack(fill="x", padx=8, pady=6)

        self.info_label = ttk.Label(bottom_frame, text="Seleccione una tarea")
        self.info_label.pack(side="left", anchor="w")

        self.count_label = ttk.Label(bottom_frame, text="Total: 0 | Pendientes: 0")
        self.count_label.pack(side="right", anchor="e")

        self.tree.bind("<<TreeviewSelect>>", self.actualizar_info)

    # --- Lógica de Tareas y Datos ---
    def _apply_deadline_filter(self, r):
        """Aplica el filtro de plazo dinámico a una tarea (r)."""
        deadline_filter = self.filter_deadline.get()
        due_date_str = r[4] # due_date (YYYY-MM-DD)
        status = r[6] # status

        if deadline_filter == "Todos":
            return True
        if status == "Completada":
            return False # Ignorar completadas para los filtros de plazo

        if not due_date_str:
            return False

        try:
            due_date = datetime.strptime(due_date_str, "%Y-%m-%d")
            today = datetime.today().replace(hour=0, minute=0, second=0, microsecond=0)
            
            if deadline_filter == "Vencidas":
                return due_date < today
            
            if deadline_filter == "Vence Hoy":
                return due_date == today
            
            if deadline_filter == "Próximos 7 Días":
                seven_days_out = today + timedelta(days=7)
                return today < due_date <= seven_days_out
            
            if deadline_filter == "Próximo Mes":
                thirty_days_out = today + timedelta(days=30)
                return today < due_date <= thirty_days_out
            
            return True
        except ValueError:
            return True # Si la fecha es inválida, mejor mostrarla
    
    def cargar_tareas(self):
        self.tree.delete(*self.tree.get_children())
        self.visual_to_real_id.clear()

        vid = 1
        pendientes = 0
        today = datetime.today().strftime("%Y-%m-%d")
        
        for r in self.db.get_all_tasks():
            real_id, title, desc, _, due, pr, st, cat = r

            # Filtrado por Estado
            if self.filter_status.get() != "Todos" and st != self.filter_status.get():
                continue
            
            # FILTRADO DE PLAZO DINÁMICO
            if not self._apply_deadline_filter(r):
                continue
            
            if st == "Pendiente":
                pendientes += 1

            self.visual_to_real_id[vid] = real_id
            
            # Aplicar tag de color según el estado
            tag_name = st.lower().replace(" ", "_")
            
            # Resaltar si está vencida (independiente del color de estado si no está completada)
            if st != "Completada" and due and due < today:
                tag_name = "vencida"

            self.tree.insert("", "end", tags=(tag_name,),
                 values=(vid, title, desc or "", due or "", pr or "", st or "", cat or ""))
            vid += 1
        
        self.count_label.config(text=f"Total: {vid - 1} | Pendientes: {pendientes}")

    def actualizar_info(self, _):
        """Visualización de la descripción completa en el info_label."""
        sel = self.tree.selection()
        if sel:
            v = self.tree.item(sel[0])["values"]
            
            # Obtener descripción completa de la base de datos (se usa el real_id)
            rid = self.visual_to_real_id.get(v[0])
            full_task = self.db.get_task(rid)
            
            # La descripción es el índice 2 de full_task
            descripcion_completa = full_task[2] if full_task and full_task[2] else "Sin descripción."
            
            # Limitar la descripción para que quepa en el label
            max_len = 100
            if len(descripcion_completa) > max_len:
                descripcion_preview = descripcion_completa[:max_len] + "..."
            else:
                descripcion_preview = descripcion_completa

            self.info_label.config(
                text=f"Título: {v[1]} | Prioridad: {v[4]} | Estado: {v[5]} | Categoría: {v[6]}\nDescripción: {descripcion_preview}"
            )

    def get_real_id(self):
        sel = self.tree.selection()
        if not sel:
            return None
        vid = self.tree.item(sel[0])["values"][0]
        return self.visual_to_real_id.get(vid)

    def agregar(self):
        TaskForm(self, self.db, self.cargar_tareas)

    def editar(self):
        rid = self.get_real_id()
        if rid:
            TaskForm(self, self.db, self.cargar_tareas, rid)

    def eliminar(self):
        rid = self.get_real_id()
        if rid and messagebox.askyesno("Confirmar", "¿Eliminar la tarea seleccionada?", parent=self):
            self.db.delete_task(rid)
            self.cargar_tareas()

    def ver_stats(self):
        tasks = self.db.get_all_tasks()
        total = len(tasks)
        comp = sum(1 for t in tasks if t[6] == "Completada") 
        percent = (comp / total * 100) if total else 0

        messagebox.showinfo(
            "Estadísticas",
            f"Total de tareas: {total}\nCompletadas: {comp}\nProgreso: {percent:.1f}%",
            parent=self
        )

    # Recordatorios de vencimiento
    def verificar_recordatorios(self):
        hoy = datetime.today().strftime("%Y-%m-%d")
        manana = (datetime.today() + timedelta(days=1)).strftime("%Y-%m-%d")
        
        alertas_hoy = []
        alertas_manana = []
        
        for r in self.db.get_all_tasks():
            if r[6] != "Completada" and r[4]: 
                if r[4] == hoy:
                    alertas_hoy.append(r[1])
                elif r[4] == manana:
                    alertas_manana.append(r[1])
        
        mensaje = ""
        if alertas_hoy:
            mensaje += "⚠️ **¡Tareas Vencen Hoy!** ⚠️\n"
            mensaje += "\n".join(f"- {t}" for t in alertas_hoy)
            mensaje += "\n\n"
        
        if alertas_manana:
            mensaje += "🔔 **Tareas Vencen Mañana** 🔔\n"
            mensaje += "\n".join(f"- {t}" for t in alertas_manana)
            mensaje += "\n\n"

        if mensaje:
            messagebox.showwarning("Recordatorio de Vencimiento", mensaje, parent=self)
            

# ================= BIENVENIDA =================
def mostrar_bienvenida():
    def ingresar():
        splash.destroy()
        app = TaskManagerApp()
        app.mainloop()

    splash = tk.Tk()
    splash.title("Bienvenido")
    splash.geometry("500x300")
    
    x = (splash.winfo_screenwidth() - 500) // 2
    y = (splash.winfo_screenheight() - 300) // 2
    splash.geometry(f"500x300+{x}+{y}")
    splash.resizable(False, False)

    # Configuración de estilo
    style = ttk.Style(splash)
    splash_bg = "#8A2BE2" 
    
    style.configure("Splash.TFrame", background=splash_bg) 
    style.configure("Splash.TLabel", background=splash_bg, foreground="white") 
    
    splash_frame = ttk.Frame(splash, style="Splash.TFrame")
    splash_frame.pack(fill="both", expand=True)

    ttk.Label(
        splash_frame,
        text="Task Manager - Gestor de Tareas Personales",
        style="Splash.TLabel",
        font=("Helvetica", 16, "bold")
    ).pack(pady=(30, 10))

    ttk.Label(
        splash_frame,
        text="¡Hola! Bienvenido al Gestor de Tareas\n\nPequeños avances diarios generan grandes resultados.",
        style="Splash.TLabel",
        font=("Helvetica", 14),
        wraplength=400,
        justify="center"
    ).pack(expand=True)

    ttk.Button(splash_frame, text="Ingresar", command=ingresar).pack(pady=20)
    splash.mainloop()

# ================= MAIN =================
if __name__ == "__main__":
    mostrar_bienvenida()
