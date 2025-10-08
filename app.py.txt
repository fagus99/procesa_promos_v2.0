import os
import pandas as pd
import numpy as np
import streamlit as st
from io import BytesIO

# Evitar errores por threads en Numpy
os.environ["OMP_NUM_THREADS"] = "1"

st.set_page_config(page_title="🎰 Promociones Casino - App Unificada", layout="wide")
st.title("🎰 App Unificada - Promociones Casino")

st.markdown("""
Esta app permite procesar archivos de **jugado y depositado por usuario**,
definir condiciones de promoción y generar un listado final con los usuarios bonificables.
""")

# === SUBIDA DE ARCHIVOS ===
col1, col2 = st.columns(2)
with col1:
    archivo_depositos = st.file_uploader("📥 Subí archivo de depósitos (CSV o Excel)", type=["csv", "xlsx"])
with col2:
    archivo_jugado = st.file_uploader("🎲 Subí archivo de jugado (CSV o Excel)", type=["csv", "xlsx"])

if archivo_depositos and archivo_jugado:
    # Lectura de archivos
    def leer_archivo(archivo):
        nombre = archivo.name.lower()
        if nombre.endswith(".csv"):
            return pd.read_csv(archivo)
        else:
            return pd.read_excel(archivo)
    
    df_dep = leer_archivo(archivo_depositos)
    df_jug = leer_archivo(archivo_jugado)

    st.success("✅ Archivos cargados correctamente.")
    st.write("**Depósitos:**", df_dep.shape, "filas")
    st.write("**Jugado:**", df_jug.shape, "filas")

    st.divider()

    # === CONFIGURACIÓN DE PROMOCIÓN ===
    st.header("⚙️ Configuración de Promoción")

    col1, col2, col3 = st.columns(3)
    with col1:
        deposito_min = st.number_input("Depósito mínimo", min_value=0.0, value=1000.0, step=100.0)
        jugado_min = st.number_input("Jugado mínimo", min_value=0.0, value=0.0, step=100.0)
    with col2:
        porcentaje_bono = st.number_input("Porcentaje de bono (%)", min_value=0.0, value=10.0, step=1.0)
        tope_bono = st.number_input("Tope máximo de bono", min_value=0.0, value=5000.0, step=500.0)
    with col3:
        tipo_deposito = st.selectbox("Tipo de depósito a considerar", ["Suma", "Máximo", "Mínimo"])
        rollover = st.checkbox("Aplicar rollover x1", value=False)

    st.divider()

    # === PROCESAMIENTO ===
    st.header("🧮 Procesamiento de usuarios bonificables")

    # Detectar columna de usuario común
    col_dep_usuario = st.selectbox("Columna de usuario en depósitos", df_dep.columns)
    col_jug_usuario = st.selectbox("Columna de usuario en jugado", df_jug.columns)

    # Calcular depósitos y jugado por usuario
    agg_func = {
        "Suma": np.sum,
        "Máximo": np.max,
        "Mínimo": np.min
    }[tipo_deposito]

    dep_por_usuario = df_dep.groupby(col_dep_usuario).sum(numeric_only=True)
    dep_por_usuario = dep_por_usuario.rename(columns=lambda c: f"DEP_{c}")

    jug_por_usuario = df_jug.groupby(col_jug_usuario).sum(numeric_only=True)
    jug_por_usuario = jug_por_usuario.rename(columns=lambda c: f"JUG_{c}")

    # Unir ambos
    resumen = dep_por_usuario.join(jug_por_usuario, how="outer").fillna(0)

    # Buscar las columnas numéricas principales
    col_dep = [c for c in resumen.columns if "DEP_" in c][0]
    col_jug = [c for c in resumen.columns if "JUG_" in c][0]

    resumen["DEPOSITO"] = resumen[col_dep]
    resumen["JUGADO"] = resumen[col_jug]

    # Aplicar condiciones
    resumen["BONIFICABLE"] = (resumen["DEPOSITO"] >= deposito_min) & (resumen["JUGADO"] >= jugado_min)
    resumen["BONO"] = np.where(
        resumen["BONIFICABLE"],
        np.minimum(resumen["DEPOSITO"] * (porcentaje_bono / 100), tope_bono),
        0
    )

    if rollover:
        resumen["BONO_CON_ROLLOVER"] = resumen["BONO"]
        resumen["BONO"] = 0

    st.dataframe(resumen[["DEPOSITO", "JUGADO", "BONO", "BONIFICABLE"]])

    # Descargar
    buffer = BytesIO()
    resumen.to_excel(buffer, index=True)
    st.download_button(
        label="💾 Descargar resultados en Excel",
        data=buffer.getvalue(),
        file_name="usuarios_bonificables.xlsx",
        mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    )
else:
    st.info("👆 Subí ambos archivos para comenzar.")

