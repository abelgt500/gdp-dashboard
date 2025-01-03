import pandas as pd
import plotly.express as px
import streamlit as st
import requests
import json
import logging
from io import BytesIO

# Configuración del logging
logging.basicConfig(level=logging.DEBUG, format='%(asctime)s - %(levelname)s - %(message)s')

# API endpoint
API_URL = 'https://datos.gob.cl/api/3/action/datastore_search?resource_id=1a8589c4-1191-4286-b49b-b247346a1d2f&limit=100000'

@st.cache_data
def get_data_from_api(url):
    """Obtiene datos desde la API REST y los maneja."""
    st.info("Obteniendo datos de la API...", icon="⏳")
    logging.debug(f"Iniciando get_data_from_api con URL: {url}")

    try:
        logging.debug("Realizando petición a la API.")
        response = requests.get(url)
        response.raise_for_status()  # Verificar si la petición fue exitosa (código 200)
        logging.debug(f"Petición exitosa. Código de estado: {response.status_code}")
        
        data = response.json()
        logging.debug("Datos JSON recibidos y parseados.")
        
        if 'result' in data and 'records' in data['result']:
            records = data['result']['records']
            if not records:
                logging.warning("La API retornó una lista vacía de registros.")
                st.warning("La API retornó una lista vacía de registros.", icon="⚠️")
                return None
        else:
            logging.error("La respuesta de la API no tiene el formato esperado.")
            st.error("Error: La respuesta de la API no tiene el formato esperado.", icon="🚨")
            return None
        
        st.success("Datos obtenidos exitosamente de la API.", icon="✅")
        return records
            
    except requests.exceptions.RequestException as e:
        logging.error(f"Error en la petición a la API: {e}")
        st.error(f"Error al conectar con la API: {e}", icon="🚨")
        return None
    except json.JSONDecodeError as e:
        logging.error(f"Error al decodificar la respuesta JSON: {e}")
        st.error(f"Error al procesar la respuesta JSON: {e}", icon="🚨")
        return None
    except Exception as e:
        logging.error(f"Error inesperado al procesar los datos: {e}")
        st.error(f"Error inesperado: {e}", icon="🚨")
        return None

def main():
    """Función principal para crear la aplicación Streamlit."""
    st.set_page_config(layout="wide", page_title="Análisis de Inversiones MOP", page_icon="📊")

    # Sidebar
    with st.sidebar:
      st.title("Análisis de Inversiones MOP")
      st.image("logouss.png", width=300)
      st.markdown("Explora y visualiza los datos de inversión del Ministerio de Obras Públicas de Chile.")
      st.markdown("Esta aplicación permite analizar la inversión a través del tiempo, regiones, provincias y servicios.")

    # Obtener datos de la API
    api_records = get_data_from_api(API_URL)

    if api_records is None:
        st.error("No se pudieron obtener los datos de la API. Verifica el error en los logs.", icon="❌")
        return

    df = pd.DataFrame(api_records)
       
    # ---  Convertir 'ANO' a tipo numérico  ---
    try:
        df['ANO'] = pd.to_numeric(df['ANO'], errors='coerce')
    except Exception as e:
        st.error(f"Error al convertir la columna 'ANO' a numérica: {e}", icon="🚨")
        return
    
    # ---  Convertir 'INVERSION (MILES DE $ DE CADA ANO)' a tipo numerico  ---
    try:
        df['INVERSION (MILES DE $ DE CADA ANO)'] = pd.to_numeric(df['INVERSION (MILES DE $ DE CADA ANO)'], errors='coerce')
    except Exception as e:
        st.error(f"Error al convertir la columna 'INVERSION (MILES DE $ DE CADA ANO)' a numérica: {e}", icon="🚨")
        return

    # --- Evolución de la Inversión a lo largo de los años ---
    st.subheader("Evolución de la Inversión a lo Largo de los Años")
    
    # Agrupar y sumar la inversión por año para poder graficar
    yearly_investment = df.groupby('ANO')['INVERSION (MILES DE $ DE CADA ANO)'].sum().reset_index()
    
    # Filtrar los datos en caso que la API no contenga registros validos para graficar
    if len(yearly_investment) > 0:
        fig_line = px.line(yearly_investment, x='ANO', y='INVERSION (MILES DE $ DE CADA ANO)',
                        title='Evolución de la Inversión Anual',
                        labels={'INVERSION (MILES DE $ DE CADA ANO)': 'Inversión (MM)'})
        st.plotly_chart(fig_line, use_container_width=True)
    else:
        st.warning("No hay datos disponibles para la gráfica de Evolución de la Inversión.", icon="⚠️")

    # --- Distribución de la Inversión por Región ---
    st.subheader("Distribución de la Inversión por Región")
    
    # Agrupar y sumar la inversión por region para poder graficar
    region_investment = df.groupby('REGION')['INVERSION (MILES DE $ DE CADA ANO)'].sum().reset_index()
    
    # Filtrar los datos en caso que la API no contenga registros validos para graficar
    if len(region_investment) > 0:
        fig_bar_region = px.bar(region_investment, x='REGION', y='INVERSION (MILES DE $ DE CADA ANO)',
                        title='Inversión por Región',
                        labels={'INVERSION (MILES DE $ DE CADA ANO)': 'Inversión (MM)'})
        st.plotly_chart(fig_bar_region, use_container_width=True)
    else:
         st.warning("No hay datos disponibles para la gráfica de Distribución de Inversión por Región.", icon="⚠️")
    
    # --- Distribución de la Inversión por Provincia ---
    st.subheader("Distribución de la Inversión por Provincia")
    
    # Agrupar y sumar la inversión por provincia para poder graficar
    provincia_investment = df.groupby('PROVINCIA')['INVERSION (MILES DE $ DE CADA ANO)'].sum().reset_index()
    
    # Filtrar por las 10 provincias principales para evitar un gráfico muy cargado
    if len(provincia_investment) > 0:
         top_provincias = provincia_investment.nlargest(10, 'INVERSION (MILES DE $ DE CADA ANO)')
         fig_bar_provincia = px.bar(top_provincias, x='PROVINCIA', y='INVERSION (MILES DE $ DE CADA ANO)',
                            title='Inversión por Provincia (Top 10)',
                            labels={'INVERSION (MILES DE $ DE CADA ANO)': 'Inversión (MM)'})
         st.plotly_chart(fig_bar_provincia, use_container_width=True)
    else:
        st.warning("No hay datos disponibles para la gráfica de Distribución de Inversión por Provincia.", icon="⚠️")

    
    # --- Comparación de Inversión en Diferentes Servicios ---
    st.subheader("Comparación de Inversión en Diferentes Servicios")
    
    # Agrupar y sumar la inversión por servicio para poder graficar
    service_investment = df.groupby('SERVICIO')['INVERSION (MILES DE $ DE CADA ANO)'].sum().reset_index()
    
    # Filtrar los datos en caso que la API no contenga registros validos para graficar
    if len(service_investment) > 0:
        fig_bar_service = px.bar(service_investment, x='SERVICIO', y='INVERSION (MILES DE $ DE CADA ANO)',
                                title='Inversión por Servicio',
                                labels={'INVERSION (MILES DE $ DE CADA ANO)': 'Inversión (MM)'})
        st.plotly_chart(fig_bar_service, use_container_width=True)
    else:
         st.warning("No hay datos disponibles para la gráfica de Comparación de Inversión por Servicio.", icon="⚠️")
    
    # --- Correlación entre Variables ---
    st.subheader("Correlación entre Inversión y Año")
    
    # Agrupar y sumar la inversión por año para poder calcular la correlación
    correlation_data = df.groupby('ANO')['INVERSION (MILES DE $ DE CADA ANO)'].sum().reset_index()
    
    # Calcular la correlación, solo si existen más de un registro para calcularla
    if len(correlation_data) > 1:
        correlation_value = correlation_data['ANO'].corr(correlation_data['INVERSION (MILES DE $ DE CADA ANO)'])
        st.write(f"Correlación entre Año e Inversión: {correlation_value:.2f}")
        
        # Mostrar un scatter plot
        fig_scatter_corr = px.scatter(correlation_data, x='ANO', y='INVERSION (MILES DE $ DE CADA ANO)',
                                        title='Relación entre Año e Inversión',
                                        labels={'INVERSION (MILES DE $ DE CADA ANO)': 'Inversión (MM)'})
        st.plotly_chart(fig_scatter_corr, use_container_width=True)
    else:
        st.warning("No hay suficientes datos para calcular la correlación.", icon="⚠️")

if __name__ == "__main__":
    # Para Colab: Usar st.set_page_config
    #st.set_page_config(layout="wide")
    main()
