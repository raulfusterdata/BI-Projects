# Business Intelligence & Performance Analytics Portfolio

Este repositorio reúne una solución analítica de **Business Intelligence** orientada al control, monitoreo y optimización de eficiencia, presencialidad y productividad en entornos operativos e industriales. Contiene tanto la documentación visual de los dashboards como las arquitecturas de modelado y fórmulas avanzadas en **DAX** (completamente anonimizadas para preservar la confidencialidad).

---

## 📊 Vistas del Dashboard

El análisis se compone de informes clave interactivos (disponibles en formato PDF en el repositorio):

1. **`DB1.pdf` — Eficiencia Global y Rendimiento (OEE)**: Monitoreo semanal del índice global de eficiencia, desglose de factores principales (Calidad, Disponibilidad y Rendimiento) y seguimiento de horas operativas.
2. **`DB2.pdf` — Análisis de Inactividad y Paros**: Matriz detallada de causas de inactividad, eventos de parada por código/categoría y evolución temporal de tiempos no operativos.
3. **`DB3.pdf` — Tendencias y Distribución Temporal**: Evolución semanal de métricas de producción, seguimiento histórico e identificación de patrones de actividad.
4. **`DB4.pdf` — Control Diario y Comparativa por Ubicación**: Seguimiento diario con bandas de tolerancia, tendencias por zona/grupo y variabilidad entre turnos.
5. **`Presencia_Productividad-5.pdf` — Ratio de Tiempo Efectivo y Aprovechamiento**: Análisis de tendencias del ratio de aprovechamiento del tiempo efectivo frente al tiempo presencial registrado.
6. **`Presencia_Productividad-7.pdf` — Presencialidad vs. Productividad**: Matriz comparativa entre horas presenciales totales, horas productivas reales y cálculo de desviaciones por operador y ubicación.

---

## 🛠️ Arquitectura DAX (Código Anonimizado)

A continuación se muestra el bloque de código consolidado utilizado para la integración de tiempos de máquina, cálculo de tareas manuales, matrices de presencia por intervalos y tablas consolidadas de productividad:

```dax
// 1. Tabla Consolidada de Eficiencia y Productividad
TablaConsolidada_Productividad = 
VAR DatosSegmentoA = 
    SELECTCOLUMNS(
        'TablaMetricas_Operador',
        "ID_Join", [ID_Operador] & "",
        "Fecha_Join", [Fecha], 
        "Elemento_Join", [ID_Elemento],
        "Ubicacion_Join", [ID_Ubicacion],
        "Prod_Val_Maq", [ValorProductividad],
        "Prod_Porc_Maq", [% Productividad],
        "Horas_Maq", [TiempoBase_H],
        "Segundos_Man", 0.0,
        "Prod_Porc_Man", 0.0,
        "Saturacion_Join", [NivelSaturacion],
        "Actividad_Join", BLANK()
    )

VAR DatosSegmentoB = 
    SELECTCOLUMNS(
        'TablaCalculos_Manuales',
        "ID_Join", [ID_Operador] & "",
        "Fecha_Join", [Fecha],
        "Elemento_Join", "SEGMENTO_MANUAL",
        "Ubicacion_Join", "SEGMENTO_MANUAL",
        "Prod_Val_Maq", 0.0,
        "Prod_Porc_Maq", 0.0,
        "Horas_Maq", 0.0,
        "Segundos_Man", [TiempoReal_Seg],
        "Prod_Porc_Man", [PorcentajeEficiencia],
        "Saturacion_Join", [NivelSaturacion],
        "Actividad_Join", [TipoActividad]
    )

VAR TablaUnion = UNION(DatosSegmentoA, DatosSegmentoB)

VAR TablaAgrupada = 
    GROUPBY(
        TablaUnion,
        [ID_Join],
        [Fecha_Join],
        [Elemento_Join],
        [Ubicacion_Join],
        [Actividad_Join],
        "Sum_Prod_Maq", SUMX(CURRENTGROUP(), [Prod_Val_Maq]),
        "Sum_Horas_Maq", SUMX(CURRENTGROUP(), [Horas_Maq]),
        "Sum_Seg_Man", SUMX(CURRENTGROUP(), [Segundos_Man]),
        "Max_Porc_Man", MAXX(CURRENTGROUP(), [Prod_Porc_Man]),
        "Max_Saturacion", MAXX(CURRENTGROUP(), [Saturacion_Join])
    )

VAR ResultadoFinal = 
    SELECTCOLUMNS(
        TablaAgrupada,
        "ID Operador", [ID_Join],
        "Fecha", [Fecha_Join],
        "Mes-Dia", FORMAT([Fecha_Join], "MM-dd"),
        "Elemento", [Elemento_Join],
        "Ubicacion", [Ubicacion_Join],
        "Actividad", [Actividad_Join],
        "Tiempo Elemento", [Sum_Horas_Maq],
        "Productividad Elemento (H)", [Sum_Prod_Maq],
        "% Eficiencia Elemento", IF([Sum_Horas_Maq] > 0, DIVIDE([Sum_Prod_Maq], [Sum_Horas_Maq]), 0),
        "Tiempo Manual (H)", [Sum_Seg_Man] / 3600,
        "Productividad Manual", [Max_Porc_Man],
        "Tiempo Total (H)", [Sum_Horas_Maq] + ([Sum_Seg_Man] / 3600),
        "% Productividad Ponderada", 
            VAR HorasProdManual = ([Max_Porc_Man] * ([Sum_Seg_Man] / 3600))
            VAR TotalHorasProducidas = [Sum_Prod_Maq] + HorasProdManual
            RETURN DIVIDE(TotalHorasProducidas, 7.75),
        "Saturacion", [Max_Saturacion]
    )

RETURN
    FILTER(
        ResultadoFinal, 
        [Tiempo Total (H)] >= 0.01
    )

// 2. Cálculo de Productividad en Procesos Manuales
TablaCalculos_Manuales = 
VAR MapaObjetivos = 
    GENERATE(
        VALUES('TablaRegistro_Manual'[ID_Componente]),
        VAR CodigoActual = [ID_Componente]
        VAR ValorReferencia = LOOKUPVALUE('TablaMaestra_Puntos'[ValorBase], 'TablaMaestra_Puntos'[Codigo], CodigoActual)
        VAR P90_Calculado = 
            CALCULATE(
                PERCENTILEX.INC(
                    'TablaRegistro_Manual',
                    DIVIDE(DATEDIFF('TablaRegistro_Manual'[HoraInicio], 'TablaRegistro_Manual'[HoraFin], SECOND), 'TablaRegistro_Manual'[UnidadesReales]),
                    0.2
                ),
                'TablaRegistro_Manual'[ID_Componente] = CodigoActual,
                'TablaRegistro_Manual'[UnidadesReales] > 5
            )
        RETURN ROW("Seg_Obj_Fijo", IF(NOT ISBLANK(ValorReferencia), ValorReferencia * 45, P90_Calculado))
    )

VAR TablaConTodo = 
    ADDCOLUMNS(
        'TablaRegistro_Manual',
        "T_Real_Total_Seg", DATEDIFF([HoraInicio], [HoraFin], SECOND),
        "S_Real_Pza", DIVIDE(DATEDIFF([HoraInicio], [HoraFin], SECOND), [UnidadesReales]),
        "S_Obj_Pza", 
            VAR Cod = [ID_Componente]
            RETURN MAXX(FILTER(MapaObjetivos, [ID_Componente] = Cod), [Seg_Obj_Fijo])
    )

RETURN 
    SELECTCOLUMNS(
        TablaConTodo,
        "Fecha", DATE(YEAR([HoraInicio]), MONTH([HoraInicio]), DAY([HoraInicio])),
        "FechaTurno", DATE(YEAR([HoraInicio]), MONTH([HoraInicio]), DAY([HoraInicio])), 
        "ID Operario", [ID_Operador],
        "Producto", [ID_Componente],
        "Unidades Reales", [UnidadesReales],
        "Valor Punto", DIVIDE([S_Obj_Pza], 45), 
        "Tiempo de Trabajo", 
            VAR TotalSegundos = [T_Real_Total_Seg]
            VAR Horas = INT(TotalSegundos / 3600)
            VAR Minutos = INT(MOD(TotalSegundos, 3600) / 60)
            RETURN FORMAT(Horas, "00") & ":" & FORMAT(Minutos, "00"),
        "Seg por Unid Objetivo", [S_Obj_Pza],
        "Seg por Unid Real", [S_Real_Pza],
        "Tiempo Real Total (Seg)", [T_Real_Total_Seg],
        "Porcentaje Productividad", 
            VAR T_Obj = [UnidadesReales] * [S_Obj_Pza]
            VAR T_Real = [T_Real_Total_Seg]
            RETURN 
                IF(
                    [TipoActividad] IN {"ACTIVIDAD_EXC_01", "ACTIVIDAD_EXC_02", "ACTIVIDAD_EXC_03", "ACTIVIDAD_EXC_04", "ACTIVIDAD_EXC_05"},
                    1.0, 
                    IF(T_Real > 0 && T_Obj > 0, DIVIDE(T_Obj, T_Real), BLANK())
                ),
        "Saturacion", 100.0,
        "TipoActividad", [TipoActividad] 
    )

// 3. Resumen Métrico por Operador y Turno
TablaMetricas_Operador = 
SUMMARIZE(
    FILTER(
        'TablaRegistro_Actividad', 
        'TablaRegistro_Actividad'[CodigoOperador] <> "ND"
    ),
    'TablaRegistro_Actividad'[CodigoOperador],
    'TablaRegistro_Actividad'[FechaTurno],
    'TablaRegistro_Actividad'[ID_Elemento],
    'TablaRegistro_Actividad'[ID_Ubicacion],
    
    "Fecha", 
        VAR TextoFecha = 'TablaRegistro_Actividad'[FechaTurno]
        VAR PosBarra = SEARCH("/", TextoFecha, 1, 3) 
        VAR Mes = VALUE(LEFT(TextoFecha, PosBarra - 1))
        VAR Dia = VALUE(MID(TextoFecha, PosBarra + 1, 2))
        RETURN DATE(2026, Mes, Dia),

    "ID_Operador", 
        VAR Texto = [CodigoOperador]
        VAR PosEspacio = SEARCH(" ", Texto, 1, 0)
        RETURN IF(PosEspacio > 0, LEFT(Texto, PosEspacio - 1), Texto),
    
    "ValorProductividad", SUM('TablaRegistro_Actividad'[ProductividadAcumulada]),
    "% Productividad", SUM('TablaRegistro_Actividad'[ProductividadAcumulada]) / 7.75,
    "TiempoBase_H", SUM('TablaRegistro_Actividad'[DuracionSegundos] / 3600),
    "NivelSaturacion", AVERAGE('TablaRegistro_Actividad'[SaturacionBase])
)

// 4. Matriz de Presencia en Intervalos de 15 Minutos
TablaPresencia_Intervalos = 
ADDCOLUMNS(
    CROSSJOIN(
        DimOperarios,
        TablaIntervalos15Min
    ),
    "Presente",
    VAR Operario = [CodigoOperador]
    VAR InicioIntervalo = [IntervaloInicio]
    VAR FinIntervalo = [IntervaloFin]
    RETURN
        IF(
            COUNTROWS(
                FILTER(
                    'TablaRegistro_Actividad',
                    'TablaRegistro_Actividad'[CodigoOperador] = Operario &&
                    'TablaRegistro_Actividad'[FechaHoraInicio] < FinIntervalo &&
                    'TablaRegistro_Actividad'[FechaHora] > InicioIntervalo
                )
            ) > 0,
            1,
            0
        ),
    "FechaTurno",
    VAR RegistroTurno =
        FILTER(
            'TablaRegistro_Actividad',
            'TablaRegistro_Actividad'[CodigoOperador] = [CodigoOperador] &&
            'TablaRegistro_Actividad'[FechaHoraInicio] < [IntervaloFin] &&
            'TablaRegistro_Actividad'[FechaHora] > [IntervaloInicio]
        )
    RETURN
        IF(COUNTROWS(RegistroTurno) > 0, MINX(RegistroTurno, 'TablaRegistro_Actividad'[FechaTurno]), BLANK())
)

// 5. Cálculo Consolidado de Presencialidad y Horas Efectivas
TablaPresencialidad_Consolidada = 
VAR ManualesPreparado =
    SELECTCOLUMNS(
        'TablaRegistro_Manual', 
        "ID_Var", CONVERT('TablaRegistro_Manual'[ID_Operador], STRING),
        "Nombre_Var", "Operador Manual", 
        "FechaTurno_Var", 
            VAR _Hora = HOUR('TablaRegistro_Manual'[HoraInicio])
            VAR _FechaSoloDia = DATE(YEAR('TablaRegistro_Manual'[HoraInicio]), MONTH('TablaRegistro_Manual'[HoraInicio]), DAY('TablaRegistro_Manual'[HoraInicio]))
            VAR _FechaAjustada = IF(_Hora < 6, _FechaSoloDia - 1, _FechaSoloDia)
            RETURN FORMAT(_FechaAjustada, "MM/dd") & " - 1",
        "Elemento_Var", "SEGMENTO_MANUAL",
        "Ubicacion_Var", "SEGMENTO_MANUAL",
        "H_Maquina", 0.0, 
        "H_Manuales", DATEDIFF('TablaRegistro_Manual'[HoraInicio], 'TablaRegistro_Manual'[HoraFin], MINUTE) / 60.0,
        "Actividad_Var", 'TablaRegistro_Manual'[TipoActividad]
    )

VAR MaquinaPreparado =
    GENERATE(
        'TablaMetricas_Operador',
        VAR OperarioActual = [CodigoOperador]
        VAR FechaActual = [FechaTurno]
        
        VAR PresenciaRealTotal = 
            CALCULATE(
                SUM(TablaPresencia_Intervalos[Presente]) / 12,
                TablaPresencia_Intervalos[CodigoOperador] = OperarioActual,
                TablaPresencia_Intervalos[FechaTurno] = FechaActual
            )
            
        VAR TiempoMaqTotalTurno = 
            CALCULATE(
                SUM('TablaMetricas_Operador'[TiempoBase_H]),
                ALLEXCEPT('TablaMetricas_Operador', 'TablaMetricas_Operador'[CodigoOperador], 'TablaMetricas_Operador'[FechaTurno])
            )
            
        VAR PresenciaRepartida = 
            IF(TiempoMaqTotalTurno > 0, 
                DIVIDE([TiempoBase_H], TiempoMaqTotalTurno) * PresenciaRealTotal, 
                0
            )
            
        RETURN ROW(
            "ID_Var", [ID_Operador] & "",
            "Nombre_Var", TRIM(MID([CodigoOperador], SEARCH(" ", [CodigoOperador] & " ") + 1, 999)),
            "FechaTurno_Var", [FechaTurno],
            "Elemento_Var", CONVERT([ID_Elemento], STRING),
            "Ubicacion_Var", CONVERT([ID_Ubicacion], STRING),
            "H_Maquina_Calculada", PresenciaRepartida,
            "H_Manuales_Var", 0.0,
            "Actividad_Var", BLANK()
        )
    )

VAR TablaUnida = 
    UNION(
        SELECTCOLUMNS(MaquinaPreparado, "ID", [ID_Var], "Nombre", [Nombre_Var], "Fecha", [FechaTurno_Var], "Maq", [Elemento_Var], "Nav", [Ubicacion_Var], "HM", [H_Maquina_Calculada], "HMan", [H_Manuales_Var], "Act", [Actividad_Var]),
        SELECTCOLUMNS(ManualesPreparado, "ID", [ID_Var], "Nombre", [Nombre_Var], "Fecha", [FechaTurno_Var], "Maq", [Elemento_Var], "Nav", [Ubicacion_Var], "HM", [H_Maquina], "HMan", [H_Manuales], "Act", [Actividad_Var])
    )

VAR ResultadoAgrupado = 
    GROUPBY(
        TablaUnida,
        [ID],
        [Fecha],
        [Maq],
        [Nav],
        [Act],
        "Nombre Operario", MAXX(CURRENTGROUP(), [Nombre]),
        "Total Horas Maquina", SUMX(CURRENTGROUP(), [HM]),
        "Total Horas Manuales", SUMX(CURRENTGROUP(), [HMan]),
        "Horas Totales", SUMX(CURRENTGROUP(), [HM] + [HMan])
    )

RETURN
    ADDCOLUMNS(
        ResultadoAgrupado,
        "Fecha Sin Turno", 
            VAR _PosGuion = SEARCH(" -", [Fecha], 1, 0)
            VAR _SoloFecha = IF(_PosGuion > 0, LEFT([Fecha], _PosGuion - 1), [Fecha])
            RETURN SUBSTITUTE(_SoloFecha, "/", "-"),
        "Actividad Realizada", [Act]
    )
