# Sistema de decisión para PagoYa — Grupo 2
Integrantes: 
* Benavides Coraima
* Cobos Jair
* Freire Alejandro
* Villamarin Kevin
  
# Problema: Detección de Fraude en Transacciones Financieras
## 1. Problema y datos
El objetivo del proyecto es detectar transacciones fraudulentas para minimizar las pérdidas financieras de la plataforma. Contamos con un dataset de 2,996 eventos y 17 variables (demográficas, transaccionales y temporales).
Durante el EDA rápido, se identificó una tasa de fraude del 10.51%, lo que evidencia un dataset desbalanceado. Para mejorar la capacidad predictiva, se realizó Feature Engineering creando variables clave como madrugada (transacciones entre 0 y 6 AM), ratio_monto_edad y monto_log (para normalizar la distribución del monto). Los gráficos de densidad demostraron que los fraudes se concentran en montos significativamente más altos y que la tasa de fraude en la madrugada (25%) casi triplica a la del resto del día.

## 2. Modelos comparados (tabla de métricas)
Se realizó una partición estratificada 70/30 (random_state=42). Se entrenaron tres modelos: una Regresión Logística (base, con escalado y balanceo de clases), un Random Forest y un LightGBM (con early stopping interno para evitar fuga de datos).

| Modelo | AUC | Balanced Accuracy | Brier Score |
| :--- | :---: | :---: | :---: |
| Regresión Logística | 0.8755 | 0.8139 | 0.1277 |
| Random Forest | 0.8978 | 0.7130 | 0.0582 |
| LightGBM (Boosting) | 0.8894 | 0.8512 | 0.2160 |

**Decisión:** Se seleccionó el Random Forest como modelo final. Aunque LightGBM tuvo una mejor exactitud balanceada, el Random Forest dominó en las métricas más críticas: obtuvo el mejor AUC (0.8978) para discriminar fraudes, y un Brier Score casi perfecto (0.0582), lo que indica que sus probabilidades están excelentemente calibradas para la toma de decisiones reales.

## 3. Qué explica el resultado (variables, segmentos, incertidumbre)
El gráfico global revela que las variables más determinantes para que el modelo detecte un fraude son, por orden de importancia:
*	Dispositivo_nuevo: Es el factor más influyente. Cuando el valor es 1 (puntos rojos), empuja fuertemente la predicción hacia el fraude. Es lógico: un defraudador suele usar un dispositivo que el sistema no ha visto antes.
*	Distancia_km y ratio: Transacciones a grandes distancias o con ratios atípicos también aumentan el riesgo.
*	Monto y monto_log: A mayor monto, mayor probabilidad de fraude, aunque su impacto es ligeramente menor que el dispositivo.
*	Hora e intentos_pin: Los intentos de PIN fallidos (puntos rojos) son un claro indicador de alerta.

### Explicación Individual (Caso Índice 2308):
El modelo evaluó una transacción específica y le asignó una probabilidad de fraude del 90.8% (f(x)=0.908). El gráfico de cascada explica por qué:
* Factores de Riesgo (suman probabilidad): El uso de un dispositivo nuevo (+0.23) fue el detonante principal. Le siguieron un ratio atípico (+0.11), una distancia elevada de 7.1 km (+0.08), un monto alto de $219.7 (+0.05) y un ratio_monto_edad elevado (+0.03).
*	Factores de Protección (restan probabilidad): Curiosamente, la transacción se realizó a las 15:00 hrs (-0.03), el usuario tiene 36 años (-0.03) y no fue en la madrugada (-0.02).
*	Conclusión del caso: A pesar de que no fue en la madrugada y el usuario tiene una edad promedio, la combinación letal de un dispositivo nuevo + monto alto + distancia inusual fue suficiente para que el modelo lo clasificara como un fraude casi seguro (90.8%).
  
## 4. Recomendación 
Decisión: Implementar el modelo Random Forest con un umbral de decisión (threshold) óptimo de 0.12 (en lugar del 0.50 por defecto).
*	Costo sin modelo (No hacer nada): Dejar pasar todas las transacciones cuesta $12,984.88 en fraudes consumados.
*	Costo con el umbral óptimo: Al aplicar el umbral de 0.12, el costo se reduce a $1,366.63.
*	Impacto: Esto genera un ahorro neto de $11,618.25 (una reducción del 89.4% en pérdidas).
Bajar el umbral a 0.12 es matemáticamente correcto porque el costo de bloquear a un cliente bueno es de solo USD 5, mientras que el costo de no detectar un fraude es perder el monto total de la compra. El gráfico de optimización demostró que este es el punto exacto donde se minimiza el costo total.

## 5. Limitaciones y próximos pasos
*	Desbalanceo extremo: Aunque usamos class_weight='balanced', el dataset sigue teniendo pocos casos de fraude real.
*	Calibración en Boosting: Si en el futuro se opta por LightGBM, es obligatorio aplicar CalibratedClassifierCV (Platt Scaling) para arreglar su alto Brier Score (0.2160).
*	Variables categóricas en el Simulador: El simulador actual asume un perfil fijo para variables como canal o provincia. El próximo paso es permitir que el usuario también pueda simular transacciones desde diferentes canales (App, Web, POS) para medir el riesgo de cada canal.
*	Despliegue en Producción: Se recomienda monitorear el Data Drift (cambio en el comportamiento de los usuarios) cada 3 meses, ya que los patrones de fraude evolucionan rápidamente.


