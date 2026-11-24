# MEDIC-MLOps | Predicción de enfermedades comunes y huérfanas

**Universidad Icesi — Machine Learning Operations (MLOps) — Semanas 5–6: Reestructuración**  
**Versión:** 2.0 (propuesta de arquitectura) · 23 de noviembre de 2026

## 1. Problema, objetivo y alcance

La disponibilidad abundante de información para enfermedades comunes contrasta con la escasez de casos confirmados de enfermedades huérfanas. Se propone un sistema de apoyo clínico que estime, a partir de síntomas y antecedentes disponibles antes del diagnóstico, la probabilidad de cada enfermedad incluida en un catálogo previamente aprobado. Como las enfermedades pueden coexistir, el problema es de clasificación multietiqueta, no de clasificación mutuamente excluyente. El sistema puede abstenerse por insuficiencia de información o evidencia. Una salida negativa no descarta otras enfermedades ni reemplaza el diagnóstico médico.

El objetivo es definir un ciclo reproducible, seguro y operable desde la adquisición hasta el monitoreo y la actualización del modelo, con inferencia local o remota. Se conserva el enfoque clínico de la semana 1 y se concretan sus mecanismos de MLOps. La demostración determinista anterior no es un modelo entrenado y sus cuatro estados ficticios no se trasladan a la solución clínica. Esta entrega describe una arquitectura futura: no afirma tener datos, modelos validados, infraestructura desplegada ni resultados experimentales.

## 2. Suposiciones y decisiones de diseño

| ID | Suposición o decisión | Implicación y alternativa si no se cumple |
|---|---|---|
| S01 | Instituciones autorizan acceso a historias y registros especializados. | Sin acuerdos, base jurídica y aprobación institucional no se ingieren datos reales; solo se permiten datos sintéticos para pruebas de ingeniería. |
| S02 | Existen síntomas fechados y diagnósticos adjudicados por especialistas. | Una etiqueta desconocida no es negativa; se excluye de la pérdida y de métricas de esa enfermedad. |
| S03 | Se define una población, contexto asistencial, catálogo y horizonte de confirmación por enfermedad antes de entrenar. | Clínica fija estos parámetros; fuera de población o catálogo se informa limitación y se abstiene cuando corresponda. |
| S04 | Es posible identificar pacientes repetidos mediante identificador seudónimo estable, incluso entre centros cuando exista autorización. | Si no puede garantizarse independencia entre cohortes, no se afirma validación externa independiente. |
| S05 | Los datos huérfanos son escasos y heterogéneos. | Se negocia integración multicéntrica; transferencia solo si es compatible y supera la referencia. Si no hay evidencia suficiente, no se habilita esa etiqueta. |
| S06 | Hay información estructurada o exportable; no se necesitan imágenes ni notas libres para la primera versión. | FHIR es preferente; CSV con contrato equivalente es alternativa. Se excluyen variables no disponibles en inferencia. |
| S07 | Entrenamiento centralizado en infraestructura institucional privada; inferencia CPU. | Se propone iniciar con 8 vCPU y 32 GB RAM en entrenamiento, ajustables tras pruebas; GPU solo para experimentos justificados. |
| S08 | El equipo médico local admite contenedores Linux, 4 núcleos y 8 GB RAM como hipótesis inicial. | Se mide consumo y latencia; si no cumple, se usa servidor autorizado. No se requiere conectividad durante inferencia local. |
| S09 | Los diagnósticos de seguimiento llegan con retraso y no siempre se confirman. | Se informa cobertura de etiquetas y tiempo de maduración; no se usan predicciones como verdad. |
| S10 | Hay responsables clínicos, de datos, ML, operación y seguridad. | La ausencia de aprobación clínica o de seguridad impide promover modelos. |
| S11 | Se dispone de conectividad segura para modo remoto y distribución periódica de versiones locales. | Sin conexión se mantiene versión local vigente; si vence su autorización se suspende la inferencia. |
| S12 | Los costos de falsos negativos y positivos difieren por enfermedad. | Umbrales, capacidad de alertas y evidencia mínima se acuerdan antes de abrir la prueba final; no se fija un valor universal clínico. |
| S13 | El volumen inicial cabe en procesamiento por lotes de un nodo. | Pandas y Airflow son suficientes como hipótesis; se reconsidera procesamiento distribuido solo si las mediciones lo exigen. |

## 3. Diagrama principal

![Pipeline integral de MLOps](diagrams/pipeline_general.svg)

**Figura 1.** Flujo end-to-end, puertas de calidad y aprobación, despliegue alternativo y retroalimentación. Fuente: elaboración propia a partir de la propuesta de semana 1. [Versión PNG](diagrams/pipeline_general.png) · [Fuente editable Graphviz](diagrams/pipeline_general.dot).

## 4. Etapas del pipeline

### E01. Diseño clínico y criterios de aceptación
**Entrada:** necesidad clínica, población y recursos (S03, S10, S12). **Proceso:** clínica y ciencia de datos definen catálogo versionado con códigos CIE/ORPHA cuando corresponda, población, unidad de análisis (encuentro), instante de predicción, ventana de síntomas, horizonte de confirmación y exclusiones. Establecen por enfermedad sensibilidad mínima, carga máxima de alertas, calibración aceptable, tamaño de muestra requerido y política de abstención. Se documenta plan estadístico antes del entrenamiento. **Tecnología:** Markdown y YAML versionados en Git, porque permiten auditar acuerdos y reutilizar criterios en pruebas. **Salida:** especificación clínica y configuración de aceptación aprobadas. **Responsable:** líder clínico y ML. **Puerta:** sin criterios firmados no se inicia adquisición para el estudio.

### E02. Adquisición e ingesta autorizada
**Entrada:** convenios, especificación E01, historias y registros especializados (S01, S02, S06). **Proceso:** extraer por lotes diarios mediante Python desde FHIR (Condition, Observation y Encounter) o CSV controlado; registrar origen, fecha, esquema, conteo y checksum. Separar identificadores en zona restringida, asignar seudónimo estable y conservar tabla de correspondencia únicamente bajo custodia institucional. Los especialistas adjudican etiquetas positivas, negativas confirmadas o desconocidas por enfermedad y fecha. Airflow ejecuta tareas idempotentes por lote; reintenta fallos transitorios hasta dos veces, alerta y no publica lotes incompletos. **Tecnología:** FHIR para interoperabilidad, Python para adaptadores y Airflow para dependencias y recuperación. **Salida:** lote seudonimizado y manifiesto. **Responsable:** ingeniería de datos y custodio clínico. **Puerta:** autorización, procedencia y contrato de extracción verificados; si fallan, detener y notificar.

### E03. Almacenamiento, calidad y versionamiento
**Entrada:** lotes E02 (S01, S04, S06). **Proceso:** almacenar objetos originales seudonimizados en zona raw privada, lotes rechazados en cuarentena y datos conformes en curated; PostgreSQL conserva manifiestos, catálogo y metadatos. Great Expectations verifica campos obligatorios, tipos, dominios, unidades, fechas, unicidad de encuentro y coherencia clínica; los faltantes permitidos se cuantifican y no se eliminan casos raros por ser atípicos. Cualquier fallo crítico bloquea publicación; ingeniería corrige origen y revalida, sin sobrescribir históricos. DVC registra instantánea, checksum y ubicación privada; Git conserva punteros y esquema, nunca registros clínicos. **Tecnología:** MinIO compatible con S3 para objetos, PostgreSQL para metadatos relacionales, Great Expectations para contratos y DVC para reproducibilidad. **Salida:** versión de datos aprobada y reporte de calidad. **Responsable:** ingeniería de datos. **Puerta:** cero violaciones críticas y manifiesto completo.

### E04. Partición y prevención de fuga
**Entrada:** instantánea E03 (S02, S04, S05). **Proceso:** reservar primero una institución externa si existe y un período posterior para prueba; asignar pacientes completos a una única partición, excluyendo del entrenamiento los pacientes reservados. En el resto, asignar aproximadamente 80/20 por paciente a entrenamiento y validación; ajustar tamaños por disponibilidad sin comprometer independencia. Verificar intersecciones vacías, prevalencia y conteos por etiqueta y centro; bloquear variables posteriores al instante de predicción. En escasez usar validación agrupada dentro del desarrollo; conservar prueba independiente cuando sea factible. No afirmar evidencia concluyente si los positivos son insuficientes. **Tecnología:** Python, pandas y divisores agrupados de scikit-learn; permiten reglas auditables. **Salida:** manifiesto congelado de particiones y reporte de fuga. **Responsable:** ciencia de datos. **Puerta:** ninguna intersección de pacientes ni variables posdiagnóstico; si falla, reconstruir particiones antes de entrenar.

### E05. Preprocesamiento e ingeniería de características
**Entrada:** particiones E04 (S06). **Proceso:** normalizar unidades y vocabularios, codificar presencia/ausencia/desconocimiento de síntomas, duración, intensidad, edad y antecedentes autorizados. Calcular únicamente características disponibles en el encuentro. Ajustar imputación numérica por mediana, codificación categórica y escalado cuando aplique exclusivamente en entrenamiento; usar indicadores de faltantes cuando sean informativos. Categorías nuevas se gestionan explícitamente y se rechazan entradas incompatibles. Empaquetar transformadores con el modelo para evitar discrepancias entrenamiento/inferencia. **Tecnología:** pandas y ColumnTransformer/Pipeline de scikit-learn; PyTorch consume la misma representación versionada. **Salida:** transformador, contrato de características y matrices por partición. **Responsable:** ciencia de datos. **Puerta:** pruebas de reproducibilidad, ausencia de fuga y equivalencia de transformaciones; si falla, corregir E05.

### E06. Entrenamiento de enfermedades comunes
**Entrada:** matrices E05 y etiquetas conocidas (S02, S03, S12). **Proceso:** entrenar clasificadores binarios por enfermedad con regresión logística como referencia y XGBoost como candidato. Excluir etiquetas desconocidas de cada ajuste, ponderar clases usando solo entrenamiento y evitar remuestreo de validación o prueba. Optuna explora profundidad, tasa de aprendizaje, regularización y pesos dentro de límites registrados; objetivo PR-AUC en validación agrupada, sujeto a requisitos clínicos. Calibrar probabilidades usando predicciones fuera de muestra del desarrollo; fijar umbrales en validación. **Tecnología:** scikit-learn, XGBoost y Optuna por su adecuación a datos tabulares, costo CPU y búsqueda reproducible. **Salida:** candidatos y referencia por enfermedad. **Responsable:** ciencia de datos. **Puerta:** ejecución reproducible y métricas calculables; fallos o insuficiencia de datos retornan a E03/E05.

### E07. Entrenamiento de enfermedades huérfanas
**Entrada:** matrices E05, casos confirmados multicéntricos y etiquetas conocidas (S02, S05). **Proceso:** comenzar con regresión logística regularizada y pesos de clase; comparar transferencia de una representación tabular aprendida con enfermedades relacionadas mediante PyTorch, únicamente si las variables y poblaciones son compatibles y no existe fuga de pacientes. Congelar capas iniciales y ajustar una cabeza binaria con pocos casos, comparando con línea base bajo las mismas particiones. Optimizar pocos parámetros mediante Optuna para reducir sobreajuste. Evaluar variación por centro, intervalos de confianza y calibración; no generar pacientes ficticios como evidencia clínica. Si la transferencia no mejora o no hay positivos suficientes según E01, deshabilitar predicción de esa enfermedad y devolver abstención con recomendación de evaluación especializada. **Tecnología:** scikit-learn, PyTorch y Optuna; PyTorch se usa condicionalmente, no se obliga un modelo complejo. **Salida:** candidato o estado explícito de evidencia insuficiente por enfermedad. **Responsable:** ciencia de datos y especialista. **Puerta:** evidencia mínima y trazabilidad; nunca promover automáticamente una etiqueta escasa.

### E08. Evaluación técnica y clínica
**Entrada:** candidatos E06/E07 y prueba congelada (S03, S05, S12). **Proceso:** seleccionar configuración y umbrales únicamente en desarrollo; ejecutar prueba final una vez por ciclo de evaluación predefinido. Comparar con referencia y modelo vigente. Informar sensibilidad, especificidad, precisión, PR-AUC, F1 macro, Brier, matriz de confusión, cobertura y abstención; reportar prevalencia, casos positivos, intervalos mediante bootstrap por paciente cuando sea estimable, y resultados por centro y subgrupo autorizado. Evaluar sensibilidad incluyendo abstenciones como no detecciones cuando corresponda y también desempeño condicionado a cobertura. Usar SHAP para revisar asociaciones, sin interpretarlas como causas. Clínica revisa falsos negativos, falsos positivos y carga de alertas. Probar latencia y recursos en equipos objetivo. **Tecnología:** scikit-learn, SHAP y Python. **Salida:** informe, ficha de modelo, lista de etiquetas habilitadas y decisión firmada. **Responsable:** validación independiente y líder clínico. **Puerta:** cumplir umbrales E01, no presentar fuga y disponer de evidencia suficiente; si falla, retornar a datos/diseño/desarrollo, con nueva prueba independiente cuando se haya usado la anterior para decidir cambios.

### E09. Seguimiento, registro y aprobación
**Entrada:** artefactos E05–E08 (S10). **Proceso:** MLflow Tracking registra desde el inicio de cada experimento el commit Git, versión DVC, entorno bloqueado, semillas, particiones, hiperparámetros y métricas. Tras evaluación se registra en MLflow Model Registry un paquete con transformador, modelos, calibradores, umbrales, catálogo, firma de entrada y ficha. Usar alias `candidate` y `champion`, sin confundir registro con autorización. Seguridad y clínica aprueban explícitamente promoción mediante registro de cambio; guardar hash y versión anterior. **Tecnología:** MLflow con PostgreSQL y MinIO privados, para relacionar experimentos, artefactos y versiones. **Salida:** paquete aprobado e inmutable o candidato rechazado. **Responsable:** ingeniería ML y responsables clínico y de seguridad. **Puerta:** expediente completo, firmas y hash verificado; sin aprobación no hay despliegue.

### E10. Integración y entrega continuas (CI/CD)
**Entrada:** cambios de código y paquetes aprobados E09 (S10, S11). **Proceso:** en pull request, GitHub Actions ejecuta Pytest (contrato, faltantes, fuga, transformaciones, API, abstención), análisis de dependencias e imagen, y pruebas con datos sintéticos. Al integrar código, construir Docker con versiones fijadas y publicar imagen por digest en registro privado. En validación ejecutar integración y pruebas de recursos; en producción exigir aprobación manual y artefacto autorizado. Separar desarrollo, validación y producción; usar ejecutor institucional para operaciones privadas y credenciales de mínimo privilegio. No enviar datos clínicos a GitHub. **Tecnología:** GitHub Actions, Pytest, Trivy y Docker/registro privado. **Salida:** release trazable con digest, pruebas y evidencia de aprobación. **Responsable:** MLOps y seguridad. **Puerta:** todas las pruebas críticas aprobadas y sin vulnerabilidades críticas no mitigadas; detener release si falla.

### E11. Despliegue local o remoto e inferencia
**Entrada:** release E10 (S03, S08, S11). **Proceso:** empaquetar FastAPI, transformador, modelos y una interfaz Streamlit que consume la API; Docker Compose configura servicios. En modo local ambos se ejecutan en el computador médico, con API interna y acceso a interfaz por loopback; no se envían datos fuera del equipo. Distribuir imagen firmada por canal institucional, verificar digest y vigencia antes de activar. En modo remoto la misma imagen se ejecuta en servidor institucional o nube autorizada; proxy HTTPS y autenticación institucional con roles limitan acceso, API no se expone directamente. Se inicia en observación y luego piloto supervisado antes de producción. Ante fallo se desactiva la versión y se vuelve a una anterior autorizada si es segura. **Tecnología:** Docker Compose, FastAPI, Pydantic, Streamlit y Nginx para proxy remoto; reutilizan el mismo contrato y artefacto. **Salida:** servicio versionado y respuesta clínica de apoyo. **Responsable:** MLOps, TI clínica y médico. **Puerta:** pruebas de salud, seguridad y paridad local/remota; objetivo operativo provisional p95 ≤ 2 s y memoria total ≤ 4 GB para inferencia local, sujetos a prueba real. Si no cumple, optimizar o usar servidor; no alterar umbrales clínicos para lograr velocidad.

### E12. Monitoreo y retroalimentación
**Entrada:** métricas de servicio, entradas agregadas y diagnósticos confirmados (S09, S11). **Proceso:** Prometheus captura latencia, disponibilidad y errores sin identificadores; Grafana presenta paneles. Evidently compara distribuciones de variables y predicciones con referencia aprobada; revisión diaria de operación y corte mensual de rendimiento con etiquetas maduras. Medir sensibilidad, calibración, abstención y subgrupos por enfermedad, junto con conteos e incertidumbre. Modo local conserva métricas mínimas cifradas y exporta agregados solo con autorización; el médico registra confirmación clínica por canal institucional separado. **Tecnología:** Prometheus/Grafana para operación y Evidently para deriva; no sustituyen evaluación clínica. **Salida:** alertas, informes y nuevas etiquetas adjudicadas. **Responsable:** operaciones y clínica. **Puerta:** incompatibilidad de esquema bloquea solicitudes; deterioro clínico confirmado activa suspensión o rollback; deriva persistente activa investigación, no promoción automática.

### E13. Entrenamiento continuo y mejora controlada (CT)
**Entrada:** alertas E12 o nuevos casos confirmados (S05, S09, S10). **Proceso:** Airflow revisa mensualmente elegibilidad de reentrenamiento y permite ejecución extraordinaria autorizada. Ante deriva persistente, degradación o nuevos casos suficientes según E01, registrar ticket, congelar nueva versión de datos E03 y repetir E04–E09. Comparar candidato con champion mediante protocolo independiente; si no mejora según criterios acordados o incumple seguridad, conservar versión vigente si sigue siendo segura. Si se aprueba, repetir E10–E11 con piloto. Conservar historial de versiones y registrar rollback. **Tecnología:** Airflow, DVC, MLflow y GitHub Actions por separación de orquestación de datos, registro y entrega. **Salida:** candidato evaluado, nueva versión autorizada o decisión documentada de no actualizar. **Responsable:** MLOps, ciencia de datos y clínica. **Puerta:** jamás promover por calendario o deriva sin evaluación y aprobación.

## 5. Contrato de inferencia y uso médico

`POST /v1/predict` acepta JSON versionado con identificador de solicitud aleatorio, versión de esquema, edad cuando sea pertinente, síntomas codificados (estado, intensidad y duración), antecedentes permitidos y contexto mínimo definido en E01. No recibe nombre, documento ni historia libre. Pydantic valida tipos, rangos y campos; datos incompatibles retornan HTTP 422, falta de autorización 401/403 y servicio no disponible 503. `GET /health` informa salud técnica; `GET /version` informa versiones del modelo y esquema a usuarios autorizados.

La respuesta incluye por enfermedad código, probabilidad calibrada cuando esté habilitada, umbral, estado (alerta, sin alerta o abstención), motivo de abstención, explicación disponible, versión y advertencia de apoyo clínico. Las etiquetas sin evidencia no entregan probabilidades aparentes. Síntomas desconocidos no equivalen a ausentes. La interfaz permite revisar y corregir entradas, muestra límites de cobertura y no prescribe tratamientos ni declara ausencia de enfermedad. La valoración y decisión final pertenecen al médico. El sistema no debe retrasar la atención urgente ni funcionar como mecanismo autónomo de triaje.

## 6. Operación, alertas y recuperación

| Señal | Regla inicial propuesta | Acción y responsable |
|---|---|---|
| Contrato inválido | Cualquier fallo crítico | Bloquear lote o solicitud, alertar a datos/TI. |
| Latencia | p95 > 2 s durante 15 min (servidor) | Operaciones revisa capacidad y versión; optimiza o revierte si el release causó el fallo. |
| Errores de servidor | >1% de solicitudes durante 15 min, con volumen suficiente | Operaciones investiga, suspende o revierte. |
| Deriva | Señal significativa en dos cortes consecutivos, con volumen mínimo preacordado | Datos investiga calidad y mezcla poblacional; CT solo tras revisión. |
| Sensibilidad/calibración | Incumplimiento de criterio E01 con evidencia suficiente | Clínica evalúa riesgo y suspende etiquetas o versión; CT autorizado. |
| Escasez de evidencia | Etiquetas maduras insuficientes | No afirmar estabilidad; ampliar seguimiento o abstenerse. |
| Versión local vencida | Fecha de vigencia aprobada superada | Bloquear inferencia y solicitar actualización autorizada. |

Los umbrales operativos son hipótesis iniciales, no estándares clínicos. El rollback restaura el digest anterior, contrato y paquete compatibles, verifica salud y registra incidente; si ninguna versión es segura se suspende el servicio. El objetivo inicial de recuperación remota es 4 horas y de pérdida máxima de metadatos 24 horas, sujetos a acuerdos institucionales y pruebas de restauración. Copias cifradas diarias y ensayos periódicos de recuperación protegen artefactos y metadatos.

## 7. Seguridad, privacidad y gobernanza transversal

Se aplica minimización de datos, separación de identificadores, cifrado en tránsito y reposo, RBAC, segregación de ambientes, auditoría de accesos y promoción, gestión de secretos fuera del código, imágenes firmadas y retención definida por la institución. Los datos seudonimizados siguen siendo sensibles; GitHub público solo contiene documentación y código sin datos reales, credenciales ni artefactos clínicos. Los logs no incluyen síntomas individuales ni identificadores; la vinculación con diagnósticos se realiza dentro del entorno autorizado. Seguridad evalúa riesgos, permisos y respuesta a incidentes antes del piloto. Para Colombia, el tratamiento de información de salud debe revisarse bajo Ley 1581 de 2012 y normativa de historia clínica aplicable; la arquitectura no equivale a certificación de cumplimiento ni autorización sanitaria. La institución determina requisitos legales y clínicos antes de uso real.

## 8. Stack tecnológico y justificación integrada

| Tecnología | Etapas | Función, elección y límite |
|---|---|---|
| Git/GitHub/Markdown/YAML | E01, E03, E09–E10 | Código, contratos y decisiones revisables; no datos clínicos. |
| Python, HL7 FHIR, CSV | E02 | Adaptadores interoperables y alternativa para centros sin FHIR. |
| Airflow | E02, E13 | DAG de lotes, dependencias, reintentos y CT; no participa en inferencia. |
| PostgreSQL | E03, E09 | Metadatos relacionales y backend MLflow; instancia privada con bases y roles separados. |
| MinIO/S3 privado | E03, E09 | Objetos versionados y artefactos; acceso restringido. |
| Great Expectations | E03 | Contratos de calidad automatizados y reportes; reglas clínicas aprobadas externamente. |
| DVC | E03–E04, E13 | Vincula snapshot y particiones al commit; remoto privado. |
| pandas/scikit-learn | E04–E08 | Transformaciones y referencias tabulares CPU; evita infraestructura distribuida prematura. |
| XGBoost | E06 | Candidato robusto para relaciones no lineales en datos tabulares. |
| PyTorch | E07 | Transferencia experimental cuando los datos lo permiten; no requisito de inferencia de todas las etiquetas. |
| Optuna | E06–E07 | Búsqueda controlada y reproducible sin acceder a prueba. |
| SHAP | E08 | Revisión de contribuciones predictivas, no causalidad. |
| MLflow | E06–E09 | Tracking y registro de paquetes; no sustituye aprobación clínica. |
| GitHub Actions/Pytest/Trivy | E10 | CI, pruebas y escaneo de imágenes; ejecutor privado para acceso institucional. |
| Docker/Compose/registro privado | E10–E11 | Artefacto portable, ejecución local o servidor sin Kubernetes inicial. |
| FastAPI/Pydantic/Streamlit/Nginx | E11 | API validada, interfaz clínica y proxy TLS remoto; no duplican lógica de modelo. |
| Evidently | E12 | Deriva estadística y reportes; requiere referencia y revisión humana. |
| Prometheus/Grafana | E12 | Métricas técnicas y visualización; no almacenan historias clínicas. |

La integración central usa un entorno privado: Airflow ejecuta ingesta/entrenamiento; DVC apunta a objetos de datos; MLflow guarda metadatos en PostgreSQL y artefactos en MinIO; CI construye imágenes que incorporan exclusivamente paquetes aprobados. El servicio local no necesita Airflow, DVC, MLflow ni base central para inferir. Las versiones concretas de dependencias se fijarán y probarán en implementación. No se presupone contratación de nube ni necesidad de Spark o Kubernetes.

## 9. Secuencia de implementación y responsabilidades

| Hito | Entregable verificable | Responsable y dependencia |
|---|---|---|
| 1 | Catálogo, contratos y criterios clínicos aprobados | Clínica/seguridad, E01 |
| 2 | DAG de ingesta, reporte de calidad y snapshot | Datos, E02–E03 |
| 3 | Particiones y transformador probado | ML, E04–E05 |
| 4 | Referencias, candidatos y tracking | ML, E06–E07/E09 |
| 5 | Informe independiente y decisión clínica | Clínica/validación, E08–E09 |
| 6 | Release probado en ambos modos | MLOps/TI, E10–E11 |
| 7 | Paneles, piloto, protocolo de incidentes y CT | Operaciones/clínica, E12–E13 |

Cada hito depende de la aceptación del anterior. El piloto no se inicia con datos reales hasta contar con autorizaciones y evaluación clínica. No se declara cumplimiento de estos hitos en la presente propuesta.

## 10. Trazabilidad de la entrega

| Criterio de rúbrica | Evidencia |
|---|---|
| Propuesta general (2,65) | Secciones 1–7 y 9: problema, supuestos, 13 etapas, controles y operación. |
| Diagrama principal (0,85) | Sección 3 y archivos SVG/PNG. |
| Tecnologías (1,00) | Justificación por etapa y sección 8. |
| Changelog (0,50) | [CHANGELOG.md](CHANGELOG.md), comparación con archivos originales de semana 1. |

## 11. Referencias técnicas para implementación

Documentación oficial de [FHIR](https://hl7.org/fhir/), [Airflow](https://airflow.apache.org/docs/), [DVC](https://dvc.org/doc), [Great Expectations](https://docs.greatexpectations.io/), [scikit-learn](https://scikit-learn.org/stable/), [XGBoost](https://xgboost.readthedocs.io/), [PyTorch](https://pytorch.org/docs/), [Optuna](https://optuna.readthedocs.io/), [MLflow](https://mlflow.org/docs/), [SHAP](https://shap.readthedocs.io/), [GitHub Actions](https://docs.github.com/actions), [Docker](https://docs.docker.com/), [FastAPI](https://fastapi.tiangolo.com/), [Evidently](https://docs.evidentlyai.com/), [Prometheus](https://prometheus.io/docs/) y [Grafana](https://grafana.com/docs/). Fuente interna de comparación: `ULI.zip`, propuesta y demostración de Taller 1, 31 de octubre de 2026.
