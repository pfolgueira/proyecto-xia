## Guión del vídeo

---

**[0:00 — 0:30] Introducción**

En este vídeo voy a explicar el proyecto que hemos desarrollado sobre
ataques adversariales contra métodos de explicación de modelos de machine
learning, concretamente contra LIME y SHAP.

La motivación del trabajo parte de una pregunta sencilla: si usamos LIME
o SHAP para auditar un modelo y comprobar que no discrimina, ¿podemos
confiar en esas explicaciones? La respuesta, como vamos a ver, es que no
necesariamente.

---

**[0:30 — 1:15] El dataset y el problema**

Hemos trabajado con el dataset Adult Income, que predice si una persona
gana más de 50.000 dólares anuales a partir de datos del censo de EE.UU.
Lo elegimos porque tiene una mezcla de variables numéricas y categóricas,
el contexto es éticamente relevante — estamos hablando de decisiones que
afectan al acceso a recursos económicos — y las explicaciones son
fácilmente interpretables sin necesidad de ser experto en el dominio.

El atributo sensible principal es la raza, aunque también tratamos el
sexo y la variable `relationship` como sensibles, porque tiene una
correlación de Spearman de -0.62 con el sexo y actúa como proxy
indirecto de género.

---

**[1:15 — 2:15] El sistema adversarial**

El núcleo del trabajo es construir un clasificador adversarial que
discrimina en producción pero engaña a LIME y SHAP. Para ello
entrenamos tres componentes.

El primero es un modelo sesgado, entrenado con datos envenenados donde
forzamos que las personas no blancas y las mujeres siempre se clasifiquen
como de bajos ingresos. Sus tres features más importantes son `race`,
`relationship` y `sex`, que acumulan el 87% de la importancia total.

El segundo es un modelo inocuo, entrenado sin las variables sensibles
y con las etiquetas originales. Usa predictores legítimos como
`capital_gain` o `education_num`, y de hecho supera al modelo sesgado
en AUC — 0.93 frente a 0.84 — lo que significa que un auditor que
compare métricas globales no detectará ninguna anomalía.

El tercero es un detector OOD, que distingue si una entrada es un dato
real del dataset o una perturbación artificial como las que generan LIME
y SHAP internamente. Para LIME el detector alcanza F1=1.0 porque sus
perturbaciones gaussianas son trivialmente distinguibles de datos reales.
Para SHAP el F1 es 0.89, porque sus perturbaciones por K-Means masking
son más realistas.

El clasificador adversarial combina los tres: si el detector identifica
un dato real, usa el modelo sesgado. Si identifica una perturbación de
LIME o SHAP, usa el modelo inocuo. El resultado es que mantiene una
accuracy de 0.855 y un AUC de 0.851 en producción, mientras engaña a
los explicadores.

---

**[2:15 — 3:00] Resultados del ataque**

Los resultados son contundentes. Con LIME estándar, el ataque es perfecto:
las variables sensibles desaparecen completamente del ranking en las 500
instancias evaluadas, pasando del 97.6% al 0%. Con SHAP, el ataque es
parcialmente efectivo, reduciendo la visibilidad del sesgo del 67% al 12%.
La diferencia se explica por el F1 de los detectores — LIME es más fácil
de engañar porque sus perturbaciones son más detectables.

También replicamos la Figura 5 del paper original, que muestra cómo la
efectividad del ataque depende de la precisión del detector OOD. Para
LIME todos los detectores alcanzan F1 alto independientemente de su
complejidad, confirmando que la gaussiana es intrínsecamente detectable.
Para SHAP sí se observa la curva decreciente esperada: a menor F1 del
detector, más sesgo visible.

---

**[3:00 — 3:45] Defensas**

La parte final del trabajo explora estrategias de defensa, que consisten
en modificar cómo LIME y SHAP generan sus perturbaciones para hacerlas
más difíciles de detectar por el OOD classifier.

Para LIME probamos dos variantes. La primera usa vecinos reales del
training set en lugar de perturbaciones sintéticas, recuperando una
detección del 37%. La segunda muestrea de una distribución normal
multivariante que respeta las correlaciones entre variables, alcanzando
el 64% — casi duplicando la efectividad de la defensa anterior y
demostrando que la calidad de las perturbaciones importa más que su
cantidad.

Para SHAP usamos el Partition Masker, que agrupa features por correlación
y las enmascara en bloques. La mejora es más modesta, del 12% al 18%,
porque el método distribuye inherentemente la importancia entre más
features, diluyendo la señal del sesgo.

---

**[3:45 — 4:20] Conclusiones**

Los resultados de este trabajo tienen una implicación clara: la
explicabilidad post-hoc no es suficiente como único mecanismo de auditoría
en contextos donde existen incentivos para la discriminación. Un modelo
adversarial construido con recursos técnicos accesibles puede superar
cualquier auditoría basada en LIME o SHAP estándar mientras discrimina
sistemáticamente en producción.

La única defensa completa no es técnica sino regulatoria: exigir
transparencia en los datos de entrenamiento y en el código, no solo en
las explicaciones de las decisiones individuales. Auditar la fachada de
un modelo sin acceso a su proceso de construcción es, a la luz de estos
resultados, insuficiente.