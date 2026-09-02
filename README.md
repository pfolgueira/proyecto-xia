# Explorando las limitaciones de LIME y SHAP mediante ataques adversarios

Proyecto de **Inteligencia Artificial Explicable (XAI)** centrado en estudiar la vulnerabilidad de los métodos de explicabilidad **LIME** y **SHAP** frente a ataques adversarios capaces de ocultar comportamientos discriminatorios de un modelo.

El proyecto parte de la propuesta presentada en *Fooling LIME and SHAP: Adversarial Attacks on Post hoc Explanation Methods* (Slack et al., 2019) y la extiende mediante un sistema capaz de engañar simultáneamente a LIME y SHAP, además de explorar diferentes estrategias de defensa.

> **Proyecto desarrollado por Sandra Conde González y Pablo Folgueira Galán**
> Máster en Inteligencia Artificial — Universidad Complutense de Madrid

## 📌 Descripción

Los métodos de explicabilidad post-hoc como LIME y SHAP permiten analizar cómo un modelo de Machine Learning toma sus decisiones. Sin embargo, las explicaciones generadas dependen de cómo estos métodos construyen las perturbaciones utilizadas para aproximar el comportamiento local del modelo.

Este proyecto estudia cómo un modelo puede aprovechar esta característica para **detectar cuándo está siendo analizado por un método de explicabilidad y modificar su comportamiento**, ocultando así un sesgo discriminatorio durante una auditoría.

Para ello se utiliza el dataset **Adult Income**, construyendo un escenario experimental en el que:

* Se entrena un modelo deliberadamente sesgado.
* Se entrena un modelo inocuo que no utiliza atributos sensibles.
* Se construyen detectores **Out-of-Distribution (OOD)** capaces de distinguir entre datos reales y perturbaciones generadas por LIME y SHAP.
* Un clasificador adversarial decide qué modelo utilizar dependiendo del tipo de entrada.
* Se evalúa hasta qué punto LIME y SHAP son capaces de detectar el sesgo.
* Se implementan diferentes estrategias de defensa para generar perturbaciones más realistas.

## 🎯 Objetivos

* Analizar las limitaciones de LIME y SHAP como herramientas de auditoría.
* Estudiar la vulnerabilidad de los métodos de explicabilidad frente a ataques adversarios.
* Construir un sistema capaz de ocultar un comportamiento discriminatorio.
* Comparar la robustez de LIME y SHAP ante este escenario.
* Analizar la influencia de los detectores OOD en la efectividad del ataque.
* Diseñar estrategias de defensa basadas en perturbaciones más realistas.
* Reflexionar sobre las implicaciones éticas de utilizar XAI para auditar modelos en contextos sensibles.

## 🗂️ Dataset

Se utiliza el dataset **Adult Income (Census 1994)**, procedente del UCI Machine Learning Repository.

El objetivo es predecir si una persona tiene unos ingresos anuales superiores a **50.000 dólares**.

Tras la limpieza de los datos se utilizan **46.033 instancias y 11 variables**:

### Variables numéricas

* `age`
* `education_num`
* `hours_per_week`
* `capital_gain`
* `capital_loss`

### Variables categóricas

* `workclass`
* `marital_status`
* `occupation`
* `relationship`
* `race`
* `sex`

Las variables `sex`, `race` y `relationship` se consideran **atributos sensibles**. En particular, `relationship` se considera también un posible proxy de género debido a su elevada correlación con `sex`.

## 🧪 Metodología

### 1. Preprocesamiento

Los datos se limpian y se eliminan las instancias con valores ausentes. Las variables categóricas se codifican mediante `LabelEncoder` y posteriormente se divide el conjunto en entrenamiento y test.

### 2. Construcción del modelo sesgado

Para crear un escenario de auditoría controlado se modifica deliberadamente una parte de las etiquetas de entrenamiento:

* Las mujeres con ingresos superiores a 50K pasan a tener una etiqueta negativa.
* Las personas no blancas con ingresos superiores a 50K pasan a tener una etiqueta negativa.

De esta forma se fuerza al modelo a aprender un comportamiento discriminatorio basado en atributos sensibles.

### 3. Modelo inocuo

Se entrena un segundo modelo utilizando únicamente las variables consideradas no sensibles.

Este modelo representa el comportamiento que aparentemente debería mostrar un sistema que no discrimina en función de `sex`, `race` o `relationship`.

### 4. Detectores OOD

Se entrenan clasificadores capaces de distinguir entre:

* **Datos reales**, procedentes del dataset.
* **Datos perturbados**, generados por los métodos de explicabilidad.

Estos detectores permiten identificar cuándo el sistema está siendo analizado mediante LIME o SHAP.

### 5. Clasificador adversarial

Se desarrolla un sistema adversarial que combina los detectores y utiliza una lógica **OR** para identificar perturbaciones generadas por cualquiera de los dos métodos.

La arquitectura utiliza dos posibles modelos:

```text
                    Entrada
                       │
                       ▼
                ┌─────────────┐
                │ Detector OOD│
                └──────┬──────┘
                       │
             ┌─────────┴─────────┐
             │                   │
          Dato real          Perturbación
             │                   │
             ▼                   ▼
       Modelo sesgado      Modelo inocuo
          f_b                    f_i
             │                   │
             └─────────┬─────────┘
                       ▼
                  Predicción
```

El objetivo del atacante es que las instancias reales sean procesadas por el modelo sesgado, mientras que las perturbaciones generadas durante la auditoría sean procesadas por el modelo inocuo.

## 🔍 Ataques sobre LIME y SHAP

Se estudian dos escenarios:

### LIME

El ataque consigue ocultar completamente el sesgo en las explicaciones generadas por la versión estándar de LIME.

En las evaluaciones realizadas, **ninguna variable sensible aparece como la variable más importante** en las explicaciones obtenidas mediante LIME bajo el ataque.

### SHAP

El ataque también consigue ocultar una parte importante del sesgo, aunque resulta menos efectivo que con LIME debido a las características de las perturbaciones generadas por SHAP.

En los experimentos, el porcentaje de instancias con una variable sensible en el Top 1 se reduce hasta aproximadamente un **11 %** frente al **65,6 %** observado al explicar directamente el modelo sesgado con SHAP estándar.

## 📊 Robustez del ataque

También se analiza cómo afecta la capacidad del detector OOD a la efectividad del ataque.

Para LIME, incluso clasificadores relativamente sencillos consiguen distinguir eficazmente entre datos reales y perturbaciones, alcanzándose valores de **F1 cercanos a 1**. A medida que mejora el detector, el sesgo resulta cada vez menos visible.

En SHAP se observa un comportamiento similar, aunque el detector presenta mayor dificultad para separar ambos tipos de datos.

Estos resultados muestran que la efectividad del ataque depende en gran medida de lo fácilmente distinguibles que sean las perturbaciones generadas por el método de explicabilidad.

## 🛡️ Estrategias de defensa

Para estudiar cómo podrían mitigarse estos ataques, se modifican las estrategias utilizadas para generar las perturbaciones.

### LIME con vecinos reales

En lugar de generar instancias sintéticas, se utilizan los **K vecinos reales más cercanos** del conjunto de entrenamiento para construir la explicación local.

Esto dificulta que el detector OOD pueda identificar las instancias como perturbaciones.

Resultado:

| Método                   | Variables sensibles en Top 1 |
| ------------------------ | ---------------------------: |
| LIME estándar (engañado) |                        0,0 % |
| LIME + vecinos reales    |                       36,4 % |

### LIME con correlaciones multivariantes

Se genera el vecindario mediante una distribución normal multivariante que tiene en cuenta las correlaciones existentes entre las variables.

De esta forma se producen perturbaciones más realistas y difíciles de detectar.

Resultado:

| Método                              | Variables sensibles en Top 1 |
| ----------------------------------- | ---------------------------: |
| LIME estándar (engañado)            |                        0,0 % |
| LIME + correlaciones multivariantes |                   **70,6 %** |

Esta estrategia resulta especialmente efectiva para revelar el sesgo del modelo.

### SHAP Partition Masker

Para SHAP se utiliza `Partition Masker`, agrupando las variables en función de sus correlaciones y enmascarándolas conjuntamente.

Aunque esta estrategia consigue que más del **80 %** de las instancias sean dirigidas al modelo sesgado durante la explicación, el sesgo no se concentra tanto en las primeras posiciones del ranking.

| Método                                 | Variables sensibles en Top 1 |
| -------------------------------------- | ---------------------------: |
| SHAP estándar (engañado)               |                       11,4 % |
| SHAP Partition Masker                  |                       18,0 % |
| SHAP estándar — modelo sesgado         |                       65,6 % |
| SHAP Partition Masker — modelo sesgado |                       58,0 % |

## 📈 Principales conclusiones

Los experimentos muestran que **LIME y SHAP no deben considerarse herramientas infalibles para auditar la ausencia de sesgos en modelos de Machine Learning**.

El sistema desarrollado demuestra que es posible construir un modelo que:

* Mantenga un rendimiento predictivo aparentemente normal.
* Utilice atributos sensibles para tomar decisiones.
* Detecte cuándo está siendo analizado.
* Modifique su comportamiento durante la auditoría.
* Oculte el sesgo de las explicaciones generadas por LIME y, en menor medida, SHAP.

El ataque resulta especialmente efectivo contra **LIME estándar**, mientras que SHAP presenta una mayor resistencia debido a las características de sus perturbaciones.

Por otro lado, las estrategias de defensa experimentadas muestran que **generar perturbaciones más realistas**, respetando la distribución y las correlaciones de los datos, puede dificultar considerablemente este tipo de ataques.

Esto pone de manifiesto la importancia de realizar auditorías de modelos desde diferentes perspectivas y no depender exclusivamente de una única técnica de explicabilidad.

## ⚠️ Implicaciones éticas

Este proyecto se plantea como un **experimento de investigación sobre seguridad, robustez y auditoría de sistemas de IA**.

El objetivo no es desarrollar sistemas discriminatorios para su utilización real, sino demostrar experimentalmente cómo podrían ser manipuladas determinadas técnicas de auditoría y por qué es necesario desarrollar métodos de explicabilidad más robustos.

Este tipo de vulnerabilidades resulta especialmente relevante en sistemas utilizados para decisiones relacionadas con **contratación, concesión de créditos, acceso a servicios u otros ámbitos de alto impacto**.

## 🚀 Trabajo futuro

Como posibles líneas de continuación se plantean:

* Explorar defensas basadas en **modelos generativos**, como GANs o VAEs, para generar perturbaciones más próximas a la distribución real.
* Extender el estudio a datos no tabulares, como **texto e imágenes**.
* Analizar otros métodos de explicabilidad post-hoc, como **Anchors** o **Counterfactual Explanations**.
* Evaluar detectores OOD más sofisticados.
* Estudiar estrategias de auditoría que combinen múltiples métodos de explicabilidad.
* Analizar la robustez de estas técnicas en datasets y escenarios reales.

## 🛠️ Tecnologías utilizadas

* **Python**
* **NumPy**
* **Pandas**
* **Scikit-learn**
* **XGBoost**
* **LIME**
* **SHAP**
* **Matplotlib**
* **Seaborn**
* **SciPy**
* **PCA**
* **Random Forest**
* **K-Means**
* **Ridge Regression**

## 📚 Documentación y notebook

El desarrollo completo del experimento, incluyendo el código, las visualizaciones, los resultados y el análisis de los experimentos, está documentado en el capítulo:

**Explorando las limitaciones de métodos de explicabilidad LIME y SHAP — XAI Stories**

👉 https://belenda.github.io/xai_stories_project/chapters/adversarios_lime_shap/capitulo.html

El capítulo forma parte de **XAI Stories**, una colección de trabajos sobre Inteligencia Artificial Explicable desarrollados en el contexto del Máster en Inteligencia Artificial de la Universidad Complutense de Madrid.

## 📖 Referencias

* Slack, D., Hilgard, S., Jia, E., Singh, S., & Lakkaraju, H. (2019). *How can we fool LIME and SHAP? Adversarial Attacks on Post hoc Explanation Methods.*
* Ribeiro, M. T., Singh, S., & Guestrin, C. (2016). *"Why Should I Trust You?" Explaining the Predictions of Any Classifier.*
* Lundberg, S. M. & Lee, S. I. (2017). *A Unified Approach to Interpreting Model Predictions.*
* Becker, B. & Kohavi, R. (1996). *Adult Dataset.* UCI Machine Learning Repository.
