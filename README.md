# Business Intelligence & Performance Analytics Portfolio

Este repositorio reúne una solución analítica de **Business Intelligence** orientada al control, monitoreo y optimización de eficiencia en entornos de producción e industriales[cite: 1, 2, 3, 4, 5]. Contiene tanto la documentación visual de los dashboards como las arquitecturas de modelado y fórmulas avanzadas en **DAX** (anonimizadas para preservar la confidencialidad)[cite: 1, 2, 3, 4, 5].

---

## Vistas del Dashboard

El análisis se compone de 4 informes clave interactivos (disponibles en formato PDF en la raíz del repositorio)[cite: 2, 3, 4, 5]:

1. **`DB1.pdf` — Eficiencia Global y Rendimiento (OEE)**: Monitoreo semanal del índice global de eficiencia, desglose de factores principales (Calidad, Disponibilidad y Rendimiento) y seguimiento de horas operativas[cite: 2].
2. **`DB2.pdf` — Análisis de Inactividad y Paros**: Matriz detallada de causas de inactividad, eventos de parada por código/categoría y evolución temporal de tiempos no operativos[cite: 3].
3. **`DB3.pdf` — Tendencias y Distribución Temporal**: Evolución semanal de métricas de producción, seguimiento histórico e identificación de patrones de actividad[cite: 4].
4. **`DB4.pdf` — Control Diario y Comparativa por Ubicación**: Seguimiento diario con bandas de tolerancia, tendencias por zona/grupo y variabilidad entre turnos[cite: 5].

---

## Arquitectura DAX (Código Anonimizado)

A continuación se muestra el bloque de código consolidado utilizado para el cálculo de tablas agregadas, matrices temporales, métricas de inactividad e índices de rendimiento global[cite: 1]:

```dax
// 1. Agregación de Valores por Identificador
TablaResumen_Costes = 
SUMMARIZE(
    'TablaOrigen',
    'TablaOrigen'[ID_Registro],
    "ImporteTotal",
        SUM('TablaOrigen'[ValorUnitario])
)

// 2. Extracción de Entidades Únicas Filtradas
TablaDimension_Estados = 
FILTER(
    DISTINCT(
        SELECTCOLUMNS(
            TablaEventos,
            "ID_Proceso", TablaEventos[ID_Proceso],
            "EstadoDescripcion", TablaEventos[EstadoDescripcion]
        )
    ),
    [EstadoDescripcion] <> "EXCLUIDO"
)

// 3. Matriz de Intervalos Temporales
TablaIntervalos = 
ADDCOLUMNS(
    SUMMARIZE(
        FILTER(
            TablaEventos, 
            TablaEventos[SubcategoriaEstado] <> "OMITIR"
        ), 
        TablaEventos[ID_Elemento], 
        TablaEventos[PeriodoSemana], 
        TablaEventos[CategoriaEstado],
        TablaEventos[SubcategoriaEstado]
    ),
    "HoraInicio", FORMAT(CALCULATE(MIN(TablaEventos[FechaHora])), "hh:mm"),
    "HoraFin", FORMAT(CALCULATE(MAX(TablaEventos[FechaHora])), "hh:mm")
)

// 4. Acumulado Diario de Inactividad por Segmento
TablaResumenDiario = 
SUMMARIZE(
    FILTER(
        TablaEventos,
        LEFT(TablaEventos[CategoriaEstado], 6) <> "ACTIVO" && TablaEventos[Estatus] <> "NO_ELEGIBLE"
    ),
    TablaEventos[ID_Elemento],
    TablaEventos[GrupoJornada],
    TablaEventos[CategoriaEstado],
    TablaEventos[CodigoEstado],
    TablaEventos[DiaNumero],
    "TotalDuracion", SUMX(
        FILTER(
            TablaEventos,
            LEFT(TablaEventos[CategoriaEstado], 6) <> "ACTIVO"
        ),
        TablaEventos[Duracion]
    )
)

// 5. Acumulado Semanal con Exclusión de Ventana Temporal
TablaResumenSemanal = 
SUMMARIZE(
    FILTER(
        TablaEventos,
        -- 1. Filtro: Criterio de estado principal
        LEFT(TablaEventos[CategoriaEstado], 6) <> "ACTIVO" &&
        
        -- 2. Filtro: Exclusión por ventana horaria
        VAR DiaReg = WEEKDAY(TablaEventos[FechaHora], 2)
        VAR HoraReg = HOUR(TablaEventos[FechaHora])
        VAR EsFueraDeHorario = 
            (DiaReg = 5 && HoraReg >= 22) || 
            (DiaReg = 6) ||                  
            (DiaReg = 7 && HoraReg < 22)     
        RETURN NOT(EsFueraDeHorario)
    ),
    TablaEventos[ID_Ubicacion],
    TablaEventos[CategoriaEstado],
    TablaEventos[CodigoEstado],
    TablaEventos[PeriodoSemana],
    "TotalDuracion", SUMX(
        FILTER(
            TablaEventos,
            LEFT(TablaEventos[CategoriaEstado], 6) <> "ACTIVO" &&
            VAR DiaReg = WEEKDAY(TablaEventos[FechaHora], 2)
            VAR HoraReg = HOUR(TablaEventos[FechaHora])
            VAR EsFueraDeHorario = 
                (DiaReg = 5 && HoraReg >= 22) || 
                (DiaReg = 6) ||               
                (DiaReg = 7 && HoraReg < 22)
            RETURN NOT(EsFueraDeHorario)
        ),
        TablaEventos[Duracion]
    )
)

// 6. Cálculo de Ratios Específicos e Índice Global por Elemento
TablaResumenKpiCalculado = 

VAR DatosBase = 

    SUMMARIZE(

        TablaEventos,

        TablaEventos[ID_Elemento],

        TablaEventos[Anio],

        TablaEventos[PeriodoSemana],

        TablaEventos[Anio_Semana],

        TablaEventos[ID_Ubicacion],

        TablaEventos[ID_Grupo],

        "FechaReferencia", MAX(TablaEventos[FechaHora]),

        "TotalTiempoActivo", SUMX(

            FILTER(TablaEventos, ISNUMBER(TablaEventos[TiempoActivo])), 

            TablaEventos[TiempoActivo]

        ),

        "TotalTiempoInactivo", SUMX(

            FILTER(

                TablaEventos, 

                ISNUMBER(TablaEventos[TiempoInactivo]) && 

                TablaEventos[Estatus] <> "NO_ELEGIBLE" &&

                VAR DiaSemana = WEEKDAY(TablaEventos[FechaHora], 2) 

                VAR Hora = HOUR(TablaEventos[FechaHora])

                VAR EsFueraDeHorario = 

                    (DiaSemana = 5 && Hora >= 22) || 

                    (DiaSemana = 6) ||               

                    (DiaSemana = 7 && Hora < 22)     

                RETURN

                (NOT TablaEventos[CategoriaEstado] IN {
                    "CATEGORIA_EXCLUIDA_01"
                })

                && 

                NOT (

                    TablaEventos[CategoriaEstado] = "CATEGORIA_EXCLUIDA_02" && 

                    EsFueraDeHorario

                )

            ), 

            TablaEventos[TiempoInactivo]

        ),

        "TotalConformes", SUMX(

            FILTER(TablaEventos, ISNUMBER(TablaEventos[CantidadConforme]) && TablaEventos[TiempoActivo] > 0), 

            TablaEventos[CantidadConforme]

        ),

        "TotalNoConformes", SUMX(

            FILTER(

                TablaEventos, 

                ISNUMBER(TablaEventos[CantidadNoConforme]) && 

                TablaEventos[CantidadNoConforme] <= 3000 && 

                TablaEventos[CategoriaEstado] <> "CATEGORIA_EXCLUIDA_03" && 

                TablaEventos[CategoriaEstado] <> "CATEGORIA_EXCLUIDA_04"

            ), 

            TablaEventos[CantidadNoConforme]

        ),

        "DesviacionTotal", SUMX(

            FILTER(TablaEventos, ISNUMBER(TablaEventos[DesviacionBase]) && TablaEventos[TiempoActivo] > 0 && TablaEventos[DesviacionBase] < 100), 

            TablaEventos[DesviacionBase]

        )

    )

VAR DatosKPIs = 

    ADDCOLUMNS(

        DatosBase,

        "RatioFactorC", 

            DIVIDE([TotalConformes], [TotalConformes] + [TotalNoConformes], 0) * 100,

        "RatioFactorA", 

            DIVIDE([TotalTiempoActivo], [TotalTiempoActivo] + [TotalTiempoInactivo], 0) * 100,

        "RatioFactorB", 

            DIVIDE([DesviacionTotal], [TotalTiempoActivo], 0) * 100

    )

RETURN

    ADDCOLUMNS(

        DatosKPIs,

        "IndiceGlobal", 

        ([RatioFactorA] * [RatioFactorB] * [RatioFactorC]) / 10000

    )

// 7. Consolidador por Ubicación y Periodo
TablaResumenUbicacion = 
SUMMARIZE(
    TablaEventos,
    TablaEventos[ID_Ubicacion],
    TablaEventos[Anio],
    TablaEventos[PeriodoSemana],
    TablaEventos[Anio_Semana],
    "FechaReferencia", MAX(TablaEventos[FechaHora]),

    "TotalTiempoActivo", SUMX(
        FILTER(TablaEventos, ISNUMBER(TablaEventos[TiempoActivo])), 
        TablaEventos[TiempoActivo]
    ),
    
    "TotalTiempoInactivo", SUMX(
        FILTER(TablaEventos, ISNUMBER(TablaEventos[TiempoInactivo]) && TablaEventos[Estatus] <> "NO_ELEGIBLE"), 
        TablaEventos[TiempoInactivo]
    ),
    
    "TotalConformes", SUMX(
        FILTER(TablaEventos, ISNUMBER(TablaEventos[CantidadConforme]) && TablaEventos[TiempoActivo] > 0), 
        TablaEventos[CantidadConforme]
    ),
    
    "TotalNoConformes", SUMX(
        FILTER(
            TablaEventos, 
            ISNUMBER(TablaEventos[CantidadNoConforme]) && 
            TablaEventos[CantidadNoConforme] <= 3000 && 
            TablaEventos[CategoriaEstado] <> "CATEGORIA_EXCLUIDA_03" && 
            TablaEventos[CategoriaEstado] <> "CATEGORIA_EXCLUIDA_04"
        ), 
        TablaEventos[CantidadNoConforme]
    ),
    
    "DesviacionTotal", SUMX(
        FILTER(TablaEventos, ISNUMBER(TablaEventos[DesviacionBase]) && TablaEventos[TiempoActivo] > 0 && TablaEventos[DesviacionBase] < 100), 
        TablaEventos[DesviacionBase]
    )
)

// 8. Resumen por Ubicación, Día y Jornada con Exclusiones Múltiples
TablaResumenUbicacion_Jornada = 
VAR DatosBase = 
    SUMMARIZE(
        FILTER(
            TablaEventos, 
            FORMAT(TablaEventos[Jornada], "General Number") <> "0" && 
            FORMAT(TablaEventos[ID_Ubicacion], "General Number") <> "4"
        ),
        TablaEventos[ID_Ubicacion],
        TablaEventos[DiaJornada],
        TablaEventos[Jornada],
        
        "TotalTiempoActivo", SUMX(
            FILTER(TablaEventos, ISNUMBER(TablaEventos[TiempoActivo])), 
            TablaEventos[TiempoActivo]
        ),

        "TotalTiempoInactivo", SUMX(
            FILTER(
                TablaEventos, 
                ISNUMBER(TablaEventos[TiempoInactivo]) && 
                TablaEventos[Estatus] <> "NO_ELEGIBLE" &&
                
                VAR DiaSemana = WEEKDAY(TablaEventos[FechaHora], 2)
                VAR Hora = HOUR(TablaEventos[FechaHora])
                VAR EsFueraDeHorario = 
                    (DiaSemana = 5 && Hora >= 22) || 
                    (DiaSemana = 6) || 
                    (DiaSemana = 7 && Hora < 22)
                
                RETURN
                (NOT TablaEventos[CategoriaEstado] IN {
                    "CATEGORIA_EXCLUIDA_01",
                    "CATEGORIA_EXCLUIDA_05",
                    "CATEGORIA_EXCLUIDA_06"
                })
                && 
                NOT (
                    TablaEventos[CategoriaEstado] = "CATEGORIA_EXCLUIDA_02" && 
                    EsFueraDeHorario
                )
            ), 
            TablaEventos[TiempoInactivo]
        ),

        "TotalConformes", SUMX(
            FILTER(TablaEventos, ISNUMBER(TablaEventos[CantidadConforme]) && TablaEventos[TiempoActivo] > 0), 
            TablaEventos[CantidadConforme]
        ),

        "TotalNoConformes", SUMX(
            FILTER(
                TablaEventos, 
                ISNUMBER(TablaEventos[CantidadNoConforme]) && 
                TablaEventos[CantidadNoConforme] <= 3000
            ), 
            TablaEventos[CantidadNoConforme]
        ),

        "DesviacionTotal", SUMX(
            FILTER(TablaEventos, ISNUMBER(TablaEventos[DesviacionBase]) && TablaEventos[TiempoActivo] > 0 && TablaEventos[DesviacionBase] < 100), 
            TablaEventos[DesviacionBase]
        )
    )

VAR DatosKPIs = 
    ADDCOLUMNS(
        DatosBase,
        "RatioFactorC", DIVIDE([TotalConformes], [TotalConformes] + [TotalNoConformes], 0) * 100,
        "RatioFactorA", DIVIDE([TotalTiempoActivo], [TotalTiempoActivo] + [TotalTiempoInactivo], 0) * 100,
        "RatioFactorB", DIVIDE([DesviacionTotal], [TotalTiempoActivo], 0) * 100
    )

RETURN
    ADDCOLUMNS(
        DatosKPIs,
        "IndiceGlobal", ([RatioFactorA] * [RatioFactorB] * [RatioFactorC]) / 10000
    )

// 9. Tabla Resumen Consolidada con Promedio Ponderado de Índices
TablaResumenConsolidado = 
VAR ActividadFiltrada = 
    FILTER (
        TablaEventos,
        TablaEventos[ID_Ubicacion] < 4
            && NOT ( ISBLANK ( RELATED ( DimensionEntidad[ID_Grupo] ) ) )
            && (
                (WEEKDAY( TablaEventos[FechaHora], 2 ) = 7 && HOUR( TablaEventos[FechaHora] ) >= 22)
                || (WEEKDAY( TablaEventos[FechaHora], 2 ) >= 1 && WEEKDAY( TablaEventos[FechaHora], 2 ) <= 5)
                || (WEEKDAY( TablaEventos[FechaHora], 2 ) = 6 && HOUR( TablaEventos[FechaHora] ) < 22)
            )
    )

RETURN
SUMMARIZE (
    ActividadFiltrada,
    DimensionEntidad[ID_Elemento],
    "PromedioIndiceGlobal", 
        CALCULATE (
            AVERAGEX (
                FILTER(
                    TablaResumenUbicacion, 
                    (TablaResumenUbicacion[RatioFactorA] * TablaResumenUbicacion[RatioFactorB] * TablaResumenUbicacion[RatioFactorC]) > 0
                ),
                DIVIDE(TablaResumenUbicacion[RatioFactorA] * TablaResumenUbicacion[RatioFactorB] * TablaResumenUbicacion[RatioFactorC], 10000)
            )
        ),
    "PromedioUtilizacion",
        VAR HorasActivas = SUM ( TablaEventos[TiempoActivo] )
        VAR NumeroSemanas = DISTINCTCOUNT ( TablaEventos[Anio_Semana] )
        RETURN 
        DIVIDE ( HorasActivas, NumeroSemanas * 120 )
)
```
