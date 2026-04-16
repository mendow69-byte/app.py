import streamlit as st
import pandas as pd
import folium
from streamlit_folium import st_folium
from geopy.geocoders import Nominatim
import datetime

# Setări Aspect Site
st.set_page_config(page_title="LOGISTIC MAP - Camioane Disponibile", layout="wide", page_icon="🚚")

# Stil personalizat
st.markdown("""
    <style>
    .main { background-color: #f5f7f9; }
    .stButton>button { width: 100%; border-radius: 5px; height: 3em; background-color: #007bff; color: white; }
    </style>
    """, unsafe_allow_html=True)

st.title("🚚 Monitorizare Camioane Disponibile - Europa")
st.write(f"Data curentă: {datetime.date.today().strftime('%d-%m-%Y')}")

# Baza de date temporară (se resetează la restart server - pentru permanentizare e nevoie de DB)
if 'db' not in st.session_state:
    st.session_state.db = pd.DataFrame(columns=['ID', 'Oraș', 'Țară', 'Tip', 'Contact', 'Status', 'Lat', 'Lon'])

# --- ZONA DE INTRODUCERE DATE ---
with st.sidebar:
    st.header("📍 Adaugă Camion")
    with st.form("form_camion"):
        oras = st.text_input("Oraș (ex: Arad, Lyon, Berlin)")
        tara = st.text_input("Țara (ex: RO, FR, DE)")
        tip = st.selectbox("Tip Camion", ["Prelată 13.6m", "Frigider", "Dubă 3.5t", "Ansamblu Jumbo"])
        contact = st.text_input("Contact (Nume/Tel)")
        status = st.select_slider("Disponibilitate", options=["Imediat", "În 24h", "În 48h"])
        
        submit = st.form_submit_button("Publică pe hartă")

    if submit and oras:
        geolocator = Nominatim(user_agent="truck_app_2026")
        loc = geolocator.geocode(f"{oras}, {tara}")
        
        if loc:
            new_truck = pd.DataFrame([{
                'ID': len(st.session_state.db) + 1,
                'Oraș': oras, 'Țară': tara, 'Tip': tip,
                'Contact': contact, 'Status': status,
                'Lat': loc.latitude, 'Lon': loc.longitude
            }])
            st.session_state.db = pd.concat([st.session_state.db, new_truck], ignore_index=True)
            st.success("Camion adăugat!")
        else:
            st.error("Nu am găsit locația. Verifică ortografia.")

# --- AFIȘARE HARTĂ ---
col1, col2 = st.columns([3, 1])

with col1:
    # Centrare hartă pe Europa
    m = folium.Map(location=[48.0, 15.0], zoom_start=4, tiles="cartodbpositron")
    
    for _, row in st.session_state.db.iterrows():
        # Culoare în funcție de status
        color = "green" if row['Status'] == "Imediat" else "orange" if row['Status'] == "În 24h" else "red"
        
        folium.Marker(
            [row['Lat'], row['Lon']],
            popup=f"<b>{row['Tip']}</b><br>Contact: {row['Contact']}<br>Status: {row['Status']}",
            tooltip=f"{row['Oraș']} - {row['Tip']}",
            icon=folium.Icon(color=color, icon="info-sign")
        ).add_to(m)
    
    st_folium(m, width="100%", height=600)

with col2:
    st.subheader("📋 Listă Rapidă")
    if not st.session_state.db.empty:
        for _, row in st.session_state.db.iterrows():
            st.info(f"**{row['Oraș']}** ({row['Țară']})\n\n{row['Tip']} | {row['Contact']}")
    else:
        st.write("Niciun camion disponibil.")

# --- TABEL DETALIAT ---
if not st.session_state.db.empty:
    st.divider()
    st.subheader("Gestionare Date")
    st.dataframe(st.session_state.db[['Oraș', 'Țară', 'Tip', 'Contact', 'Status']], use_container_width=True)
    if st.button("Resetează Harta (Șterge tot)"):
        st.session_state.db = pd.DataFrame(columns=['ID', 'Oraș', 'Țară', 'Tip', 'Contact', 'Status', 'Lat', 'Lon'])
        st.rerun()
