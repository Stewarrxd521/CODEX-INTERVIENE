# Bot Trading Riesgo - Reversión short con EMA y volumen en Binance Futures

Aplicación web lista para Render que monitorea contratos USDT-M de Binance y busca **reversiones bajistas confirmadas** antes de abrir tramos *short*. No existe una estrategia con rentabilidad garantizada: este bot es una implementación configurable para investigación y *paper trading*, no una promesa de resultados.

## Estrategia implementada

- Actualiza los ganadores de Binance Futures por `priceChangePercent` de 24h con WebSocket y ciclos de filtrado controlados.
- Usa WebSocket de Binance (`!ticker@arr`) para mantener precios, cambios 24h y volumen en tiempo real entre escaneos, evitando polling agresivo.
- Descarga y mantiene velas de 1 minuto para calcular la señal. Un short solo se permite cuando una **vela cerrada** cumple simultáneamente:
  - EMA rápida (9 por defecto) por debajo de EMA lenta (21).
  - EMA lenta por debajo de la EMA de tendencia (55).
  - Vela roja cuyo cierre rompe el mínimo de las 5 velas anteriores.
  - Volumen de la vela de ruptura de al menos 1.5× la media de las 20 velas anteriores.
- Si esa confirmación existe, abre tramos configurables cuando el cambio 24h supera estos niveles:
  - `50%, 75%, 100%, 150%, 200%, 250%`
- Tamaño de cada tramo:
  - `5, 5, 10, 20, 40, 80 USDT`
- Cierra toda la posición cuando la ganancia no realizada llega al 12.5% del capital colocado por defecto.
- Muestra en una página web:
  - Ganadores detectados.
  - Posiciones abiertas.
  - PnL no realizado.
  - Operaciones cerradas.
  - Eventos del bot.

## Seguridad

El bot arranca por defecto en **PAPER_MODE=true**, por lo que simula las órdenes y no envía operaciones reales.

Para operar real en Binance Futures debes configurar todas estas variables de entorno:

```bash
PAPER_MODE=false
LIVE_TRADING=true
BINANCE_API_KEY=tu_api_key
BINANCE_API_SECRET=tu_api_secret
```

> Usa primero paper trading y valida la estrategia con datos fuera de muestra, comisiones, *slippage* y distintos regímenes de mercado. Un short contra monedas que suben 100%-250% puede liquidarse si no hay control de margen, apalancamiento y pérdidas.

## Variables de entorno principales

| Variable | Default | Descripción |
| --- | --- | --- |
| `PAPER_MODE` | `true` | Simula órdenes si está en `true`. |
| `LIVE_TRADING` | `false` | Habilita órdenes reales si también `PAPER_MODE=false`. |
| `ENTRY_LEVELS` | `50,75,100,150,200,250` | Niveles de subida 24h para abrir tramos. |
| `ENTRY_NOTIONALS` | `5,5,10,20,40,80` | USDT por tramo. |
| `TAKE_PROFIT_FRACTION` | `0.125` | Ganancia objetivo sobre el notional total. |
| `EMA_FAST_PERIOD` | `9` | Período de la EMA rápida de confirmación bajista. |
| `EMA_SLOW_PERIOD` | `21` | Período de la EMA lenta de confirmación bajista. |
| `EMA_TREND_PERIOD` | `55` | Período de la EMA que filtra la tendencia. |
| `VOLUME_LOOKBACK` | `20` | Velas usadas para la media de volumen. |
| `MIN_VOLUME_RATIO` | `1.5` | Múltiplo mínimo de volumen para aceptar la ruptura. |
| `BREAKDOWN_LOOKBACK` | `5` | Velas previas cuyo mínimo debe romper el cierre. |
| `KLINE_HISTORY_CANDLES` | `120` | Historial de velas de 1 minuto que mantiene el caché. Debe ser mayor que la EMA más larga. |
| `SCAN_INTERVAL_SECS` | `2` | Frecuencia de evaluación de las señales ya presentes en WebSocket. |
| `MIN_GAIN_TO_SHOW` | `0` | Filtro mínimo de porcentaje para mostrar ganadores en la tabla. |
| `INCLUDE_SPOT_WINNERS` | `false` | Conservado solo para el fallback manual REST; el escaneo operativo usa futures por WebSocket. |
| `LEVERAGE` | `1` | Apalancamiento que intentará configurar en modo real. |
| `STATE_FILE` | `/tmp/bottradingriesgo_state.json` | Archivo usado para compartir el último estado útil entre reinicios/workers. |

## Ejecutar local

```bash
pip install -r requirements.txt
python app.py
```

Abre `http://localhost:8000`.

## Diagnóstico de pantalla vacía

Si el bot abre posiciones en los logs pero la página no las muestra, revisa en la web el bloque **Estado API crudo**. La página ahora renderiza un snapshot inicial del servidor y luego refresca `/api/status`; si falla JavaScript, fetch o el endpoint, el error queda visible en **Último error / diagnóstico**.

## Deploy en Render

El archivo `render.yaml` incluye el servicio web y fija `PYTHON_VERSION=3.12.13` para evitar que Render use Python 3.14, donde dependencias con extensiones nativas pueden compilar desde fuente y fallar. En Render configura las variables de entorno necesarias y despliega el repositorio.
