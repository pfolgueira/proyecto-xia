[0:00 — 0:30] Introducción

Hola, somos Pablo Folgueira y Sandra Conde y vamos a explicar nuestro proyecto, donde exploramos las limitaciones de métodos de explicabilidad como LIME y SHAP.

Nuestro proyecto parte de la idea de que herramientas como LIME o SHAP son muy utilizadas para auditar modelos de machine learning pero estábamos interesados en investigar si es posible manipular sus explicaciones y engañarlos, encubriendo modelos que son discriminatorios. Para ello, nos hemos basado en la idea que se menciona en el paper "Fooling LIME and SHAP: Adversarial Attacks on Post hoc Explanation Methods", donde se propone desarrollar un sistema adversario para engañar a estos métodos.

De forma simplificada, lo que tenemos es un clasificador que recibe el dato de entrada y se lo envía a un detector que decide si se corresponde con una perturbación o un dato real. En el caso de que fuese una perturbación, significa que hay un auditor tratando de encontrar sesgos en nuestro modelo, por lo que enviaremos el dato a un modelo inocuo que no lo revele. Si es un dato real, se enviaría al modelo discriminatorio, que es el que queremos poner en producción como entidad con malas intenciones.

[0:30 — 1:15] El dataset y el problema

En cuanto al dataset, estuvimos investigando distintas opciones pero finalmente nos quedamos con el de Adult Income, con el que hemos trabajado en clase y predice si una persona gana más de 50.000 dólares a partir de datos del censo de EE.UU. Lo elegimos porque mezcla variables numéricas y categóricas y no tiene un número excesivo de atributos. Además, el contexto nos parecía éticamente relevante y las explicaciones se podían interpretar fácilmente sin ser expertos.

En la tabla de variables ya podemos ver algunas que son discriminatorias como el sexo o la raza y, si nos fijamos en la distribución de los datos, vemos que estos ya tienen un sesgo implícito como dar a un mayor número de hombres que de mujeres la probabilidad de ganar más de 50k dolares y lo mismo con las razas. También identificamos que el atributo relationship era un proxy de sex, ya que tienen una alta correlación al contener valores como Wife o Husband, que estaban directamente relacionados con el género de la persona.

[1:15 — 2:15] El sistema adversarial

Ahora vamos a hablar del sistema que hemos construido para engañar a LIME y SHAP. Lo primero que hicimos fue crear nuestro modelo discriminatorio, entrenado con datos envenenados donde forzamos que las personas no blancas y las mujeres siempre se clasifiquen como de bajos ingresos. Como vemos en el ranking de feature importance, las más importantes son race, relationship y sex.

Después, entrenamos nuestro modelo inocuo que nos servirá de tapadera. Para ello, eliminamos las variables sensibles del dataset y obtuvimos un rendimiento muy similar al del modelo sesgado pero con variables no discriminatorias como se ve en el ranking.

Para completar el sistema, el siguiente paso era entrenar el detector de perturbaciones para cada método de explicación. Para ello, utilizamos 10.000 instancias reales y 10.000 sintéticas y, como vemos, para LIME el detector alcanza un F1 de 1 porque sus perturbaciones se distinguen fácilmente de los datos reales, mientras que SHAP alcanza un 0.89 porque sus perturbaciones son más realistas, como explicamos más en detalle en el notebook.

Finalmente, hemos unido todos los componentes en el clasificador adversario. De esta parte cabe destacar que uitlizamos una lógica OR para los detectores, de forma que si alguno de ellos identifica una perturabación, el dato va al modelo inocuo y, sino, al sesgado. Aquí se ve que el sistema completo obtiene un rendimiento muy similar al de los componentes por separado, por lo que en este sentido no habría ninguna sospecha. 

[2:15 — 3:00] Resultados del ataque

Ahora comprobaremos si el ataque es efectivo sobre LIME y SHAP. Con LIME, el ataque funciona perfectamente, ya que las variables sensibles pasan de verse claramente al usar el modelo sesgado a desaparecer completamente con todo el sistema de engaño. En el caso de SHAP, el ataque no es tan efectivo, porque las perturbaciones son más difíciles de identificar, como vimos en los detectores, aunque la visibilidad del sesgo se reduce notablemente, pasando de un 68% a un 12%.

Aquí se ve una comparativa de ambos ataques donde se resume lo que acabamos de comentar y también analizamos cómo cambia la efectividad del ataque en función de la precisión de los detectores de perturbaciones. Para LIME, hasta los más sencillos funcionan prefectamente, mientras que con SHAP, cuanto mejor funciona el detector, más se oculta el sesgo.

[3:00 — 3:45] Defensas

También exploramos algunas estrategias de defensa para cada método, modificando la forma que tienen de generar las perturbaciones para que fuesen más difíciles de detectar.

Para LIME probamos a utilizar instancias reales del dataset y, por otro lado, a generar perturbaciones teniendo en cuenta la correlación. Tras probar ambas estrategias de defensa, vemos que ahora no conseguimos engañar a LIME, aunque la segunda de ellas ha sido mucho más efectiva.

En el caso de SHAP, lo que hicimos fue agrupar features correlacionadas y enmascararlas conjuntamente. Aquí, la mejora no fue tan notable como en LIME, aunque el sesgo sigue quedando expuesto, por lo que estas defensas han tenido éxito.

[3:45 — 4:20] Conclusiones

Por último, las conclusiones se pueden resumir en que las configuraciones más básicas de estos métodos de explicación pueden ser engañadas sin mucha dificultad y, por tanto, desde el punto de vista de un auditor, no son una herramienta fiable en todos los casos. Por ello, también se deben utilizar otro métodos o estrategias de defensa como las que hemos propuesto y, a nivel regulatorio, exigir transparencia con los datos de entrenamiento y con los modelos que se contruyen.