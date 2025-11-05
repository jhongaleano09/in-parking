# Diccionario de Datos - Dataset de Transacciones de Parqueo

## Información General

**Nombre del Dataset:** Sistema de Transacciones de Parqueo  
**Versión:** 1.0  
**Fecha de Creación:** Noviembre 2025  
**Grano del Dataset:** Una fila por sesión de parqueo (transacción)  
**Fuente de Datos:** Sistema de gestión de parqueaderos  
**Período de Datos:** Datos históricos de transacciones de parqueo  

## Descripción del Dataset

Este dataset contiene información detallada sobre las transacciones de parqueo, incluyendo datos temporales, geográficos, financieros y de vehículos. Cada registro representa una sesión completa de parqueo desde el inicio hasta el final del período pagado.

## Campos del Dataset

| Nombre del Campo | Tipo de Dato | Descripción | Restricciones/Valores | Uso en Modelos | Notas Técnicas |
|------------------|--------------|-------------|----------------------|----------------|----------------|
| **session_id** | Integer (Entero) | Identificador único para cada transacción o sesión de parqueo | Valores únicos, no nulos | **Clave Primaria (PK)**. No se usa como feature predictora, pero esencial para: <br>- Conteo de sesiones (COUNT(DISTINCT session_id)) <br>- Unión de datos <br>- Análisis de cardinalidad | **Crítico:** Es el grano del dataset - cada fila representa una sesión única |
| **start_time** | Timestamp (DateTime) | Momento exacto en que inicia la sesión de parqueo y el pago | Formato: YYYY-MM-DD HH:MM:SS | **Feature Crítica** para engineering temporal: <br>- `hour_of_day` (0-23): Patrones diarios <br>- `day_of_week` (0-6): Patrones semanales <br>- `is_weekend` (0/1): Binario fin de semana <br>- `month_of_year` (1-12): Estacionalidad <br>- Agregaciones por ventanas temporales | **Hipótesis 1:** Variable clave para detección de anomalías en series de tiempo. Base para crear el timestamp de agregación horaria |
| **end_time** | Timestamp (DateTime) | Momento exacto en que finaliza la sesión de parqueo pagada | Formato: YYYY-MM-DD HH:MM:SS <br>Debe ser >= start_time | **Variable derivada** para calcular: <br>- `duration_minutes = (end_time - start_time)` <br>- **No se usa como feature predictora** (no disponible al inicio) | **Hipótesis 2:** Esencial para calcular la duración, variable objetivo en modelos de regresión |
| **zone_number** | Categorical (Integer) | Identificador de la zona, lote o parqueadero específico | Valores enteros únicos por zona <br>Ej: 1, 2, 3, 6, 7, etc. | **Feature Categórica Clave**: <br>- Agrupación para entrenar modelos por zona (H1) <br>- Predictor fuerte de duración e ingresos (H2) <br>- **Encoding:** One-Hot o Target Encoding | **Importante:** Aunque es numérico, debe tratarse como categoría. Cada zona puede tener características operativas diferentes |
| **parking_fee** | Numeric (Float) | Costo base del parqueo sin incluir tarifas adicionales | Valores >= 0 <br>Separador decimal: punto (.) | **Variable objetivo secundaria** o componente del net_revenue. Predictor en modelos de ingresos | Ingreso principal antes de comisiones y tarifas |
| **convenience_fee** | Numeric (Float) | Tarifa de conveniencia asociada al método de transacción | Valores >= 0 <br>**Limpieza requerida:** Reemplazar coma (,) por punto (.) | **Feature financiera** - Componente del ingreso total. Puede correlacionar con transaction_method | Tarifa típicamente cobrada por usar aplicaciones móviles vs. estaciones de pago |
| **transaction_fee** | Numeric (Float) | Tarifa fija por procesar la transacción | Valores >= 0 <br>**Limpieza requerida:** Reemplazar coma (,) por punto (.) | **Feature de costo** - Componente que reduce el ingreso neto | Costo operativo del procesamiento de pagos |
| **net_revenue** | Numeric (Float) | Ingreso neto final = (parking_fee + convenience_fee) - transaction_fee | **Puede ser negativo** (ej. payment_type = 'Free') <br>**Limpieza requerida:** Reemplazar coma (,) por punto (.) | **Variable Objetivo Principal (Y)** para: <br>- Hipótesis 2: Predicción de ingresos <br>- Modelos de regresión <br>- Detección de anomalías en ingresos | **Crítico:** Valores negativos son válidos (transacciones gratuitas). Es la métrica financiera clave del negocio |
| **car_id** | Categorical (Integer) | Identificador anonimizado del vehículo | Identificadores únicos por vehículo | **Feature de comportamiento**: <br>- Análisis de recurrencia (clientes nuevos vs. recurrentes) <br>- Segmentación de usuarios <br>- **No usar como feature numérica directa** | Sustituto anonimizado de la placa del vehículo. Permite análisis de patrones de uso sin comprometer privacidad |
| **vehicle_state** | Categorical (String) | Estado de EE.UU. asociado a la placa del vehículo | Códigos de 2 letras: 'FL', 'NJ', 'NY', etc. <br>Cardinalidad: ~50 estados | **Feature geográfica**: <br>- Patrones de turismo vs. locales <br>- Segmentación geográfica <br>- **Encoding:** Target Encoding recomendado por alta cardinalidad | **Insight de negocio:** 'FL' puede indicar residentes locales vs. 'NY'/'NJ' turistas con patrones de uso diferentes |
| **transaction_method** | Categorical (String) | Método utilizado para realizar la transacción | Valores observados: 'app' <br>Posibles: 'paystation', 'mobile', etc. | **Feature operativa**: <br>- Predictor de convenience_fee <br>- Comportamiento del usuario <br>- **Encoding:** One-Hot | Corresponde al "System Type" mencionado por Paula. En muestra actual solo 'app', pero puede expandirse |
| **payment_type** | Categorical (String) | Tipo de pago utilizado para la transacción | Valores observados: 'Credit Card', 'Free' <br>Posibles: 'Debit', 'Cash', etc. | **Feature financiera crítica**: <br>- Predictor fuerte de net_revenue <br>- 'Free' → revenue negativo <br>- **Encoding:** One-Hot | **Regla de negocio:** 'Free' es categoría válida (no error), puede relacionarse con promociones o validaciones |

## Variables Derivadas (Feature Engineering)

| Variable Derivada | Fórmula/Cálculo | Tipo | Uso en Modelos |
|------------------|------------------|------|----------------|
| **duration_minutes** | `(end_time - start_time).total_seconds() / 60` | Numeric | Variable objetivo para modelos de duración (H2) |
| **hour_of_day** | `start_time.hour` | Integer (0-23) | Feature temporal - patrones diarios |
| **day_of_week** | `start_time.dayofweek` | Integer (0-6) | Feature temporal - patrones semanales |
| **is_weekend** | `1 if day_of_week >= 5 else 0` | Binary | Feature temporal - comportamiento fin de semana |
| **month_of_year** | `start_time.month` | Integer (1-12) | Feature temporal - estacionalidad |
| **start_hour** | `start_time.floor('H')` | Timestamp | Agregación temporal para series de tiempo |

## Agregaciones para Series de Tiempo (Hipótesis 1)

Para la detección de anomalías, los datos se agregan por `[zone_number, start_hour]`:

| Variable Agregada | Cálculo | Descripción |
|------------------|---------|-------------|
| **transaction_count** | `COUNT(session_id)` | Número de sesiones que iniciaron en esa hora |
| **total_net_revenue** | `SUM(net_revenue)` | Suma de ingresos netos de las transacciones |
| **avg_duration_minutes** | `MEAN(duration_minutes)` | Duración promedio de las sesiones |

## Calidad de Datos y Validaciones

### Reglas de Negocio
1. `end_time >= start_time` (duración no negativa)
2. `session_id` debe ser único
3. `net_revenue` puede ser negativo (válido para payment_type = 'Free')
4. `zone_number` debe existir en catálogo de zonas

### Limpieza Requerida
1. **Campos numéricos:** Reemplazar coma (,) por punto (.) en convenience_fee, transaction_fee, net_revenue
2. **Fechas:** Convertir start_time y end_time a datetime
3. **Categorías:** Normalizar valores de vehicle_state a códigos estándar

### Valores Faltantes
- Monitorear completitud de campos clave
- Evaluar estrategias de imputación para campos opcionales

## Consideraciones para Modelado

### Hipótesis 1: Detección de Anomalías
- **Grano temporal:** Agregación horaria por zona
- **Features clave:** transaction_count, total_net_revenue, hour_of_day, day_of_week, is_weekend
- **Algoritmos sugeridos:** Isolation Forest, SARIMA, Control Charts

### Hipótesis 2: Predicción de Ingresos/Duración
- **Variables objetivo:** net_revenue, duration_minutes
- **Features predictoras:** zone_number, hour_of_day, day_of_week, vehicle_state, payment_type, transaction_method
- **Consideraciones:** Encoding de categorías, normalización de features numéricas

## Metadatos del Dataset

- **Cardinalidad estimada:** Variable según período de datos
- **Frecuencia de actualización:** Tiempo real (transaccional)
- **Retención de datos:** Según políticas de la organización
- **Privacidad:** car_id está anonimizado para proteger identidad

---

**Versión del Documento:** 1.0  
**Última Actualización:** Noviembre 2025  
**Responsable:** Equipo de Data Science  
**Contacto:** [Información de contacto]