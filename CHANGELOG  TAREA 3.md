# CHANGELOG — Semana 1 → Semanas 5–6

## Versión 2.0 — 23 de noviembre de 2026

Comparación documental con la entrega original `ULI.zip` (31 de octubre de 2026), específicamente `entrega_mlops/pipeline/PIPELINE.md` y `entrega_mlops/README.md`. Se distingue lo que ya estaba propuesto de lo que ahora se concreta. Los cambios son de diseño, no funcionalidades implementadas.

| Componente | Semana 1 (evidencia original) | Semanas 5–6 | Razón |
|---|---|---|---|
| Alcance | PIPELINE, Objetivo: multietiqueta y abstención | Se conserva; catálogo, horizonte y contrato explícitos (E01, §5) | Evitar ambigüedad clínica |
| Diseño | PIPELINE etapa 1: población y umbrales | E01: criterios previos, tamaño de muestra y plan estadístico | Puerta verificable |
| Adquisición | Etapa 2: historias y registros especializados autorizados | E02: FHIR/CSV, lotes Airflow, manifiesto y reintentos | Integración repetible |
| Etiquetas | Etapa 2: distinguir ausencia de diagnóstico de ausencia de enfermedad | E02/E06/E07: etiquetas desconocidas excluidas del ajuste y métricas | Evitar falsos negativos artificiales |
| Privacidad | Etapa 3: seudonimizar y limitar acceso | E02/E03 y §7: custodia separada, cifrado, roles y auditoría | Operacionalizar protección |
| Almacenamiento y calidad | Etapa 3: unidades, duplicados, faltantes, contrato | E03: MinIO, PostgreSQL, Great Expectations, cuarentena | Automatizar controles |
| Versionamiento | Etapa 3: instantáneas, esquemas y transformaciones | E03/E04: DVC, manifiestos y Git | Reproducibilidad verificable |
| División | Etapa 4: paciente, tiempo e institución | E04: orden de reserva, reglas de independencia y comprobación de intersecciones | Evitar fuga |
| Preprocesamiento | Etapa 4: imputación y selección solo con entrenamiento | E05: Pipeline/ColumnTransformer y paridad de inferencia | Reducir discrepancias |
| Enfermedades comunes | Etapa 5: regresión logística, árboles potenciados, pesos | E06: línea base y XGBoost, Optuna y calibración | Comparación sistemática |
| Enfermedades huérfanas | Etapa 5: integración multicéntrica, posible transferencia y abstención | E07: referencia regularizada, transferencia condicional PyTorch y deshabilitación por evidencia insuficiente | Evitar sobreajuste y afirmaciones sin soporte |
| Experimentos | Etapa 5: registrar código, semilla, datos y métricas | E09: MLflow Tracking con DVC y entorno fijado | Trazabilidad central |
| Evaluación | Etapa 6: sensibilidad, precisión, PR-AUC, calibración, subgrupos | E08: métricas ampliadas, cobertura, bootstrap, SHAP y protocolo de nueva prueba | Evaluación más completa |
| Registro | Etapa 7: versión aprobada y preprocesamiento | E09: MLflow Registry, alias, hash y aprobaciones | Gestión formal de versiones |
| CI/CD | Etapa 7: pruebas de contrato, integración y salud, Docker | E10: GitHub Actions, Pytest, Trivy, ambientes y aprobación manual | Automatización y seguridad de entrega |
| Despliegue local | README original: Docker loopback puerto 5000, simulador | E11: paquete de inferencia real futuro, Compose, interfaz y validación de recursos | Conservar opción local sin confundir simulación con clínica |
| Despliegue remoto | README: despliegue clínico necesita arquitectura adicional | E11: servidor autorizado, Nginx, TLS y autenticación | Completar alternativa remota |
| Monitoreo | Etapa 8 y Ciclo de mejora: revisión diaria y mensual | E12 y §6: Evidently, Prometheus/Grafana, reglas y acciones | Observabilidad operable |
| CT | Ciclo de mejora: nuevos casos o degradación, sin promoción automática | E13: Airflow, ticket, snapshot y comparación champion/candidate | Automatizar sin perder control clínico |
| Rollback | Etapa 7: conservar imagen anterior | §6: digest, compatibilidad, verificación y suspensión si no hay versión segura | Recuperación verificable |
| Suposiciones | Condiciones distribuidas en etapas originales | §2: matriz S01–S13 con alternativas; referencias por etapa | Hacer explícitas las implicaciones |
| Diagrama | PIPELINE: ocho etapas y bucle de mejora | SVG/PNG: 13 etapas, puertas, rutas local/remota y CT | Alinear visualmente arquitectura y decisiones |
| Demostración | README: cuatro estados deterministas, sin validez clínica | Se excluye el simulador del nuevo repositorio de diseño; §1 conserva advertencia | La rúbrica pide propuesta, no implementación |

## Versión 1.0 — 31 de octubre de 2026

Diseño de ocho etapas y demostración académica HTTP/Docker con función determinista. No se entrenó ni validó un modelo clínico. La reestructuración preserva los principios válidos y especifica tecnologías, artefactos, puertas y responsabilidades.
