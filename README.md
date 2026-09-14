# REVALORIZACIÓN DE COSTOS DE PRODUCCIÓN

## 1. Resumen

Durante la revisión del historial de producción en **Odoo 18**, se detectó que **20 órdenes de producción** cerradas presentaron una asignación errónea en la distribución de sus costos de fabricación entre los tres subproductos obtenidos:
- **[SE-0001] YEMA FILTRADA**
- **[SE-0004] CLARA FILTRADA**
- **[SE-0002] HUEVO ENTERO PROCESO 1. BAJO BRIX**

El modelo técnico y financiero estándar de la compañía establece que los costos de producción (materia prima, insumos y mano de obra/operaciones) deben distribuirse estrictamente bajo el esquema **72% Yema, 18% Clara y 10% Huevo Entero**.
Sin embargo, los costos se distribuyeron de manera incorrecta (asignando hasta un ~68% a la Clara y ~14% a la Yema), lo que provocó una subvaloración artificial de la Yema y una sobrevaloración severa de la Clara y el Huevo Entero.

## 2. Impacto y Regularización de Costos Unitarios de los Productos

En Odoo, el método de valoración de inventario está configurado bajo **Costo Promedio Ponderado (AVCO)** con valoración automatizada en tiempo real. Debido a las 20 órdenes mal distribuidas, el costo unitario (`standard_price`) en la ficha de cada producto quedó fuertemente distorsionado.

Se realizó el recálculo analítico del **Costo Promedio Ponderado Real** considerando la totalidad de las 53 órdenes de producción cerradas con el reparto exacto (72% / 18% / 10%):

| Código | Nombre del Semielaborado | Reparto Teórico | Coste Actual en Odoo | Nuevo Coste Real Ponderado | Variación Unitaria | Diagnóstico de Desviación |
| :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| **`SE-0001`** | **YEMA FILTRADA** | 72.0% | `Bs. 6,446.76 / kg` | **`Bs. 4,997.17 / kg`** | `-1,449.59 Bs.` | Subvalorado en órdenes afectadas |
| **`SE-0004`** | **CLARA FILTRADA** | 18.0% | `Bs. 3,448.81 / kg` | **`Bs. 646.33 / kg`** | `-2,802.48 Bs.` | Sobrevalorado ~5.3 veces |
| **`SE-0002`** | **HUEVO ENTERO PROCESO 1. BAJO BRIX** | 10.0% | `Bs. 8,692.95 / kg` | **`Bs. 3,515.56 / kg`** | `-5,177.39 Bs.` | Sobrevalorado ~2.5 veces |

> **Comportamiento Nativo en Odoo:** Al registrar las capas de valoración (`stock.valuation.layer`), Odoo deriva e impacta el coste de los productos y la valoración de inventario de forma automática y transparente a través de su motor de costeo promedio (AVCO), sin necesidad de alterar manualmente las fichas técnicas de los productos.

## 3. Metodología

Para asegurar que los ajustes no alteren indebidamente los balances contables ni dejen brechas entre el inventario físico y la contabilidad financiera, se implementó un modelo de **trazabilidad descendente exhaustiva** lote por lote:

```
 ┌─────────────────────────────────────────────────────────────────────────────┐
 │               FASE 1: ORDEN DE PRODUCCIÓN PRIMARIA (SE-0001)                │
 │        Costo Total de Fabricación = Materia Prima + Mano de Obra + CF       │
 └──────────────────────────────────────┬──────────────────────────────────────┘
                                        │
              Lotes de Semielaborados (SE-0001, SE-0004, SE-0002)
                                        │
 ┌──────────────────────────────────────┴──────────────────────────────────────┐
 │          FASE 2: TRANSFORMACIÓN EN PRODUCTO TERMINADO (PASTEURIZACIÓN)      │
 │        Consumo de lotes en órdenes de pasteurización (Nivel 2 y 3)          │
 └──────────────────────────────────────┬──────────────────────────────────────┘
                                        │
               Lotes de PT Generados (PTL-0001-R, PTL-0022-C, etc.)
                                        │
        ┌───────────────────────────────┴───────────────────────────────┐
        ▼                                                               ▼
   [DESTINO A: VENDIDO A CLIENTES]                             [DESTINO B: EN INVENTARIO (0% VENDIDO)]
   • Albaranes de salida (ventas)                              • Stock en tanques / almacén / cuarentena
   • Criterio: El producto ya salió de la empresa              • Criterio: El producto sigue en la empresa
   • Imputación contable:                                      • Imputación contable:
     5110000001 COSTO DE BIENES VENDIDOS                         [NO REQUIERE ASIENTO A COSTO DE VENTAS]
     (Ajusta el margen en Estado de Resultados)                  (Pasa semielaborado contra semielaborado;
     vs 1420000002 PRODUCTOS SEMIELABORADOS                      efecto neto patrimonial = Bs. 0.00)
                                                               • Capas de Valoración (SVL):
                                                                 Crea capas stock.valuation.layer
```

## 4. Ordenes de producción afectadas

A continuación se presenta el desglose técnico, trazabilidad multinivel, capas de valoración y asiento contable propuesto para cada una de las 20 órdenes afectadas.

### 4.1 Orden de Producción: `OVO/PRO/SE/00111`
- **Fecha de Cierre de Fabricación:** `2026-09-11 13:32:57`
- **Fecha de Registro Contable / Asiento:** `2026-09-11`
- **Costo Total Fabricación:** **Bs. 57,837,332.94** (Componentes/Materia Prima: `Bs. 49,554,919.18` | Mano de Obra y Operaciones: `Bs. 8,282,413.76`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 5653.0 kg | Bs. 10,804,013.79 (18.68%) | Bs. 41,642,879.72 (72.0%) | **+30,838,865.92 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 11117.0 kg | Bs. 38,340,368.01 (66.29%) | Bs. 10,410,719.93 (18.0%) | **-27,929,648.08 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 1000.0 kg | Bs. 8,692,951.14 (15.03%) | Bs. 5,783,733.29 (10.0%) | **-2,909,217.85 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260911-0052`, `5653.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00344-001, OVO/PRO/PTL/00363-001, OVO/PRO/PTL/00344-002
  - **Lotes PT Derivados:** `PTLYE-26-0302, PTLYE-26-0301, PTLYE-26-0300, SEYERE-20260911-0059`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **5653.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260911-0052`, `11117.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00364
  - **Lotes PT Derivados:** `SECLRE-20260911-0053`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **11117.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260911-0052`, `1000.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00365
  - **Lotes PT Derivados:** `SEHERE-20260911-0055`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **1000.0 kg EN INVENTARIO (100.00%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+30,838,865.92 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00111 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-27,929,648.08 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00111 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-2,909,217.85 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00111 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
> **[ℹ️ Inventario 100% - No requiere asiento contable a Costo de Ventas]**  
> Justificación: Ningún lote derivado fue entregado a clientes. La producción permanece en almacenes/tanques; la revalorización queda 100% reflejada en el Balance General mediante las capas de valoración (`stock.valuation.layer`).

---

### 4.2 Orden de Producción: `OVO/PRO/SE/00108`
- **Fecha de Cierre de Fabricación:** `2026-09-10 15:51:25`
- **Fecha de Registro Contable / Asiento:** `2026-09-10`
- **Costo Total Fabricación:** **Bs. 54,863,650.38** (Componentes/Materia Prima: `Bs. 47,184,872.64` | Mano de Obra y Operaciones: `Bs. 7,678,777.73`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 5241.0 kg | Bs. 9,414,602.40 (17.16%) | Bs. 39,501,828.27 (72.0%) | **+30,087,225.87 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 10987.0 kg | Bs. 37,142,691.30 (67.7%) | Bs. 9,875,457.07 (18.0%) | **-27,267,234.24 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 935.0 kg | Bs. 8,306,356.67 (15.14%) | Bs. 5,486,365.04 (10.0%) | **-2,819,991.63 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260910-0051`, `5241.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00336-002, OVO/PRO/PTL/00344-001, OVO/PRO/PTL/00363-001, OVO/PRO/PTL/00336-001
  - **Lotes PT Derivados:** `PTLYE-26-0302, PTLYE-26-0298, PTLYE-26-0299, PTLYE-26-0300, SEYERE-20260911-0058`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **5241.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260910-0051`, `10987.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00356, OVO/PRO/PTL/00351
  - **Lotes PT Derivados:** `PTLCL-26-0025, SECLRE-20260911-0052`
  - **Entregas a Clientes:** OVO/OUT/00523 (336.0 kg)
  - **Estado Final:** **336.0 kg VENDIDO (3.06%)** | **10651.0 kg EN INVENTARIO (96.94%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260910-0051`, `935.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00357
  - **Lotes PT Derivados:** `SEHERE-20260911-0054`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **935.0 kg EN INVENTARIO (100.00%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+30,087,225.87 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00108 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-27,267,234.24 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00108 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-2,819,991.63 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00108 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-09-10` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00108`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00108 - [SE-0004] CLARA (3.06% vendido) |  | Bs. 834,377.37 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00108 | Bs. 834,377.37 |  |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 834,377.37** | **Bs. 834,377.37** |

---

### 4.3 Orden de Producción: `OVO/PRO/SE/00104`
- **Fecha de Cierre de Fabricación:** `2026-09-09 21:30:26`
- **Fecha de Registro Contable / Asiento:** `2026-09-09`
- **Costo Total Fabricación:** **Bs. 35,892,899.17** (Componentes/Materia Prima: `Bs. 31,137,067.74` | Mano de Obra y Operaciones: `Bs. 4,755,831.43`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 3246.0 kg | Bs. 2,630,949.51 (7.33%) | Bs. 25,842,887.40 (72.0%) | **+23,211,937.89 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 7521.0 kg | Bs. 25,071,190.07 (69.85%) | Bs. 6,460,721.85 (18.0%) | **-18,610,468.22 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 960.0 kg | Bs. 8,190,759.59 (22.82%) | Bs. 3,589,289.92 (10.0%) | **-4,601,469.67 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260909-0050`, `3246.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00334-001, OVO/PRO/PTL/00334-002, OVO/PRO/PTL/00344-001
  - **Lotes PT Derivados:** `PTLYE-26-0300, PTLYE-26-0297, PTLYE-26-0296, SEYERE-20260910-0057`
  - **Entregas a Clientes:** OVO/OUT/00531 (2200.0 kg)
  - **Estado Final:** **1976.0 kg VENDIDO (60.87%)** | **1270.0 kg EN INVENTARIO (39.13%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260909-0050`, `7521.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00348
  - **Lotes PT Derivados:** `SECLRE-20260909-0051`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **7521.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260909-0050`, `960.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00362-001, OVO/PRO/PTL/00362-002, OVO/PRO/PTL/00349
  - **Lotes PT Derivados:** `SEHERE-20260909-0053, PTLHE-26-0267, PTLHE-26-0268`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **960.0 kg EN INVENTARIO (100.00%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+23,211,937.89 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00104 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-18,610,468.22 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00104 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-4,601,469.67 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00104 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-09-09` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00104`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00104 - [SE-0001] YEMA (60.87% vendido) | Bs. 14,129,106.60 |  |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00104 |  | Bs. 14,129,106.60 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 14,129,106.60** | **Bs. 14,129,106.60** |

---

### 4.4 Orden de Producción: `OVO/PRO/SE/00106`
- **Fecha de Cierre de Fabricación:** `2026-09-09 18:15:01`
- **Fecha de Registro Contable / Asiento:** `2026-09-09`
- **Costo Total Fabricación:** **Bs. 43,508,569.79** (Componentes/Materia Prima: `Bs. 37,165,996.08` | Mano de Obra y Operaciones: `Bs. 6,342,573.71`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 4329.0 kg | Bs. 8,214,417.98 (18.88%) | Bs. 31,326,170.25 (72.0%) | **+23,111,752.27 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 8225.0 kg | Bs. 28,506,814.93 (65.52%) | Bs. 7,831,542.56 (18.0%) | **-20,675,272.36 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 800.0 kg | Bs. 6,787,336.89 (15.6%) | Bs. 4,350,856.98 (10.0%) | **-2,436,479.91 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260909-0049`, `4329.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00329, OVO/PRO/PTL/00344-001, OVO/PRO/PTL/00334-002
  - **Lotes PT Derivados:** `SEYERE-20260910-0057, SEYERE-20260909-0056, SEYERE-20260909-0055, PTLYE-26-0297, PTLYE-26-0295, PTLYE-26-0300`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **4329.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260909-0049`, `8225.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTD/00011, OVO/PRO/PTL/00342, OVO/PRO/PTL/00341
  - **Lotes PT Derivados:** `PTLCL-26-0024, SECLRE-20260909-0050, PTDAL-26-257-0021`
  - **Entregas a Clientes:** OVO/OUT/00528 (48.0 kg)
  - **Estado Final:** **48.0 kg VENDIDO (0.58%)** | **8177.0 kg EN INVENTARIO (99.42%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260909-0049`, `800.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00343, OVO/PRO/PTL/00362-001, OVO/PRO/PTL/00335-002, OVO/PRO/PTL/00337-002
  - **Lotes PT Derivados:** `PTLHE-26-0264, PTLHE-26-0266, PTLHE-26-0267, SEHERE-20260909-0052`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **800.0 kg EN INVENTARIO (100.00%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+23,111,752.27 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00106 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-20,675,272.36 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00106 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-2,436,479.91 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00106 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-09-09` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00106`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00106 - [SE-0004] CLARA (0.58% vendido) |  | Bs. 119,916.58 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00106 | Bs. 119,916.58 |  |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 119,916.58** | **Bs. 119,916.58** |

---

### 4.5 Orden de Producción: `OVO/PRO/SE/00101`
- **Fecha de Cierre de Fabricación:** `2026-09-08 09:54:53`
- **Fecha de Registro Contable / Asiento:** `2026-09-08`
- **Costo Total Fabricación:** **Bs. 49,982,243.98** (Componentes/Materia Prima: `Bs. 42,222,883.77` | Mano de Obra y Operaciones: `Bs. 7,759,360.21`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 5296.0 kg | Bs. 10,921,120.31 (21.85%) | Bs. 35,987,215.67 (72.0%) | **+25,066,095.36 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 9415.0 kg | Bs. 31,988,636.15 (64.0%) | Bs. 8,996,803.92 (18.0%) | **-22,991,832.23 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 873.0 kg | Bs. 7,072,487.52 (14.15%) | Bs. 4,998,224.40 (10.0%) | **-2,074,263.13 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260908-0048`, `5296.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00326-002, OVO/PRO/PTL/00326-001
  - **Lotes PT Derivados:** `PTLYE-26-0294, PTLYE-26-0293`
  - **Entregas a Clientes:** OVO/OUT/00530 (2200.0 kg), OVO/OUT/00531 (3000.0 kg)
  - **Estado Final:** **5296.0 kg VENDIDO (100.0%)** | **0.0 kg EN INVENTARIO (0.0%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260908-0048`, `9415.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00330, BIODA/MAQ/00011
  - **Lotes PT Derivados:** `PTDAL-26-254-0020, SECLRE-20260908-0049`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **9415.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260908-0048`, `873.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00337-002, OVO/PRO/PTL/00331, OVO/PRO/PTL/00337-001
  - **Lotes PT Derivados:** `PTLHE-26-0265, PTLHE-26-0266, SEHERE-20260908-0050`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **873.0 kg EN INVENTARIO (100.00%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+25,066,095.36 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00101 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-22,991,832.23 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00101 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-2,074,263.13 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00101 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-09-08` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00101`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00101 - [SE-0001] YEMA (100.0% vendido) | Bs. 25,066,095.36 |  |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00101 |  | Bs. 25,066,095.36 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 25,066,095.36** | **Bs. 25,066,095.36** |

---

### 4.6 Orden de Producción: `OVO/PRO/SE/00097`
- **Fecha de Cierre de Fabricación:** `2026-09-04 23:13:19`
- **Fecha de Registro Contable / Asiento:** `2026-09-04`
- **Costo Total Fabricación:** **Bs. 48,009,286.22** (Componentes/Materia Prima: `Bs. 41,385,406.40` | Mano de Obra y Operaciones: `Bs. 6,623,879.82`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 4521.0 kg | Bs. 4,930,553.69 (10.27%) | Bs. 34,566,686.08 (72.0%) | **+29,636,132.38 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 8897.0 kg | Bs. 31,834,957.69 (66.31%) | Bs. 8,641,671.52 (18.0%) | **-23,193,286.17 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 1383.0 kg | Bs. 11,243,774.83 (23.42%) | Bs. 4,800,928.62 (10.0%) | **-6,442,846.21 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260904-0047`, `4521.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00318-002, OVO/PRO/PTL/00344-001, OVO/PRO/PTL/00334-002, OVO/PRO/PTL/00318-001
  - **Lotes PT Derivados:** `SEYERE-20260910-0057, PTLYE-26-0297, PTLYE-26-0291, PTLYE-26-0290, SEYERE-20260904-0054, PTLYE-26-0300`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **4521.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260904-0047`, `8897.0 kg`):
  - **Órdenes de Transformación:** BIODA/MAQ/00011, OVO/PRO/PTL/00319
  - **Lotes PT Derivados:** `PTDAL-26-254-0020, SECLRE-20260904-0048`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **8897.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260904-0047`, `1383.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00328, OVO/PRO/PTL/00335-001, OVO/PRO/PTL/00337-001, OVO/PRO/PTL/00320, OVO/PRO/PTL/00332
  - **Lotes PT Derivados:** `PTLHE-26-0265, PTLHE-26-0263, SEHERE-20260904-0049, PTLHE-26-0262, PTLHE-26-0258`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **1383.0 kg EN INVENTARIO (100.00%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+29,636,132.38 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00097 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-23,193,286.17 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00097 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-6,442,846.21 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00097 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
> **[ℹ️ Inventario 100% - No requiere asiento contable a Costo de Ventas]**  
> Justificación: Ningún lote derivado fue entregado a clientes. La producción permanece en almacenes/tanques; la revalorización queda 100% reflejada en el Balance General mediante las capas de valoración (`stock.valuation.layer`).

---

### 4.7 Orden de Producción: `OVO/PRO/SE/00095`
- **Fecha de Cierre de Fabricación:** `2026-09-04 22:32:19`
- **Fecha de Registro Contable / Asiento:** `2026-09-04`
- **Costo Total Fabricación:** **Bs. 40,281,795.00** (Componentes/Materia Prima: `Bs. 34,261,551.21` | Mano de Obra y Operaciones: `Bs. 6,020,243.79`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 4109.0 kg | Bs. 5,627,366.76 (13.97%) | Bs. 29,002,892.40 (72.0%) | **+23,375,525.64 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 8125.0 kg | Bs. 26,751,140.06 (66.41%) | Bs. 7,250,723.10 (18.0%) | **-19,500,416.96 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 1003.0 kg | Bs. 7,903,288.18 (19.62%) | Bs. 4,028,179.50 (10.0%) | **-3,875,108.68 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260904-0046`, `4109.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00344-001, OVO/PRO/PTL/00326-001, OVO/PRO/PTL/00334-002, OVO/PRO/PTL/00309-001, OVO/PRO/PTL/00309-002
  - **Lotes PT Derivados:** `PTLYE-26-0288, SEYERE-20260910-0057, SEYERE-20260904-0053, PTLYE-26-0297, PTLYE-26-0293, PTLYE-26-0289, PTLYE-26-0300`
  - **Entregas a Clientes:** OVO/OUT/00515 (2200.0 kg), OVO/OUT/00502 (2200.0 kg), OVO/OUT/00530 (2200.0 kg) y 1 entrega(s) más
  - **Estado Final:** **4048.75 kg VENDIDO (98.53%)** | **60.25 kg EN INVENTARIO (1.47%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260904-0046`, `8125.0 kg`):
  - **Órdenes de Transformación:** BIODA/MAQ/00010, OVO/PRO/PTL/00310, BIODA/MAQ/00011
  - **Lotes PT Derivados:** `PTDAL-26-254-0020, PTDAL-26-251-0018, SECLRE-20260904-0047`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **8125.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260904-0046`, `1003.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00328, OVO/PRO/PTL/00325-002, OVO/PRO/PTL/00325-001, OVO/PRO/PTL/00332, OVO/PRO/PTL/00311
  - **Lotes PT Derivados:** `SEHERE-20260904-0048, PTLHE-26-0257, PTLHE-26-0262, PTLHE-26-0258, PTLHE-26-0256`
  - **Entregas a Clientes:** OVO/OUT/00516 (3200.0 kg), OVO/OUT/00530 (2800.0 kg), OVO/OUT/00531 (600.0 kg)
  - **Estado Final:** **675.41 kg VENDIDO (67.34%)** | **327.59 kg EN INVENTARIO (32.66%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+23,375,525.64 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00095 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-19,500,416.96 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00095 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-3,875,108.68 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00095 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-09-04` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00095`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00095 - [SE-0001] YEMA (98.53% vendido) | Bs. 23,031,905.41 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00095 - [SE-0002] HUEVO ENTERO (67.34% vendido) |  | Bs. 2,609,498.18 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00095 |  | Bs. 20,422,407.23 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 23,031,905.41** | **Bs. 23,031,905.41** |

---

### 4.8 Orden de Producción: `OVO/PRO/SE/00094`
- **Fecha de Cierre de Fabricación:** `2026-09-04 21:10:59`
- **Fecha de Registro Contable / Asiento:** `2026-09-04`
- **Costo Total Fabricación:** **Bs. 41,849,331.30** (Componentes/Materia Prima: `Bs. 35,443,756.75` | Mano de Obra y Operaciones: `Bs. 6,405,574.56`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 4372.0 kg | Bs. 4,632,720.98 (11.07%) | Bs. 30,131,518.54 (72.0%) | **+25,498,797.56 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 8279.0 kg | Bs. 27,386,202.40 (65.44%) | Bs. 7,532,879.63 (18.0%) | **-19,853,322.77 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 1342.0 kg | Bs. 9,830,407.92 (23.49%) | Bs. 4,184,933.13 (10.0%) | **-5,645,474.79 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260904-0045`, `4372.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00299-002, OVO/PRO/PTL/00299-001, OVO/PRO/PTL/00326-001
  - **Lotes PT Derivados:** `PTLYE-26-0286, PTLYE-26-0293, PTLYE-26-0287, SEYERE-20260904-0052`
  - **Entregas a Clientes:** OVO/OUT/00525 (800.0 kg), OVO/OUT/00530 (2200.0 kg), OVO/OUT/00531 (800.0 kg)
  - **Estado Final:** **2729.41 kg VENDIDO (62.43%)** | **1642.59 kg EN INVENTARIO (37.57%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260904-0045`, `8279.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTD/00010, BIODA/MAQ/00010, OVO/PRO/PTL/00302, OVO/PRO/PTL/00301
  - **Lotes PT Derivados:** `SECLRE-20260904-0046, PTLCL-26-0023, PTDAL-26-251-0018, PTDAL-26-251-0019`
  - **Entregas a Clientes:** OVO/OUT/00485 (608.0 kg), OVO/OUT/00486 (480.0 kg), OVO/OUT/00507 (96.0 kg)
  - **Estado Final:** **1184.0 kg VENDIDO (14.3%)** | **7095.0 kg EN INVENTARIO (85.7%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260904-0045`, `1342.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00317-002, OVO/PRO/PTL/00308-003, OVO/PRO/PTL/00303, OVO/PRO/PTL/00325-001
  - **Lotes PT Derivados:** `SEHERE-20260904-0047, PTLHE-26-0256, PTLHE-26-0253, PTLHE-26-0255`
  - **Entregas a Clientes:** OVO/OUT/00503 (1000.0 kg), OVO/OUT/00516 (3200.0 kg), OVO/OUT/00530 (200.0 kg)
  - **Estado Final:** **1342.0 kg VENDIDO (100.0%)** | **0.0 kg EN INVENTARIO (0.0%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+25,498,797.56 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00094 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-19,853,322.77 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00094 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-5,645,474.79 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00094 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-09-04` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00094`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00094 - [SE-0001] YEMA (62.43% vendido) | Bs. 15,918,899.32 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00094 - [SE-0004] CLARA (14.3% vendido) |  | Bs. 2,839,025.16 |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00094 - [SE-0002] HUEVO ENTERO (100.0% vendido) |  | Bs. 5,645,474.79 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00094 |  | Bs. 7,434,399.37 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 15,918,899.32** | **Bs. 15,918,899.32** |

---

### 4.9 Orden de Producción: `OVO/PRO/SE/00088`
- **Fecha de Cierre de Fabricación:** `2026-08-31 14:56:08`
- **Fecha de Registro Contable / Asiento:** `2026-08-31`
- **Costo Total Fabricación:** **Bs. 42,905,447.33** (Componentes/Materia Prima: `Bs. 40,822,153.42` | Mano de Obra y Operaciones: `Bs. 2,083,293.91`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 5123.0 kg | Bs. 8,216,393.16 (19.15%) | Bs. 30,891,922.08 (72.0%) | **+22,675,528.91 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 10124.0 kg | Bs. 28,489,217.03 (66.4%) | Bs. 7,722,980.52 (18.0%) | **-20,766,236.51 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 865.0 kg | Bs. 6,199,837.14 (14.45%) | Bs. 4,290,544.73 (10.0%) | **-1,909,292.41 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260831-0044`, `5123.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00275-002, OVO/PRO/PTL/00275-001, OVO/PRO/PTL/00318-001
  - **Lotes PT Derivados:** `PTLYE-26-0290, PTLYE-26-0285, SEYERE-20260831-0050, PTLYE-26-0284`
  - **Entregas a Clientes:** OVO/OUT/00480 (200.0 kg), OVO/OUT/00484 (2000.0 kg), OVO/OUT/00492 (1400.0 kg) y 1 entrega(s) más
  - **Estado Final:** **4521.42 kg VENDIDO (88.26%)** | **601.58 kg EN INVENTARIO (11.74%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260831-0044`, `10124.0 kg`):
  - **Órdenes de Transformación:** BIODA/MAQ/00010, OVO/PRO/PTL/00286, OVO/PRO/PTD/00010
  - **Lotes PT Derivados:** `SECLRE-20260831-0044, PTDAL-26-251-0018, PTDAL-26-251-0019`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **10124.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260831-0044`, `865.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00317-001, OVO/PRO/PTL/00308-001, OVO/PRO/PTL/00287, OVO/PRO/PTL/00317-002, OVO/PRO/PTL/00308-002
  - **Lotes PT Derivados:** `PTLHE-26-0255, SEHERE-20260831-0046, PTLHE-26-0252, PTLHE-26-0254, PTLHE-26-0251`
  - **Entregas a Clientes:** OVO/OUT/00495 (1200.0 kg), OVO/OUT/00504 (2400.0 kg), OVO/OUT/00514 (1000.0 kg) y 1 entrega(s) más
  - **Estado Final:** **865.0 kg VENDIDO (100.0%)** | **0.0 kg EN INVENTARIO (0.0%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+22,675,528.91 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00088 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-20,766,236.51 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00088 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-1,909,292.41 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00088 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-31` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00088`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00088 - [SE-0001] YEMA (88.26% vendido) | Bs. 20,013,421.82 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00088 - [SE-0002] HUEVO ENTERO (100.0% vendido) |  | Bs. 1,909,292.41 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00088 |  | Bs. 18,104,129.41 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 20,013,421.82** | **Bs. 20,013,421.82** |

---

### 4.10 Orden de Producción: `OVO/PRO/SE/00086`
- **Fecha de Cierre de Fabricación:** `2026-08-28 17:49:47`
- **Fecha de Registro Contable / Asiento:** `2026-08-28`
- **Costo Total Fabricación:** **Bs. 45,200,120.77** (Componentes/Materia Prima: `Bs. 43,206,697.64` | Mano de Obra y Operaciones: `Bs. 1,993,423.14`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 4902.0 kg | Bs. 6,052,296.17 (13.39%) | Bs. 32,544,086.96 (72.0%) | **+26,491,790.79 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 10636.0 kg | Bs. 30,939,482.67 (68.45%) | Bs. 8,136,021.74 (18.0%) | **-22,803,460.93 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 1088.0 kg | Bs. 8,208,341.93 (18.16%) | Bs. 4,520,012.08 (10.0%) | **-3,688,329.86 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260828-0043`, `4902.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00275-001, OVO/PRO/PTL/00272-002, OVO/PRO/PTL/00272-001
  - **Lotes PT Derivados:** `PTLYE-26-0282, PTLYE-26-0283, PTLYE-26-0284`
  - **Entregas a Clientes:** OVO/OUT/00493 (3000.0 kg), OVO/OUT/00525 (1800.0 kg), OVO/OUT/00480 (200.0 kg) y 2 entrega(s) más
  - **Estado Final:** **4487.06 kg VENDIDO (91.54%)** | **414.94 kg EN INVENTARIO (8.46%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260828-0043`, `10636.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00284, BIODA/MAQ/00010
  - **Lotes PT Derivados:** `SECLRE-20260831-0043, PTDAL-26-251-0018`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **10636.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260828-0043`, `1088.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00298-001, OVO/PRO/PTL/00308-001, OVO/PRO/PTL/00285, OVO/PRO/PTL/00298-002
  - **Lotes PT Derivados:** `PTLHE-26-0250, PTLHE-26-0251, SEHERE-20260831-0045, PTLHE-26-0249`
  - **Entregas a Clientes:** OVO/OUT/00492 (2600.0 kg), OVO/OUT/00495 (1200.0 kg), OVO/OUT/00504 (2200.0 kg)
  - **Estado Final:** **1088.0 kg VENDIDO (100.0%)** | **0.0 kg EN INVENTARIO (0.0%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+26,491,790.79 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00086 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-22,803,460.93 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00086 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-3,688,329.86 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00086 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-28` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00086`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00086 - [SE-0001] YEMA (91.54% vendido) | Bs. 24,250,585.28 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00086 - [SE-0002] HUEVO ENTERO (100.0% vendido) |  | Bs. 3,688,329.86 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00086 |  | Bs. 20,562,255.42 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 24,250,585.28** | **Bs. 24,250,585.28** |

---

### 4.11 Orden de Producción: `OVO/PRO/SE/00084`
- **Fecha de Cierre de Fabricación:** `2026-08-27 13:51:51`
- **Fecha de Registro Contable / Asiento:** `2026-08-27`
- **Costo Total Fabricación:** **Bs. 40,108,489.52** (Componentes/Materia Prima: `Bs. 38,124,419.45` | Mano de Obra y Operaciones: `Bs. 1,984,070.07`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 4879.0 kg | Bs. 7,680,775.74 (19.15%) | Bs. 28,878,112.46 (72.0%) | **+21,197,336.71 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 9477.0 kg | Bs. 26,475,613.93 (66.01%) | Bs. 7,219,528.11 (18.0%) | **-19,256,085.82 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 850.0 kg | Bs. 5,952,099.85 (14.84%) | Bs. 4,010,848.95 (10.0%) | **-1,941,250.89 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260827-0042`, `4879.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00275-002, OVO/PRO/PTL/00263-001, OVO/PRO/PTL/00263-002, OVO/PRO/PTL/00318-001
  - **Lotes PT Derivados:** `PTLYE-26-0285, SEYERE-20260827-0049, PTLYE-26-0281, PTLYE-26-0290, SEYERE-20260831-0050, PTLYE-26-0280`
  - **Entregas a Clientes:** OVO/OUT/00443 (600.0 kg), OVO/OUT/00474 (1200.0 kg), OVO/OUT/00493 (1800.0 kg) y 3 entrega(s) más
  - **Estado Final:** **4858.69 kg VENDIDO (99.58%)** | **20.31 kg EN INVENTARIO (0.42%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260827-0042`, `9477.0 kg`):
  - **Órdenes de Transformación:** BIODA/MAQ/00010, OVO/PRO/PTL/00276
  - **Lotes PT Derivados:** `SECLRE-20260827-0042, PTDAL-26-251-0018`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **9477.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260827-0042`, `850.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00277, OVO/PRO/PTL/00271, OVO/PRO/PTL/00273, OVO/PRO/PTL/00288
  - **Lotes PT Derivados:** `PTLHE-26-0248, PTLHE-26-0247, SEHERE-20260828-0044, PTLHE-26-0246`
  - **Entregas a Clientes:** OVO/OUT/00454 (200.0 kg), OVO/OUT/00458 (2400.0 kg), OVO/OUT/00481 (2400.0 kg) y 1 entrega(s) más
  - **Estado Final:** **850.0 kg VENDIDO (100.0%)** | **0.0 kg EN INVENTARIO (0.0%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+21,197,336.71 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00084 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-19,256,085.82 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00084 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-1,941,250.89 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00084 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-27` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00084`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00084 - [SE-0001] YEMA (99.58% vendido) | Bs. 21,108,307.90 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00084 - [SE-0002] HUEVO ENTERO (100.0% vendido) |  | Bs. 1,941,250.89 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00084 |  | Bs. 19,167,057.01 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 21,108,307.90** | **Bs. 21,108,307.90** |

---

### 4.12 Orden de Producción: `OVO/PRO/SE/00082`
- **Fecha de Cierre de Fabricación:** `2026-08-26 14:17:46`
- **Fecha de Registro Contable / Asiento:** `2026-08-26`
- **Costo Total Fabricación:** **Bs. 40,796,311.18** (Componentes/Materia Prima: `Bs. 38,628,839.67` | Mano de Obra y Operaciones: `Bs. 2,167,471.51`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 5330.0 kg | Bs. 8,330,606.74 (20.42%) | Bs. 29,373,344.05 (72.0%) | **+21,042,737.31 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 9956.0 kg | Bs. 26,570,637.47 (65.13%) | Bs. 7,343,336.01 (18.0%) | **-19,227,301.46 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 900.0 kg | Bs. 5,895,066.97 (14.45%) | Bs. 4,079,631.12 (10.0%) | **-1,815,435.85 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260826-0041`, `5330.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00256-001, OVO/PRO/PTL/00318-001, OVO/PRO/PTL/00326-001, OVO/PRO/PTL/00275-002, OVO/PRO/PTL/C/00011, OVO/PRO/PTL/00256-002
  - **Lotes PT Derivados:** `PTLYE-26-0279, PTLYE-26-0285, PTLYE-26-0278, PTLYE-26-0292, SEYERE-20260904-0051, SEYERE-20260826-0048, PTLYE-26-0293, PTLYE-26-0290, SEYERE-20260831-0050`
  - **Entregas a Clientes:** OVO/OUT/00442 (2600.0 kg), OVO/OUT/00443 (400.0 kg), OVO/OUT/00473 (1500.0 kg) y 4 entrega(s) más
  - **Estado Final:** **5303.55 kg VENDIDO (99.5%)** | **26.45 kg EN INVENTARIO (0.5%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260826-0041`, `9956.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTD/00009, BIODA/MAQ/00010, OVO/PRO/PTL/00267, OVO/PRO/PTL/00264
  - **Lotes PT Derivados:** `PTDAL-26-251-0018, PTLCL-26-0022, SECLRE-20260826-0041, PTDAL-26-243-0017`
  - **Entregas a Clientes:** OVO/OUT/00437 (16.0 kg), OVO/OUT/00485 (32.0 kg)
  - **Estado Final:** **48.0 kg VENDIDO (0.48%)** | **9908.0 kg EN INVENTARIO (99.52%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260826-0041`, `900.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00268, OVO/PRO/PTL/00274, OVO/PRO/PTL/00271
  - **Lotes PT Derivados:** `SEHERE-20260826-0043, PTLHE-26-0246, PTLHE-26-0245`
  - **Entregas a Clientes:** OVO/OUT/00473 (3168.0 kg), OVO/OUT/00454 (200.0 kg), OVO/OUT/00458 (2400.0 kg) y 1 entrega(s) más
  - **Estado Final:** **870.2 kg VENDIDO (96.69%)** | **29.8 kg EN INVENTARIO (3.31%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+21,042,737.31 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00082 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-19,227,301.46 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00082 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-1,815,435.85 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00082 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-26` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00082`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00082 - [SE-0001] YEMA (99.5% vendido) | Bs. 20,937,523.62 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00082 - [SE-0004] CLARA (0.48% vendido) |  | Bs. 92,291.05 |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00082 - [SE-0002] HUEVO ENTERO (96.69% vendido) |  | Bs. 1,755,344.92 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00082 |  | Bs. 19,089,887.65 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 20,937,523.62** | **Bs. 20,937,523.62** |

---

### 4.13 Orden de Producción: `OVO/PRO/SE/00080`
- **Fecha de Cierre de Fabricación:** `2026-08-25 14:23:49`
- **Fecha de Registro Contable / Asiento:** `2026-08-25`
- **Costo Total Fabricación:** **Bs. 25,283,974.44** (Componentes/Materia Prima: `Bs. 23,961,938.82` | Mano de Obra y Operaciones: `Bs. 1,322,035.62`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 3251.0 kg | Bs. 4,214,838.54 (16.67%) | Bs. 18,204,461.60 (72.0%) | **+13,989,623.06 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 7189.0 kg | Bs. 17,410,544.80 (68.86%) | Bs. 4,551,115.40 (18.0%) | **-12,859,429.40 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 550.0 kg | Bs. 3,658,591.10 (14.47%) | Bs. 2,528,397.44 (10.0%) | **-1,130,193.66 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260825-0040`, `3251.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00318-001, OVO/PRO/PTL/00254, OVO/PRO/PTL/00326-001, OVO/PRO/PTL/00275-002, OVO/PRO/PTL/C/00011, OVO/PRO/PTL/00272-001, OVO/PRO/PTL/00272-002
  - **Lotes PT Derivados:** `PTLYE-26-0285, PTLYE-26-0290, SEYERE-20260825-0046, PTLYE-26-0292, SEYERE-20260825-0047, SEYERE-20260904-0051, PTLYE-26-0282, PTLYE-26-0283, PTLYE-26-0293, PTLYE-26-0277, SEYERE-20260831-0050`
  - **Entregas a Clientes:** OVO/OUT/00441 (800.0 kg), OVO/OUT/00442 (2200.0 kg), OVO/OUT/00473 (1500.0 kg) y 6 entrega(s) más
  - **Estado Final:** **3194.8 kg VENDIDO (98.27%)** | **56.2 kg EN INVENTARIO (1.73%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260825-0040`, `7189.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTD/00009, BIODA/MAQ/00009, OVO/PRO/PTL/00257
  - **Lotes PT Derivados:** `PTDAL-26-243-0016, SECLRE-20260825-0040, PTDAL-26-243-0017`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **7189.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260825-0040`, `550.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00273, OVO/PRO/PTL/00262-002, OVO/PRO/PTL/00288, OVO/PRO/PTL/00271, OVO/PRO/PTL/00258
  - **Lotes PT Derivados:** `PTLHE-26-0244, SEHERE-20260825-0042, PTLHE-26-0247, PTLHE-26-0246, PTLHE-26-0248`
  - **Entregas a Clientes:** OVO/OUT/00443 (3200.0 kg), OVO/OUT/00454 (200.0 kg), OVO/OUT/00458 (2400.0 kg) y 2 entrega(s) más
  - **Estado Final:** **536.0 kg VENDIDO (97.46%)** | **14.0 kg EN INVENTARIO (2.54%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+13,989,623.06 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00080 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-12,859,429.40 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00080 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-1,130,193.66 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00080 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-25` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00080`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00080 - [SE-0001] YEMA (98.27% vendido) | Bs. 13,747,602.58 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00080 - [SE-0002] HUEVO ENTERO (97.46% vendido) |  | Bs. 1,101,486.74 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00080 |  | Bs. 12,646,115.84 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 13,747,602.58** | **Bs. 13,747,602.58** |

---

### 4.14 Orden de Producción: `OVO/PRO/SE/00078`
- **Fecha de Cierre de Fabricación:** `2026-08-24 13:38:22`
- **Fecha de Registro Contable / Asiento:** `2026-08-24`
- **Costo Total Fabricación:** **Bs. 42,483,212.33** (Componentes/Materia Prima: `Bs. 40,333,227.00` | Mano de Obra y Operaciones: `Bs. 2,149,985.34`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 5287.0 kg | Bs. 8,972,454.45 (21.12%) | Bs. 30,587,912.88 (72.0%) | **+21,615,458.44 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 9843.0 kg | Bs. 27,639,577.95 (65.06%) | Bs. 7,646,978.22 (18.0%) | **-19,992,599.72 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 848.0 kg | Bs. 5,871,179.94 (13.82%) | Bs. 4,248,321.23 (10.0%) | **-1,622,858.71 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260824-0039`, `5287.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00318-001, OVO/PRO/PTL/00231, OVO/PRO/PTL/00275-002, OVO/PRO/PTL/00234-001, OVO/PRO/PTL/00234-002, OVO/PRO/PTL/00275-001
  - **Lotes PT Derivados:** `PTLYE-26-0285, SEYERE-20260824-0045, PTLYE-26-0276, PTLYE-26-0275, PTLYE-26-0274, PTLYE-26-0284, PTLYE-26-0290, SEYERE-20260831-0050`
  - **Entregas a Clientes:** OVO/OUT/00421 (928.0 kg), OVO/OUT/00456 (928.0 kg), OVO/OUT/00425 (2600.0 kg) y 5 entrega(s) más
  - **Estado Final:** **5273.71 kg VENDIDO (99.75%)** | **13.29 kg EN INVENTARIO (0.25%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260824-0039`, `9843.0 kg`):
  - **Órdenes de Transformación:** BIODA/MAQ/00009, OVO/PRO/PTL/00246
  - **Lotes PT Derivados:** `PTDAL-26-243-0016, SECLRE-20260824-0039`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **9843.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260824-0039`, `848.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00262-001, OVO/PRO/PTL/00247, OVO/PRO/PTL/00253-002
  - **Lotes PT Derivados:** `SEHERE-20260824-0040, PTLHE-26-0243, PTLHE-26-0241`
  - **Entregas a Clientes:** OVO/OUT/00441 (3400.0 kg), OVO/OUT/00443 (3400.0 kg)
  - **Estado Final:** **848.0 kg VENDIDO (100.0%)** | **0.0 kg EN INVENTARIO (0.0%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+21,615,458.44 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00078 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-19,992,599.72 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00078 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-1,622,858.71 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00078 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-24` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00078`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00078 - [SE-0001] YEMA (99.75% vendido) | Bs. 21,561,419.79 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00078 - [SE-0002] HUEVO ENTERO (100.0% vendido) |  | Bs. 1,622,858.71 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00078 |  | Bs. 19,938,561.08 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 21,561,419.79** | **Bs. 21,561,419.79** |

---

### 4.15 Orden de Producción: `OVO/PRO/SE/00076`
- **Fecha de Cierre de Fabricación:** `2026-08-21 21:43:39`
- **Fecha de Registro Contable / Asiento:** `2026-08-21`
- **Costo Total Fabricación:** **Bs. 36,798,875.26** (Componentes/Materia Prima: `Bs. 34,939,648.29` | Mano de Obra y Operaciones: `Bs. 1,859,226.96`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 4572.0 kg | Bs. 4,835,372.21 (13.14%) | Bs. 26,495,190.19 (72.0%) | **+21,659,817.98 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 8759.0 kg | Bs. 24,176,861.04 (65.7%) | Bs. 6,623,797.55 (18.0%) | **-17,553,063.50 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 1227.0 kg | Bs. 7,786,642.00 (21.16%) | Bs. 3,679,887.53 (10.0%) | **-4,106,754.48 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260821-0038`, `4572.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00275-001, OVO/PRO/PTL/00232-001, OVO/PRO/PTL/00232-002
  - **Lotes PT Derivados:** `PTLYE-26-0273, PTLYE-26-0284, SEYERE-20260821-0043, PTLYE-26-0272`
  - **Entregas a Clientes:** OVO/OUT/00417 (1600.0 kg), OVO/OUT/00408 (1600.0 kg), OVO/OUT/00480 (200.0 kg) y 2 entrega(s) más
  - **Estado Final:** **4568.34 kg VENDIDO (99.92%)** | **3.66 kg EN INVENTARIO (0.08%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260821-0038`, `8759.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00240, BIODA/MAQ/00009, OVO/PRO/PTL/00242
  - **Lotes PT Derivados:** `SECLRE-20260821-0038, PTDAL-26-243-0016, PTLCL-26-0021`
  - **Entregas a Clientes:** OVO/OUT/00523 (400.0 kg)
  - **Estado Final:** **400.0 kg VENDIDO (4.57%)** | **8359.0 kg EN INVENTARIO (95.43%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260821-0038`, `1227.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00243, OVO/PRO/PTL/00253-001, OVO/PRO/PTL/00255, OVO/PRO/PTL/00253-002
  - **Lotes PT Derivados:** `PTLHE-26-0240, PTLHE-26-0242, SEHERE-20260821-0039, PTLHE-26-0241`
  - **Entregas a Clientes:** OVO/OUT/00474 (3400.0 kg), OVO/OUT/00441 (3400.0 kg)
  - **Estado Final:** **1227.0 kg VENDIDO (100.0%)** | **0.0 kg EN INVENTARIO (0.0%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+21,659,817.98 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00076 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-17,553,063.50 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00076 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-4,106,754.48 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00076 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-21` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00076`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00076 - [SE-0001] YEMA (99.92% vendido) | Bs. 21,642,490.12 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00076 - [SE-0004] CLARA (4.57% vendido) |  | Bs. 802,175.00 |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00076 - [SE-0002] HUEVO ENTERO (100.0% vendido) |  | Bs. 4,106,754.48 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00076 |  | Bs. 16,733,560.64 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 21,642,490.12** | **Bs. 21,642,490.12** |

---

### 4.16 Orden de Producción: `OVO/PRO/SE/00074`
- **Fecha de Cierre de Fabricación:** `2026-08-20 20:38:38`
- **Fecha de Registro Contable / Asiento:** `2026-08-20`
- **Costo Total Fabricación:** **Bs. 33,450,106.81** (Componentes/Materia Prima: `Bs. 31,814,133.47` | Mano de Obra y Operaciones: `Bs. 1,635,973.33`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 4023.0 kg | Bs. 4,917,165.70 (14.7%) | Bs. 24,084,076.90 (72.0%) | **+19,166,911.20 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 8394.0 kg | Bs. 22,612,272.20 (67.6%) | Bs. 6,021,019.22 (18.0%) | **-16,591,252.98 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 865.0 kg | Bs. 5,920,668.90 (17.7%) | Bs. 3,345,010.68 (10.0%) | **-2,575,658.22 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260820-0037`, `4023.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00227, OVO/PRO/PTL/00318-001, OVO/PRO/PTL/00232-001, OVO/PRO/PTL/00275-002, OVO/PRO/PTL/00234-002, OVO/PRO/PTL/00275-001
  - **Lotes PT Derivados:** `PTLYE-26-0285, SEYERE-20260824-0045, PTLYE-26-0272, PTLYE-26-0276, SEYERE-20260820-0041, PTLYE-26-0284, PTLYE-26-0290, SEYERE-20260831-0050, SEYERE-20260820-0042, PTLYE-26-0271`
  - **Entregas a Clientes:** OVO/OUT/00493 (3200.0 kg), OVO/OUT/00417 (3000.0 kg), OVO/OUT/00408 (200.0 kg) y 5 entrega(s) más
  - **Estado Final:** **3964.49 kg VENDIDO (98.55%)** | **58.51 kg EN INVENTARIO (1.45%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260820-0037`, `8394.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00235, BIODA/MAQ/00009
  - **Lotes PT Derivados:** `PTDAL-26-243-0016, SECLRE-20260820-0037`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **8394.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260820-0037`, `865.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00327-001, OVO/PRO/PTL/00327-002, OVO/PRO/PTL/00253-001, OVO/PRO/PTL/00233-001, OVO/PRO/PTL/00259, OVO/PRO/PTL/00236, OVO/PRO/PTL/C/00012, OVO/PRO/PTL/00337-002, OVO/PRO/PTL/00233-002
  - **Lotes PT Derivados:** `PTLHE-26-0239, PTLHE-26-0237, PTLHE-26-0238, PTLHE-26-0240, PTLHE-26-0259, SEHERE-20260908-0051, PTLHE-26-0266, SEHERE-20260820-0038, PTLHE-26-0260`
  - **Entregas a Clientes:** OVO/OUT/00434 (3200.0 kg), OVO/OUT/00531 (1400.0 kg), OVO/OUT/00532 (2400.0 kg) y 4 entrega(s) más
  - **Estado Final:** **863.87 kg VENDIDO (99.87%)** | **1.13 kg EN INVENTARIO (0.13%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+19,166,911.20 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00074 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-16,591,252.98 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00074 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-2,575,658.22 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00074 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-20` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00074`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00074 - [SE-0001] YEMA (98.55% vendido) | Bs. 18,888,990.99 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00074 - [SE-0002] HUEVO ENTERO (99.87% vendido) |  | Bs. 2,572,309.87 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00074 |  | Bs. 16,316,681.12 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 18,888,990.99** | **Bs. 18,888,990.99** |

---

### 4.17 Orden de Producción: `OVO/PRO/SE/00071`
- **Fecha de Cierre de Fabricación:** `2026-08-19 12:17:19`
- **Fecha de Registro Contable / Asiento:** `2026-08-19`
- **Costo Total Fabricación:** **Bs. 39,322,068.76** (Componentes/Materia Prima: `Bs. 37,146,464.16` | Mano de Obra y Operaciones: `Bs. 2,175,604.61`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 5350.0 kg | Bs. 5,764,615.28 (14.66%) | Bs. 28,311,889.51 (72.0%) | **+22,547,274.23 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 10401.0 kg | Bs. 25,964,362.01 (66.03%) | Bs. 7,077,972.38 (18.0%) | **-18,886,389.63 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 1280.0 kg | Bs. 7,593,091.48 (19.31%) | Bs. 3,932,206.88 (10.0%) | **-3,660,884.60 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260819-0036`, `5350.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00214-002, OVO/PRO/PTL/00318-001, OVO/PRO/PTL/00231, OVO/PRO/PTL/00202, OVO/PRO/PTL/00275-002, OVO/PRO/PTL/00272-001, OVO/PRO/PTL/00214-001, OVO/PRO/PTL/00234-002, OVO/PRO/PTL/00275-001
  - **Lotes PT Derivados:** `PTLYE-26-0285, SEYERE-20260831-0050, SEYERE-20260824-0045, PTLYE-26-0276, SEYERE-20260818-0039, PTLYE-26-0274, PTLYE-26-0269, PTLYE-26-0282, SEYERE-20260819-0040, PTLYE-26-0284, PTLYE-26-0290, PTLYE-26-0270, PTLYE-26-0268`
  - **Entregas a Clientes:** OVO/OUT/00390 (976.0 kg), OVO/OUT/00449 (512.0 kg), OVO/OUT/00526 (976.0 kg) y 12 entrega(s) más
  - **Estado Final:** **5296.91 kg VENDIDO (99.01%)** | **53.09 kg EN INVENTARIO (0.99%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260819-0036`, `10401.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00219, BIODA/MAQ/00009, OVO/PRO/PTL/00228
  - **Lotes PT Derivados:** `PTDAL-26-243-0016, PTLCL-26-0019, SECLRE-20260819-0036`
  - **Entregas a Clientes:** OVO/OUT/00523 (32.0 kg)
  - **Estado Final:** **32.0 kg VENDIDO (0.31%)** | **10369.0 kg EN INVENTARIO (99.69%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260819-0036`, `1280.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00327-001, OVO/PRO/PTL/00327-002, OVO/PRO/PTL/00230, OVO/PRO/PTL/00233-001, OVO/PRO/PTL/00337-002, OVO/PRO/PTL/C/00012, OVO/PRO/PTL/00229, OVO/PRO/PTL/00220
  - **Lotes PT Derivados:** `PTLHE-26-0236, PTLHE-26-0237, SEHERE-20260819-0037, PTLHE-26-0259, PTLHE-26-0235, SEHERE-20260908-0051, PTLHE-26-0266, PTLHE-26-0260`
  - **Entregas a Clientes:** OVO/OUT/00408 (1400.0 kg), OVO/OUT/00412 (1200.0 kg), OVO/OUT/00425 (200.0 kg) y 5 entrega(s) más
  - **Estado Final:** **1163.38 kg VENDIDO (90.89%)** | **116.62 kg EN INVENTARIO (9.11%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+22,547,274.23 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00071 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-18,886,389.63 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00071 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-3,660,884.60 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00071 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-19` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00071`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00071 - [SE-0001] YEMA (99.01% vendido) | Bs. 22,324,056.21 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00071 - [SE-0004] CLARA (0.31% vendido) |  | Bs. 58,547.81 |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00071 - [SE-0002] HUEVO ENTERO (90.89% vendido) |  | Bs. 3,327,378.01 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00071 |  | Bs. 18,938,130.39 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 22,324,056.21** | **Bs. 22,324,056.21** |

---

### 4.18 Orden de Producción: `OVO/PRO/SE/00069`
- **Fecha de Cierre de Fabricación:** `2026-08-18 12:20:37`
- **Fecha de Registro Contable / Asiento:** `2026-08-18`
- **Costo Total Fabricación:** **Bs. 24,141,502.60** (Componentes/Materia Prima: `Bs. 22,913,404.30` | Mano de Obra y Operaciones: `Bs. 1,228,098.30`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 3020.0 kg | Bs. 1,863,724.00 (7.72%) | Bs. 17,381,881.87 (72.0%) | **+15,518,157.87 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 6720.0 kg | Bs. 16,655,222.64 (68.99%) | Bs. 4,345,470.47 (18.0%) | **-12,309,752.18 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 917.0 kg | Bs. 5,622,555.96 (23.29%) | Bs. 2,414,150.26 (10.0%) | **-3,208,405.70 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260818-0035`, `3020.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00231, OVO/PRO/PTL/00202, OVO/PRO/PTL/00212
  - **Lotes PT Derivados:** `PTLYE-26-0274, PTLYE-26-0267, PTLYE-26-0268, SEYERE-20260818-0039`
  - **Entregas a Clientes:** OVO/OUT/00403 (32.0 kg), OVO/OUT/00465 (16.0 kg), OVO/OUT/00463 (64.0 kg) y 7 entrega(s) más
  - **Estado Final:** **2275.41 kg VENDIDO (75.34%)** | **744.59 kg EN INVENTARIO (24.66%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260818-0035`, `6720.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00216, OVO/PRO/PTL/00215, BIODA/MAQ/00009, OVO/PRO/PTL/00210
  - **Lotes PT Derivados:** `PTDAL-26-243-0016, PTLCL-26-0018, SECLRE-20260818-0035, PTLCL-26-0017`
  - **Entregas a Clientes:** OVO/OUT/00374 (800.0 kg), OVO/OUT/00378 (32.0 kg), OVO/OUT/00414 (352.0 kg) y 3 entrega(s) más
  - **Estado Final:** **2592.0 kg VENDIDO (38.57%)** | **4128.0 kg EN INVENTARIO (61.43%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260818-0035`, `917.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00327-001, OVO/PRO/PTL/00327-002, OVO/PRO/PTL/C/00012, OVO/PRO/PTL/00211, OVO/PRO/PTL/00229, OVO/PRO/PTL/00337-002
  - **Lotes PT Derivados:** `PTLHE-26-0236, PTLHE-26-0259, SEHERE-20260818-0036, SEHERE-20260908-0051, PTLHE-26-0266, PTLHE-26-0260`
  - **Entregas a Clientes:** OVO/OUT/00408 (1400.0 kg), OVO/OUT/00412 (1200.0 kg), OVO/OUT/00425 (200.0 kg) y 2 entrega(s) más
  - **Estado Final:** **863.17 kg VENDIDO (94.13%)** | **53.83 kg EN INVENTARIO (5.87%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+15,518,157.87 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00069 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-12,309,752.18 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00069 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-3,208,405.70 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00069 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-18` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00069`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00069 - [SE-0001] YEMA (75.34% vendido) | Bs. 11,691,380.14 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00069 - [SE-0004] CLARA (38.57% vendido) |  | Bs. 4,747,871.41 |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00069 - [SE-0002] HUEVO ENTERO (94.13% vendido) |  | Bs. 3,020,072.28 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00069 |  | Bs. 3,923,436.45 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 11,691,380.14** | **Bs. 11,691,380.14** |

---

### 4.19 Orden de Producción: `OVO/PRO/SE/00068`
- **Fecha de Cierre de Fabricación:** `2026-08-17 20:16:55`
- **Fecha de Registro Contable / Asiento:** `2026-08-17`
- **Costo Total Fabricación:** **Bs. 37,291,997.54** (Componentes/Materia Prima: `Bs. 35,166,818.16` | Mano de Obra y Operaciones: `Bs. 2,125,179.38`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 5226.0 kg | Bs. 4,538,436.10 (12.17%) | Bs. 26,850,238.23 (72.0%) | **+22,311,802.13 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 10213.0 kg | Bs. 24,668,656.37 (66.15%) | Bs. 6,712,559.56 (18.0%) | **-17,956,096.82 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 1447.0 kg | Bs. 8,084,905.07 (21.68%) | Bs. 3,729,199.75 (10.0%) | **-4,355,705.31 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260817-0034`, `5226.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00227, OVO/PRO/PTL/00318-001, OVO/PRO/PTL/00231, OVO/PRO/PTL/00232-001, OVO/PRO/PTL/00178-002, OVO/PRO/PTL/00275-002, OVO/PRO/PTL/00214-001, OVO/PRO/PTL/00234-002, OVO/PRO/PTL/00275-001, OVO/PRO/PTL/00178-001
  - **Lotes PT Derivados:** `PTLYE-26-0285, SEYERE-20260817-0038, SEYERE-20260824-0045, PTLYE-26-0272, PTLYE-26-0276, PTLYE-26-0265, PTLYE-26-0274, PTLYE-26-0269, PTLYE-26-0266, SEYERE-20260820-0041, PTLYE-26-0284, PTLYE-26-0290, SEYERE-20260831-0050, SEYERE-20260820-0042, PTLYE-26-0271`
  - **Entregas a Clientes:** OVO/OUT/00411 (2200.0 kg), OVO/OUT/00416 (1200.0 kg), OVO/OUT/00425 (2600.0 kg) y 9 entrega(s) más
  - **Estado Final:** **5220.52 kg VENDIDO (99.9%)** | **5.48 kg EN INVENTARIO (0.1%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260817-0034`, `10213.0 kg`):
  - **Órdenes de Transformación:** BIODA/MAQ/00009, OVO/PRO/PTL/00206
  - **Lotes PT Derivados:** `PTDAL-26-243-0016, SECLRE-20260817-0034`
  - **Estado Final:** **0.0 kg VENDIDO (0.00%)** | **10213.0 kg EN INVENTARIO (100.00%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260817-0034`, `1447.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00327-001, OVO/PRO/PTL/00225-002, OVO/PRO/PTL/00327-002, OVO/PRO/PTL/00207, OVO/PRO/PTL/00225-001, OVO/PRO/PTL/C/00012, OVO/PRO/PTL/00229, OVO/PRO/PTL/00337-002, OVO/PRO/PTL/00226, OVO/PRO/PTL/00213
  - **Lotes PT Derivados:** `PTLHE-26-0236, PTLHE-26-0231, PTLHE-26-0232, PTLHE-26-0259, SEHERE-20260817-0035, PTLHE-26-0260, SEHERE-20260908-0051, PTLHE-26-0266, PTLHE-26-0234, PTLHE-26-0233`
  - **Entregas a Clientes:** OVO/OUT/00400 (1800.0 kg), OVO/OUT/00401 (2200.0 kg), OVO/OUT/00408 (1400.0 kg) y 5 entrega(s) más
  - **Estado Final:** **1447.0 kg VENDIDO (100.0%)** | **0.0 kg EN INVENTARIO (0.0%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+22,311,802.13 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00068 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-17,956,096.82 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00068 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-4,355,705.31 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00068 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-08-17` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00068`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00068 - [SE-0001] YEMA (99.9% vendido) | Bs. 22,289,490.33 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00068 - [SE-0002] HUEVO ENTERO (100.0% vendido) |  | Bs. 4,355,705.31 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00068 |  | Bs. 17,933,785.02 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 22,289,490.33** | **Bs. 22,289,490.33** |

---

### 4.20 Orden de Producción: `OVO/PRO/SE/00040`
- **Fecha de Cierre de Fabricación:** `2026-07-24 20:28:51`
- **Fecha de Registro Contable / Asiento:** `2026-07-24`
- **Costo Total Fabricación:** **Bs. 24,997,295.99** (Componentes/Materia Prima: `Bs. 23,197,034.01` | Mano de Obra y Operaciones: `Bs. 1,800,261.98`)

#### A. Comparativa de Reparto de Costos (Inicial vs. 72/18/10)
| Producto Semielaborado | Cantidad Obtenida | Costo Inicial Registrado (% Inicial) | Costo Real Teórico (72/18/10) | Ajuste Patrimonial (Delta) |
| :--- | :---: | :---: | :---: | :---: |
| **[SE-0001] YEMA FILTRADA** | 4427.0 kg | Bs. 5,046,954.06 (20.19%) | Bs. 17,998,053.11 (72.0%) | **+12,951,099.05 Bs.** ⚠️ |
| **[SE-0004] CLARA FILTRADA** | 8512.0 kg | Bs. 16,445,721.03 (65.79%) | Bs. 4,499,513.28 (18.0%) | **-11,946,207.75 Bs.** ⚠️ |
| **[SE-0002] HUEVO ENTERO (P1)** | 722.0 kg | Bs. 3,504,620.90 (14.02%) | Bs. 2,499,729.60 (10.0%) | **-1,004,891.30 Bs.** ⚠️ |

#### B. Trazabilidad Multinivel de Lotes y Destino Comercial
- **`[SE-0001] YEMA FILTRADA`** (Lote: `SEYEP1-20260724-0020`, `4427.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00256-001, OVO/PRO/PTL/00090-001, OVO/PRO/PTL/00275-001, OVO/PRO/PTL/00318-001, OVO/PRO/PTL/00263-001, OVO/PRO/PTL/00254, OVO/PRO/PTL/00113-001, OVO/PRO/PTL/00326-001, OVO/PRO/PTL/00275-002, OVO/PRO/PTL/C/00009, OVO/PRO/PTL/C/00011, OVO/PRO/PTL/00108-001, OVO/PRO/PTL/00272-001, OVO/PRO/PTL/00090-002, OVO/PRO/PTL/00272-002
  - **Lotes PT Derivados:** `PTLYE-26-0244, PTLYE-26-0236, SEYERE-20260825-0047, SEYERE-20260904-0051, PTLYE-26-0282, PTLYE-26-0293, PTLYE-26-0284, PTLYE-26-0237, PTLYE-26-0285, SEYERE-20260724-0023, SEYERE-20260824-0044, PTLYE-26-0242, PTLYE-26-0283, PTLYE-26-0277, PTLYE-26-0280, PTLYE-26-0278, PTLYE-26-0292, PTLYE-26-0290, SEYERE-20260831-0050, SEYERE-20260825-0046`
  - **Entregas a Clientes:** OVO/OUT/00265 (2000.0 kg), OVO/OUT/00370 (3200.0 kg), OVO/OUT/00317 (2800.0 kg) y 13 entrega(s) más
  - **Estado Final:** **4426.98 kg VENDIDO (100.0%)** | **0.02 kg EN INVENTARIO (0.0%)**
- **`[SE-0004] CLARA FILTRADA`** (Lote: `SECLP1-20260724-0020`, `8512.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00101, OVO/PRO/PTD/00005, BIODA/MAQ/00006, OVO/PRO/PTL/00098
  - **Lotes PT Derivados:** `PTDAL-26-212-0010, PTLCL-26-0008, SECLRE-20260724-0020, PTDAL-26-212-0009`
  - **Entregas a Clientes:** OVO/OUT/00214 (624.0 kg)
  - **Estado Final:** **624.0 kg VENDIDO (7.33%)** | **7888.0 kg EN INVENTARIO (92.67%)**
- **`[SE-0002] HUEVO ENTERO PROCESO 1`** (Lote: `SEHEP1-20260724-0020`, `722.0 kg`):
  - **Órdenes de Transformación:** OVO/PRO/PTL/00095, OVO/PRO/PTL/00172, OVO/PRO/PTL/00105-002, OVO/PRO/PTL/C/00006, OVO/PRO/PTL/00177-001, OVO/PRO/PTL/00102, OVO/PRO/PTL/00164-002, OVO/PRO/PTL/00105-001
  - **Lotes PT Derivados:** `PTLHE-26-0226, PTLHE-26-0203, PTLHE-26-0224, PTLHE-26-0223, PTLHE-26-0206, SEHERE-20260724-0020, PTLHE-26-0205, SEHERE-20260811-0030`
  - **Entregas a Clientes:** OVO/OUT/00245 (1200.0 kg), OVO/OUT/00275 (2200.0 kg), OVO/OUT/00372 (1800.0 kg) y 3 entrega(s) más
  - **Estado Final:** **619.0 kg VENDIDO (85.73%)** | **103.0 kg EN INVENTARIO (14.27%)**

#### C. Capas de Valoración de Inventario (`stock.valuation.layer`) a Generar
| Producto | Cantidad | Impacto en Valoración | Descripción de la Capa en Odoo |
| :--- | :---: | :---: | :--- |
| **[SE-0001] YEMA** | 0.0 kg | **+12,951,099.05 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00040 (72% esperado) - [SE-0001] YEMA |
| **[SE-0004] CLARA** | 0.0 kg | **-11,946,207.75 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00040 (18% esperado) - [SE-0004] CLARA |
| **[SE-0002] HUEVO** | 0.0 kg | **-1,004,891.30 Bs.** | Revalorización ajuste costo MO OVO/PRO/SE/00040 (10% esperado) - [SE-0002] HUEVO ENTERO |

#### D. Asiento Contable Propuesto (`account.move`)
**Fecha del Asiento:** `2026-07-24` | **Diario:** `V.INV (Inventory Valuation)` | **Referencia:** `Ajuste Reparto Costo MO OVO/PRO/SE/00040`

| Código Cuenta | Nombre Cuenta | Concepto / Detalle de la Línea | Débito (Bs.) | Crédito (Bs.) |
| :---: | :--- | :--- | :---: | :---: |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00040 - [SE-0001] YEMA (100.0% vendido) | Bs. 12,951,099.05 |  |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00040 - [SE-0004] CLARA (7.33% vendido) |  | Bs. 875,657.03 |
| `5110000001` | **COSTO DE BIENES VENDIDOS** | Ajuste Costo Ventas MO OVO/PRO/SE/00040 - [SE-0002] HUEVO ENTERO (85.73% vendido) |  | Bs. 861,493.31 |
| `1420000002` | **PRODUCTOS SEMIELABORADOS** | Reclasificación Inventario por Ventas MO OVO/PRO/SE/00040 |  | Bs. 11,213,948.71 |
| **TOTALES** | | **Cuadre Exacto del Asiento** | **Bs. 12,951,099.05** | **Bs. 12,951,099.05** |
