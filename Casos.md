# ESG POC — Casos de Prueba Multi-Categoría

## Leyenda

| Icono | Significado |
|---|---|
| ✅ POSITIVO | Impacto ambiental positivo — elegible como crédito verde |
| ❌ NEGATIVO | Impacto ambiental negativo — no elegible |
| ⚪ NEUTRO | Sin impacto ambiental — no aplica |
| ⛔ NO APLICA | Código 0 — excluido por reglas |
| 🔀 MÚLTIPLE | Aplica a más de una categoría ESG |
| 🧠 APRENDE | FuzzySharp fallará — Groq aprende keywords nuevas |

---

## 1. Casos Positivos — Una sola categoría

| # | Estado | Descripción | Código esperado | Categoría |
|---|---|---|---|---|
| 1 | ✅ POSITIVO | Financiamiento para instalación de paneles solares en planta industrial | 201 | Energía Solar |
| 2 | ✅ POSITIVO | Crédito para compra de aerogeneradores de eje horizontal en zona costera | 203 | Energía Eólica |
| 3 | ✅ POSITIVO | Préstamo para adquisición de baterías de litio para almacenamiento solar | 209 | Almacenamiento Energía |
| 4 | ✅ POSITIVO | Financiamiento para flota de autobuses eléctricos de transporte masivo | 401 | Transporte Público |
| 5 | ✅ POSITIVO | Crédito para compra de vehículo eléctrico de batería para uso particular | 405 | Transporte Particular |
| 6 | ✅ POSITIVO | Préstamo para planta de tratamiento de aguas residuales con biodigestor | 503 | Tratamiento Agua |
| 7 | ✅ POSITIVO | Financiamiento para compostadora industrial de fracción orgánica | 104 | Compostaje |
| 8 | ✅ POSITIVO | Crédito para reforestación y conservación de cuencas hidrográficas | 901 | Gestión Desastres |
| 9 | ✅ POSITIVO | Préstamo para conservación de arrecifes de coral y manglares | 1001 | Activos Naturales |
| 10 | ✅ POSITIVO | Financiamiento para producción de bioplásticos y polímeros reciclados | 607 | Industria Verde |

---

## 2. Casos Positivos — Aprendizaje requerido

| # | Estado | Descripción | Código esperado | Vocabulario nuevo |
|---|---|---|---|---|
| 1 | 🧠 APRENDE | Crédito para sistema de generación distribuida fotovoltaica en techo | 201 | generación distribuida fotovoltaica |
| 2 | 🧠 APRENDE | Financiamiento para electromovilidad corporativa de última milla | 405 | electromovilidad, última milla |
| 3 | 🧠 APRENDE | Préstamo para retrofit energético con mejora de envolvente térmica | 302 | retrofit, envolvente térmica |
| 4 | 🧠 APRENDE | Crédito para implementación de sistema MRV de medición de emisiones | 108 | sistema MRV |
| 5 | 🧠 APRENDE | Financiamiento para electrolizador de producción de H2 verde | 220 | electrolizador, H2 verde |
| 6 | 🧠 APRENDE | Préstamo para planta de valorización material de RSU | 105 | valorización material, RSU |
| 7 | 🧠 APRENDE | Crédito para membranas de ósmosis inversa en planta potabilizadora | 503 | ósmosis inversa, potabilizadora |
| 8 | 🧠 APRENDE | Financiamiento para certificación EDGE de edificio comercial | 303 | certificación EDGE |
| 9 | 🧠 APRENDE | Préstamo para cogeneración de calor residual en proceso industrial | 219 | cogeneración calor residual |
| 10 | 🧠 APRENDE | Crédito para infraestructura de bajo consumo para data center verde | 701 | data center verde |

---

## 3. Casos Múltiples — Dos categorías

| # | Estado | Descripción | Códigos esperados | Categorías |
|---|---|---|---|---|
| 1 | 🔀 MÚLTIPLE ✅ | Financiamiento para edificio con paneles solares y certificación LEED | 201 + 301 | Energía Solar + Construcción Verde |
| 2 | 🔀 MÚLTIPLE ✅ | Crédito para vehículo eléctrico e instalación de cargador domiciliario | 405 + 403 | Transporte Particular + Infraestructura |
| 3 | 🔀 MÚLTIPLE ✅ | Préstamo para sistema de riego sostenible con bombeo solar | 201 + 504 | Energía Solar + Eficiencia Agua |
| 4 | 🔀 MÚLTIPLE ✅ | Financiamiento para biodigestor que genera biogás y trata aguas residuales | 207 + 503 | Bioenergía + Tratamiento Agua |
| 5 | 🔀 MÚLTIPLE ✅ | Crédito para planta de compostaje con captura de gases de efecto invernadero | 104 + 108 | Compostaje + Captura GEI |
| 6 | 🔀 MÚLTIPLE ✅ | Préstamo para techos verdes y sistema de captación de agua pluvial | 803 + 504 | Drenaje Urbano + Eficiencia Agua |
| 7 | 🔀 MÚLTIPLE ✅ | Financiamiento para flota de bicicletas eléctricas y estaciones de carga | 402 + 403 | Micromovilidad + Infraestructura |
| 8 | 🔀 MÚLTIPLE ✅ | Crédito para producción de biogás a partir de residuos orgánicos | 103 + 207 | Digestión Orgánicos + Bioenergía |

---

## 4. Casos Múltiples — Tres o más categorías

| # | Estado | Descripción | Códigos esperados | Categorías |
|---|---|---|---|---|
| 1 | 🔀 MÚLTIPLE ✅ | Financiamiento para edificio con paneles solares, reutilización de agua y certificación LEED | 201 + 301 + 504 | Energía Solar + Construcción + Agua |
| 2 | 🔀 MÚLTIPLE ✅ | Crédito para planta industrial eficiente con cogeneración, tratamiento de efluentes y paneles solares | 201 + 219 + 503 | Solar + Calor Residual + Agua |
| 3 | 🔀 MÚLTIPLE ✅ | Préstamo para parque eólico con baterías de almacenamiento y red de distribución verde | 203 + 209 + 208 | Eólica + Almacenamiento + Transmisión |
| 4 | 🔀 MÚLTIPLE ✅ | Financiamiento para hub de movilidad eléctrica con electrolineras, bicicletas y autobuses eléctricos | 401 + 402 + 403 | Transporte Público + Micro + Infraestructura |
| 5 | 🔀 MÚLTIPLE ✅ | Crédito para proyecto de economía circular con reciclaje, compostaje y biogás | 105 + 104 + 207 | Reciclaje + Compostaje + Bioenergía |

---

## 5. Casos Negativos — Impacto ambiental negativo

| # | Estado | Descripción | Impacto esperado |
|---|---|---|---|
| 1 | ❌ NEGATIVO | Financiamiento para perforación de pozos petroleros en zona costera | Negativo — fósil |
| 2 | ❌ NEGATIVO | Crédito para ampliación de planta de carbón para generación eléctrica | Negativo — fósil |
| 3 | ❌ NEGATIVO | Préstamo para compra de flota de camiones diésel de carga pesada | Negativo — transporte contaminante |
| 4 | ❌ NEGATIVO | Financiamiento para gasoducto de gas natural no renovable | Negativo — fósil |
| 5 | ❌ NEGATIVO | Crédito para refinería de petróleo en zona industrial | Negativo — fósil |

---

## 6. Casos Neutros — Sin impacto ambiental

| # | Estado | Descripción | Código esperado |
|---|---|---|---|
| 1 | ⚪ NEUTRO | Financiamiento para equipos de cómputo para oficinas administrativas | 0 |
| 2 | ⚪ NEUTRO | Crédito para remodelación de sucursales bancarias sin criterios de eficiencia | 0 |
| 3 | ⚪ NEUTRO | Préstamo para adquisición de maquinaria industrial de producción general | 0 |
| 4 | ⚪ NEUTRO | Financiamiento para infraestructura de telecomunicaciones convencional | 0 |
| 5 | ⚪ NEUTRO | Crédito para compra de terreno sin proyecto ambiental definido | 0 |

---

## 7. Casos No Aplica — Excluidos por reglas

| # | Estado | Descripción | Regla de exclusión |
|---|---|---|---|
| 1 | ⛔ NO APLICA | Préstamo para capital de trabajo de empresa de alimentos | Keyword: capital de trabajo |
| 2 | ⛔ NO APLICA | Crédito para nómina de personal de empresa solar | Keyword: nómina |
| 3 | ⛔ NO APLICA | Financiamiento para pintar las oficinas de color verde | Keyword: pintar |
| 4 | ⛔ NO APLICA | Préstamo para reciclaje de datos y gestión de información digital | Keyword: reciclaje de datos |
| 5 | ⛔ NO APLICA | Crédito para publicidad de productos ecológicos | Sin actividad ambiental concreta |
| 6 | ⛔ NO APLICA | Financiamiento para mobiliario de oficina ecológica | Sin actividad ambiental concreta |

---

## 8. Casos Trampa — Deben clasificarse como No Aplica

| # | Estado | Descripción | Por qué es trampa |
|---|---|---|---|
| 1 | ⛔ NO APLICA | Crédito para empresa Verde S.A. para capital de trabajo | El nombre "Verde" no es criterio ESG |
| 2 | ⛔ NO APLICA | Préstamo para jardín decorativo con plantas en fachada | No es infraestructura ambiental |
| 3 | ⛔ NO APLICA | Financiamiento para estudio de factibilidad solar sin ejecución | Sin actividad concreta financiada |
| 4 | ⛔ NO APLICA | Crédito para vehículo eléctrico de gerente para uso personal mixto | El uso personal mixto no califica |
| 5 | ⛔ NO APLICA | Préstamo para empresa de reciclaje para pagar deudas | La actividad financiada no es ambiental |

---

## 9. Casos Ambiguos — Requieren revisión manual

| # | Estado | Descripción | Ambigüedad |
|---|---|---|---|
| 1 | ⚠️ AMBIGUO | Financiamiento para biodigestor de aguas residuales industriales | 101 vs 103 vs 503 |
| 2 | ⚠️ AMBIGUO | Crédito para construcción de bodega industrial sostenible | 301 vs 0 — depende de certificación |
| 3 | ⚠️ AMBIGUO | Préstamo para planta de tratamiento de residuos orgánicos e industriales | 103 vs 503 |
| 4 | ⚠️ AMBIGUO | Financiamiento para proyecto de energía con componente de consultoría | 201 vs 221 |
| 5 | ⚠️ AMBIGUO | Crédito para empresa de transporte que compra un vehículo eléctrico y diésel | 405 vs 0 |

---

## Resumen de casos

| Tipo | Total | FuzzySharp | Groq | Códigos |
|---|---|---|---|---|
| ✅ Positivos una categoría | 10 | | | Varios |
| 🧠 Positivos aprendizaje | 10 | | | Varios |
| 🔀 Múltiples dos categorías | 8 | | | Múltiples |
| 🔀 Múltiples tres+ categorías | 5 | | | Múltiples |
| ❌ Negativos | 5 | | | 0 |
| ⚪ Neutros | 5 | | | 0 |
| ⛔ No aplica | 6 | | | 0 |
| ⛔ Trampa | 5 | | | 0 |
| ⚠️ Ambiguos | 5 | | | 0 |
| **Total** | **59** | | | |

---

## Consultas de verificación post-pruebas

```sql
-- Ver todo lo aprendido ordenado por fecha
SELECT
    s.code,
    s.label,
    l.keyword,
    l.source_text,
    ROUND(l.llm_confidence * 100) AS confidence_pct,
    l.created_at
FROM esg_keyword_learning_log l
JOIN esg_taxonomy_subcategory s ON s.code = l.subcategory_code
ORDER BY l.created_at DESC;

-- Casos multi-categoría — verificar keywords aprendidas por código
SELECT
    s.code,
    s.label,
    array_length(s.keywords, 1) AS total_keywords,
    s.updated_at
FROM esg_taxonomy_subcategory s
ORDER BY s.updated_at DESC
LIMIT 20;

-- Revertir keyword incorrecta si es necesario
-- UPDATE esg_taxonomy_subcategory
-- SET keywords = array_remove(keywords, 'keyword incorrecta')
-- WHERE code = 201;
```
