import pandas as pd
import plotly.graph_objects as go
import streamlit as st

# Set page config
st.set_page_config(layout="wide")
st.title("📊 KPI Dashboard - Januari vs Februari")

# Upload file
uploaded_file = st.file_uploader("Upload file Excel", type=["xlsx"])
if uploaded_file:
    # Baca data dari file
    df = pd.read_excel(uploaded_file, sheet_name="Dulu")
    
    # Pilih kolom yang dipakai
    df = df[['Perspective', 'KPI', 'PIC', 'Target Jan', 'Actual Jan', 'Achv Jan', 'Target Feb', 'Actual Feb', 'Achv Feb']].copy()

    # Fungsi untuk menghitung status (traffic light) menggunakan emoji
    def get_status(achv, target):
        if pd.isna(achv) or pd.isna(target):
            return '🛑'  # Hitam (NA)
        r = achv/target if target else 0
        if r > 1:    
            return '🟢'  # Hijau (>100%)
        elif r >= 0.7:
            return '🟡'  # Kuning (70%-99%)
        return '🔴'  # Merah (<70%)

    df['Status'] = df.apply(lambda row: get_status(row['Achv Jan'], row['Target Jan']), axis=1)

    # Summary per Status
    status_order = ['🔴', '🟡', '🟢', '🛑']
    overall = df['Status'].value_counts().reindex(status_order, fill_value=0)

    # Summary per Perspective
    persp = (
        df.groupby(['Perspective','Status'])
        .size()
        .unstack(fill_value=0)
        .reindex(columns=status_order, fill_value=0)
    )

    # Tampilkan hasil summary di Streamlit
    col1, col2 = st.columns(2)
    with col1:
        st.subheader("📍 Total KPI per Traffic Light")
        st.bar_chart(overall)

    with col2:
        st.subheader("📍 KPI per Perspective dan Status")
        st.dataframe(persp)

    # Filter interaktif
    st.subheader("🔎 Lihat KPI berdasarkan warna")
    selected_status = st.selectbox("Pilih warna status:", status_order)
    filtered_data = df[df['Status'] == selected_status]
    kpis = filtered_data['KPI'].unique()

    # Dropdown untuk memilih KPI
    selected_kpi = st.selectbox("Pilih KPI:", kpis)

    # Ambil data untuk KPI yang dipilih
    kpi_data = filtered_data[filtered_data['KPI'] == selected_kpi]

    # Menampilkan perbandingan Januari vs Februari serta Target
    if not kpi_data.empty:
        target_jan = kpi_data['Target Jan'].values[0]
        target_feb = kpi_data['Target Feb'].values[0]
        actual_jan = kpi_data['Actual Jan'].values[0]
        actual_feb = kpi_data['Actual Feb'].values[0]
        achv_jan = kpi_data['Achv Jan'].values[0]
        achv_feb = kpi_data['Achv Feb'].values[0]

        # Perbandingan grafik
        fig = go.Figure()

        fig.add_trace(go.Bar(
            x=['Januari', 'Februari'],
            y=[actual_jan, actual_feb],
            name='Actual',
            marker=dict(color='blue'),
        ))

        fig.add_trace(go.Bar(
            x=['Januari', 'Februari'],
            y=[target_jan, target_feb],
            name='Target',
            marker=dict(color='grey'),
        ))

        fig.update_layout(
            title=f'Perbandingan Actual vs Target untuk KPI: {selected_kpi}',
            xaxis_title='Bulan',
            yaxis_title='Nilai',
            barmode='group',
            template='plotly_dark'
        )

        st.plotly_chart(fig)

        # Menampilkan Achievement untuk Januari dan Februari
        st.subheader("📈 Achievement (%)")
        st.write(f"Achievement Januari: {achv_jan * 100:.2f}%")
        st.write(f"Achievement Februari: {achv_feb * 100:.2f}%")
else:
    st.info("Silakan upload file Excel terlebih dahulu.")
