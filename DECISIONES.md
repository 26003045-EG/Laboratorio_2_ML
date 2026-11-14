# Decisiones técnicas — NovaMercado

## Datos defectuosos
- `tienda_id` numérico en texto y formato T05: **reparar**, después validar contra catálogo.
- Familia con mayúsculas, acentos, espacios y nombre visible: **reparar** por equivalencia única.
- Cantidad entera en texto o decimal exacto: **reparar**; no redondear fracciones.
- Fechas AAAA-MM-DD, DD/MM/AAAA y fecha ISO con 00:00:00: **reparar** cuando no hay ambigüedad.
- Familias faltantes en pronóstico: **advertir**, completar con promoción 0, sin afirmar que esa promoción ocurrió.
- Familia duplicada: **rechazar 422** porque no es inequívoco sumar ni sobrescribir.
- Alias `tienda`: **reparar** únicamente si no existe `tienda_id`.
- Venta negativa: **rechazar 422**; no convertir devoluciones en ventas ficticias.
- Observación idéntica repetida: **idempotencia**; retorna 0 modificaciones.
- Cantidad con coma `1,234` o `812,0`: **reparar** según regla explícita (grupos de 3=separador miles; otros=decimal), enteros exactos.
- Familia, tienda o fecha irrecuperable: **rechazar 422**.

## Variables disponibles al pronosticar
Tienda, familia, día, mes, día del año, promociones de la solicitud y feriados del calendario; nunca ventas futuras. Las observaciones posteriores se usan solo al reentrenar.

## Cantidad sugerida
Costo asimétrico `cfaltante*max(real-sugerido,0)+csobrante*max(sugerido-real,0)`.
Cuantil por familia `cfaltante/(cfaltante+csobrante)`; aproximación Normal sobre residual validación y mínimo 10% de predicción; inventario existente desconocido.

## Modelo y reemplazo
Baseline: mediana histórica por familia. Alternativa: HistGradientBoostingRegressor, con partición temporal final 28 días. MLflow registra todos los entrenamientos y versiones. Si existe campeón, nuevo candidato se promueve si reduce coste validación al menos 0.5%. Si falla, mantiene campeón. En bootstrap inicial se promueve candidato para asegurar disponibilidad, incluso si no supera baseline; esa excepción se reporta en métricas (limitación del requisito de mejora).

## Arquitectura
FastAPI se apoya en PostgreSQL propia y retiene el modelo cacheado para continuar prediciendo si MLflow cae. Sondeo del alias `champion` cada 15 segundos al recibir peticiones. Airflow se ejecuta aparte con otra BD y dispara tareas con API REST básica. Compose mantiene volúmenes persistentes.

## Límites
La inferencia no utiliza lags de observaciones en tiempo real. El entrenamiento utiliza promociones reales del histórico; observaciones incrementales carecen de promociones y se rellenan a cero (supuesto documentado). Datos originales no versionados públicamente; docente debe colocar CSV en `data/entrada`.
