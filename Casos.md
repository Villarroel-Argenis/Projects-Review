# ESG POC — Pruebas Finales Integradas

## Instrucciones
1. Ejecutar cada caso en el clasificador
2. Verificar motor usado (FuzzySharp vs Groq) y resultado
3. Marcar ✅ si el resultado es correcto

---

## 1. Elegibles Verdes — Una Categoría

| # | Input | Código | Motor esperado | Resultado | ✓ |
|---|---|---|---|---|---|
| 1 | `Instalación de paneles solares en techo industrial` | 201 | FuzzySharp | | |
| 2 | `Compra de aerogeneradores para parque eólico` | 203 | FuzzySharp | | |
| 3 | `Planta de tratamiento de aguas residuales con biodigestor` | 503 | FuzzySharp | | |
| 4 | `Adquisición de vehículo eléctrico de batería` | 405 | FuzzySharp | | |
| 5 | `Reforestación y conservación de cuencas hidrográficas` | 901 | FuzzySharp | | |
| 6 | `Compostaje de residuos orgánicos de restaurante` | 104 | FuzzySharp | | |
| 7 | `Baterías de litio para almacenamiento de energía solar` | 209 | FuzzySharp | | |
| 8 | `Conservación de arrecifes de coral y manglares` | 1001 | FuzzySharp | | |
| 9 | `Producción de bioplásticos y polímeros reciclados` | 607 | FuzzySharp | | |
| 10 | `Pavimentos permeables y techos verdes para drenaje urbano` | 803 | FuzzySharp | | |

---

## 2. Elegibles Verdes — Múltiples Categorías

| # | Input | Códigos esperados | Motor esperado | Resultado | ✓ |
|---|---|---|---|---|---|
| 1 | `Edificio con paneles solares y certificación LEED` | 201 + 301 | Groq | | |
| 2 | `Vehículo eléctrico con instalación de cargador domiciliario` | 405 + 403 | Groq | | |
| 3 | `Sistema de riego sostenible con bombeo solar` | 201 + 504 | Groq | | |
| 4 | `Biodigestor que genera biogás y trata aguas residuales` | 207 + 503 | Groq | | |
| 5 | `Planta de compostaje con captura de gases de efecto invernadero` | 104 + 108 | Groq | | |
| 6 | `Flota de bicicletas eléctricas y estaciones de carga` | 402 + 403 | Groq | | |
| 7 | `Parque eólico con baterías de almacenamiento y red de distribución` | 203 + 209 + 208 | Groq | | |
| 8 | `Edificio con paneles solares, reutilización de agua y certificación LEED` | 201 + 301 + 504 | Groq | | |

---

## 3. Impacto Negativo — Actividades Dañinas

> FuzzySharp debe resolver sin Groq después del primer aprendizaje.

| # | Input | Impacto | Motor 1ra vez | Motor 2da vez | ✓ |
|---|---|---|---|---|---|
| 1 | `Financiamiento para refinería de petróleo en zona industrial` | Negativo 🚫 | Groq | FuzzySharp | |
| 2 | `Crédito para flota de camiones diésel de carga pesada` | Negativo 🚫 | Groq | FuzzySharp | |
| 3 | `Préstamo para planta de generación con carbón` | Negativo 🚫 | Groq | FuzzySharp | |
| 4 | `Perforación de pozos petroleros en zona costera` | Negativo 🚫 | Groq | FuzzySharp | |
| 5 | `Proyecto de fracturación hidráulica fracking` | Negativo 🚫 | Groq | FuzzySharp | |

---

## 4. Impacto Neutro — No Ambiental

> FuzzySharp debe resolverlo con exclusiones.

| # | Input | Resultado esperado | Motor esperado | ✓ |
|---|---|---|---|---|
| 1 | `Financiamiento para equipos de cómputo de oficina` | ⚪ No ambiental | FuzzySharp | |
| 2 | `Crédito para nómina de personal administrativo` | ⚪ No ambiental | FuzzySharp | |
| 3 | `Préstamo para capital de trabajo de empresa` | ⚪ No ambiental | FuzzySharp | |
| 4 | `Crédito para pintar oficinas de color verde` | ⚪ No ambiental | FuzzySharp | |
| 5 | `Financiamiento para oficinas administrativas nuevas` | ⚪ No ambiental | FuzzySharp | |
| 6 | `Préstamo para reciclaje de datos y gestión digital` | ⚪ No ambiental | FuzzySharp | |
| 7 | `Crédito para remodelación sin criterios de eficiencia` | ⚪ No ambiental | FuzzySharp | |

---

## 5. Aprendizaje de Keywords — Groq enseña a FuzzySharp

> Probar dos veces cada caso. La primera invoca Groq; la segunda debe resolverla FuzzySharp.

| # | Input | Código | 1ra vez | 2da vez | Keyword aprendida | ✓ |
|---|---|---|---|---|---|---|
| 1 | `Sistema de generación distribuida fotovoltaica en techo` | 201 | Groq | FuzzySharp | generación distribuida fotovoltaica | |
| 2 | `Electromovilidad corporativa para flota de última milla` | 405 | Groq | FuzzySharp | electromovilidad, última milla | |
| 3 | `Retrofit energético con mejora de envolvente térmica` | 302 | Groq | FuzzySharp | retrofit, envolvente térmica | |
| 4 | `Sistema MRV de medición y reporte de emisiones` | 108 | Groq | FuzzySharp | sistema mrv | |
| 5 | `Electrolizador para producción de H2 verde` | 220 | Groq | FuzzySharp | electrolizador, h2 verde | |
| 6 | `Planta de valorización material de RSU` | 105 | Groq | FuzzySharp | valorización material, rsu | |
| 7 | `Membranas de ósmosis inversa en planta potabilizadora` | 503 | Groq | FuzzySharp | ósmosis inversa, potabilizadora | |
| 8 | `Certificación EDGE para edificio comercial` | 303 | Groq | FuzzySharp | certificación edge | |

---

## 6. Corrección Ortográfica — Antes de FuzzySharp

| # | Input con error | Corrección esperada | Código | ✓ |
|---|---|---|---|---|
| 1 | `Panales solares industriales` | paneles solares | 201 | |
| 2 | `Gaces de inbernader` | gases de invernadero | 108 | |
| 3 | `Refinera de petroleo` | refinería de petróleo | Negativo | |
| 4 | `Biogas de residuos organicos` | biogás de residuos orgánicos | 207 | |
| 5 | `Reciclaje de plastico y latas` | reciclaje de plástico | 105 | |
| 6 | `Vehiculo electrico de bateria` | vehículo eléctrico de batería | 405 | |
| 7 | `Compostadora industrial de residuos organicos` | compostadora industrial | 104 | |

---

## 7. Casos Trampa — No Deben Clasificarse como Verde

| # | Input | Por qué es trampa | Resultado esperado | ✓ |
|---|---|---|---|---|
| 1 | `Empresa Verde S.A. para nómina de empleados` | Nombre no es criterio ESG | ⚪ No ambiental | |
| 2 | `Jardín decorativo con plantas en fachada` | No es infraestructura ambiental | ⚪ No ambiental | |
| 3 | `Estudio de factibilidad solar sin ejecución` | Sin actividad concreta financiada | ⚪ No ambiental | |
| 4 | `Reciclaje de datos del sistema de información` | Reciclaje digital no es ESG | ⚪ No ambiental | |
| 5 | `Préstamo para empresa de reciclaje para pagar deudas` | La actividad financiada no es ambiental | ⚪ No ambiental | |

---

## Consultas de verificación post-pruebas

```sql
-- Keywords aprendidas ordenadas por fecha
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

-- Reglas de exclusión aprendidas
SELECT id, description, keyword, is_harmful, created_at
FROM esg_classification_rules
WHERE rule_type = 'negative' AND keyword IS NOT NULL
ORDER BY created_at DESC;

-- Vocabulario ortográfico aprendido
SELECT word, source, created_at
FROM esg_spell_vocabulary
WHERE source = 'learned'
ORDER BY created_at DESC;

-- Resumen de aprendizaje por subcategoría
SELECT
    s.code,
    s.label,
    COUNT(l.id) AS keywords_aprendidas,
    array_length(s.keywords, 1) AS total_keywords
FROM esg_taxonomy_subcategory s
LEFT JOIN esg_keyword_learning_log l ON l.subcategory_code = s.code
GROUP BY s.code, s.label, s.keywords
HAVING COUNT(l.id) > 0
ORDER BY s.code;
```
