# Laboratorio 2 — Servicio ML en producción para NovaMercado

Maestría en Data Science · Product Development · 2026

## Archivos de datos (obligatorios)
Colocar en `data/entrada/` los archivos oficiales descargados del GES:
`tiendas.csv`, `familias.csv`, `historico_ventas.csv`, `feriados.csv`, `costos.csv`.
Los archivos no están versionados en Git por ser material proporcionado para evaluación.

## Iniciar desde cero
Requiere Docker Desktop, Docker Compose v2 y al menos 6 GB libres de RAM para contenedores.

1. `python scripts/preparar_env.py` crea `.env` con secretos aleatorios locales y una clave Fernet válida.
2. Colocar los cinco CSV oficiales en `data/entrada/` (con esos nombres exactos).
3. `docker compose up --build -d`
4. Consultar `http://localhost:8000/health` y `http://localhost:8000/docs`.

Los servicios deben conservar su estado con `docker compose down` y `docker compose up -d` (sin `-v`).

## Servicios
- API: http://localhost:8000/docs · `GET /health` · `GET /modelo`
- MLflow: http://localhost:5000
- Airflow: http://localhost:8080 (usuario definido en `.env`)
- PostgreSQL propia + PostgreSQL de Airflow (separadas)

## Probar API
Desde el directorio raíz, con PowerShell y los ejemplos oficiales colocados en `data/entrada`:

```powershell
$body = Get-Content .\data\ejemplos\prediccion_valida.json -Raw
Invoke-RestMethod http://localhost:8000/v1/prediccion -Method Post -ContentType 'application/json' -Body $body
```

`POST /v1/observaciones` registra ventas por familia con clave única tienda/fecha/familia; duplicados idénticos cuentan 0.
`POST /admin/reentrenar` activa el DAG `novamercado_reentrenamiento`, retorna id para consultar su estado con GET.

## Modelo
Baseline por mediana histórica por familia. Candidato ML HistGradientBoostingRegressor, validación temporal de 28 días, MAE y costo de negocio por faltantes/sobrantes. MLflow conserva experimentos, versiones y alias `champion`. Bootstrap entrena al primer inicio. La API recarga la nueva versión bajo demanda sin reinicio y retiene el modelo previo si MLflow se interrumpe.

## Tests esenciales
`python -m pytest tests/ -q` (no requiere MLflow ni servidor de API).

## Resultados de validación histórica
La validación temporal de febrero de 2026, ejecutada contra los datos oficiales, obtuvo MAE baseline **109.8521** y MAE ML **47.1731**. Los costos medios fueron **96.0882** y **37.6515**, respectivamente (mejora de costo **60.8157%**). La evidencia numérica generada está en `evidencias/validacion_modelo.json`. Para reproducirla: `python -m scripts.validacion_offline`.

Se ejecutaron **6 pruebas de contrato**, todas satisfactorias, sin servicios externos. MLflow, Airflow y Docker se proporcionan como código y configuración; no se adjuntan capturas de una ejecución no realizada. GitHub Actions ejecuta pruebas de contrato en cada push.

## Entrega
Repositorio público, tag obligatorio: `entrega-lab2` (crear después de verificar la versión final).

## Decisiones y comportamiento
Políticas de limpieza, criterios de promoción del modelo y supuestos del producto: `DECISIONES.md`.
