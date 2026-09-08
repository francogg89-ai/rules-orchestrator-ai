# REGLAS DEL ORQUESTADOR

Comportamiento mecánico del orquestador de un trabajo gobernado por REVOLUTIONS — ORCHESTRA.

Está escrito para poder implementarse mediante código determinista, sin un modelo razonador.

Cada sección numerada mecánica empieza con un bloque de obligaciones etiquetadas. Ese bloque es
la única superficie normativa del documento. Todo lo que sigue a un marcador `Nota.` explica y no
obliga. La forma exacta de esa convención está en `13`.

## Qué gobierna este documento y qué no

Gobierna lo que el orquestador hace: validar la forma de lo que recibe, elegir instancia,
entregar, detenerse y reportar.

No gobierna el contrato de transporte. La forma del sobre `revolutions-hop/v1`, sus campos, sus
combinaciones admitidas, y la forma que debe tener la respuesta de un actor, son autoridad de
`revolutions-orchestra-ai : metodo/REVOLUTIONS.md`. Este documento no la reproduce, no la amplía
y no le agrega campos: describe qué comprueba el orquestador sobre ella.

Tampoco gobierna cómo se produce el paquete de constitución, ni la política de relevo de un
trabajo. Ambas son autoridad de `metodo-manifiestos-ai : METODO-MANIFIESTOS.md`.

## Principio

> El orquestador transporta.

Si para transportar hiciera falta interpretar el trabajo, la interfaz está mal diseñada y
corresponde detenerse, no suplirla.

---

# 1. Dos situaciones distintas

```text
R-1-dos-caminos          el arranque externo del trabajo y los pases internos del loop se tratan
                         por caminos distintos
```

Nota. Confundirlos es el error que produce un trabajo sin constitución, o un pase sin origen.

```text
ARRANQUE EXTERNO DEL TRABAJO   una sola vez, sin sobre anterior
PASES INTERNOS DEL LOOP        todas las veces siguientes, siempre desde un sobre
```

## 1.1. Arranque externo

```text
R-1.1-recibe-init        el orquestador recibe un locator de arranque conforme a auditor-init/v1
R-1.1-abre-auditor      abre una instancia inicial nueva de AUDITOR
R-1.1-forma             el locator se transporta como una unica linea ASCII:
                         AUDITOR_INIT_V1|WORK_ID=<id>|CARRIL=<carril>|CONSTITUTION_REPO=<owner/repo>|
                         CONSTITUTION_PATH=<path-relativo>|CONSTITUTION_SHA=<sha40>
R-1.1-canonica          para parsear el locator se permite ignorar un unico U+FEFF inicial y
                         espacios ASCII exteriores al string completo; no se permite ninguna otra
                         reescritura, inferencia ni normalizacion
R-1.1-valida            valida mecanicamente protocolo, WORK_ID, CARRIL, repo owner/repo, path
                         relativo no vacio y CONSTITUTION_SHA hexadecimal de 40 caracteres
R-1.1-sin-url           CONSTITUTION_REPO usa slug owner/repo, no una URL completa; el AUDITOR
                         reconstruye la localizacion Git desde esas coordenadas
R-1.1-entrega-init      entrega al AUDITOR el locator canonico validado; no transporta la
                         constitucion completa por la GUI
R-1.1-git               la constitucion compleja vive en Git y queda identificada exclusivamente
                         por CONSTITUTION_REPO, CONSTITUTION_PATH y CONSTITUTION_SHA
R-1.1-entra-al-loop     recibe la salida del AUDITOR y entra en el loop ordinario
R-1.1-no-interpreta     no valida el contenido metodologico de la constitucion, no la completa,
                         resume ni reescribe
R-1.1-primer-turn-id    el primer sobre producido en el arranque externo lleva turn_id igual a 1
```

Nota. El locator inicial es deliberadamente distinto de `next_prompt`. Su objetivo es sobrevivir
interfaces que puedan autolinkear URLs, insertar BOM o alterar presentacion. La semantica completa
del trabajo no viaja por la GUI: viaja por Git.

Nota. Los pases internos del loop mantienen su contrato de literalidad e integridad. La tolerancia
canonica de `auditor-init/v1` no se aplica a `next_prompt`.

---

# 2. Forma de la respuesta y extracción del sobre

```text
R-2-unicidad             una salida con mas de un bloque json produce sobre invalido
R-2-sin-posterior        contenido posterior al bloque, distinto de espacios en blanco, produce
                         sobre invalido
R-2-parseo               una salida sin bloque json, o cuyo bloque no parsea como JSON, produce
                         sobre invalido
R-2-toma-el-bloque       comprobadas esas tres condiciones, el orquestador toma ese bloque
R-2-no-prosa             no interpreta la prosa anterior al bloque
R-2-no-escanear-prompt   no inspecciona next_prompt en busca de instrucciones
R-2-no-reconstruye       no reconstruye un sobre ausente ni deduce sus campos de la prosa
```

Nota. La forma que debe tener la respuesta de un actor es autoridad de
`revolutions-orchestra-ai : metodo/REVOLUTIONS.md`, §4.2: la respuesta termina con un bloque
`json`, ese bloque es el único bloque JSON de la respuesta y no existe contenido posterior.

Nota. Tomar el último bloque es correcto únicamente porque la unicidad ya fue comprobada. Un
orquestador que buscara el último sin comprobarla aceptaría una respuesta que la autoridad
declara mal formada y transportaría un sobre que nunca debió pasar. La unicidad es lo que hace
mecánica la extracción; no es una tolerancia.

Nota. Los espacios en blanco y saltos de línea que siguen al cierre del bloque no son contenido.
Cualquier otro carácter lo es.

---

# 3. Validaciones mínimas

```text
R-3-orden                las validaciones corren en el orden declarado y la primera que falla
                         detiene
R-V1                     protocol es exactamente revolutions-hop/v1
R-V2                     work_id es exactamente el del trabajo que el orquestador transporta
R-V3                     estan presentes todos los campos del contrato
R-V4                     cada campo tiene el tipo que el contrato le asigna
R-V4a                    actor tiene un valor admitido: AUDITOR o CONSTRUCTOR
R-V4b                    actor coincide exactamente con el rol de la instancia de la que se
                         obtuvo la respuesta
R-V4c                    next_actor tiene un valor admitido: AUDITOR, CONSTRUCTOR o null
R-V4d                    commit es una cadena hexadecimal de exactamente 40 caracteres
R-V5                     turn_id es el sucesor exacto del ultimo turn_id transportado
R-V6                     next_instance tiene un valor admitido: current, fresh o null
R-V7                     next_instance es null si y solo si next_actor es null
R-V8                     la combinacion de human_need, final, next_actor, next_instance y
                         next_prompt es una de las formas admitidas por el contrato
R-3-no-lee-fuentes       ninguna validacion lee el contenido de next_prompt, el mensaje de un
                         commit, el repositorio ni Git
```

Nota. `R-V2` existe porque un sobre entregado al loop equivocado es indistinguible de uno
correcto si nadie compara el identificador del trabajo.

Nota. `R-V4b` compara dos datos que el orquestador ya posee: el rol asociado a la instancia
receptora de la intervención y el campo `actor` de su respuesta. No interpreta el trabajo ni
consulta Git. `R-V4d` comprueba únicamente la forma del SHA completo que esta versión del
contrato transporta; no comprueba por esa vía qué significa el commit.

Nota. `R-V8` no enumera las formas admitidas: son autoridad del contrato. El orquestador
comprueba que la combinación recibida sea una de ellas, y ninguna otra.

---

# 4. Sobre inválido

```text
R-4-detiene              un sobre invalido detiene el transporte
R-4-reporte              el reporte identifica que validacion fallo y que se recibio
R-4-no-repara            no propone una correccion, no infiere la intencion del actor y no
                         completa el campo faltante
```

Nota. Reparar exigiría saber qué ocurrió, y eso es interpretar el trabajo.

---

# 5. `turn_id`

```text
R-5-sucesor              se exige que el turn_id recibido sea el sucesor exacto del ultimo
                         transportado
R-5-otro-detiene         cualquier otro valor detiene y se reporta
R-5-no-adivina           no se intenta adivinar el numero correcto
R-5-no-reinicio          un relevo no reinicia el contador
R-5-git-prevalece        si el contador contradice a Git, el orquestador reporta la contradiccion
                         y no la resuelve
```

Nota. Una repetición, un salto o un retroceso son una omisión, una duplicación o un pase fuera de
secuencia. El orquestador los detecta comparando dos enteros.

```text
AUDITOR      -> CONSTRUCTOR   1
CONSTRUCTOR  -> AUDITOR       2
AUDITOR      -> CONSTRUCTOR   3
```

Nota. El contador pertenece al trabajo, no al actor que lo ocupa.

---

# 6. `next_instance`

```text
R-6-current              current usa la unica instancia actualmente activa de next_actor
R-6-fresh                fresh abre una instancia nueva de next_actor
R-6-null                 null significa que no existe actor siguiente
R-6-fresh-a-current      una instancia abierta como fresh pasa a ser la current de su rol solo
                         cuando el adaptador confirma la entrega de su primer prompt
R-6-current-unico        existe como maximo un handle current elegible por rol
R-6-reemplazo            al confirmar la primera entrega a una instancia fresh, su handle
                         reemplaza atomicamente al handle current anterior de ese rol
R-6-retira-anterior      desde ese reemplazo el handle anterior queda retirado y no puede volver
                         a ser seleccionado por ningun pase current posterior
R-6-cierre-opcional      cerrar fisicamente la ventana o sesion anterior es opcional; impedir su
                         reutilizacion es obligatorio
R-6-solo-next-instance   la eleccion de instancia lee next_instance y ningun otro campo
R-6-handle-efimero       el handle de instancia puede contener la metadata tecnica necesaria
                         para reencontrar exactamente esa instancia; no es estado autoritativo
                         del trabajo y no se publica en Git
```

Nota. Los pases posteriores hacia una instancia abierta como `fresh` usan `current` hasta que
otro sobre indique `fresh`.

## 6.1. `current` perdido

```text
R-6.1-detiene            si no puede satisfacerse literalmente current porque se perdio esa
                         instancia, el orquestador detiene y reporta la imposibilidad
R-6.1-no-degrada         nunca convierte current en fresh
R-6.1-no-inventa         no inventa continuidad conversacional
```

Nota. Esa transformación fabricaría un actor sin la continuidad que el emisor del sobre decidió,
y quien decide un relevo es un actor del método, no el transporte.

## 6.2. Adaptadores de runtime

```text
R-6.2-frontera           cada runtime se accede mediante un adaptador mecanico; las reglas del
                         orquestador no dependen de una interfaz concreta de ChatGPT, Claude,
                         terminal, navegador, API ni proveedor
R-6.2-fresh              ante fresh el adaptador debe abrir una instancia nueva y devolver un
                         handle que permita identificarla como la instancia abierta
R-6.2-fresh-distinta     el handle obtenido para fresh debe ser distinto de cualquier handle
                         existente del mismo rol; reutilizar la instancia current no satisface
                         fresh aunque el prompt entregado sea correcto
R-6.2-fresh-preflight    antes de entregar el primer prompt a fresh, el adaptador debe poder
                         comprobar mecanicamente que la instancia destino es distinta de la
                         current anterior; si no puede, detiene y reporta
R-6.2-current            el adaptador puede reencontrar una instancia current unicamente cuando
                         puede identificar que es exactamente la misma instancia
R-6.2-envia              el adaptador recibe un string ya resuelto por el orquestador y lo entrega
                         sin resumirlo, reescribirlo, completarlo ni regenerarlo
R-6.2-verifica-destino   si la interfaz de destino permite leer de vuelta mecanicamente el valor
                         efectivamente preparado antes del envio, el adaptador compara longitud
                         UTF-8 y SHA-256 contra el string fuente
R-6.2-sin-readback       la ausencia de una primitiva de readback no obliga por si sola a detener:
                         el adaptador puede continuar usando su primitiva normal de insercion de
                         texto siempre que no observe transformacion, ambiguedad ni artefactos
                         laterales y no invente mecanismos auxiliares como file:// para intentar
                         demostrar igualdad
R-6.2-anomalia           si durante la insercion se observa una transformacion, adjunto, archivo,
                         upload, perdida de caracteres, ambiguedad o cualquier evidencia de que el
                         texto preparado no coincide con el string fuente, no envia y reporta
R-6.2-sin-artefactos     la insercion no puede crear adjuntos, archivos, uploads ni otros artefactos
                         laterales que no formen parte del string fuente; si aparecen, no envia
R-6.2-espera             el adaptador no entrega una salida al extractor mientras el actor siga
                         generando una respuesta
R-6.2-recibe             cuando la respuesta termino, el adaptador devuelve la salida completa
                         que corresponde a esa intervencion
R-6.2-reanuda            un mecanismo nativo de reanudacion del runtime solo puede aplicarse
                         cuando conserva la misma instancia current
R-6.2-no-inventa         si el adaptador no puede demostrar continuidad con la misma instancia
                         current, reporta perdida de instancia y se aplica 6.1
```

Nota. En un runtime conversacional, una instancia `fresh` significa una conversación o sesión
nueva, no una nueva intervención dentro de la conversación current. En un runtime de terminal,
significa una sesión nueva identificable por un handle distinto. La interfaz concreta puede variar;
la propiedad exigida es que la identidad de instancia sea nueva y mecánicamente distinguible.

Nota. El handle es opaco para las reglas. Una implementación concreta puede necesitar, por
ejemplo, identificadores de navegador, pestaña, conversación, proceso, sesión, terminal o
directorio de trabajo. Esos datos sirven para recuperar una instancia técnica; no describen el
estado metodológico del trabajo, no forman parte de `revolutions-hop/v1` y no se publican en Git.

Nota. Un comando como `resume` puede ser correcto para un runtime concreto, pero no es una regla
universal. La regla universal es conservar la misma instancia cuando `next_instance` exige
`current`.

---

# 7. Loop ordinario

```text
R-7-necesidad            human_need distinto de null detiene y se muestra la necesidad
R-7-final                final en true detiene y se muestra el cierre
R-7-unit                 unit distinto de null se muestra como transicion
R-7-orden                esas tres comprobaciones ocurren en ese orden, antes de entregar
R-7-entrega-literal      next_prompt se entrega literalmente a la instancia resuelta
R-7-ruta-directa         despues del parseo y la validacion, next_prompt se conserva como valor
                         de runtime y ese mismo valor se pasa directamente al adaptador de destino
R-7-no-regenera          ningun modelo, humano ni etapa de generacion de texto vuelve a emitir,
                         transcribir o reconstruir next_prompt para efectuar el salto
R-7-integridad           antes de entregar, el orquestador calcula longitud UTF-8 y SHA-256 sobre
                         next_prompt y sobre el valor preparado para envio; cualquier diferencia
                         detiene antes de la entrega
R-7-integridad-efimera   longitud y SHA-256 son comprobaciones transitorias del mismo string; no
                         se agregan al sobre, no se publican en Git y no crean estado durable
R-7-unit-no-decide       mostrar unit no cierra ni abre una unidad
R-7-no-consulta-git      el orquestador no consulta Git para reconstruir, completar o mejorar
                         next_prompt
```

## 7.1. Reanudación de `human_need`

Cuando un sobre válido llega con `human_need != null`, el ORQUESTADOR conserva ese sobre
íntegro como parte de su estado efímero y muestra la necesidad al HUMANO.

Si después recibe una resolución humana para esa necesidad, no solicita al HUMANO un
`turn_id`, no lo deriva de Git y no inventa un `next_prompt`.

Toma mecánicamente del sobre detenido:

```text
resume_actor = human_need.resume_actor
incoming_turn_id = turn_id
```

y entrega a la misma instancia `current` del `resume_actor` dos entradas separadas:

```text
RESUME_CONTEXT:
INCOMING_TURN_ID=<turn_id exacto del sobre detenido>

HUMAN_RESOLUTION_LITERAL:
<resolución humana recibida sin reescribir>
```

La forma concreta del adaptador puede variar, pero debe conservar inequívocamente la separación
entre metadata de reanudación y resolución humana literal.

```text
R-7.1-sobre              la reanudacion usa exactamente el sobre detenido que origino human_need
R-7.1-turn               INCOMING_TURN_ID es copia exacta de turn_id de ese sobre
R-7.1-no-humano-turn     el HUMANO no proporciona ni corrige INCOMING_TURN_ID
R-7.1-no-incrementa      el ORQUESTADOR no incrementa turn_id; el resume_actor emite el sucesor
R-7.1-no-git             no consulta Git para reconstruir metadata de reanudacion
R-7.1-literal            la resolucion humana se entrega literalmente y separada de la metadata
R-7.1-current            resume_actor debe poder satisfacerse como la misma instancia current;
                         si se perdio, se aplica 6.1 y se detiene
```

Nota. El ciclo completo:

```text
recibir salida del actor
        ↓
extraer el bloque json comprobando su forma
        ↓
validar
        ↓
si human_need != null   -> detener y mostrar la necesidad
        ↓
si final == true        -> detener y mostrar el cierre
        ↓
si unit != null         -> mostrar la transición
        ↓
leer next_actor y next_instance
        ↓
tomar next_prompt como valor de runtime
        ↓
comprobar integridad del valor preparado para envio
        ↓
pasar ese mismo valor al adaptador y entregarlo
        ↓
repetir
```

Nota. `unit` es una notificación del actor competente. Git lo consultan los actores, según el
método.

Nota. La ruta directa evita que la literalidad dependa de que un modelo copie bien un prompt
largo. El control de hash y longitud protege esa propiedad dentro del runtime; como ambos valores
son derivables del propio `next_prompt`, incorporarlos al contrato sólo duplicaría información.

---

# 8. Lo que el orquestador no hace

```text
R-8-no-decide            no decide si una auditoria es correcta, si el constructor debe corregir,
                         si una unidad termino, si un relevo corresponde, si un actor dejo
                         material suficiente, que significa una evidencia, si existe una
                         necesidad humana, que repositorio modificar, arquitectura, diseno,
                         permisos, alcance, riesgos ni prioridades
R-8-no-elige-modelo      no decide que modelo de IA se utiliza
R-8-no-cadencia          no cuenta entregas ni intervenciones, no deriva ninguna cadencia de
                         relevo y no consulta Git para esa decision
R-8-no-toca-prompt       no resume, no reescribe, no mejora y no agrega contexto a next_prompt
R-8-no-copia-git         no copia dentro de next_prompt resultados que deberian obtenerse desde
                         Git
R-8-no-extiende          no agrega al contrato next_model, next_runtime, seleccion automatica de
                         modelo, cambio de modelo por unidad ni reglas de costo
```

Nota. La política de relevo es autoridad de `metodo-manifiestos-ai : METODO-MANIFIESTOS.md` y la
aplican los actores dentro de sus autoridades. El orquestador sólo ejecuta `next_instance`.

---

# 9. `DETENER`

```text
R-9-no-necesidad         DETENER no se convierte en human_need
R-9-no-git               DETENER no modifica Git ni el manifiesto
R-9-no-durable           DETENER no crea estado durable del trabajo
R-9-flag-efimero         puede existir un indicador efimero de runtime equivalente a
                         stop_requested
```

Nota. Es una orden de control del orquestador emitida por el humano.

## 9.1. Frontera segura

```text
R-9.1-frontera           con un sobre valido del CONSTRUCTOR hacia el AUDITOR todavia no
                         entregado, DETENER pausa en ese punto exacto: conserva el sobre y no lo
                         entrega al AUDITOR
R-9.1-constructor-1      DETENER durante el CONSTRUCTOR: se permite que termine
R-9.1-constructor-2      DETENER durante el CONSTRUCTOR: se recibe y valida su sobre
R-9.1-constructor-3      DETENER durante el CONSTRUCTOR: se detiene antes de entregarlo al AUDITOR
R-9.1-auditor-1          DETENER durante el AUDITOR: se permite que termine
R-9.1-auditor-2          DETENER durante el AUDITOR: si su resultado normal continua hacia el
                         CONSTRUCTOR, se entrega al CONSTRUCTOR correspondiente
R-9.1-auditor-3          DETENER durante el AUDITOR: se permite que el CONSTRUCTOR termine, se
                         recibe y valida su sobre, y se detiene antes de entregarlo al AUDITOR
R-9.1-natural            si el AUDITOR termina con human_need distinto de null o con final en
                         true, esa detencion prevalece y no se fuerza una intervencion adicional
R-9.1-pendiente          DETENER con un sobre CONSTRUCTOR hacia AUDITOR ya pendiente detiene
                         inmediatamente, sin entregarlo
```

Nota. Es la frontera donde el trabajo material ya está preservado en Git y ninguna intervención
queda cortada por la mitad.

## 9.2. `CONTINUAR`

```text
R-9.2-preserva           la detencion preserva integramente el ultimo sobre recibido
R-9.2-literal            CONTINUAR entrega el next_prompt pendiente exactamente como fue emitido,
                         a la instancia que indican su next_actor y su next_instance
R-9.2-no-reconstruye     no reconstruye el pase, no lo actualiza y no lo revalida contra un
                         estado nuevo
```

## 9.3. Directivas humanas durante la pausa

```text
R-9.3-no-aplica-relevo   el orquestador no aplica el relevo
R-9.3-canal-separado     entrega al actor competente dos entradas diferenciadas:
                         ACTOR_PROMPT_LITERAL y HUMAN_DIRECTIVE_LITERAL
R-9.3-no-modifica        la directiva no modifica, concatena, reinterpreta ni falsifica el
                         next_prompt emitido
R-9.3-no-en-json         en ningun momento del transporte la directiva aparece dentro de un
                         campo del sobre: el sobre observado despues de emitirla es identico al
                         recibido
R-9.3-no-saltea          un relevo nunca saltea una entrega que todavia no fue auditada
```

Nota. Durante una pausa el humano puede emitir directivas como `RELEVAR CONSTRUCTOR` o
`RELEVAR AUDITOR`. La decisión pasa por REVOLUTIONS y la toma el actor con autoridad. Una interfaz
equivalente inequívoca al par de entradas diferenciadas es admisible.

---

# 10. Estado efímero

```text
R-10-estado-admitido     el estado efimero del orquestador es exactamente: handle de instancia
                         current de AUDITOR, handle de instancia current de CONSTRUCTOR, ultimo
                         turn_id transportado, ultimo sobre pendiente o sobre detenido por
                         human_need, y stop_requested
R-10-handle-opaco        cada handle current puede encapsular solo la metadata tecnica necesaria
                         para recuperar exactamente esa instancia mediante su adaptador
R-10-persistencia-local  el estado efimero admitido puede serializarse fuera de Git para sobrevivir
                         un reinicio del orquestador; esa persistencia sigue siendo runtime y no
                         autoridad sobre el trabajo
R-10-no-paralelo         no se crean como fuentes de verdad current_unit, approved_work_sha,
                         latest_audit, relay_pending, work_status, constructor_count ni
                         auditor_count
```

Nota. Cada uno de esos estados prohibidos sería una segunda fuente sobre algo que el protocolo de
derivación reconstruye desde Git. En cambio, un handle técnico conserva una propiedad que Git no
puede derivar: cómo reencontrar una conversación o proceso concreto. Persistir ese handle permite
recuperar el runtime; no permite inferir aprobación, unidad, vigencia, veredictos ni ningún otro
estado metodológico.

## 10.1. Fallas y reinicios

```text
R-10.1-fail-closed       ante una falla, un reinicio o cualquier situacion en la que no pueda
                         cumplirse literalmente el salto que el sobre indica, el orquestador se
                         detiene, reporta y espera resolucion
R-10.1-no-degrada        no degrada el salto a una alternativa mas comoda y no continua con una
                         suposicion
R-10.1-entrega-ambigua   si despues de una falla no puede determinarse mecanicamente si un prompt
                         pendiente ya fue entregado, el orquestador se detiene y reporta la
                         ambiguedad
R-10.1-no-replay         no repite automaticamente una entrega cuya ejecucion pueda haber comenzado
```

Nota. Fail-closed significa que la conducta ante lo imprevisto es detenerse, no elegir la
alternativa más cómoda. Un orquestador que ante una falla continúa con una suposición es más
difícil de diagnosticar que uno que se detiene, porque su estado deja de corresponder a ninguna
decisión que un actor del método haya tomado.

Nota. Un identificador adicional de entrega no produce por sí solo semántica de exactamente una
vez cuando el runtime receptor no participa de un protocolo de idempotencia. Ante un resultado
ambiguo, detenerse preserva mejor la secuencia que repetir a ciegas.

---

# 11. Secretos

```text
R-11-no-necesita         el orquestador no necesita valores secretos para transportar
R-11-literal             una referencia segura a una credencial se transporta literalmente, como
                         cualquier otro texto
R-11-no-resuelve         no la resuelve, no la expande y no la convierte en el valor
R-11-no-logs             no registra valores secretos en logs
```

Nota. Un sobre nunca transporta un secreto.

---

# 12. Relación con los demás documentos

```text
revolutions-orchestra-ai   gobierna la ejecución y el contrato revolutions-hop/v1
metodo-manifiestos-ai      produce el paquete de constitución y la política de relevo
manifiestos-trabajo-ai     conserva la intención humana aprobada y su identidad exacta
```

Este documento describe únicamente el transporte. Cuando necesita una regla ajena la referencia
por repositorio, path y contrato, sin reproducir su texto y sin congelar un SHA de esos
documentos.

La cadena completa del sistema es autoridad de `metodo-manifiestos-ai : METODO-MANIFIESTOS.md`.

---

# 13. Superficie normativa de este documento

Esta sección declara dónde vive lo normativo, para que la cobertura de una verificación pueda
medirse contra el documento y no contra una lista paralela.

## Obligaciones

Una obligación es una línea etiquetada dentro del bloque de obligaciones de una sección mecánica:

```text
<identificador><espacios><enunciado>
```

El identificador empieza con `R-`. Una línea que empieza con cuatro o más espacios continúa el
enunciado de la obligación anterior.

El conjunto de obligaciones de este documento es el conjunto de esas líneas. No existe una
segunda lista, ni aquí ni en ninguna implementación. Una obligación no puede quedar fuera del
conjunto sin dejar de estar enunciada.

## Forma de una sección mecánica

Una sección mecánica tiene exactamente esta forma:

```text
1  el encabezado de la seccion
2  el bloque de obligaciones, primer bloque cercado de la seccion
3  opcionalmente, a partir del primer marcador Nota., contenido libre hasta el proximo encabezado
```

Cualquier contenido no vacío entre el encabezado y el bloque de obligaciones, o entre el bloque de
obligaciones y el primer marcador `Nota.`, está fuera de esa forma y es un defecto del documento.

## Notas

El contenido que sigue a un marcador `Nota.` explica, ilustra, cita autoridades y da razones. No
enuncia obligaciones y no tiene fuerza normativa. Una conducta que este documento pretenda exigir
y que sólo aparezca en una nota, no está exigida por este documento.

Esa declaración es lo que hace completo al conjunto de obligaciones: no depende de que alguien
haya inspeccionado bien cada sección, sino de que fuera del bloque etiquetado no hay lugar donde
una obligación pueda existir.

## Secciones no mecánicas

```text
SECCIONES_NO_MECANICAS   12  13
```

Toda otra sección numerada es mecánica y contiene su bloque de obligaciones.

El encabezado del documento y las secciones sin número —`Qué gobierna este documento y qué no` y
`Principio`— delimitan alcance y no pertenecen a la superficie normativa.
