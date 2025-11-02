"""
Generador HTML interactivo mejorado de series tarifarias.

Características:
- Visualización consolidada: todos los segmentos de una tarifa en un solo gráfico
- Interfaz mejorada con múltiples filtros y controles
- Exportación de datos a CSV
- Gráficos comparativos y estadísticas
- Manejo robusto de errores
- Código optimizado y documentado

Autor: Sistema GTR
Versión: 2.0
"""

import pandas as pd
import numpy as np
import json
from pathlib import Path
from datetime import datetime
import logging

# Configuración de logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# ===== CONFIGURACIÓN =====
PATH_XLSX = r"C:\DesarrollosGTR\Tarifas\Cuadro_TarifarioMT_AT.xlsx"
OUTPUT_HTML = Path("tarifas_interactivo_v2.html")
SHEET_NAME = "Base"

# Columnas que NO son fechas
COLUMNAS_METADATA = {
    "Tarifa", "Segmento", "Unidades", "Concepto", 
    "Division", "Intervalo_Horario", "Intervalo Horario"
}


class TarifasProcessor:
    """Clase para procesar y transformar datos de tarifas."""
    
    def __init__(self, excel_path, sheet_name="Base"):
        self.excel_path = Path(excel_path)
        self.sheet_name = sheet_name
        self.df_raw = None
        self.df_melt = None
        
    def load_data(self):
        """Carga datos del archivo Excel con validación."""
        try:
            if not self.excel_path.exists():
                raise FileNotFoundError(f"Archivo no encontrado: {self.excel_path}")
            
            logger.info(f"Cargando archivo: {self.excel_path}")
            self.df_raw = pd.read_excel(
                self.excel_path, 
                sheet_name=self.sheet_name, 
                engine="openpyxl"
            )
            
            if self.df_raw.empty:
                raise ValueError("El archivo Excel está vacío")
            
            logger.info(f"✓ Datos cargados: {len(self.df_raw)} filas, {len(self.df_raw.columns)} columnas")
            return self
            
        except Exception as e:
            logger.error(f"Error al cargar datos: {e}")
            raise
    
    def _identify_date_columns(self):
        """Identifica columnas que contienen fechas."""
        date_cols = []
        col_to_dt = {}
        
        for col in self.df_raw.columns:
            # Saltar columnas de metadata
            if col in COLUMNAS_METADATA:
                continue
            
            # Intentar parsear como fecha
            try:
                col_str = str(col)
                
                # Verificar si contiene separadores de fecha
                if "/" in col_str or "-" in col_str:
                    # Intentar parsear con día primero (formato dd/mm/yyyy)
                    dt = pd.to_datetime(col_str, dayfirst=True, errors="coerce")
                    
                    if pd.isna(dt):
                        # Intentar otros formatos
                        dt = pd.to_datetime(col_str, errors="coerce")
                    
                    if not pd.isna(dt):
                        date_cols.append(col)
                        col_to_dt[col] = dt.strftime("%Y-%m-%d")
                        
            except Exception as e:
                logger.debug(f"No se pudo parsear columna '{col}': {e}")
                continue
        
        if not date_cols:
            logger.warning("No se encontraron columnas de fecha explícitas. Usando heurística...")
            # Fallback: columnas después de las primeras 6
            potential_cols = list(self.df_raw.columns[6:])
            for col in potential_cols:
                try:
                    dt = pd.to_datetime(str(col), errors="coerce")
                    if not pd.isna(dt):
                        date_cols.append(col)
                        col_to_dt[col] = dt.strftime("%Y-%m-%d")
                except:
                    pass
        
        if not date_cols:
            raise RuntimeError("No se encontraron columnas de fecha válidas. Verifica el formato del archivo.")
        
        logger.info(f"✓ Identificadas {len(date_cols)} columnas de fecha")
        return date_cols, col_to_dt
    
    def transform_to_long_format(self):
        """Transforma datos de formato ancho a largo (melt)."""
        try:
            date_cols, col_to_dt = self._identify_date_columns()
            
            # Identificar columnas de metadata presentes
            id_vars = [col for col in self.df_raw.columns if col not in date_cols]
            
            # Melt: convertir columnas de fecha en filas
            logger.info("Transformando datos a formato largo...")
            self.df_melt = self.df_raw.melt(
                id_vars=id_vars,
                value_vars=date_cols,
                var_name="FechaOriginal",
                value_name="Valor"
            )
            
            # Mapear fechas a formato ISO
            self.df_melt["Fecha"] = self.df_melt["FechaOriginal"].map(
                lambda x: col_to_dt.get(x, x)
            )
            self.df_melt["Fecha"] = pd.to_datetime(self.df_melt["Fecha"], errors="coerce")
            
            # Eliminar filas con fechas inválidas
            invalid_dates = self.df_melt["Fecha"].isna().sum()
            if invalid_dates > 0:
                logger.warning(f"Eliminando {invalid_dates} filas con fechas inválidas")
                self.df_melt = self.df_melt.dropna(subset=["Fecha"])
            
            # Crear identificador de serie
            self.df_melt["Serie"] = self.df_melt.apply(
                lambda r: self._build_serie_name(r), axis=1
            )
            
            # Ordenar
            self.df_melt = self.df_melt.sort_values(
                ["Tarifa", "Division", "Serie", "Fecha"]
            ).reset_index(drop=True)
            
            logger.info(f"✓ Datos transformados: {len(self.df_melt)} registros")
            return self
            
        except Exception as e:
            logger.error(f"Error en transformación: {e}")
            raise
    
    def _build_serie_name(self, row):
        """Construye nombre descriptivo de serie."""
        parts = []
        
        # Segmento
        if pd.notna(row.get("Segmento")):
            parts.append(str(row["Segmento"]))
        
        # Concepto
        if pd.notna(row.get("Concepto")):
            parts.append(str(row["Concepto"]))
        
        # Intervalo horario (si existe)
        intervalo = row.get("Intervalo_Horario") or row.get("Intervalo Horario")
        if pd.notna(intervalo):
            parts.append(f"[{intervalo}]")
        
        # Unidades
        if pd.notna(row.get("Unidades")):
            parts.append(f"({row['Unidades']})")
        
        return " - ".join(parts) if parts else "Sin nombre"
    
    def calculate_statistics(self):
        """Calcula estadísticas por serie."""
        if self.df_melt is None:
            raise ValueError("Primero debes transformar los datos")
        
        logger.info("Calculando estadísticas...")
        
        # Agrupar por serie y calcular métricas
        stats = []
        for (tarifa, division, serie), group in self.df_melt.groupby(["Tarifa", "Division", "Serie"]):
            valores = group["Valor"].dropna()
            
            if len(valores) > 0:
                # Calcular cambios porcentuales
                pct_changes = valores.pct_change() * 100
                
                stat = {
                    "tarifa": tarifa,
                    "division": division,
                    "serie": serie,
                    "count": len(valores),
                    "min": float(valores.min()),
                    "max": float(valores.max()),
                    "mean": float(valores.mean()),
                    "std": float(valores.std()) if len(valores) > 1 else 0,
                    "first": float(valores.iloc[0]),
                    "last": float(valores.iloc[-1]),
                    "change_abs": float(valores.iloc[-1] - valores.iloc[0]) if len(valores) > 0 else 0,
                    "change_pct": float((valores.iloc[-1] / valores.iloc[0] - 1) * 100) if valores.iloc[0] != 0 else 0,
                    "avg_monthly_change": float(pct_changes.mean()) if len(pct_changes) > 1 else 0
                }
                stats.append(stat)
        
        logger.info(f"✓ Estadísticas calculadas para {len(stats)} series")
        return pd.DataFrame(stats)


class DataStructureBuilder:
    """Construye estructura JSON para visualización."""
    
    def __init__(self, df_melt):
        self.df_melt = df_melt
        
    def build_consolidated_structure(self):
        """
        Estructura consolidada: todos los segmentos de cada tarifa en un solo nivel.
        
        Retorna:
        {
          "tarifas": {
            "GDMTH": {
              "divisiones": ["Centro", "Norte", ...],
              "series": [
                {
                  "nombre": "Energía - Básico ($/kWh)",
                  "division": "Centro",
                  "segmento": "Energía",
                  "concepto": "Básico",
                  "fechas": ["2018-01-01", ...],
                  "valores": [1.234, ...],
                  "pct": [null, 2.5, ...]
                },
                ...
              ]
            },
            ...
          },
          "metadata": {
            "fecha_generacion": "...",
            "total_tarifas": 5,
            "rango_fechas": ["2018-01-01", "2024-12-01"]
          }
        }
        """
        logger.info("Construyendo estructura JSON consolidada...")
        
        estructura = {"tarifas": {}, "metadata": {}}
        
        # Procesar por tarifa
        tarifas = sorted(self.df_melt["Tarifa"].dropna().unique())
        
        for tarifa in tarifas:
            df_tarifa = self.df_melt[self.df_melt["Tarifa"] == tarifa]
            
            # Obtener divisiones únicas
            divisiones = sorted(df_tarifa["Division"].dropna().unique())
            
            # Construir series
            series_list = []
            
            for (division, serie), group in df_tarifa.groupby(["Division", "Serie"]):
                # Ordenar por fecha
                group = group.sort_values("Fecha")
                
                # Extraer componentes del nombre de serie
                segmento = group["Segmento"].iloc[0] if "Segmento" in group.columns else ""
                concepto = group["Concepto"].iloc[0] if "Concepto" in group.columns else ""
                
                # Preparar datos
                fechas = group["Fecha"].dt.strftime("%Y-%m-%d").tolist()
                valores = [
                    None if pd.isna(v) else float(v) 
                    for v in group["Valor"].tolist()
                ]
                
                # Calcular cambio porcentual mensual
                pct = self._calculate_pct_change(valores)
                
                series_list.append({
                    "nombre": serie,
                    "division": division,
                    "segmento": str(segmento),
                    "concepto": str(concepto),
                    "fechas": fechas,
                    "valores": valores,
                    "pct": pct
                })
            
            estructura["tarifas"][tarifa] = {
                "divisiones": divisiones,
                "series": series_list
            }
        
        # Agregar metadata
        all_fechas = self.df_melt["Fecha"].dropna()
        estructura["metadata"] = {
            "fecha_generacion": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "total_tarifas": len(tarifas),
            "total_registros": len(self.df_melt),
            "rango_fechas": [
                all_fechas.min().strftime("%Y-%m-%d"),
                all_fechas.max().strftime("%Y-%m-%d")
            ] if len(all_fechas) > 0 else []
        }
        
        logger.info(f"✓ Estructura creada para {len(tarifas)} tarifas")
        return estructura
    
    def _calculate_pct_change(self, valores):
        """Calcula cambio porcentual entre valores consecutivos."""
        pct = []
        for i in range(len(valores)):
            if i == 0 or valores[i] is None or valores[i-1] is None or valores[i-1] == 0:
                pct.append(None)
            else:
                pct.append(100.0 * (valores[i] - valores[i-1]) / valores[i-1])
        return pct


class HTMLGenerator:
    """Genera archivo HTML interactivo mejorado."""
    
    def __init__(self, data_structure, stats_df=None):
        self.data = data_structure
        self.stats = stats_df
        
    def generate(self, output_path):
        """Genera archivo HTML completo."""
        logger.info("Generando HTML interactivo...")
        
        data_js = json.dumps(self.data, ensure_ascii=False, indent=2)
        stats_js = json.dumps(
            self.stats.to_dict('records') if self.stats is not None else [],
            ensure_ascii=False
        )
        
        html_content = self._build_html_template(data_js, stats_js)
        
        output_path.write_text(html_content, encoding="utf-8")
        logger.info(f"✓ HTML generado: {output_path.resolve()}")
        
    def _build_html_template(self, data_js, stats_js):
        """Construye template HTML completo."""
        return f"""<!doctype html>
<html lang="es">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Análisis de Tarifas Eléctricas - GTR</title>
  <script src="https://cdn.plot.ly/plotly-2.26.1.min.js"></script>
  <style>
    * {{
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }}
    
    body {{
      font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      padding: 20px;
      min-height: 100vh;
    }}
    
    .container {{
      max-width: 1400px;
      margin: 0 auto;
      background: white;
      border-radius: 12px;
      box-shadow: 0 10px 40px rgba(0,0,0,0.2);
      overflow: hidden;
    }}
    
    .header {{
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      color: white;
      padding: 30px;
      text-align: center;
    }}
    
    .header h1 {{
      font-size: 2.5em;
      margin-bottom: 10px;
      text-shadow: 2px 2px 4px rgba(0,0,0,0.2);
    }}
    
    .header .subtitle {{
      font-size: 1.1em;
      opacity: 0.9;
    }}
    
    .controls-panel {{
      background: #f8f9fa;
      padding: 25px;
      border-bottom: 2px solid #e0e0e0;
    }}
    
    .control-row {{
      display: flex;
      gap: 20px;
      margin-bottom: 15px;
      flex-wrap: wrap;
      align-items: center;
    }}
    
    .control-group {{
      flex: 1;
      min-width: 200px;
    }}
    
    .control-group label {{
      display: block;
      font-weight: 600;
      margin-bottom: 5px;
      color: #333;
      font-size: 0.9em;
    }}
    
    .control-group select,
    .control-group input {{
      width: 100%;
      padding: 10px;
      border: 2px solid #ddd;
      border-radius: 6px;
      font-size: 1em;
      transition: all 0.3s;
    }}
    
    .control-group select:focus,
    .control-group input:focus {{
      outline: none;
      border-color: #667eea;
      box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
    }}
    
    .button-group {{
      display: flex;
      gap: 10px;
      flex-wrap: wrap;
    }}
    
    button {{
      padding: 10px 20px;
      border: none;
      border-radius: 6px;
      font-size: 1em;
      font-weight: 600;
      cursor: pointer;
      transition: all 0.3s;
      display: inline-flex;
      align-items: center;
      gap: 8px;
    }}
    
    .btn-primary {{
      background: #667eea;
      color: white;
    }}
    
    .btn-primary:hover {{
      background: #5568d3;
      transform: translateY(-2px);
      box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
    }}
    
    .btn-secondary {{
      background: #48bb78;
      color: white;
    }}
    
    .btn-secondary:hover {{
      background: #38a169;
      transform: translateY(-2px);
      box-shadow: 0 4px 12px rgba(72, 187, 120, 0.4);
    }}
    
    .btn-info {{
      background: #4299e1;
      color: white;
    }}
    
    .btn-info:hover {{
      background: #3182ce;
      transform: translateY(-2px);
      box-shadow: 0 4px 12px rgba(66, 153, 225, 0.4);
    }}
    
    .tabs {{
      display: flex;
      background: #e0e0e0;
      padding: 0;
      margin: 0;
    }}
    
    .tab {{
      flex: 1;
      padding: 15px;
      background: #e0e0e0;
      border: none;
      cursor: pointer;
      font-size: 1em;
      font-weight: 600;
      transition: all 0.3s;
      border-bottom: 3px solid transparent;
    }}
    
    .tab:hover {{
      background: #d0d0d0;
    }}
    
    .tab.active {{
      background: white;
      border-bottom-color: #667eea;
      color: #667eea;
    }}
    
    .tab-content {{
      display: none;
      padding: 25px;
    }}
    
    .tab-content.active {{
      display: block;
    }}
    
    #plotDiv {{
      width: 100%;
      height: 600px;
      margin-top: 20px;
    }}
    
    .stats-grid {{
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
      gap: 20px;
      margin-top: 20px;
    }}
    
    .stat-card {{
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      color: white;
      padding: 20px;
      border-radius: 8px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    }}
    
    .stat-card .label {{
      font-size: 0.9em;
      opacity: 0.9;
      margin-bottom: 5px;
    }}
    
    .stat-card .value {{
      font-size: 2em;
      font-weight: bold;
    }}
    
    .info-box {{
      background: #e3f2fd;
      border-left: 4px solid #2196f3;
      padding: 15px;
      margin-top: 20px;
      border-radius: 4px;
    }}
    
    .info-box h3 {{
      color: #1976d2;
      margin-bottom: 10px;
    }}
    
    .checkbox-group {{
      display: flex;
      flex-wrap: wrap;
      gap: 15px;
      margin-top: 10px;
    }}
    
    .checkbox-item {{
      display: flex;
      align-items: center;
      gap: 5px;
    }}
    
    .checkbox-item input[type="checkbox"] {{
      width: auto;
      cursor: pointer;
    }}
    
    .loading {{
      display: none;
      text-align: center;
      padding: 40px;
      font-size: 1.2em;
      color: #667eea;
    }}
    
    .loading.active {{
      display: block;
    }}
    
    @media (max-width: 768px) {{
      .control-row {{
        flex-direction: column;
      }}
      
      .header h1 {{
        font-size: 1.8em;
      }}
      
      #plotDiv {{
        height: 400px;
      }}
    }}
  </style>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1>📊 Análisis de Tarifas Eléctricas</h1>
      <div class="subtitle">Sistema de Visualización Interactiva - GTR</div>
      <div class="subtitle" style="margin-top:10px; font-size:0.9em;">
        Generado: {self.data['metadata']['fecha_generacion']}
      </div>
    </div>
    
    <div class="controls-panel">
      <div class="control-row">
        <div class="control-group">
          <label for="selTarifa">🔹 Tarifa:</label>
          <select id="selTarifa"></select>
        </div>
        
        <div class="control-group">
          <label for="selDivision">🔹 División:</label>
          <select id="selDivision">
            <option value="TODAS">Todas las divisiones</option>
          </select>
        </div>
        
        <div class="control-group">
          <label for="selSegmento">🔹 Segmento:</label>
          <select id="selSegmento">
            <option value="TODOS">Todos los segmentos</option>
          </select>
        </div>
      </div>
      
      <div class="control-row">
        <div class="control-group">
          <label for="dateStart">📅 Fecha inicio:</label>
          <input type="date" id="dateStart">
        </div>
        
        <div class="control-group">
          <label for="dateEnd">📅 Fecha fin:</label>
          <input type="date" id="dateEnd">
        </div>
      </div>
      
      <div class="control-row">
        <div class="button-group">
          <button class="btn-primary" onclick="actualizarGrafico()">
            🔄 Actualizar Gráfico
          </button>
          <button class="btn-secondary" onclick="exportarDatos()">
            💾 Exportar CSV
          </button>
          <button class="btn-info" onclick="mostrarEstadisticas()">
            📈 Ver Estadísticas
          </button>
        </div>
      </div>
    </div>
    
    <div class="tabs">
      <button class="tab active" onclick="cambiarTab('grafico')">📊 Gráfico Principal</button>
      <button class="tab" onclick="cambiarTab('comparativa')">🔄 Comparativa</button>
      <button class="tab" onclick="cambiarTab('estadisticas')">📈 Estadísticas</button>
      <button class="tab" onclick="cambiarTab('ayuda')">❓ Ayuda</button>
    </div>
    
    <div id="tabGrafico" class="tab-content active">
      <div class="loading" id="loading">⏳ Cargando datos...</div>
      <div id="plotDiv"></div>
    </div>
    
    <div id="tabComparativa" class="tab-content">
      <h2>Comparativa de Tarifas por Segmento</h2>
      
      <div class="control-row" style="margin-top: 20px;">
        <div class="control-group">
          <label>🔹 Modo de comparación:</label>
          <select id="modoComparativa">
            <option value="promedio">Promedio General de Tarifa</option>
            <option value="segmento">Por Segmento Específico</option>
            <option value="concepto">Por Concepto Específico</option>
          </select>
        </div>
        
        <div class="control-group" id="filtroSegmentoComp" style="display:none;">
          <label>🔹 Segmento a comparar:</label>
          <select id="segmentoComparativa"></select>
        </div>
        
        <div class="control-group" id="filtroConceptoComp" style="display:none;">
          <label>🔹 Concepto a comparar:</label>
          <select id="conceptoComparativa"></select>
        </div>
        
        <div class="control-group" id="filtroDivisionComp">
          <label>🔹 División:</label>
          <select id="divisionComparativa">
            <option value="TODAS">Todas (Promedio)</option>
          </select>
        </div>
      </div>
      
      <div class="control-row">
        <div class="control-group">
          <label>✅ Seleccionar tarifas a comparar:</label>
          <div id="tarifasCompare" class="checkbox-group"></div>
        </div>
      </div>
      
      <div class="control-row">
        <button class="btn-primary" onclick="actualizarComparativa()">
          🔄 Actualizar Comparativa
        </button>
        <button class="btn-secondary" onclick="exportarComparativa()">
          💾 Exportar Comparativa CSV
        </button>
      </div>
      
      <div id="infoComparativa" class="info-box" style="margin-top: 20px; display:none;">
        <h3>📊 Información de la Comparativa</h3>
        <div id="infoComparativaTexto"></div>
      </div>
      
      <div id="plotCompare" style="width:100%; height:600px; margin-top:20px;"></div>
    </div>
    
    <div id="tabEstadisticas" class="tab-content">
      <h2>Estadísticas Generales</h2>
      <div class="stats-grid" id="statsGrid"></div>
      <div id="statsTable" style="margin-top: 30px; overflow-x: auto;"></div>
    </div>
    
    <div id="tabAyuda" class="tab-content">
      <div class="info-box">
        <h3>📖 Guía de Uso</h3>
        <p><strong>Controles principales:</strong></p>
        <ul style="margin: 15px 0 15px 20px; line-height: 1.8;">
          <li><strong>Tarifa:</strong> Selecciona la tarifa a visualizar</li>
          <li><strong>División:</strong> Filtra por división específica o muestra todas</li>
          <li><strong>Segmento:</strong> Filtra por segmento (Energía, Distribución, etc.)</li>
          <li><strong>Fechas:</strong> Establece un rango temporal específico</li>
        </ul>
        
        <p><strong>Interacción con gráficos:</strong></p>
        <ul style="margin: 15px 0 15px 20px; line-height: 1.8;">
          <li>🖱️ <strong>Hover:</strong> Pasa el mouse sobre las líneas para ver valores detallados</li>
          <li>🔍 <strong>Zoom:</strong> Arrastra para hacer zoom en una región</li>
          <li>📌 <strong>Click en leyenda:</strong> Oculta/muestra series específicas</li>
          <li>🏠 <strong>Resetear:</strong> Doble click para restaurar vista original</li>
        </ul>
        
        <p><strong>Características avanzadas:</strong></p>
        <ul style="margin: 15px 0 15px 20px; line-height: 1.8;">
          <li>💾 <strong>Exportar:</strong> Descarga los datos filtrados en formato CSV</li>
          <li>📊 <strong>Comparativa:</strong> Compara múltiples tarifas simultáneamente</li>
          <li>📈 <strong>Estadísticas:</strong> Analiza métricas clave de cada serie</li>
        </ul>
      </div>
      
      <div class="info-box" style="margin-top: 20px; background: #fff3cd; border-left-color: #ffc107;">
        <h3 style="color: #856404;">💡 Consejos</h3>
        <ul style="margin: 15px 0 15px 20px; line-height: 1.8;">
          <li>Para mejor visualización, filtra por división o segmento específico</li>
          <li>Los cambios porcentuales mensuales se muestran al pasar el mouse</li>
          <li>Usa la comparativa para analizar diferencias entre tarifas</li>
          <li>Las estadísticas incluyen promedios, máximos, mínimos y tendencias</li>
        </ul>
      </div>
    </div>
  </div>

<script>
const data = {data_js};
const statsData = {stats_js};

// Variables globales
let currentTarifa = null;
let currentDivision = 'TODAS';
let currentSegmento = 'TODOS';
let currentDateStart = null;
let currentDateEnd = null;

// ===== INICIALIZACIÓN =====
function init() {{
  poblarTarifas();
  configurarFechas();
  if (Object.keys(data.tarifas).length > 0) {{
    const primeraTarifa = Object.keys(data.tarifas)[0];
    document.getElementById('selTarifa').value = primeraTarifa;
    actualizarFiltrosDependientes();
    actualizarGrafico();
  }}
  generarComparativa();
  mostrarEstadisticas();
}}

function poblarTarifas() {{
  const sel = document.getElementById('selTarifa');
  sel.innerHTML = '';
  
  Object.keys(data.tarifas).sort().forEach(tarifa => {{
    const opt = document.createElement('option');
    opt.value = tarifa;
    opt.text = tarifa;
    sel.appendChild(opt);
  }});
  
  sel.addEventListener('change', () => {{
    actualizarFiltrosDependientes();
    actualizarGrafico();
  }});
}}

function configurarFechas() {{
  const fechas = data.metadata.rango_fechas;
  if (fechas && fechas.length === 2) {{
    document.getElementById('dateStart').value = fechas[0];
    document.getElementById('dateEnd').value = fechas[1];
    currentDateStart = fechas[0];
    currentDateEnd = fechas[1];
  }}
}}

function actualizarFiltrosDependientes() {{
  currentTarifa = document.getElementById('selTarifa').value;
  
  if (!currentTarifa || !data.tarifas[currentTarifa]) return;
  
  // Actualizar divisiones
  const selDiv = document.getElementById('selDivision');
  selDiv.innerHTML = '<option value="TODAS">Todas las divisiones</option>';
  
  data.tarifas[currentTarifa].divisiones.forEach(div => {{
    const opt = document.createElement('option');
    opt.value = div;
    opt.text = div;
    selDiv.appendChild(opt);
  }});
  
  // Actualizar segmentos
  const selSeg = document.getElementById('selSegmento');
  selSeg.innerHTML = '<option value="TODOS">Todos los segmentos</option>';
  
  const segmentos = new Set();
  data.tarifas[currentTarifa].series.forEach(s => {{
    if (s.segmento) segmentos.add(s.segmento);
  }});
  
  Array.from(segmentos).sort().forEach(seg => {{
    const opt = document.createElement('option');
    opt.value = seg;
    opt.text = seg;
    selSeg.appendChild(opt);
  }});
}}

// ===== GRÁFICO PRINCIPAL =====
function actualizarGrafico() {{
  const loading = document.getElementById('loading');
  loading.classList.add('active');
  
  currentTarifa = document.getElementById('selTarifa').value;
  currentDivision = document.getElementById('selDivision').value;
  currentSegmento = document.getElementById('selSegmento').value;
  currentDateStart = document.getElementById('dateStart').value;
  currentDateEnd = document.getElementById('dateEnd').value;
  
  if (!currentTarifa || !data.tarifas[currentTarifa]) {{
    loading.classList.remove('active');
    return;
  }}
  
  const series = data.tarifas[currentTarifa].series;
  
  // Filtrar series según criterios
  const seriesFiltradas = series.filter(s => {{
    const divMatch = currentDivision === 'TODAS' || s.division === currentDivision;
    const segMatch = currentSegmento === 'TODOS' || s.segmento === currentSegmento;
    return divMatch && segMatch;
  }});
  
  // Construir traces para Plotly
  const traces = seriesFiltradas.map(s => {{
    // Filtrar por rango de fechas
    const indices = s.fechas.map((f, i) => {{
      if (currentDateStart && f < currentDateStart) return null;
      if (currentDateEnd && f > currentDateEnd) return null;
      return i;
    }}).filter(i => i !== null);
    
    const fechasFiltradas = indices.map(i => s.fechas[i]);
    const valoresFiltrados = indices.map(i => s.valores[i]);
    const pctFiltrados = indices.map(i => s.pct[i]);
    
    return {{
      x: fechasFiltradas,
      y: valoresFiltrados,
      name: s.nombre,
      mode: 'lines+markers',
      line: {{ width: 2 }},
      marker: {{ size: 6 }},
      text: name,
      customdata: pctFiltrados,
      hovertemplate:
        '<b>%{{text}}</b><br>' +
        'División: ' + s.division + '<br>' +
        'Fecha: %{{x}}<br>' +
        'Valor: %{{y:.4f}}<br>' +
        'Cambio mensual: %{{customdata:.2f}}%' +
        '<extra></extra>',
      connectgaps: false
    }};
  }});
  
  const layout = {{
    title: {{
      text: `Tarifa: ${{currentTarifa}} | División: ${{currentDivision}} | Segmento: ${{currentSegmento}}`,
      font: {{ size: 20, color: '#333' }}
    }},
    xaxis: {{
      title: 'Fecha',
      tickangle: -45,
      gridcolor: '#e0e0e0'
    }},
    yaxis: {{
      title: 'Valor ($/kWh)',
      gridcolor: '#e0e0e0'
    }},
    legend: {{
      orientation: 'v',
      yanchor: 'top',
      y: 1,
      xanchor: 'left',
      x: 1.02,
      bgcolor: 'rgba(255,255,255,0.8)',
      bordercolor: '#ccc',
      borderwidth: 1
    }},
    hovermode: 'closest',
    plot_bgcolor: '#fafafa',
    paper_bgcolor: '#fff',
    margin: {{ l: 80, r: 250, t: 100, b: 100 }},
    font: {{ family: 'Segoe UI, sans-serif' }}
  }};
  
  const config = {{
    responsive: true,
    displayModeBar: true,
    displaylogo: false,
    modeBarButtonsToRemove: ['lasso2d', 'select2d']
  }};
  
  Plotly.newPlot('plotDiv', traces, layout, config);
  loading.classList.remove('active');
}}

// ===== COMPARATIVA =====
function generarComparativa() {{
  const container = document.getElementById('tarifasCompare');
  container.innerHTML = '';
  
  Object.keys(data.tarifas).sort().forEach(tarifa => {{
    const label = document.createElement('label');
    label.className = 'checkbox-item';
    
    const checkbox = document.createElement('input');
    checkbox.type = 'checkbox';
    checkbox.value = tarifa;
    checkbox.id = `cmp_${{tarifa}}`;
    
    label.appendChild(checkbox);
    label.appendChild(document.createTextNode(` ${{tarifa}}`));
    container.appendChild(label);
  }});
  
  // Poblar opciones de segmentos y conceptos
  poblarOpcionesComparativa();
  
  // Eventos de cambio de modo
  document.getElementById('modoComparativa').addEventListener('change', cambiarModoComparativa);
}}

function poblarOpcionesComparativa() {{
  // Obtener todos los segmentos y conceptos únicos
  const segmentos = new Set();
  const conceptos = new Set();
  const divisiones = new Set();
  
  Object.values(data.tarifas).forEach(tarifa => {{
    tarifa.series.forEach(s => {{
      if (s.segmento) segmentos.add(s.segmento);
      if (s.concepto) conceptos.add(s.concepto);
      if (s.division) divisiones.add(s.division);
    }});
  }});
  
  // Poblar segmentos
  const selSegmento = document.getElementById('segmentoComparativa');
  selSegmento.innerHTML = '';
  Array.from(segmentos).sort().forEach(seg => {{
    const opt = document.createElement('option');
    opt.value = seg;
    opt.text = seg;
    selSegmento.appendChild(opt);
  }});
  
  // Poblar conceptos
  const selConcepto = document.getElementById('conceptoComparativa');
  selConcepto.innerHTML = '';
  Array.from(conceptos).sort().forEach(con => {{
    const opt = document.createElement('option');
    opt.value = con;
    opt.text = con;
    selConcepto.appendChild(opt);
  }});
  
  // Poblar divisiones
  const selDivision = document.getElementById('divisionComparativa');
  selDivision.innerHTML = '<option value="TODAS">Todas (Promedio)</option>';
  Array.from(divisiones).sort().forEach(div => {{
    const opt = document.createElement('option');
    opt.value = div;
    opt.text = div;
    selDivision.appendChild(opt);
  }});
}}

function cambiarModoComparativa() {{
  const modo = document.getElementById('modoComparativa').value;
  
  // Mostrar/ocultar filtros según el modo
  document.getElementById('filtroSegmentoComp').style.display = 
    modo === 'segmento' ? 'block' : 'none';
  document.getElementById('filtroConceptoComp').style.display = 
    modo === 'concepto' ? 'block' : 'none';
}}

function actualizarComparativa() {{
  const checkboxes = document.querySelectorAll('#tarifasCompare input[type="checkbox"]:checked');
  const tarifasSeleccionadas = Array.from(checkboxes).map(cb => cb.value);
  
  if (tarifasSeleccionadas.length === 0) {{
    document.getElementById('plotCompare').innerHTML = 
      '<p style="text-align:center; padding:40px; color:#999;">Selecciona al menos una tarifa para comparar</p>';
    document.getElementById('infoComparativa').style.display = 'none';
    return;
  }}
  
  const modo = document.getElementById('modoComparativa').value;
  const divisionSeleccionada = document.getElementById('divisionComparativa').value;
  
  let traces = [];
  let tituloGrafico = '';
  let infoTexto = '';
  
  if (modo === 'promedio') {{
    // Modo: Promedio General
    const result = compararPromedioGeneral(tarifasSeleccionadas, divisionSeleccionada);
    traces = result.traces;
    tituloGrafico = 'Comparativa de Tarifas - Promedio General';
    infoTexto = `Comparando <strong>${{tarifasSeleccionadas.length}}</strong> tarifa(s) por promedio general`;
    if (divisionSeleccionada !== 'TODAS') {{
      tituloGrafico += ` (División: ${{divisionSeleccionada}})`;
      infoTexto += ` en la división <strong>${{divisionSeleccionada}}</strong>`;
    }}
    
  }} else if (modo === 'segmento') {{
    // Modo: Por Segmento
    const segmento = document.getElementById('segmentoComparativa').value;
    const result = compararPorSegmento(tarifasSeleccionadas, segmento, divisionSeleccionada);
    traces = result.traces;
    tituloGrafico = `Comparativa de Tarifas - Segmento: ${{segmento}}`;
    infoTexto = `Comparando segmento <strong>${{segmento}}</strong> entre <strong>${{tarifasSeleccionadas.length}}</strong> tarifa(s)`;
    
    if (divisionSeleccionada !== 'TODAS') {{
      tituloGrafico += ` (División: ${{divisionSeleccionada}})`;
      infoTexto += ` en la división <strong>${{divisionSeleccionada}}</strong>`;
    }}
    infoTexto += `<br>Series encontradas: <strong>${{result.seriesEncontradas}}</strong>`;
    
  }} else if (modo === 'concepto') {{
    // Modo: Por Concepto
    const concepto = document.getElementById('conceptoComparativa').value;
    const result = compararPorConcepto(tarifasSeleccionadas, concepto, divisionSeleccionada);
    traces = result.traces;
    tituloGrafico = `Comparativa de Tarifas - Concepto: ${{concepto}}`;
    infoTexto = `Comparando concepto <strong>${{concepto}}</strong> entre <strong>${{tarifasSeleccionadas.length}}</strong> tarifa(s)`;
    
    if (divisionSeleccionada !== 'TODAS') {{
      tituloGrafico += ` (División: ${{divisionSeleccionada}})`;
      infoTexto += ` en la división <strong>${{divisionSeleccionada}}</strong>`;
    }}
    infoTexto += `<br>Series encontradas: <strong>${{result.seriesEncontradas}}</strong>`;
  }}
  
  if (traces.length === 0) {{
    document.getElementById('plotCompare').innerHTML = 
      '<p style="text-align:center; padding:40px; color:#f56565;">⚠️ No se encontraron datos para los filtros seleccionados</p>';
    document.getElementById('infoComparativa').style.display = 'none';
    return;
  }}
  
  // Mostrar información
  document.getElementById('infoComparativa').style.display = 'block';
  document.getElementById('infoComparativaTexto').innerHTML = infoTexto;
  
  const layout = {{
    title: {{
      text: tituloGrafico,
      font: {{ size: 18 }}
    }},
    xaxis: {{ 
      title: 'Fecha', 
      tickangle: -45,
      gridcolor: '#e0e0e0'
    }},
    yaxis: {{ 
      title: 'Valor ($/kWh)',
      gridcolor: '#e0e0e0'
    }},
    hovermode: 'x unified',
    legend: {{
      orientation: 'v',
      yanchor: 'top',
      y: 1,
      xanchor: 'left',
      x: 1.02,
      bgcolor: 'rgba(255,255,255,0.9)',
      bordercolor: '#ccc',
      borderwidth: 1
    }},
    plot_bgcolor: '#fafafa',
    margin: {{ l: 80, r: 220, t: 80, b: 100 }}
  }};
  
  const config = {{
    responsive: true,
    displayModeBar: true,
    displaylogo: false
  }};
  
  Plotly.newPlot('plotCompare', traces, layout, config);
}}

function compararPromedioGeneral(tarifas, division) {{
  const traces = [];
  
  tarifas.forEach(tarifa => {{
    const series = data.tarifas[tarifa].series;
    
    // Filtrar por división si es necesario
    const seriesFiltradas = division === 'TODAS' 
      ? series 
      : series.filter(s => s.division === division);
    
    if (seriesFiltradas.length === 0) return;
    
    // Agrupar valores por fecha
    const valoresPorFecha = {{}};
    seriesFiltradas.forEach(s => {{
      s.fechas.forEach((f, i) => {{
        if (s.valores[i] !== null) {{
          if (!valoresPorFecha[f]) valoresPorFecha[f] = [];
          valoresPorFecha[f].push(s.valores[i]);
        }}
      }});
    }});
    
    // Calcular promedios
    const fechas = Object.keys(valoresPorFecha).sort();
    const promedios = fechas.map(f => {{
      const vals = valoresPorFecha[f];
      return vals.reduce((a, b) => a + b, 0) / vals.length;
    }});
    
    traces.push({{
      x: fechas,
      y: promedios,
      name: tarifa,
      mode: 'lines+markers',
      line: {{ width: 3 }},
      marker: {{ size: 7 }},
      hovertemplate: '<b>%{{fullData.name}}</b><br>Fecha: %{{x}}<br>Promedio: %{{y:.4f}}<extra></extra>'
    }});
  }});
  
  return {{ traces }};
}}

function compararPorSegmento(tarifas, segmento, division) {{
  const traces = [];
  let totalSeriesEncontradas = 0;
  
  tarifas.forEach(tarifa => {{
    const series = data.tarifas[tarifa].series;
    
    // Filtrar por segmento y división
    const seriesFiltradas = series.filter(s => {{
      const segMatch = s.segmento === segmento;
      const divMatch = division === 'TODAS' || s.division === division;
      return segMatch && divMatch;
    }});
    
    if (seriesFiltradas.length === 0) return;
    
    totalSeriesEncontradas += seriesFiltradas.length;
    
    // Si hay múltiples series del mismo segmento, mostrarlas todas o promediar
    if (seriesFiltradas.length === 1) {{
      // Una sola serie: mostrarla directamente
      const s = seriesFiltradas[0];
      traces.push({{
        x: s.fechas,
        y: s.valores,
        name: `${{tarifa}} - ${{s.nombre}}`,
        mode: 'lines+markers',
        line: {{ width: 2.5 }},
        marker: {{ size: 6 }},
        hovertemplate: '<b>%{{fullData.name}}</b><br>Fecha: %{{x}}<br>Valor: %{{y:.4f}}<extra></extra>'
      }});
    }} else {{
      // Múltiples series: crear un trace por cada una
      seriesFiltradas.forEach(s => {{
        traces.push({{
          x: s.fechas,
          y: s.valores,
          name: `${{tarifa}} - ${{s.division}} - ${{s.concepto}}`,
          mode: 'lines+markers',
          line: {{ width: 2 }},
          marker: {{ size: 5 }},
          hovertemplate: '<b>%{{fullData.name}}</b><br>Fecha: %{{x}}<br>Valor: %{{y:.4f}}<extra></extra>'
        }});
      }});
    }}
  }});
  
  return {{ traces, seriesEncontradas: totalSeriesEncontradas }};
}}

function compararPorConcepto(tarifas, concepto, division) {{
  const traces = [];
  let totalSeriesEncontradas = 0;
  
  tarifas.forEach(tarifa => {{
    const series = data.tarifas[tarifa].series;
    
    // Filtrar por concepto y división
    const seriesFiltradas = series.filter(s => {{
      const conMatch = s.concepto === concepto;
      const divMatch = division === 'TODAS' || s.division === division;
      return conMatch && divMatch;
    }});
    
    if (seriesFiltradas.length === 0) return;
    
    totalSeriesEncontradas += seriesFiltradas.length;
    
    // Mostrar cada serie encontrada
    seriesFiltradas.forEach(s => {{
      traces.push({{
        x: s.fechas,
        y: s.valores,
        name: `${{tarifa}} - ${{s.segmento}} [${{s.division}}]`,
        mode: 'lines+markers',
        line: {{ width: 2 }},
        marker: {{ size: 5 }},
        hovertemplate: '<b>%{{fullData.name}}</b><br>Fecha: %{{x}}<br>Valor: %{{y:.4f}}<extra></extra>'
      }});
    }});
  }});
  
  return {{ traces, seriesEncontradas: totalSeriesEncontradas }};
}}

function exportarComparativa() {{
  const checkboxes = document.querySelectorAll('#tarifasCompare input[type="checkbox"]:checked');
  const tarifasSeleccionadas = Array.from(checkboxes).map(cb => cb.value);
  
  if (tarifasSeleccionadas.length === 0) {{
    alert('Por favor selecciona al menos una tarifa para exportar');
    return;
  }}
  
  const modo = document.getElementById('modoComparativa').value;
  const division = document.getElementById('divisionComparativa').value;
  
  let csv = 'Tarifa,Division,Segmento,Concepto,Serie,Fecha,Valor\\n';
  
  tarifasSeleccionadas.forEach(tarifa => {{
    let seriesFiltradas = data.tarifas[tarifa].series;
    
    // Aplicar filtros según el modo
    if (modo === 'segmento') {{
      const segmento = document.getElementById('segmentoComparativa').value;
      seriesFiltradas = seriesFiltradas.filter(s => s.segmento === segmento);
    }} else if (modo === 'concepto') {{
      const concepto = document.getElementById('conceptoComparativa').value;
      seriesFiltradas = seriesFiltradas.filter(s => s.concepto === concepto);
    }}
    
    if (division !== 'TODAS') {{
      seriesFiltradas = seriesFiltradas.filter(s => s.division === division);
    }}
    
    seriesFiltradas.forEach(s => {{
      s.fechas.forEach((f, i) => {{
        if (s.valores[i] !== null) {{
          csv += `"${{tarifa}}","${{s.division}}","${{s.segmento}}","${{s.concepto}}","${{s.nombre}}",${{f}},${{s.valores[i]}}\\n`;
        }}
      }});
    }});
  }});
  
  // Descargar
  const blob = new Blob([csv], {{ type: 'text/csv;charset=utf-8;' }});
  const link = document.createElement('a');
  const url = URL.createObjectURL(blob);
  
  const timestamp = new Date().toISOString().split('T')[0];
  link.setAttribute('href', url);
  link.setAttribute('download', `comparativa_tarifas_${{modo}}_${{timestamp}}.csv`);
  link.style.visibility = 'hidden';
  
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}}

// ===== ESTADÍSTICAS =====
function mostrarEstadisticas() {{
  const grid = document.getElementById('statsGrid');
  const table = document.getElementById('statsTable');
  
  // Estadísticas generales
  const totalTarifas = Object.keys(data.tarifas).length;
  const totalSeries = Object.values(data.tarifas).reduce((sum, t) => sum + t.series.length, 0);
  
  let totalValores = 0;
  let sumaValores = 0;
  Object.values(data.tarifas).forEach(t => {{
    t.series.forEach(s => {{
      s.valores.forEach(v => {{
        if (v !== null) {{
          totalValores++;
          sumaValores += v;
        }}
      }});
    }});
  }});
  
  const promedioGlobal = totalValores > 0 ? (sumaValores / totalValores).toFixed(4) : 0;
  
  grid.innerHTML = `
    <div class="stat-card">
      <div class="label">Total de Tarifas</div>
      <div class="value">${{totalTarifas}}</div>
    </div>
    <div class="stat-card">
      <div class="label">Total de Series</div>
      <div class="value">${{totalSeries}}</div>
    </div>
    <div class="stat-card">
      <div class="label">Total de Datos</div>
      <div class="value">${{totalValores.toLocaleString()}}</div>
    </div>
    <div class="stat-card">
      <div class="label">Promedio Global</div>
      <div class="value">${{promedioGlobal}} $/kWh</div>
    </div>
  `;
  
  // Tabla detallada
  if (statsData && statsData.length > 0) {{
    let tableHTML = `
      <h3 style="margin-bottom: 15px;">Estadísticas Detalladas por Serie</h3>
      <table style="width:100%; border-collapse: collapse; font-size: 0.9em;">
        <thead>
          <tr style="background: #667eea; color: white;">
            <th style="padding: 12px; text-align: left; border: 1px solid #ddd;">Tarifa</th>
            <th style="padding: 12px; text-align: left; border: 1px solid #ddd;">División</th>
            <th style="padding: 12px; text-align: left; border: 1px solid #ddd;">Serie</th>
            <th style="padding: 12px; text-align: right; border: 1px solid #ddd;">Datos</th>
            <th style="padding: 12px; text-align: right; border: 1px solid #ddd;">Mínimo</th>
            <th style="padding: 12px; text-align: right; border: 1px solid #ddd;">Máximo</th>
            <th style="padding: 12px; text-align: right; border: 1px solid #ddd;">Promedio</th>
            <th style="padding: 12px; text-align: right; border: 1px solid #ddd;">Cambio Total</th>
          </tr>
        </thead>
        <tbody>
    `;
    
    statsData.slice(0, 50).forEach((stat, i) => {{
      const bgColor = i % 2 === 0 ? '#f9f9f9' : 'white';
      const changeColor = stat.change_pct >= 0 ? '#48bb78' : '#f56565';
      
      tableHTML += `
        <tr style="background: ${{bgColor}};">
          <td style="padding: 10px; border: 1px solid #ddd;">${{stat.tarifa}}</td>
          <td style="padding: 10px; border: 1px solid #ddd;">${{stat.division}}</td>
          <td style="padding: 10px; border: 1px solid #ddd; max-width: 300px; overflow: hidden; text-overflow: ellipsis;">${{stat.serie}}</td>
          <td style="padding: 10px; border: 1px solid #ddd; text-align: right;">${{stat.count}}</td>
          <td style="padding: 10px; border: 1px solid #ddd; text-align: right;">${{stat.min.toFixed(4)}}</td>
          <td style="padding: 10px; border: 1px solid #ddd; text-align: right;">${{stat.max.toFixed(4)}}</td>
          <td style="padding: 10px; border: 1px solid #ddd; text-align: right;">${{stat.mean.toFixed(4)}}</td>
          <td style="padding: 10px; border: 1px solid #ddd; text-align: right; color: ${{changeColor}}; font-weight: bold;">
            ${{stat.change_pct >= 0 ? '+' : ''}}${{stat.change_pct.toFixed(2)}}%
          </td>
        </tr>
      `;
    }});
    
    tableHTML += '</tbody></table>';
    
    if (statsData.length > 50) {{
      tableHTML += `<p style="margin-top: 15px; color: #666; text-align: center;">Mostrando las primeras 50 series de ${{statsData.length}} totales</p>`;
    }}
    
    table.innerHTML = tableHTML;
  }}
}}

// ===== EXPORTAR DATOS =====
function exportarDatos() {{
  if (!currentTarifa || !data.tarifas[currentTarifa]) {{
    alert('Por favor selecciona una tarifa primero');
    return;
  }}
  
  const series = data.tarifas[currentTarifa].series;
  
  // Filtrar series
  const seriesFiltradas = series.filter(s => {{
    const divMatch = currentDivision === 'TODAS' || s.division === currentDivision;
    const segMatch = currentSegmento === 'TODOS' || s.segmento === currentSegmento;
    return divMatch && segMatch;
  }});
  
  // Construir CSV
  let csv = 'Tarifa,Division,Segmento,Serie,Fecha,Valor,Cambio_Mensual_%\\n';
  
  seriesFiltradas.forEach(s => {{
    s.fechas.forEach((f, i) => {{
      if (currentDateStart && f < currentDateStart) return;
      if (currentDateEnd && f > currentDateEnd) return;
      
      const valor = s.valores[i] !== null ? s.valores[i] : '';
      const pct = s.pct[i] !== null ? s.pct[i].toFixed(2) : '';
      
      csv += `${{currentTarifa}},"${{s.division}}","${{s.segmento}}","${{s.nombre}}",${{f}},${{valor}},${{pct}}\\n`;
    }});
  }});
  
  // Descargar
  const blob = new Blob([csv], {{ type: 'text/csv;charset=utf-8;' }});
  const link = document.createElement('a');
  const url = URL.createObjectURL(blob);
  
  link.setAttribute('href', url);
  link.setAttribute('download', `tarifas_${{currentTarifa}}_${{new Date().toISOString().split('T')[0]}}.csv`);
  link.style.visibility = 'hidden';
  
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}}

// ===== NAVEGACIÓN TABS =====
function cambiarTab(tabName) {{
  // Ocultar todos los contenidos
  document.querySelectorAll('.tab-content').forEach(tab => {{
    tab.classList.remove('active');
  }});
  
  // Desactivar todos los botones
  document.querySelectorAll('.tab').forEach(btn => {{
    btn.classList.remove('active');
  }});
  
  // Activar tab seleccionado
  document.getElementById(`tab${{tabName.charAt(0).toUpperCase() + tabName.slice(1)}}`).classList.add('active');
  event.target.classList.add('active');
  
  // Actualizar gráficos si es necesario
  if (tabName === 'comparativa') {{
    actualizarComparativa();
  }}
}}

// Inicializar al cargar
window.addEventListener('DOMContentLoaded', init);
</script>

</body>
</html>"""


def main():
    """Función principal."""
    try:
        print("="*60)
        print("🚀 GENERADOR DE TARIFAS INTERACTIVO V2.0")
        print("="*60)
        
        # 1. Cargar y procesar datos
        processor = TarifasProcessor(PATH_XLSX, SHEET_NAME)
        processor.load_data().transform_to_long_format()
        
        # 2. Calcular estadísticas
        stats_df = processor.calculate_statistics()
        
        # 3. Construir estructura JSON
        builder = DataStructureBuilder(processor.df_melt)
        data_structure = builder.build_consolidated_structure()
        
        # 4. Generar HTML
        generator = HTMLGenerator(data_structure, stats_df)
        generator.generate(OUTPUT_HTML)
        
        print("="*60)
        print(f"✅ PROCESO COMPLETADO EXITOSAMENTE")
        print(f"📁 Archivo generado: {OUTPUT_HTML.resolve()}")
        print(f"📊 Total de tarifas: {data_structure['metadata']['total_tarifas']}")
        print(f"📈 Total de registros: {data_structure['metadata']['total_registros']:,}")
        print(f"📅 Rango de fechas: {' - '.join(data_structure['metadata']['rango_fechas'])}")
        print("="*60)
        print("\n🌐 Abre el archivo HTML en tu navegador para ver los resultados")
        
    except FileNotFoundError as e:
        logger.error(f"❌ Archivo no encontrado: {e}")
        logger.info("💡 Verifica que la ruta del archivo Excel sea correcta")
    except Exception as e:
        logger.error(f"❌ Error inesperado: {e}", exc_info=True)
        raise


if __name__ == "__main__":
    main()
