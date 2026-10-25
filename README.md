import base64
import calendar
from datetime import date, datetime, time
import json
import os
import sqlite3
import pandas as pd
import streamlit as st

# Configuración de la página
st.set_page_config(
    page_title="Master Global de Horarios - RIO",
    layout="wide",
)

DB_NAME = "horarios_rio.db"

DIAS_SEMANA = ["Lun", "Mar", "Mié", "Jue", "Vie", "Sáb", "Dom"]

MESES = {
    1: "Enero",
    2: "Febrero",
    3: "Marzo",
    4: "Abril",
    5: "Mayo",
    6: "Junio",
    7: "Julio",
    8: "Agosto",
    9: "Septiembre",
    10: "Octubre",
    11: "Noviembre",
    12: "Diciembre",
}

TURNOS_RIO = {
    "A": {
        "nombre": "Apertura",
        "bg": "#97C900",
        "text": "#0c2e12",
        "desc": "08:00 AM - 04:00 PM",
        "inicio": time(8, 0),
        "fin": time(16, 0),
        "activo": True,
    },
    "I": {
        "nombre": "Intermedio",
        "bg": "#126332",
        "text": "#ffffff",
        "desc": "11:00 AM - 07:00 PM",
        "inicio": time(11, 0),
        "fin": time(19, 0),
        "activo": True,
    },
    "C": {
        "nombre": "Cierre",
        "bg": "#0095D9",
        "text": "#ffffff",
        "desc": "02:00 PM - 11:00 PM",
        "inicio": time(14, 0),
        "fin": time(23, 0),
        "activo": True,
    },
    "R": {
        "nombre": "Redoble",
        "bg": "#F58220",
        "text": "#ffffff",
        "desc": "08:00 AM - 11:00 PM",
        "inicio": time(8, 0),
        "fin": time(23, 0),
        "activo": True,
    },
    "V": {
        "nombre": "Vacaciones",
        "bg": "#D63883",
        "text": "#ffffff",
        "desc": "Período de Vacaciones",
        "inicio": None,
        "fin": None,
        "activo": False,
    },
    "L": {
        "nombre": "Libre",
        "bg": "#E9ECEF",
        "text": "#6C757D",
        "desc": "Día Libre / Descanso",
        "inicio": None,
        "fin": None,
        "activo": False,
    },
}


# ---------------------------------------------------------
# LOGO RIO VECTORIAL Y MINIATURA PARA SIDEBAR
# ---------------------------------------------------------
def obtener_logo_html(ancho=105, alto=78):
  for nombre_archivo in [
      "logo_rio.png",
      "Captura de pantalla 2026-10-05 144132.png",
  ]:
    if os.path.exists(nombre_archivo):
      with open(nombre_archivo, "rb") as f:
        b64 = base64.b64encode(f.read()).decode("utf-8")
      return f'<img src="data:image/png;base64,{b64}" class="rio-logo-img" style="width:{ancho}px;height:auto;" />'

  svg_rio = f"""<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 380 280" width="{ancho}" height="{alto}">
    <g fill="none" stroke="#C6DEC9" stroke-linecap="round" stroke-linejoin="round">
      <path d="M 140 42 L 106 42 L 112 100 L 214 108 L 248 20 C 252 12 260 12 268 15 L 282 21" stroke-width="12"/>
    </g>
    <g fill="#C6DEC9">
      <polygon points="126,52 143,52 144,98 128,97"/>
      <polygon points="150,32 169,32 169,100 150,99"/>
      <polygon points="179,21 198,21 191,102 173,101"/>
      <polygon points="212,11 231,11 209,104 195,103"/>
      <circle cx="137" cy="122" r="12"/>
      <ellipse cx="199" cy="126" rx="20" ry="14"/>
      <path d="M 30 260 L 30 132 L 80 132 L 80 260 Z"/>
      <path d="M 80 132 L 105 132 C 142 132 155 152 155 178 C 155 204 140 222 102 222 L 88 222 L 88 210 L 102 210 C 128 210 141 198 141 178 C 141 158 128 145 102 145 L 80 145 Z"/>
      <path d="M 103 218 L 114 218 L 130 245 C 135 253 142 254 152 254 L 152 262 C 136 263 125 259 117 246 Z"/>
      <rect x="175" y="158" width="50" height="102"/>
      <path fill-rule="evenodd" d="M 298 154 C 330 154 354 177 354 208 C 354 239 330 262 298 262 C 266 262 242 239 242 208 C 242 177 266 154 298 154 Z M 298 164 L 298 252 C 324 252 341 233 341 208 C 341 183 324 164 298 164 Z"/>
      <text x="350" y="261" font-family="Arial, sans-serif" font-size="24" font-weight="bold" fill="#C6DEC9">®</text>
    </g>
  </svg>"""
  b64_svg = base64.b64encode(svg_rio.encode("utf-8")).decode("utf-8")
  return f'<img src="data:image/svg+xml;base64,{b64_svg}" style="width:{ancho}px;height:auto;display:block;" />'


# ---------------------------------------------------------
# ESTILOS CSS PROFESIONALES Y MATRIZ EXPANDIBLE
# ---------------------------------------------------------
st.markdown(
    """<style>
/* Hace transparente el header nativo de Streamlit para que NUNCA corte las esquinas del banner */
header[data-testid="stHeader"] {
    background-color: transparent !important;
}

.block-container {
    padding-top: 3.2rem !important;
    padding-bottom: 1rem !important;
}

section[data-testid="stSidebar"] {
    background-color: #f8fafc;
    border-right: 1px solid #e2e8f0;
}

section[data-testid="stSidebar"] .stTextInput input, 
section[data-testid="stSidebar"] .stNumberInput input,
section[data-testid="stSidebar"] .stSelectbox div[data-baseweb="select"] > div {
    background-color: #ffffff !important;
    color: #0f172a !important;
    border-radius: 8px !important;
    font-weight: 600 !important;
}

section[data-testid="stSidebar"] div.stForm {
    background-color: #ffffff;
    padding: 14px;
    border-radius: 12px;
    border: 1px solid #cbd5e1;
    box-shadow: 0 4px 12px rgba(0,0,0,0.04);
}

section[data-testid="stSidebar"] .stButton > button,
section[data-testid="stSidebar"] .stFormSubmitButton > button {
    background-color: #126332 !important;
    color: #ffffff !important;
    font-weight: 700 !important;
    border-radius: 8px !important;
    border: none !important;
    width: 100% !important;
}
section[data-testid="stSidebar"] .stButton > button:hover,
section[data-testid="stSidebar"] .stFormSubmitButton > button:hover {
    background-color: #0e4e27 !important;
}

/* BANNER CON las 4 ESQUINAS PERFECTAMENTE REDONDEADAS */
.rio-banner {
    background-color: #126332;
    padding: 20px 30px;
    border-radius: 20px !important;
    overflow: hidden;
    color: white;
    display: flex;
    justify-content: space-between;
    align-items: center;
    box-shadow: 0 8px 22px rgba(18, 99, 50, 0.22);
    margin-top: 4px;
    margin-bottom: 12px;
    border-bottom: 6px solid #97C900;
}
.rio-banner-left { display: flex; align-items: center; gap: 24px; }
.rio-logo-img { height: 72px; width: auto; object-fit: contain; display: block; }
.rio-divider { width: 2px; height: 54px; background-color: rgba(198, 222, 201, 0.4); border-radius: 2px; }
.rio-title { font-size: 32px; font-weight: 800; margin: 0; letter-spacing: -0.5px; color: #ffffff; line-height: 1.1; }
.rio-subtitle { font-size: 16px; font-weight: 600; margin-top: 4px; color: #C6DEC9; }
.rio-badge-branch { background: rgba(255, 255, 255, 0.12); padding: 10px 20px; border-radius: 14px; text-align: right; border: 1px solid rgba(198, 222, 201, 0.3); }
.color-bar { display: flex; height: 6px; border-radius: 999px; overflow: hidden; margin-bottom: 18px; }

.kpi-card { background: #ffffff; border-radius: 14px; padding: 16px 18px; box-shadow: 0 4px 12px rgba(0,0,0,0.05); border: 1px solid #f0f2f5; min-height: 118px; display: flex; flex-direction: column; justify-content: space-between; }
.kpi-title { font-size: 12px; font-weight: 700; color: #475569; text-transform: uppercase; letter-spacing: 0.5px; }
.kpi-value { font-size: 32px; font-weight: 800; color: #0f172a; margin-top: 2px; }

.live-dot { display: inline-block; width: 10px; height: 10px; background-color: #22c55e; border-radius: 50%; margin-right: 6px; box-shadow: 0 0 10px #22c55e; }
.off-dot { display: inline-block; width: 10px; height: 10px; background-color: #94a3b8; border-radius: 50%; margin-right: 6px; }
.guardia-chip { display: flex; align-items: center; justify-content: space-between; background: #f0fdf4; border: 1px solid #bbf7d0; padding: 6px 12px; border-radius: 8px; font-size: 14px; font-weight: 700; color: #14532d; margin-top: 6px; }
.guardia-tag { padding: 3px 8px; border-radius: 6px; font-size: 12px; font-weight: 800; }

.legend-container { display: flex; gap: 12px; flex-wrap: wrap; margin: 14px 0 18px 0; }
.legend-pill { display: inline-flex; align-items: center; gap: 8px; padding: 7px 15px; border-radius: 999px; font-size: 13px; font-weight: 700; box-shadow: 0 2px 6px rgba(0,0,0,0.06); }

.tech-card { background: #ffffff; border-radius: 12px; padding: 18px; border-left: 6px solid #126332; box-shadow: 0 3px 10px rgba(0,0,0,0.06); margin-bottom: 14px; }
.tech-name { font-size: 18px; font-weight: 700; color: #1e293b; margin-bottom: 4px; }
.tech-phone { font-size: 14px; color: #64748b; margin-bottom: 10px; font-weight: 600; }

/* -------------------------------------------------------------
   ESTILOS DE LA MATRIZ MENSUAL EXPANDIDA (OCUPA ESPACIO VACÍO)
   ------------------------------------------------------------- */
.matrix-wrapper {
    background: #ffffff;
    border-radius: 16px;
    border: 1px solid #cbd5e1;
    box-shadow: 0 10px 25px rgba(0,0,0,0.06);
    overflow-x: auto;
    height: calc(100vh - 435px);
    min-height: 420px;
    display: flex;
    flex-direction: column;
}

.rio-matrix-table {
    width: 100%;
    height: 100%;
    border-collapse: separate;
    border-spacing: 5px;
    table-layout: fixed;
    padding: 8px;
}

.rio-matrix-table thead tr {
    height: 56px;
}

.rio-matrix-table th {
    background: #f8fafc;
    color: #334155;
    font-weight: 800;
    text-align: center;
    border: 1px solid #e2e8f0;
    border-radius: 8px;
    padding: 4px 2px;
    vertical-align: middle;
}

.rio-matrix-table th.col-tech-header {
    width: 230px;
    text-align: left;
    padding-left: 16px;
    font-size: 15px;
    background: #f1f5f9;
    color: #0f172a;
}

.rio-matrix-table th.col-total-header {
    width: 150px;
    font-size: 14px;
    background: #f1f5f9;
    color: #0f172a;
}

.rio-matrix-table th.th-hoy {
    background: #dcfce7;
    border: 2px solid #126332;
    color: #126332;
}

.th-day-num {
    font-size: 15px;
    font-weight: 900;
    line-height: 1.1;
}

.th-day-dow {
    font-size: 10px;
    text-transform: uppercase;
    color: #64748b;
    font-weight: 700;
}

.rio-matrix-table tbody tr.tech-row {
    height: 1%;
}

.rio-matrix-table td {
    border-radius: 10px;
    text-align: center;
    vertical-align: middle;
    transition: transform 0.12s ease, box-shadow 0.12s ease;
}

.rio-matrix-table td.shift-cell:hover {
    transform: scale(1.06);
    z-index: 5;
    box-shadow: 0 6px 15px rgba(0,0,0,0.18);
}

.td-tech-info {
    background: #ffffff;
    border: 1px solid #e2e8f0;
    text-align: left !important;
    padding: 12px 16px !important;
}

.td-tech-name {
    font-size: 18px;
    font-weight: 800;
    color: #0f172a;
    line-height: 1.2;
}

.td-tech-phone {
    font-size: 13px;
    font-weight: 600;
    color: #64748b;
    margin-top: 4px;
}

.shift-code {
    font-size: 24px;
    font-weight: 900;
    letter-spacing: -0.5px;
    line-height: 1;
}

.td-progress-box {
    background: #ffffff;
    border: 1px solid #e2e8f0;
    padding: 10px 14px !important;
}

.prog-val {
    font-size: 17px;
    font-weight: 800;
    color: #0f172a;
    margin-bottom: 6px;
}

.prog-bar-bg {
    width: 100%;
    height: 10px;
    background: #e2e8f0;
    border-radius: 999px;
    overflow: hidden;
}

.prog-bar-fill {
    height: 100%;
    background: linear-gradient(90deg, #126332, #97C900);
    border-radius: 999px;
}

.rio-matrix-table tr.total-row {
    height: 62px;
}

.td-total-label {
    background: #0f172a !important;
    color: #ffffff !important;
    font-size: 14px;
    font-weight: 800;
    text-align: left !important;
    padding-left: 16px !important;
    letter-spacing: 0.5px;
}

.td-total-cell {
    background: #e6f4ea;
    color: #126332;
    font-size: 19px;
    font-weight: 900;
    border: 1px solid #bbf7d0;
}

.td-total-zero {
    background: #f1f5f9;
    color: #94a3b8;
    font-size: 16px;
    font-weight: 700;
}
</style>""",
    unsafe_allow_html=True,
)


# ---------------------------------------------------------
# 1. BASE DE DATOS SQLITE
# ---------------------------------------------------------
def init_db():
  with sqlite3.connect(DB_NAME) as conn:
    cursor = conn.cursor()
    cursor.execute(
        "SELECT name FROM sqlite_master WHERE type='table' AND name='turnos_mensuales'"
    )
    col_check = cursor.execute(
        "PRAGMA table_info(turnos_mensuales)"
    ).fetchall()
    tiene_tecnico_id = any(col[1] == "tecnico_id" for col in col_check)

    if col_check and not tiene_tecnico_id:
      cursor.execute("DROP TABLE IF EXISTS turnos_mensuales")
      cursor.execute("DROP TABLE IF EXISTS tecnicos")
      cursor.execute("DROP TABLE IF EXISTS sucursales")

    cursor.execute("""
            CREATE TABLE IF NOT EXISTS sucursales (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nombre TEXT UNIQUE NOT NULL,
                horario TEXT DEFAULT 'Apertura 8:00am - Cierre 12:00pm'
            )
        """)

    cursor.execute("""
            CREATE TABLE IF NOT EXISTS tecnicos (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                sucursal_id INTEGER NOT NULL,
                nombre_completo TEXT NOT NULL,
                FOREIGN KEY (sucursal_id) REFERENCES sucursales(id) ON DELETE CASCADE
            )
        """)

    cursor.execute("""
            CREATE TABLE IF NOT EXISTS turnos_mensuales (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                tecnico_id INTEGER NOT NULL,
                anio INTEGER NOT NULL,
                mes INTEGER NOT NULL,
                turnos_json TEXT NOT NULL,
                FOREIGN KEY (tecnico_id) REFERENCES tecnicos(id) ON DELETE CASCADE
            )
        """)

    cursor.execute("SELECT COUNT(*) FROM sucursales")
    if cursor.fetchone()[0] == 0:
      cursor.execute(
          "INSERT INTO sucursales (nombre, horario) VALUES (?, ?)",
          ("Sucursal Lomas del Sol", "Apertura 8:00am - Cierre 12:00pm"),
      )
      suc_id = cursor.lastrowid
      cursor.execute(
          "INSERT INTO tecnicos (sucursal_id, nombre_completo) VALUES (?, ?)",
          (suc_id, "Luis Chavarri / 04142283092"),
      )
      t1_id = cursor.lastrowid
      cursor.execute(
          "INSERT INTO turnos_mensuales (tecnico_id, anio, mes, turnos_json)"
          " VALUES (?, ?, ?, ?)",
          (t1_id, 2026, 10, json.dumps({})),
      )
      conn.commit()


init_db()


def get_sucursales_df():
  with sqlite3.connect(DB_NAME) as conn:
    return pd.read_sql_query("SELECT * FROM sucursales", conn)


def agregar_sucursal(nombre, horario):
  with sqlite3.connect(DB_NAME) as conn:
    conn.execute(
        "INSERT INTO sucursales (nombre, horario) VALUES (?, ?)",
        (nombre.strip(), horario.strip()),
    )
    conn.commit()


def agregar_tecnico(sucursal_id, nombre_completo, anio, mes):
  with sqlite3.connect(DB_NAME) as conn:
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO tecnicos (sucursal_id, nombre_completo) VALUES (?, ?)",
        (sucursal_id, nombre_completo.strip()),
    )
    tec_id = cursor.lastrowid
    cursor.execute(
        "INSERT INTO turnos_mensuales (tecnico_id, anio, mes, turnos_json)"
        " VALUES (?, ?, ?, ?)",
        (tec_id, anio, mes, "{}"),
    )
    conn.commit()


def get_datos_mes_sucursal(sucursal_id, anio, mes):
  with sqlite3.connect(DB_NAME) as conn:
    query = """
            SELECT T.id as tecnico_id, S.nombre as sucursal, S.horario as horario_sucursal, 
            T.nombre_completo as empleado, M.id as turno_id, M.turnos_json
            FROM tecnicos T
            JOIN sucursales S ON T.sucursal_id = S.id
            LEFT JOIN turnos_mensuales M ON T.id = M.tecnico_id AND M.anio = ? AND M.mes = ?
            WHERE S.id = ?
        """
    return pd.read_sql_query(query, conn, params=(anio, mes, sucursal_id))


def get_datos_todos_global(anio, mes):
  with sqlite3.connect(DB_NAME) as conn:
    query = """
            SELECT T.id as tecnico_id, S.nombre as sucursal, S.horario as horario_sucursal, 
            T.nombre_completo as empleado, M.id as turno_id, M.turnos_json
            FROM tecnicos T
            JOIN sucursales S ON T.sucursal_id = S.id
            LEFT JOIN turnos_mensuales M ON T.id = M.tecnico_id AND M.anio = ? AND M.mes = ?
        """
    return pd.read_sql_query(query, conn, params=(anio, mes))


def actualizar_turnos_lote(actualizaciones):
  with sqlite3.connect(DB_NAME) as conn:
    cursor = conn.cursor()
    for tecnico_id, nombre_emp, turnos_dict, anio, mes in actualizaciones:
      cursor.execute(
          "UPDATE tecnicos SET nombre_completo = ? WHERE id = ?",
          (nombre_emp.strip(), tecnico_id),
      )
      cursor.execute(
          "SELECT id FROM turnos_mensuales WHERE tecnico_id = ? AND anio = ? AND"
          " mes = ?",
          (tecnico_id, anio, mes),
      )
      row = cursor.fetchone()
      if row:
        cursor.execute(
            "UPDATE turnos_mensuales SET turnos_json = ? WHERE id = ?",
            (json.dumps(turnos_dict), row[0]),
        )
      else:
        cursor.execute(
            "INSERT INTO turnos_mensuales (tecnico_id, anio, mes, turnos_json)"
            " VALUES (?, ?, ?, ?)",
            (tecnico_id, anio, mes, json.dumps(turnos_dict)),
        )
    conn.commit()


def eliminar_tecnico(tecnico_id):
  with sqlite3.connect(DB_NAME) as conn:
    conn.execute("DELETE FROM tecnicos WHERE id = ?", (tecnico_id,))
    conn.commit()


def renombrar_sucursal(sucursal_id, nuevo_nombre, nuevo_horario):
  with sqlite3.connect(DB_NAME) as conn:
    conn.execute(
        "UPDATE sucursales SET nombre = ?, horario = ? WHERE id = ?",
        (nuevo_nombre.strip(), nuevo_horario.strip(), sucursal_id),
    )
    conn.commit()


def eliminar_sucursal(sucursal_id):
  with sqlite3.connect(DB_NAME) as conn:
    conn.execute("DELETE FROM sucursales WHERE id = ?", (sucursal_id,))
    conn.commit()


# ---------------------------------------------------------
# 2. MENÚ LATERAL CON LOGO RIO Y SELECTOR DE MES VISIBLE
# ---------------------------------------------------------
ahora = datetime.now()

logo_sidebar = obtener_logo_html(ancho=85, alto=62)
st.sidebar.markdown(
    f"""<div style="display: flex; align-items: center; gap: 12px; margin-bottom: 8px;">
{logo_sidebar}
<div>
<div style="font-size: 18px; font-weight: 800; color: #126332; line-height: 1.1;">Master RIO</div>
<div style="font-size: 11px; color: #64748b; font-weight: 600;">Control de Horarios</div>
</div>
</div>""",
    unsafe_allow_html=True,
)
st.sidebar.caption(
    f"🕒 **{ahora.strftime('%I:%M %p')}** ({ahora.strftime('%d/%m/%Y')})"
)

st.sidebar.markdown("##### 📅 Período de Planificación")
col_mes_top, col_anio_top = st.sidebar.columns([1.4, 1])
with col_mes_top:
  selected_mes = st.selectbox(
      "Mes de Horario:",
      options=list(MESES.keys()),
      format_func=lambda x: MESES[x],
      index=ahora.month - 1,
  )
with col_anio_top:
  selected_anio = st.number_input(
      "Año:", min_value=2024, max_value=2030, value=ahora.year
  )

df_sucursales = get_sucursales_df()

# SECCIÓN 1: SELECCIÓN Y GESTIÓN DE SEDES
with st.sidebar.expander("📍 Seleccionar y Administrar Sedes", expanded=True):
  opciones_radio = ["🏢 Vista Global (Todas las Sucursales)"] + (
      df_sucursales["nombre"].tolist() if not df_sucursales.empty else []
  )
  selected_sucursal_nombre = st.radio(
      "Sede activa:", opciones_radio, label_visibility="collapsed"
  )

  st.markdown("---")
  st.markdown("##### ➕ Crear Nueva Sucursal")
  with st.form("form_nueva_suc", clear_on_submit=True):
    nueva_suc_nombre = st.text_input("Nombre de la Sede")
    nueva_suc_horario = st.text_input(
        "Horario de Tienda", value="Apertura 8:00am - Cierre 11:00pm"
    )
    if st.form_submit_button("Guardar Sucursal"):
      if nueva_suc_nombre.strip():
        try:
          agregar_sucursal(nueva_suc_nombre, nueva_suc_horario)
          st.success("¡Sucursal creada!")
          st.rerun()
        except Exception:
          st.error("Esa sucursal ya existe.")
      else:
        st.warning("Escribe un nombre.")

# SECCIÓN 2: CONFIGURACIÓN DE RANGOS HORARIOS
with st.sidebar.expander("⚙️ Configurar Horas de Turnos", expanded=False):
  st.caption("Rangos de guardia en tiempo real:")
  c_a1, c_a2 = st.columns(2)
  TURNOS_RIO["A"]["inicio"] = c_a1.time_input("Apertura Inicio", time(8, 0))
  TURNOS_RIO["A"]["fin"] = c_a2.time_input("Apertura Fin", time(16, 0))

  c_c1, c_c2 = st.columns(2)
  TURNOS_RIO["C"]["inicio"] = c_c1.time_input("Cierre Inicio", time(14, 0))
  TURNOS_RIO["C"]["fin"] = c_c2.time_input("Cierre Fin", time(23, 0))

  dia_evaluar = st.number_input(
      "Día a evaluar como 'Hoy':",
      min_value=1,
      max_value=31,
      value=ahora.day,
  )

num_dias = calendar.monthrange(selected_anio, selected_mes)[1]
nombre_mes = MESES[selected_mes]

columnas_dias = {}
for d in range(1, num_dias + 1):
  dow = DIAS_SEMANA[date(selected_anio, selected_mes, d).weekday()]
  columnas_dias[str(d)] = f"{d:02d} {dow}"

# ---------------------------------------------------------
# MODO 1: VISTA GLOBAL (TODAS LAS SUCURSALES)
# ---------------------------------------------------------
if selected_sucursal_nombre == "🏢 Vista Global (Todas las Sucursales)":
  logo_html = obtener_logo_html()
  st.markdown(
      f"""<div class="rio-banner">
<div class="rio-banner-left">
{logo_html}
<div class="rio-divider"></div>
<div>
<div class="rio-title">Master Global de Horarios</div>
<div class="rio-subtitle">Consolidado de todas las sucursales de la red</div>
</div>
</div>
<div class="rio-badge-branch">
<div style="font-size:24px;font-weight:800;">{nombre_mes} {selected_anio}</div>
<div style="font-size:13px;color:#C6DEC9;font-weight:600;">Panel Gerencial</div>
</div>
</div>
<div class="color-bar">
<div style="flex:1;background:#D63883;"></div>
<div style="flex:1;background:#97C900;"></div>
<div style="flex:1;background:#F58220;"></div>
<div style="flex:1;background:#0095D9;"></div>
<div style="flex:1;background:#126332;"></div>
</div>""",
      unsafe_allow_html=True,
  )

  df_todos = get_datos_todos_global(selected_anio, selected_mes)
  if df_todos.empty:
    st.info(
        "No hay registros cargados en ninguna sucursal para este período."
    )
  else:
    st.markdown("### ⚡ Quiénes están de guardia AHORA MISMO por Sucursal")
    sucursales_activas = df_todos["sucursal"].unique()
    hora_actual = ahora.time()
    dia_num_eval = min(dia_evaluar, num_dias)
    dia_str = str(dia_num_eval)

    cols_global = st.columns(min(len(sucursales_activas), 3) or 1)
    for i, suc in enumerate(sucursales_activas):
      df_s = df_todos[df_todos["sucursal"] == suc]
      activos_suc = []

      for _, row in df_s.iterrows():
        t_json = row["turnos_json"]
        t_dict = json.loads(t_json) if t_json else {}
        turno_hoy = str(t_dict.get(dia_str, "")).upper().strip()
        if turno_hoy in ["A", "I", "C", "R"]:
          info_t = TURNOS_RIO[turno_hoy]
          nombre_corto = row["empleado"].split("/")[0].strip()
          if info_t["inicio"] <= hora_actual <= info_t["fin"]:
            activos_suc.append((nombre_corto, turno_hoy, info_t))

      chips_s = ""
      if activos_suc:
        for nom, cod, inf in activos_suc:
          chips_s += (
              f'<div class="guardia-chip"><span>{nom}</span><span'
              ' class="guardia-tag"'
              f' style="background:{inf["bg"]};color:{inf["text"]};">{cod} •'
              f' {inf["nombre"]}</span></div>'
          )
      else:
        chips_s = '<div style="font-size:13px;color:#64748b;margin-top:8px;">Ningún técnico activo a esta hora</div>'

      with cols_global[i % len(cols_global)]:
        st.markdown(
            f"""<div class="kpi-card" style="border-top: 5px solid #126332; margin-bottom: 16px;">
<div class="kpi-title"><span class="live-dot"></span>{suc}</div>
<div style="margin-top:6px;">{chips_s}</div>
<div style="font-size:11px;color:#94a3b8;margin-top:12px;">Total personal en sede: {len(df_s)}</div>
</div>""",
            unsafe_allow_html=True,
        )

    st.markdown("---")
    st.markdown("### 📂 Base de Datos Consolidada General")
    st.dataframe(
        df_todos[["sucursal", "horario_sucursal", "empleado"]],
        use_container_width=True,
        hide_index=True,
    )

# ---------------------------------------------------------
# MODO 2: SUCURSAL ESPECÍFICA SELECCIONADA
# ---------------------------------------------------------
else:
  suc_row = df_sucursales[
      df_sucursales["nombre"] == selected_sucursal_nombre
  ].iloc[0]
  sucursal_id = int(suc_row["id"])
  horario_texto = suc_row["horario"]

  df_mes = get_datos_mes_sucursal(sucursal_id, selected_anio, selected_mes)

  lista_tecnicos = (
      ["Todos"] + df_mes["empleado"].tolist()
      if not df_mes.empty
      else ["Todos"]
  )
  filtro_tecnico = st.sidebar.selectbox("👤 Filtrar Técnico:", lista_tecnicos)

  st.sidebar.markdown("---")

  # SECCIÓN 3: REGISTRO Y ADMINISTRACIÓN DE PERSONAL / SEDE
  with st.sidebar.expander(
      f"👥 Gestión de Sede ({selected_sucursal_nombre})", expanded=True
  ):
    st.markdown("##### ➕ Agregar Técnico")
    with st.form("form_nuevo_emp", clear_on_submit=True):
      emp_nombre = st.text_input(
          "Nombre / Teléfono", placeholder="Ej. Juan Perez / 0412..."
      )
      if st.form_submit_button("Registrar en Sede"):
        if emp_nombre.strip():
          agregar_tecnico(
              sucursal_id, emp_nombre, selected_anio, selected_mes
          )
          st.success("¡Técnico agregado!")
          st.rerun()
        else:
          st.error("Escribe un nombre.")

    st.markdown("---")
    st.markdown("##### ⚙️ Modificar Sede")
    with st.form("form_edit_suc"):
      nuevo_nombre_suc = st.text_input("Nombre Sede", value=selected_sucursal_nombre)
      nuevo_horario_suc = st.text_input("Horario Tienda", value=horario_texto)
      if st.form_submit_button("Actualizar Sede"):
        if nuevo_nombre_suc.strip():
          renombrar_sucursal(sucursal_id, nuevo_nombre_suc, nuevo_horario_suc)
          st.success("¡Actualizado!")
          st.rerun()

    with st.popover("🗑️ Eliminar Sede", use_container_width=True):
      st.warning("⚠️ Borrará la sede y sus técnicos.")
      conf_txt = st.text_input(f"Escribe '{selected_sucursal_nombre}' para confirmar:")
      if st.button("Borrar definitivamente", type="secondary"):
        if conf_txt.strip() == selected_sucursal_nombre.strip():
          eliminar_sucursal(sucursal_id)
          st.success("Eliminado.")
          st.rerun()
        else:
          st.error("No coincide.")

  # Cabecera Sucursal
  logo_html = obtener_logo_html()
  st.markdown(
      f"""<div class="rio-banner">
<div class="rio-banner-left">
{logo_html}
<div class="rio-divider"></div>
<div>
<div class="rio-title">{selected_sucursal_nombre}</div>
<div class="rio-subtitle">{horario_texto}</div>
</div>
</div>
<div class="rio-badge-branch">
<div style="font-size:24px;font-weight:800;">{nombre_mes} {selected_anio}</div>
<div style="font-size:13px;color:#C6DEC9;font-weight:600;">Planificación de Soporte Técnico</div>
</div>
</div>
<div class="color-bar">
<div style="flex:1;background:#D63883;"></div>
<div style="flex:1;background:#97C900;"></div>
<div style="flex:1;background:#F58220;"></div>
<div style="flex:1;background:#0095D9;"></div>
<div style="flex:1;background:#126332;"></div>
</div>""",
      unsafe_allow_html=True,
  )

  if df_mes.empty:
    st.info(
        "No hay técnicos registrados en esta sucursal. Usa la sección 'Gestión"
        " de Sede' en la barra lateral para agregar el primero."
    )
  else:
    df_vista = (
        df_mes[df_mes["empleado"] == filtro_tecnico]
        if filtro_tecnico != "Todos"
        else df_mes.copy()
    )

    total_apertura = total_cierre = total_redoble = total_vacaciones = 0
    en_guardia_ahora = []
    programados_hoy = []
    hora_actual = ahora.time()
    dia_num_eval = min(dia_evaluar, num_dias)
    dia_str = str(dia_num_eval)
    etiqueta_dia_hoy = columnas_dias[dia_str]

    for _, row in df_vista.iterrows():
      t_json = row["turnos_json"]
      t_dict = json.loads(t_json) if t_json else {}
      for val in t_dict.values():
        v = str(val).upper().strip()
        if v == "A":
          total_apertura += 1
        elif v == "C":
          total_cierre += 1
        elif v == "R":
          total_redoble += 1
        elif v == "V":
          total_vacaciones += 1

      turno_hoy = str(t_dict.get(dia_str, "")).upper().strip()
      if turno_hoy in ["A", "I", "C", "R"]:
        info_t = TURNOS_RIO[turno_hoy]
        nombre_corto = row["empleado"].split("/")[0].strip()
        programados_hoy.append((nombre_corto, turno_hoy, info_t))
        if info_t["inicio"] <= hora_actual <= info_t["fin"]:
          en_guardia_ahora.append((nombre_corto, turno_hoy, info_t))

    if en_guardia_ahora:
      chips_html = ""
      for nom, cod, inf in en_guardia_ahora:
        chips_html += (
            f'<div class="guardia-chip"><span>{nom}</span><span'
            ' class="guardia-tag"'
            f' style="background:{inf["bg"]};color:{inf["text"]};">{cod} •'
            f' {inf["nombre"]}</span></div>'
        )
      html_guardia_card = (
          '<div class="kpi-card" style="border-top: 4px solid #126332;"><div '
          f'class="kpi-title"><span class="live-dot"></span>EN GUARDIA AHORA '
          f'({etiqueta_dia_hoy} • {ahora.strftime("%I:%M %p")})</div><div>{chips_html}</div></div>'
      )
    else:
      if programados_hoy:
        resumen_prog = ", ".join([f"{n} ({c})" for n, c, _ in programados_hoy])
        sub_msg = f"Programados hoy: {resumen_prog}"
      else:
        sub_msg = f"Sin turnos activos el {etiqueta_dia_hoy}"

      html_guardia_card = (
          '<div class="kpi-card" style="border-top: 4px solid #64748b;"><div '
          f'class="kpi-title"><span class="off-dot"></span>EN GUARDIA AHORA '
          f'({etiqueta_dia_hoy} • {ahora.strftime("%I:%M %p")})</div><div '
          'style="font-size:16px;font-weight:800;color:#475569;margin-top:6px;">Ningún '
          'técnico activo ahora</div><div '
          f'style="font-size:12px;color:#64748b;margin-top:4px;">{sub_msg}</div></div>'
      )

    # Tarjetas KPI
    k1, k2, k3, k4, k5 = st.columns([1.45, 1, 1, 1, 1])
    with k1:
      st.markdown(html_guardia_card, unsafe_allow_html=True)
    with k2:
      st.markdown(
          f'<div class="kpi-card" style="border-top: 4px solid #97C900;"><div'
          ' class="kpi-title">Turnos Apertura (A)</div><div class="kpi-value"'
          f' style="color:#6b8e00;">{total_apertura}</div><div'
          ' style="font-size:12px;color:#64748b;font-weight:600;">'
          f'{TURNOS_RIO["A"]["inicio"].strftime("%I:%M %p")} -'
          f' {TURNOS_RIO["A"]["fin"].strftime("%I:%M %p")}</div></div>',
          unsafe_allow_html=True,
      )
    with k3:
      st.markdown(
          f'<div class="kpi-card" style="border-top: 4px solid #0095D9;"><div'
          ' class="kpi-title">Turnos Cierre (C)</div><div class="kpi-value"'
          f' style="color:#0095D9;">{total_cierre}</div><div'
          ' style="font-size:12px;color:#64748b;font-weight:600;">'
          f'{TURNOS_RIO["C"]["inicio"].strftime("%I:%M %p")} -'
          f' {TURNOS_RIO["C"]["fin"].strftime("%I:%M %p")}</div></div>',
          unsafe_allow_html=True,
      )
    with k4:
      st.markdown(
          f'<div class="kpi-card" style="border-top: 4px solid #F58220;"><div'
          ' class="kpi-title">Redobles (R)</div><div class="kpi-value"'
          f' style="color:#F58220;">{total_redoble}</div><div'
          ' style="font-size:12px;color:#64748b;font-weight:600;">'
          f'{TURNOS_RIO["R"]["inicio"].strftime("%I:%M %p")} -'
          f' {TURNOS_RIO["R"]["fin"].strftime("%I:%M %p")}</div></div>',
          unsafe_allow_html=True,
      )
    with k5:
      st.markdown(
          f'<div class="kpi-card" style="border-top: 4px solid #D63883;"><div'
          ' class="kpi-title">Días Vacaciones (V)</div><div class="kpi-value"'
          f' style="color:#D63883;">{total_vacaciones}</div><div'
          f' style="font-size:12px;color:#64748b;font-weight:600;">Personal en sede: {len(df_vista)}</div></div>',
          unsafe_allow_html=True,
      )

    st.markdown(
        f"""<div class="legend-container">
<span class="legend-pill" style="background:#97C900;color:#0c2e12;">A • Apertura ({TURNOS_RIO['A']['inicio'].strftime('%I:%M %p')} - {TURNOS_RIO['A']['fin'].strftime('%I:%M %p')})</span>
<span class="legend-pill" style="background:#126332;color:#ffffff;">I • Intermedio ({TURNOS_RIO['I']['inicio'].strftime('%I:%M %p')} - {TURNOS_RIO['I']['fin'].strftime('%I:%M %p')})</span>
<span class="legend-pill" style="background:#0095D9;color:#ffffff;">C • Cierre ({TURNOS_RIO['C']['inicio'].strftime('%I:%M %p')} - {TURNOS_RIO['C']['fin'].strftime('%I:%M %p')})</span>
<span class="legend-pill" style="background:#F58220;color:#ffffff;">R • Redoble ({TURNOS_RIO['R']['inicio'].strftime('%I:%M %p')} - {TURNOS_RIO['R']['fin'].strftime('%I:%M %p')})</span>
<span class="legend-pill" style="background:#D63883;color:#ffffff;">V • Vacaciones</span>
<span class="legend-pill" style="background:#E9ECEF;color:#495057;">L • Libre</span>
</div>""",
        unsafe_allow_html=True,
    )

    # Pestañas principales
    tab1, tab2, tab3 = st.tabs([
        "Matriz Mensual Interactiva",
        "Consulta de Guardia por Día",
        "Editar Horarios y Personal",
    ])

    with tab1:
      conteo_diario = {d: 0 for d in range(1, num_dias + 1)}

      ths_dias = ""
      for d in range(1, num_dias + 1):
        dow = DIAS_SEMANA[date(selected_anio, selected_mes, d).weekday()]
        clase_hoy = "th-hoy" if d == dia_num_eval else ""
        ths_dias += (
            f'<th class="{clase_hoy}">'
            f'<div class="th-day-num">{d:02d}</div>'
            f'<div class="th-day-dow">{dow}</div>'
            f'</th>'
        )

      filas_html = ""
      gran_total = 0

      for _, row in df_vista.iterrows():
        t_json = row["turnos_json"]
        t_dict = json.loads(t_json) if t_json else {}
        partes_emp = row["empleado"].split("/")
        nom_emp = partes_emp[0].strip()
        tlf_emp = partes_emp[1].strip() if len(partes_emp) > 1 else ""

        dias_activos = 0
        tds_dias = ""

        for d in range(1, num_dias + 1):
          codigo = str(t_dict.get(str(d), "")).upper().strip()
          if codigo in TURNOS_RIO:
            bg = TURNOS_RIO[codigo]["bg"]
            fg = TURNOS_RIO[codigo]["text"]
            nom_turno = TURNOS_RIO[codigo]["nombre"]
            if codigo in ["A", "I", "C", "R"]:
              dias_activos += 1
              conteo_diario[d] += 1
            tds_dias += (
                f'<td class="shift-cell" title="{nom_emp} • Día {d:02d}: {nom_turno}" '
                f'style="background-color:{bg};color:{fg};">'
                f'<div class="shift-code">{codigo}</div>'
                f'</td>'
            )
          else:
            tds_dias += (
                '<td class="shift-cell" style="background-color:#f8fafc;color:#cbd5e1;'
                'border:1px dashed #e2e8f0;"><div class="shift-code" style="font-size:16px;">—</div></td>'
            )

        gran_total += dias_activos
        porcentaje = min(100, int((dias_activos / num_dias) * 100))

        filas_html += (
            f'<tr class="tech-row">'
            f'<td class="td-tech-info">'
            f'<div class="td-tech-name">{nom_emp}</div>'
            f'<div class="td-tech-phone">{tlf_emp}</div>'
            f'</td>'
            f'{tds_dias}'
            f'<td class="td-progress-box">'
            f'<div class="prog-val">{dias_activos} días</div>'
            f'<div class="prog-bar-bg"><div class="prog-bar-fill" style="width:{porcentaje}%;"></div></div>'
            f'</td>'
            f'</tr>'
        )

      tds_totales = ""
      for d in range(1, num_dias + 1):
        cant = conteo_diario[d]
        clase_tot = "td-total-cell" if cant > 0 else "td-total-zero"
        tds_totales += f'<td class="{clase_tot}">{cant}</td>'

      porcentaje_total = min(
          100, int((gran_total / max(1, len(df_vista) * num_dias)) * 100)
      )

      fila_total_html = (
          f'<tr class="total-row">'
          f'<td class="td-total-label">TOTAL ACTIVOS ({nombre_mes.upper()})</td>'
          f'{tds_totales}'
          f'<td class="td-progress-box" style="background:#f8fafc;">'
          f'<div class="prog-val" style="color:#126332;">{gran_total} días</div>'
          f'<div class="prog-bar-bg"><div class="prog-bar-fill" style="width:{porcentaje_total}%;"></div></div>'
          f'</td>'
          f'</tr>'
      )

      tabla_completa_html = (
          f'<div class="matrix-wrapper">'
          f'<table class="rio-matrix-table">'
          f'<thead><tr>'
          f'<th class="col-tech-header">Técnico / Contacto</th>'
          f'{ths_dias}'
          f'<th class="col-total-header">Días Laborados</th>'
          f'</tr></thead>'
          f'<tbody>'
          f'{filas_html}'
          f'{fila_total_html}'
          f'</tbody>'
          f'</table>'
          f'</div>'
      )

      st.markdown(tabla_completa_html, unsafe_allow_html=True)

    with tab2:
      st.markdown("#### Consulta de Cobertura por Día")
      col_dia_sel, col_resumen_dia = st.columns([1, 3])

      with col_dia_sel:
        dia_consulta = st.slider(
            "Selecciona el día del mes:",
            min_value=1,
            max_value=num_dias,
            value=min(ahora.day, num_dias),
        )
        dow_nombre = DIAS_SEMANA[
            date(selected_anio, selected_mes, dia_consulta).weekday()
        ]
        st.info(
            f"Consultando: **{dow_nombre} {dia_consulta:02d} de {nombre_mes}"
            f" {selected_anio}**"
        )

      with col_resumen_dia:
        c_cards = st.columns(3)
        for idx, row in df_mes.iterrows():
          t_json = row["turnos_json"]
          t_dict = json.loads(t_json) if t_json else {}
          cod = str(t_dict.get(str(dia_consulta), "")).upper().strip()
          info = TURNOS_RIO.get(
              cod,
              {
                  "nombre": "Sin turno asignado",
                  "bg": "#cbd5e1",
                  "text": "#334155",
                  "desc": "Descanso / No asignado",
                  "inicio": None,
                  "fin": None,
              },
          )

          partes = row["empleado"].split("/")
          nombre_emp = partes[0].strip()
          tlf_emp = partes[1].strip() if len(partes) > 1 else "Sin teléfono"

          esta_activo = (
              info["inicio"] is not None
              and info["inicio"] <= hora_actual <= info["fin"]
              and dia_consulta == dia_num_eval
          )
          badge_vivo = (
              '<span style="color:#16a34a;font-size:13px;font-weight:800;">● EN'
              " TURNO AHORA</span>"
              if esta_activo
              else ""
          )

          with c_cards[idx % 3]:
            st.markdown(
                f"""<div class="tech-card" style="border-left-color: {info['bg']};">
<div style="display:flex;justify-content:space-between;align-items:center;">
<div class="tech-name">{nombre_emp}</div>
{badge_vivo}
</div>
<div class="tech-phone">Tel: {tlf_emp}</div>
<span class="legend-pill" style="background:{info['bg']};color:{info['text']};">{cod if cod else '—'} • {info['nombre']}</span>
<div style="font-size:13px;color:#64748b;margin-top:8px;font-weight:600;">Horario: {info['desc']}</div>
</div>""",
                unsafe_allow_html=True,
            )

    with tab3:
      st.markdown("#### Modificar Turnos y Personal del Mes")
      filas_editor = []
      for _, row in df_mes.iterrows():
        t_json = row["turnos_json"]
        t_dict = json.loads(t_json) if t_json else {}
        item = {"ID": row["tecnico_id"], "Técnico": row["empleado"]}
        for d in range(1, num_dias + 1):
          col_label = columnas_dias[str(d)]
          val_actual = t_dict.get(str(d), "")
          item[col_label] = val_actual if val_actual else ""
        filas_editor.append(item)

      df_editor = pd.DataFrame(filas_editor)

      config_columnas_editor = {
          "ID": st.column_config.NumberColumn("ID", width="small"),
          "Técnico": st.column_config.TextColumn(
              "Técnico / Teléfono", width="medium"
          ),
      }
      for d in range(1, num_dias + 1):
        col_label = columnas_dias[str(d)]
        config_columnas_editor[col_label] = st.column_config.SelectboxColumn(
            label=col_label,
            help=f"Turno para el día {col_label} de {nombre_mes}",
            width="small",
            options=["", "A", "I", "C", "R", "V", "L"],
            required=False,
        )

      edited_df = st.data_editor(
          df_editor,
          use_container_width=True,
          hide_index=True,
          disabled=["ID"],
          column_config=config_columnas_editor,
      )

      col_b1, col_b2 = st.columns([1, 2])
      with col_b1:
        if st.button(
            "Guardar Cambios en Base de Datos",
            type="primary",
            use_container_width=True,
        ):
          actualizaciones = []
          for _, r in edited_df.iterrows():
            nuevos_turnos = {}
            for d in range(1, num_dias + 1):
              col_label = columnas_dias[str(d)]
              raw_val = r[col_label]
              val = (
                  str(raw_val).strip().upper()
                  if pd.notna(raw_val) and str(raw_val) != "None"
                  else ""
              )
              if val in TURNOS_RIO:
                nuevos_turnos[str(d)] = val
            actualizaciones.append(
                (
                    r["ID"],
                    str(r["Técnico"]),
                    nuevos_turnos,
                    selected_anio,
                    selected_mes,
                )
            )
          actualizar_turnos_lote(actualizaciones)
          st.success("Horarios y datos de personal actualizados exitosamente.")
          st.rerun()

      with col_b2:
        with st.popover("Eliminar un técnico de esta sucursal"):
          emp_borrar = st.selectbox(
              "Selecciona el técnico a retirar:",
              options=df_mes["tecnico_id"].tolist(),
              format_func=lambda x: (
                  df_mes[df_mes["tecnico_id"] == x]["empleado"].values[0]
              ),
          )
          if st.button("Confirmar eliminación del técnico", type="secondary"):
            eliminar_tecnico(emp_borrar)
            st.success("Técnico eliminado de la base de datos.")
            st.rerun()
