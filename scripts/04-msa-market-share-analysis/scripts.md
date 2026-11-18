# 4. SCRIPTS MSA (MARKET SHARE ANALYSIS)

**Nota:** *Los siguientes scripts están diseñados para el análisis de datos orientado a la estimación de market share, la generación de visualizaciones analíticas y la identificación de insights estratégicos a partir de datos históricos obtenidos de TradeMap. Se sugiere implementar este flujo de trabajo en entornos de ejecución Python tales como:*

- [Google Colab](https://colab.research.google.com/)
- [Python-Fiddle](https://python-fiddle.com/)
- [Kaggle Notebooks](https://www.kaggle.com/code)
- [Deepnote](https://deepnote.com/)

## 4.1. Importar e instalar librerías

```
# Instalar librerías si es necesario
!pip install openpyxl

import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import numpy as np
from google.colab import files
import warnings
warnings.filterwarnings('ignore')

# Configurar estilo de gráficos
plt.style.use('seaborn-v0_8')
sns.set_palette("husl")
```

## 4.2. Cargar archivos fuente (Trademap)

```

import pandas as pd
from google.colab import files
import re

# --- 0. CARGA DE ARCHIVO ---
print("📂 Por favor, seleccione el archivo Excel exportado desde Trade Map:")
uploaded = files.upload()
filename = list(uploaded.keys())[0]

# Leer datos de la primera hoja
df_raw = pd.read_excel(filename, sheet_name=0)

# Limpiar espacios en blanco en los nombres de las columnas
df_raw.columns = df_raw.columns.astype(str).str.strip()

print(f"\n📁 Archivo cargado exitosamente: {filename}")
print(f"📋 Columnas disponibles en el archivo:")
for idx, col in enumerate(df_raw.columns):
    print(f"  [{idx}] {col}")
print("-" * 50)

# --- 1. DETECCIÓN AUTOMÁTICA DE LA COLUMNA DE PAÍSES/SOCIOS ---
posibles_cabeceras_pais = [
    'exportadores', 'importadores', 'mercados', 'socios', 'país', 'pais',
    'exporters', 'importers', 'markets', 'partners', 'country', 'countries',
    'partner country', 'país socio'
]

col_pais = next(
    (col for col in df_raw.columns if any(palabra in str(col).lower() for palabra in posibles_cabeceras_pais)),
    None
)

# --- 2. ASIGNACIÓN MANUAL (FALLBACK) SI LA DETECCIÓN AUTOMÁTICA FALLA ---
if col_pais is None:
    print("\n⚠️ ADVERTENCIA: No se pudo detectar automáticamente la columna de países/socios.")
    print("Por favor, seleccione manualmente la columna ingresando su número de índice de la lista superior:")
    
    while True:
        try:
            idx_manual = int(input("Ingrese el número de la columna de países: "))
            if 0 <= idx_manual < len(df_raw.columns):
                col_pais = df_raw.columns[idx_manual]
                print(f"✅ Columna seleccionada manualmente con éxito: '{col_pais}'")
                break
            else:
                print(f"❌ Error: El índice debe estar entre 0 y {len(df_raw.columns)-1}. Intente de nuevo.")
        except ValueError:
            print("❌ Entrada inválida. Por favor, ingrese un número entero válido.")
else:
    print(f"✅ Columna de países detectada automáticamente: '{col_pais}'")

# --- 3. DETECCIÓN AUTOMÁTICA DE COLUMNAS DE AÑOS ---
year_columns = [col for col in df_raw.columns if re.search(r'\b(19|20)\d{2}\b', str(col))]

if not year_columns:
    print("\n⚠️ ADVERTENCIA: No se detectaron columnas de años automáticamente.")
    print("Columnas disponibles:", df_raw.columns.tolist())
    raise ValueError("❌ No se encontraron columnas de años válidas (formato 19xx o 20xx) en el archivo.")

print(f"✅ Columnas de años detectadas ({len(year_columns)} en total): {year_columns}\n")

# --- 4. TRANSFORMACIÓN Y NORMALIZACIÓN DE DATOS (MELT) ---
df_melted = df_raw.melt(
    id_vars=[col_pais],
    value_vars=year_columns,
    var_name='year_column',
    value_name='trade_value_usd'
)

# Extraer el año numérico y renombrar la columna de país
df_melted['year'] = df_melted['year_column'].str.extract(r'(\d{4})').astype(int)
df_melted = df_melted.rename(columns={col_pais: 'partner_country'})

# Limpieza y conversión estricta de valores comerciales a numéricos
df_melted['trade_value_usd'] = pd.to_numeric(
    df_melted['trade_value_usd'].astype(str).str.replace(r'[^0-9.\-]', '', regex=True), 
    errors='coerce'
)

df = df_melted[['year', 'partner_country', 'trade_value_usd']].copy()

# Eliminar nulos y filas de totales ("Mundo", "World", "Total")
df = df.dropna(subset=['trade_value_usd'])
excluidos = ['mundo', 'world', 'total', 'grand total', 'total general']
df = df[~df['partner_country'].astype(str).str.strip().str.lower().isin(excluidos)].reset_index(drop=True)

# --- 5. RESULTADO FINAL Y VERIFICACIÓN ---
print(f"📊 Datos procesados correctamente: {len(df)} registros resultantes.")
print(f"🌍 Países únicos analizados: {df['partner_country'].nunique()}")
print(f"📅 Rango de años: {sorted(df['year'].unique())}")
print("\nPrimeros 10 registros normalizados:")
print(df.head(10))
```

## 4.3. Calcular Market Share, Ranking y aplicar filtros

```
# Calcular market share por año y país
market_evolution = df.groupby(['year', 'partner_country'])['trade_value_usd'].sum().reset_index()

# Calcular market share total por año
yearly_totals = market_evolution.groupby('year')['trade_value_usd'].sum()
market_evolution['market_share'] = market_evolution.apply(
    lambda x: (x['trade_value_usd'] / yearly_totals[x['year']]) * 100, axis=1
)

# Identificar top países
top_countries = df.groupby('partner_country')['trade_value_usd'].sum().nlargest(8).index

# Filtrar solo top países
market_evolution_top = market_evolution[market_evolution['partner_country'].isin(top_countries)]
```

## 4.4.  Generar Tablas Pivot para análisis

```
# TABLA 1: Market Share por año y país
print("📊 TABLA 1: MARKET SHARE POR AÑO (%)")
print("="*60)
pivot_share = market_evolution_top.pivot(index='year', columns='partner_country', values='market_share')
pivot_share_rounded = pivot_share.round(1)
print(pivot_share_rounded.to_string())

# TABLA 2: Valores de comercio (Millones USD)
print("\n\n💰 TABLA 2: VALORES DE COMERCIO (Millones USD)")
print("="*60)
pivot_value = market_evolution_top.pivot(index='year', columns='partner_country', values='trade_value_usd')
pivot_value_millions = (pivot_value / 1000).round(1)  # Convertir a millones
print(pivot_value_millions.to_string())

# TABLA 3: Crecimiento anual por país
print("\n\n📈 TABLA 3: CRECIMIENTO ANUAL COMPUESTO (CAGR %)")
print("="*60)
growth_data = market_evolution_top.groupby('partner_country').apply(
    lambda x: ((x['trade_value_usd'].iloc[-1] / x['trade_value_usd'].iloc[0]) ** (1/(len(x)-1)) - 1) * 100
).reset_index(name='cagr')
growth_summary = growth_data.set_index('partner_country')['cagr'].round(1).sort_values(ascending=False)
print(growth_summary.to_string())

# TABLA 4: Ranking por año
print("\n\n🏆 TABLA 4: RANKING DE PAÍSES POR AÑO (Top 5)")
print("="*60)
for year in sorted(df['year'].unique()):
    year_data = market_evolution_top[market_evolution_top['year'] == year].nlargest(5, 'market_share')
    print(f"\n{year}:")
    for i, (_, row) in enumerate(year_data.iterrows(), 1):
        print(f"  {i}. {row['partner_country']}: {row['market_share']:.1f}% ({row['trade_value_usd']/1000:.0f}M USD)")
```

## 4.5. Generar Gráficos analíticos

```
import ipywidgets as widgets
from IPython.display import display, clear_output
import matplotlib.pyplot as plt
import seaborn as sns
import pandas as pd
import re

# ==========================================
# 1. DEFINICIÓN DE PALETAS PREDETERMINADAS (HEX)
# ==========================================
paletas_preset = {
    "1. Ejecutivo Clásico": ["#1f4e78", "#2e75b6", "#5b9bd5", "#8ea9db", "#b4c6e7", "#d9e1f2"],
    "2. Comercio Global": ["#134f5c", "#2986cc", "#38761d", "#6aa84f", "#93c47d", "#b6d7a8"],
    "3. Corporativo Contemporáneo": ["#4a3b32", "#8c6d53", "#c29b7c", "#d9bfa9", "#f2eae1", "#594f43"]
}

# Widgets de la interfaz interactiva con selección directa (RadioButtons)
selector_paleta = widgets.RadioButtons(
    options=list(paletas_preset.keys()) + ["4. Personalizada (CSS)"],
    value="1. Ejecutivo Clásico",
    description='🎨 Paleta:',
    style={'description_width': 'initial'}
)

input_css = widgets.Textarea(
    value='',
    placeholder='Pega aquí tu código CSS de Coolors (ej: --primary: #1f4e78; --secondary: #2e75b6; ... o códigos HEX directos)',
    description='Código CSS:',
    disabled=True,
    layout=widgets.Layout(width='100%', height='100px')
)

boton_generar = widgets.Button(
    description='Aplicar Paleta y Generar Gráficos',
    button_style='success',
    icon='check',
    layout=widgets.Layout(width='280px', height='40px')
)

output_graficos = widgets.Output()

# Habilitar o deshabilitar caja de texto según la selección en los RadioButtons
def al_cambiar_opcion(change):
    if change['type'] == 'change' and change['name'] == 'value':
        input_css.disabled = (change['new'] != "4. Personalizada (CSS)")

selector_paleta.observe(al_cambiar_opcion)

# ==========================================
# 2. FUNCIÓN QUE GENERA LOS 4 GRÁFICOS AL HACER CLIC
# ==========================================
def al_hacer_clic_generar(b):
    with output_graficos:
        clear_output()
        
        # Obtener paleta seleccionada
        opcion = selector_paleta.value
        if opcion != "4. Personalizada (CSS)":
            colores_activos = paletas_preset[opcion]
            print(f"✅ Paleta predeterminada aplicada: '{opcion}'")
        else:
            css_text = input_css.value.strip()
            if not css_text:
                print("⚠️ ADVERTENCIA: La caja de código CSS está vacía. Se utilizará la paleta '1. Ejecutivo Clásico' por defecto.")
                colores_activos = paletas_preset["1. Ejecutivo Clásico"]
            else:
                # Extraer códigos HEX (soporta formatos de 6 u 8 caracteres)
                hex_brutos = re.findall(r'#([A-Fa-f0-9]{8}|[A-Fa-f0-9]{6})', css_text)
                if not hex_brutos:
                    print("❌ ERROR: No se encontraron colores HEX válidos en el código CSS proporcionado. Se utilizará la paleta '1. Ejecutivo Clásico' por defecto.")
                    colores_activos = paletas_preset["1. Ejecutivo Clásico"]
                else:
                    hex_limpios = [f"#{h[:6].lower()}" for h in hex_brutos]
                    colores_activos = list(dict.fromkeys(hex_limpios)) # Eliminar duplicados manteniendo orden
                    print(f"✅ ¡Código CSS procesado con éxito! Se extrajeron {len(colores_activos)} colores únicos.")

        # Aplicar configuración visual a Seaborn y Matplotlib
        sns.set_theme(style="whitegrid")
        sns.set_palette(colores_activos)
        plt.rcParams['figure.figsize'] = (12, 6)

        # Validar existencia del DataFrame 'df' del apartado 4.2
        if 'df' not in globals():
            print("❌ ERROR CRÍTICO: No se encuentra el DataFrame 'df'. Asegúrate de ejecutar primero el apartado 4.2.")
            return

        print("\n📊 Generando visualizaciones analíticas con la paleta seleccionada...")

        # --- Gráfico 1: Market Share Trends ---
        plt.figure()
        pivot_trends = df.pivot(index='year', columns='partner_country', values='trade_value_usd').fillna(0)
        market_share = pivot_trends.div(pivot_trends.sum(axis=1), axis=0) * 100

        for i, col in enumerate(market_share.columns[:len(colores_activos)]):
            plt.plot(market_share.index, market_share[col], marker='o', linewidth=2.5, color=colores_activos[i % len(colores_activos)], label=col)

        plt.title('Market Share Trends (%) por País Socio', fontsize=14, fontweight='bold')
        plt.xlabel('Año', fontsize=12)
        plt.ylabel('Participación de Mercado (%)', fontsize=12)
        plt.legend(bbox_to_anchor=(1.05, 1), loc='upper left')
        plt.tight_layout()
        plt.show()

        # --- Gráfico 2: Trade Value Evolution ---
        plt.figure()
        pivot_trends.iloc[:, :len(colores_activos)].plot(kind='bar', stacked=True, color=colores_activos)
        plt.title('Trade Value Evolution (USD)', fontsize=14, fontweight='bold')
        plt.xlabel('Año', fontsize=12)
        plt.ylabel('Valor Comercial (USD)', fontsize=12)
        plt.legend(bbox_to_anchor=(1.05, 1), loc='upper left')
        plt.tight_layout()
        plt.show()

        # --- Gráfico 3: Market Share Actual ---
        plt.figure()
        ultimo_anio = market_share.index.max()
        top_actual = market_share.loc[ultimo_anio].nlargest(min(5, len(colores_activos)))
        plt.pie(top_actual, labels=top_actual.index, colors=colores_activos[:len(top_actual)], autopct='%1.1f%%', startangle=140)
        plt.title(f'Market Share Actual ({ultimo_anio})', fontsize=14, fontweight='bold')
        plt.tight_layout()
        plt.show()

        # --- Gráfico 4: CAGR ---
        paises_unicos = df['partner_country'].unique()
        cagr_data = {}
        for p in paises_unicos:
            sub = df[df['partner_country'] == p].sort_values('year')
            if len(sub) >= 2:
                val_inicial = sub.iloc[0]['trade_value_usd']
                val_final = sub.iloc[-1]['trade_value_usd']
                n_anios = sub.iloc[-1]['year'] - sub.iloc[0]['year']
                if val_inicial > 0 and n_anios > 0:
                    cagr = ((val_final / val_inicial) ** (1 / n_anios) - 1) * 100
                    cagr_data[p] = cagr

        if cagr_data:
            cagr_series = pd.Series(cagr_data).nlargest(min(6, len(colores_activos)))
            plt.figure()
            cagr_series.plot(kind='barh', color=colores_activos[:len(cagr_series)])
            plt.title('CAGR - Crecimiento Anual Compuesto (%) por País', fontsize=14, fontweight='bold')
            plt.xlabel('Tasa de Crecimiento Anual Compuesta (%)', fontsize=12)
            plt.ylabel('País', fontsize=12)
            plt.axvline(0, color='gray', linestyle='--', linewidth=1)
            plt.tight_layout()
            plt.show()

boton_generar.on_click(al_hacer_clic_generar)

# Mostrar la interfaz completa en Google Colab
ui_completa = widgets.VBox([
    widgets.HTML("<h3>🎨 Sección 4.5: Selección de Paleta y Gráficos Analíticos (MSA)</h3>"),
    widgets.HTML("<p>Selecciona una paleta predeterminada o introduce código CSS personalizado:</p>"),
    selector_paleta, 
    input_css, 
    widgets.HTML("<br>"),
    boton_generar, 
    output_graficos
])
display(ui_completa)
```

## 4.6. Generar Insights estratégicos o "Descubrimientos relevantes" 

**Nota:** *Reemplazar "producto analizado" (Line 2) y "Producto" (Line 35) por el número de partida y nombre del producto analizado antes de proceder con la ejecución del script.*

```
# Generar insights automáticos
def generate_insights(df, market_evolution_top, product_name="producto analizado"):
    latest_year = df['year'].max()
    first_year = df['year'].min()

    print(f"📊 INSIGHTS ESTRATÉGICOS - {product_name.upper()}")
    print("="*60)

    # Top 3 países
    top3 = market_evolution_top[market_evolution_top['year'] == latest_year].nlargest(3, 'market_share')
    print(f"\n🏆 TOP 3 PAÍSES ({latest_year}):")
    for i, (_, row) in enumerate(top3.iterrows(), 1):
        print(f"  {i}. {row['partner_country']}: {row['market_share']:.1f}% market share")

    # Mayor crecimiento
    growth_analysis = market_evolution_top.groupby('partner_country').apply(
        lambda x: x['trade_value_usd'].iloc[-1] / x['trade_value_usd'].iloc[0] - 1
    ).sort_values(ascending=False)

    print(f"\n📈 MAYOR CRECIMIENTO ({first_year}-{latest_year}):")
    print(f"  {growth_analysis.index[0]}: +{growth_analysis.iloc[0]:.1%}")

    # Oportunidades
    declining = growth_analysis[growth_analysis < 0]
    if len(declining) > 0:
        print(f"\n⚠️  MERCADOS EN DECLIVE:")
        for country, decline in declining.head(3).items():
            print(f"  {country}: {decline:.1%}")

    # Tamaño total del mercado
    total_market = df[df['year'] == latest_year]['trade_value_usd'].sum()
    print(f"\n💰 TAMAÑO TOTAL DEL MERCADO ({latest_year}): ${total_market/1e9:.1f}B USD")

# Define the product_name variable
product_name = "Producto"

# Aplicar análisis con nombre de producto dinámico
generate_insights(df, market_evolution_top, product_name)
```

## 4.7. Generación de Promts para análisis posteriores

**Nota:** *Reemplazar "market_name" (Line 7) por el mercado objetivo (país-destino) antes de proceder con la ejecución del script.*

```
# GENERAR PROMPT AUTOMÁTICO PARA ANÁLISIS
def generate_analysis_prompt(df, market_evolution_top, product_name):
    latest_year = df['year'].max()
    first_year = df['year'].min()

    # Detectar mercado objetivo (desde columnas o contexto)
    market_name = "Estados Unidos"  # Por defecto, se podría dinamizar

    # Generar estadísticas clave
    top3 = market_evolution_top[market_evolution_top['year'] == latest_year].nlargest(3, 'market_share')
    total_market = df[df['year'] == latest_year]['trade_value_usd'].sum()

    prompt = f"""
    Actúa como consultor en inteligencia de mercados y analiza estos datos de market share de {product_name.lower()} en {market_name} ({first_year}-{latest_year}). Proporciona insights estratégicos específicos:

    CONTEXTO DEL MERCADO:
    - Producto: {product_name}
    - Mercado objetivo: {market_name}
    - Período: {first_year}-{latest_year}
    - Tamaño total del mercado ({latest_year}): ${total_market/1000:.0f} millones USD

    TOP 3 LÍDERES ACTUALES ({latest_year}):
    """

    for i, (_, row) in enumerate(top3.iterrows(), 1):
        prompt += f"\n    {i}. {row['partner_country']}: {row['market_share']:.1f}% market share"

    prompt += f"""

    DATOS PARA ANÁLISIS:
    [Pegar aquí las 4 tablas generadas por el script anterior]

    PREGUNTAS ESPECÍFICAS A RESPONDER:

    1. ANÁLISIS COMPETITIVO:
       - ¿Cuáles son las tendencias de los líderes del mercado?
       - ¿Qué países están ganando/perdiendo participación?
       - ¿Hay oportunidades por concentración/fragmentación del mercado?

    2. OPORTUNIDADES DE ENTRADA:
       - ¿Qué nichos o segmentos presentan menos competencia?
       - ¿Existen países que han reducido su participación creando vacíos?
       - ¿Cuál sería el mejor timing para entrar al mercado?

    3. BENCHMARKING REGIONAL:
       - ¿Cómo se comportan países con perfil similar al de tu empresa?
       - ¿Qué países de América Latina tienen mejor performance?
       - ¿Cuáles son las mejores prácticas observables?

    4. ESTRATEGIA DE POSICIONAMIENTO:
       - ¿Competir directamente con líderes o buscar nichos?
       - ¿Qué ventajas competitivas serían más relevantes?
       - ¿Cuál sería un market share objetivo realista en 2-3 años?

    5. RIESGOS Y AMENAZAS:
       - ¿Qué competidores emergentes representan amenazas?
       - ¿Hay signos de saturación o madurez del mercado?
       - ¿Qué factores externos podrían afectar la industria?

    FORMATO DE RESPUESTA:
    - Insights concisos y accionables
    - Recomendaciones estratégicas específicas
    - Identificación de 2-3 oportunidades prioritarias
    - Métricas clave a monitorear
    """

    return prompt

# Generar y mostrar el prompt
analysis_prompt = generate_analysis_prompt(df, market_evolution_top, product_name)
print("="*80)
print("📋 PROMPT PARA ANÁLISIS ESTRATÉGICO:")
print("="*80)
print(analysis_prompt)
print("\n" + "="*80)
print("💡 INSTRUCCIONES:")
print("1. Copia este prompt")
print("2. Pega las 4 tablas de datos generadas arriba")
print("3. Úsalo con cualquier chatbot para obtener insights estratégicos")
print("="*80)
```

## 4.8. Helpers (opcionales) para depuración y personalización
### 4.8.1. Promt para análisis de oportunidad: Origen (Proveedor)-Destino

```
Con base en lo anterior, redacta un texto argumentativo, en tono formal y académico en tercera persona, máximo una cuartilla (250 palabras) donde evidencies los resultados de este estudio de mercados. Enfatiza en el nivel de oportunidad que tendría [COLOMBIA] para ingresar a este mercado desde el punto de vista comercial basándose únicamente en los datos suministrados.
```

### 4.8.2. Promt para personalizar colores en los gráficos

```
Con base en la siguiente paleta de colores:

[CSS.CODE]

Ajusta el siguiente trozo de código en consecuencia, da por hecho que las librerías ya se encuentran instaladas:

[PY.CODE.4.5]
```

### 4.8.3. Ajuste para datos faltantes o sin datos históricos (No calcula CAGR)

```
# TABLA 1: Market Share por año y país
print("📊 TABLA 1: MARKET SHARE POR AÑO (%)")
print("="*60)
pivot_share = market_evolution_top.pivot(index='year', columns='partner_country', values='market_share')
pivot_share_rounded = pivot_share.round(1)
print(pivot_share_rounded.to_string())

# TABLA 2: Valores de comercio (Millones USD)
print("\n\n💰 TABLA 2: VALORES DE COMERCIO (Millones USD)")
print("="*60)
pivot_value = market_evolution_top.pivot(index='year', columns='partner_country', values='trade_value_usd')
pivot_value_millions = (pivot_value / 1000).round(1)  # Convertir a millones
print(pivot_value_millions.to_string())

# TABLA 3: Crecimiento anual por país
# El cálculo del CAGR requiere al menos dos años de datos. Como solo se tiene un año (2024),
# esta tabla no se puede generar con los datos actuales.
# print("\n\n📈 TABLA 3: CRECIMIENTO ANUAL COMPUESTO (CAGR %)")
# print("="*60)
# growth_data = market_evolution_top.groupby('partner_country').apply(
#     lambda x: ((x['trade_value_usd']. iloc[-1] / x['trade_value_usd'].iloc[0]) ** (1/(len(x)-1)) - 1) * 100
# ).reset_index(name='cagr')
# growth_summary = growth_data.set_index('partner_country')['cagr'].round(1).sort_values(ascending=False)
# print(growth_summary.to_string())
print("\n\n📈 TABLA 3: CRECIMIENTO ANUAL COMPUESTO (CAGR %)")
print("="*60)
print("No se puede calcular el CAGR. Se requiere al menos dos años de datos para este cálculo.")

# TABLA 4: Ranking por año
print("\n\n🏆 TABLA 4: RANKING DE PAÍSES POR AÑO (Top 5)")
print("="*60)
for year in sorted(df['year'].unique()):
    year_data = market_evolution_top[market_evolution_top['year'] == year].nlargest(5, 'market_share')
    print(f"\n{year}:")
    for i, (_, row) in enumerate(year_data.iterrows(), 1):
        print(f"  {i}. {row['partner_country']}: {row['market_share']:.1f}% ({row['trade_value_usd']/1000:.0f}M USD)")
```

### 4.8.4. Ajuste para datos faltantes (Calcula CAGR omitiendo datos vacíos o nulos)

```
# TABLA 1: Market Share por año y país
print("📊 TABLA 1: MARKET SHARE POR AÑO (%)")
print("="*60)
pivot_share = market_evolution_top.pivot(index='year', columns='partner_country', values='market_share')
pivot_share_rounded = pivot_share.round(1)
print(pivot_share_rounded.to_string())

# TABLA 2: Valores de comercio (Millones USD)
print("\n\n💰 TABLA 2: VALORES DE COMERCIO (Millones USD)")
print("="*60)
pivot_value = market_evolution_top.pivot(index='year', columns='partner_country', values='trade_value_usd')
pivot_value_millions = (pivot_value / 1000).round(1)  # Convertir a millones
print(pivot_value_millions.to_string())

# TABLA 3: Crecimiento anual por país
print("\n\n📈 TABLA 3: CRECIMIENTO ANUAL COMPUESTO (CAGR %)")
print("="*60)
growth_data = market_evolution_top.groupby('partner_country').apply(
    lambda x: ((x['trade_value_usd'].iloc[-1] / x['trade_value_usd'].iloc[0]) ** (1/(len(x)-1)) - 1) * 100 if len(x) > 1 else np.nan
).reset_index(name='cagr')
growth_summary = growth_data.set_index('partner_country')['cagr'].round(1).sort_values(ascending=False)
print(growth_summary.to_string())

# TABLA 4: Ranking por año
print("\n\n🏆 TABLA 4: RANKING DE PAÍSES POR AÑO (Top 5)")
print("="*60)
for year in sorted(df['year'].unique()):
    year_data = market_evolution_top[market_evolution_top['year'] == year].nlargest(5, 'market_share')
    print(f"\n{year}:")
    for i, (_, row) in enumerate(year_data.iterrows(), 1):
        print(f"  {i}. {row['partner_country']}: {row['market_share']:.1f}% ({row['trade_value_usd']/1000:.0f}M USD)")
```
