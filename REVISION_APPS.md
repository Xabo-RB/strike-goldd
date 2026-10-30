# Revisión de las apps MATLAB de STRIKE-GOLDD

Revisión de las cinco apps de App Designer. **No se ha modificado ningún archivo del proyecto**: este documento es lo único añadido.

## Resumen

Las apps funcionan en el caso habitual (por ejemplo, FISPO con `C2M`), pero tienen tres problemas de fondo:

1. **Pueden producir un resultado incorrecto sin avisar.** Es lo más grave en una herramienta de análisis: una salida escrita como `x1 +x2` se convierte en dos salidas, y un `[0 10]` se lee como `[0 1 0]`.
2. **Hay funciones rotas para entradas válidas.** *Save options file* falla siempre. Run falla con 28 modelos sin `u` y con 82 modelos de subcarpetas. La app de controlabilidad termina el análisis y pierde el resultado si no existe `results/`.
3. **La causa de muchos fallos es la arquitectura.** Las apps generan archivos `.m`, los ejecutan y cambian de carpeta con `cd`. Además, el análisis de texto está duplicado en muchos sitios.

El rendimiento de la interfaz **no** es un problema importante. La mejora con más retorno es de mantenimiento: tres métodos de `STRIKE_GOLDD_APP` suman 1544 líneas, el 55 % del archivo, y podrían reducirse mucho.

| Bloque | Elementos |
|---|---:|
| Resultados incorrectos sin aviso (sección 2) | 7 |
| Funciones rotas o errores (sección 3) | 23 |
| Mejoras de arquitectura (sección 4) | 6 |

## 1. Cómo se ha comprobado

- Se extrajo el código de cada `.mlapp` (`matlab/document.xml`). **Los números de línea (L) son los de ese código** y deberían coincidir con la vista de código de App Designer.
- Las apps se ejecutaron de verdad en MATLAB R2026b (modo batch, Windows), siempre sobre una **copia temporal**, pulsando sus callbacks con distintas entradas.
- Para probar muchas entradas rápido, una de las copias usaba un `STRIKE_GOLDD` sustituto que solo capturaba las opciones recibidas. El análisis real se ejecutó en otra copia.
- En las tablas: **R** = reproducido ejecutando; **L** = deducido leyendo el código.

Límites de la revisión:

- No se ha probado en Linux ni macOS, ni en versiones anteriores de MATLAB.
- No se ha comprobado el aspecto visual, porque las apps corrieron sin pantalla.
- Solo se ha revisado el resto del toolbox en lo que interactúa con las apps.

Abreviaturas: **SG** = `STRIKE_GOLDD_APP`, **NL** = `NLControllability_APP`, **NM** = `newmodel`, **PP** = `prevPars`, **MA** = `MainApp`.

## 2. Prioridad 1: resultados incorrectos sin aviso

Ninguno de estos casos muestra un error, y por eso son los más peligrosos.

| ID | Dónde | Qué pasa | Comp. | Arreglo |
|---|---|---|---|---|
| 1.1 | NM L167 | La salida `x1 +x2` crea **dos** salidas (`[x1; x2]`), porque se escribe tal cual dentro de `[...]`. Las ecuaciones (L223) se escriben igual, así que el riesgo es el mismo (L). | R | Parche inmediato: escribir cada expresión entre paréntesis, `[(x1 +x2), (x3)]` (probado). Definitivo: sección 4.4. |
| 1.2 | NM L219-225 | Acepta un número de ecuaciones distinto del de estados. Se probó 1 ecuación con 3 estados y guardó el modelo. | R | Validar `numel(f) == numel(x)`. |
| 1.3 | NM L249 | `isnan(app.ics{i})` se aplica a texto, así que nunca detecta `NaN`. Con `[NaN,0,0]` guarda `known_ics = [1 1 1]`, y `Lie_Symmetry` trata esa condición inicial como conocida ([Lie_Symmetry.m:61](STRIKE-GOLDD/functions/Lie_Symmetry.m#L61)). | R | Convertir a número antes: `known_ics = ~isnan(valores)`. |
| 1.4 | SG L505-513 | La matriz multiexperimento se lee **dígito a dígito**: `[0 10]` da `[0 1 0]` y `[inf 1]` da `[0 NaN NaN 1]`. Con líneas en blanco al principio borra la fila equivocada: el valor `[0 1]` acaba como `0`. | R | Parser único (sección 4.1). |
| 1.5 | SG L152, L975 | La casilla **Calculate rank numerically** no hace nada: `opts.numeric` no llega al archivo de opciones y ningún algoritmo lo usa. La opción figura como obsoleta en [options.m:70](STRIKE-GOLDD/options.m#L70). | R | Quitar la casilla (decisión 2). |
| 1.6 | SG L172, L1319-1336 | Las opciones ocultas se siguen aplicando. Con el interruptor multiexperimento activado y SIM seleccionado, la app dice que está deshabilitado, pero llega `opts.multiexp = 1` y [SIM.m:14](STRIKE-GOLDD/functions/SIM.m#L14) lo usa. Lo mismo ocurre con *Known parameters*, oculto en SIM (L). | R | Enviar valores neutros en las opciones que no apliquen al algoritmo. |
| 1.7 | NM L272-274 | **Sobrescribe un modelo existente sin preguntar**: al crear uno llamado `C2M` se reemplazó `C2M.mat`. Con el nombre vacío guarda `models.mat` en la raíz de STRIKE-GOLDD, y con espacios crea `mi modelo.mat`. | R | Validar el nombre y pedir confirmación con `uiconfirm`. |

## 3. Prioridad 2: funciones rotas y errores

### Identificabilidad (`STRIKE_GOLDD_APP`)

| ID | Dónde | Qué pasa | Comp. | Arreglo |
|---|---|---|---|---|
| 2.1 | L1964-1976 | **Save options file falla siempre**: `app.icsMulti` está vacío y `erase` da error. Hay cinco defectos más detrás (ver debajo). | R | Reescribir con la lógica común de Run. |
| 2.2 | L387 | **28 modelos no se pueden ejecutar**: todos los de `1D_BIG_model_variants`. No tienen `u`, y `VectorU = u` falla aunque el motor sí admite ese caso. | R | Comprobar con `whos('-file', ...)`. |
| 2.3 | L385 | **82 modelos aparecen en la lista pero no cargan** en una sesión nueva: 19 de `Cancer` y 63 de `NLcontr_models`. `install` no añade esas carpetas y el `load` ocurre antes de que `STRIKE_GOLDD` las añada. | R | `addpath(genpath(...))` al arrancar, o usar la ruta completa. |
| 2.4 | L139 | Run con *Select algorithm*: genera `opts.algorithm = ;`, da un error de sintaxis confuso y deja archivos temporales. | R | Validar el algoritmo como ya se valida el modelo (L372). |
| 2.5 | L506, L519 | Dejar vacío el campo multiexperimento, o pulsar Intro dos veces en él, rompe Run **aunque el multiexperimento esté desactivado**. | R | Parser único, usado solo si la opción aplica. |
| 2.6 | L844, L851 | Los espacios cuelan `NaN`: `[2  3]` da `[2 NaN 3]`, `[2 3 ]` da `[2 3 NaN]` y ` 2` da `[NaN 2]`. Este último rompe FISPO con un error interno de `sym`. | R | Parser único (sección 4.1). |
| 2.7 | L535-561, L607-637 | Falso positivo: con multiexperimento activo y 3 experimentos, exige corregir el campo de derivadas aunque **no esté marcado como manual** y no se vaya a usar. | R | Validar solo si `app.knowMulti` o `app.uknowMulti` están activos. |
| 2.8 | L608 | SIM_PDE con el interruptor multiexperimento activado: `ExisteW` no existe y falla. | R | Se resuelve con 1.6. |
| 2.9 | L1061-1075 | En **FISPO** se muestra la etiqueta del campo de derivadas de `w` en cada experimento, pero **no el campo**, que sí aparece en Prob_Obs_Test y ORC-DF. Falta `knowderivTextArea_2.Visible = 'on'`. | R | Añadir esa línea, o aplicar la sección 4.2. |
| 2.10 | PP L47-63; [STRIKE_GOLDD.m:109](STRIKE-GOLDD/STRIKE_GOLDD.m#L109) | Un parámetro conocido que no existe en el modelo provoca `p(0) = []` y un error críptico; igual en [SIM.m:40](STRIKE-GOLDD/functions/SIM.m#L40). La validación tampoco detecta `k1,k2` en una línea, que falla más tarde. | R | Validar con `isvarname` y contra `p` antes de ejecutar. |
| 2.11 | L2209, L2214 | El refresco de la lista no es recursivo y sobrescribe por posición: al añadir un modelo, otro **desaparece** de la lista (`1D_out_BG_par_agcs` en la prueba). | R | Función única de listado (sección 4.5). |

**Defectos adicionales de Save options file (2.1)**, ocultos porque el error ocurre antes:

- L1859: si no se tocó el desplegable, el nombre del modelo está vacío.
- L1867-1870: **la numeración de algoritmos es incorrecta**. Lie_Symmetry se guarda como 4 (que es SIM), AutoRepar como 5 (que es Lie_Symmetry), y faltan SIM y SIM_PDE.
- L2122: declara cinco salidas, incluida `submodels`, que nunca se asigna. `STRIKE_GOLDD` espera cuatro.
- L2135: depende de `prevIdPars()`, que se vacía cada vez que arranca la app, así que el archivo guardado pierde los parámetros conocidos.

### Controlabilidad (`NLControllability_APP`)

| ID | Dónde | Qué pasa | Comp. | Arreglo |
|---|---|---|---|---|
| 2.12 | [ctrl_analysis_MAIN.m:146](STRIKE-GOLDD/functions/ctrl_analysis_MAIN.m#L146) | **Sin `results/` se pierde el resultado**: el análisis termina y falla al guardar. Esa carpeta no está en el repositorio y esta app nunca ejecuta `install`. | R | Crear la carpeta al arrancar la app. |
| 2.13 | L64 | Usa `delete(app.SG)`, pero esa propiedad no existe aquí (es `app.NLCont`). En vez de cerrarse con el aviso previsto, da otro error. | R | Corregir el nombre. |
| 2.14 | L116-123 | RUN sin elegir modelo intenta cargar `"Models in the folder"` y la lámpara se queda en rojo. | R | Copiar la validación de SG L372. |
| 2.15 | L77 | **Solo lista 12 de los 75 modelos de `NLcontr_models`**, precisamente los pensados para esta app. El listado no es recursivo. | R | Función única de listado (sección 4.5). |

### Creación de modelos (`newmodel`)

| ID | Dónde | Qué pasa | Comp. | Arreglo |
|---|---|---|---|---|
| 2.16 | L89, L120, L279 | Marcar *Symbolic initial conditions* con condiciones solo numéricas falla, **deja la carpeta actual en `functions\aux_APP` y el archivo abierto**. A partir de ahí los `cd` relativos dejan de funcionar. | R | No usar `cd`; escribir el modelo sin generar `.m` (sección 4.3). |
| 2.17 | L112 | Con la casilla simbólica, `1e-3` se trata como variable porque contiene una letra, y `syms` falla. | R | Decidir con `str2double`, no buscando letras. |

### Transversales

| ID | Dónde | Qué pasa | Comp. | Arreglo |
|---|---|---|---|---|
| 2.18 | SG y NL, Run | Cualquier error deja la lámpara en **rojo para siempre**, y en SG también deja `options_aux.m` y `current_options.m`. El botón sigue activo durante el análisis. | R | `try`/`catch` y `onCleanup` (sección 4.3). |
| 2.19 | MA L71, L84 | Pulsar dos veces un botón abre dos ventanas. La primera se pierde: queda huérfana al cerrar MainApp, y ambas comparten archivos temporales. | R | Si la ventana sigue abierta, enfocarla con `figure(...)`. |
| 2.20 | SG L254, L260, L1797; PP L72; NM L89, L272; SG L240, NL L56 | **Rutas solo para Windows** (`'functions\aux_APP'`, `'\'`). Fuera de Windows, SG no arrancaría por el `cd` de L260, y NL tampoco: su comprobación con `'\STRIKE-GOLDD'` falla siempre y acaba en 2.13. El commit `175877f` indica que Linux importa. | R (`strcmp`), L | `fullfile` y `filesep`. |
| 2.21 | MA L40, NL L55, SG L239 | `pwd(end-12:end)` da error si la ruta tiene menos de 13 caracteres. Además, todas las apps dependen de la carpeta actual. | R | Partir de `which(class(app))`, que devuelve la ruta del `.mlapp` (probado con SG; desde `functions/` hay que subir un nivel). |
| 2.22 | SG L1039 | En FISPO genera un aviso, porque `STRIKE_GOLDD` ya borró `current_options.m`. | R | Desaparece con la sección 4.3. |
| 2.23 | SG L275 | La lista muestra 12 nombres repetidos, indistinguibles entre sí, y al elegir cualquiera se carga el primero según el path. No altera resultados: 9 son idénticos y 3 difieren en detalles (`HIV` y `HIV_known_u` usan `delta` o `delta1`; `Fujita` tiene `p` en fila o en columna). | R | Dejar una sola copia (decisión 5). |

## 4. Mejoras de arquitectura

Las funciones `parseNumericMatrix`, `parseSymList`, `parseExprList` y `parseExprLines` se han ejecutado tal como aparecen aquí, en R2026b. Los fragmentos de 4.2, 4.3 y 4.4 son esqueletos: se han probado sus piezas clave, pero no completos dentro de las apps.

Las funciones auxiliares conviene ponerlas como `.m` en `functions/aux_APP`, fuera del `.mlapp`, para poder probarlas por separado y compartirlas entre apps.

### 4.1. Un único parser de números

Hoy hay parsers repetidos para la misma tarea en Run y en Save (por ejemplo, L496-524, L574-600, L794-846, L877-930, L1898-1926, L2027-2065), con comportamientos distintos entre sí. Esto los sustituye a todos. Se probó con 16 casos, incluidos los problemáticos de las secciones 2 y 3: los 14 válidos se interpretan bien y los 2 inválidos se rechazan con un mensaje claro.

```matlab
function M = parseNumericMatrix(lines)
    lines = strtrim(string(lines));
    lines = lines(strlength(lines) > 0);
    if isempty(lines)
        M = [];
        return
    end
    rows = cell(numel(lines),1);
    for k = 1:numel(lines)
        rows{k} = str2double(split(strtrim(regexprep(lines(k),'[\[\],]',' ')))).';
    end
    if numel(unique(cellfun(@numel,rows))) > 1
        error('All rows must have the same number of elements.');
    end
    M = vertcat(rows{:});
    if any(isnan(M),'all')
        error('Only numbers or inf are allowed.');
    end
end
```

Junto con 4.4 y 4.5, elimina prácticamente todos los avisos `AGROW` de Code Analyzer.

### 4.2. Visibilidad por grupos

`AlgorithmDropDownValueChanged` tiene 518 líneas, y `STRIKE_GOLDD_APP` acumula 448 líneas con `.Visible`. Son el origen de 2.9. Basta con definir grupos una vez en `startupFcn` y decidir por algoritmo:

```matlab
% ItemsData hace que Value sea directamente el número del algoritmo
app.AlgorithmDropDown.Items     = {'Select algorithm','FISPO','Prob_Obs_Test','ORC DF','SIM','Lie_Symmetry','AutoRepar','SIM_PDE'};
app.AlgorithmDropDown.ItemsData = 0:7;

function updateVisibleOptions(app)
    alg = app.AlgorithmDropDown.Value;
    setGroup(app.sioControls,   app.TheseoptionsaredisabledforthisalgorithmLabel_5, ismember(alg,[1 2 3 5 6]));
    setGroup(app.multiControls, app.TheseoptionsaredisabledforthisalgorithmLabel_2, ismember(alg,[1 2 3]));
    setGroup(app.orcControls,   app.TheseoptionsaredisabledforthisalgorithmLabel_3, alg == 3);
    setGroup(app.lieControls,   app.TheseoptionsaredisabledforthisalgorithmLabel_4, ismember(alg,[5 6]));
end

function setGroup(controls, notice, visible)
    set(controls,'Visible',matlab.lang.OnOffSwitchState(visible));
    notice.Visible = ~visible;
end
```

En App Designer, `updateVisibleOptions` y `setGroup` serían métodos privados, y `setGroup` recibiría también `app`. Falta un detalle: Prob_Obs_Test oculta además `MaxLieTimeEditField` y `SymbNumCheckBox`, que irían en un grupo propio.

`ItemsData` elimina además las tres tablas de `if/elseif` que convierten nombre en número (L399-413, L1046-1559, L1861-1871). Que estén repetidas es lo que causó el error de numeración de 2.1.

### 4.3. Dejar de generar archivos `.m`

Es la mejora de mayor alcance. Hoy el flujo es: construir texto, escribir `options_aux.m`, copiarlo a `current_options.m`, ejecutarlo y borrarlo. Lo curioso es que `RunButtonPushed` **ya construye la estructura `opts` completa (L973-1018) y luego la descarta**, porque `overwrite_options` no la recibe.

Pasarla directamente elimina `options_aux.m`, `current_options.m`, `prevIdPars.m`, `CreateNewModel.m`, los `cd`, el `clear`/`rehash` y varios errores de esta lista.

```matlab
function RunButtonPushed(app, event)
    app.RunButton.Enable = 'off';
    app.Lamp.Color = 'red';
    dlg = uiprogressdlg(app.SG,'Title','STRIKE-GOLDD', ...
        'Message','Running analysis...','Indeterminate','on');
    restore = onCleanup(@() app.finishRun(dlg));    %#ok<NASGU>
    try
        cfg = app.collectOptions();                 % valida y devuelve una estructura
        STRIKE_GOLDD(cfg);
    catch ME
        uialert(app.SG,ME.message,'Analysis failed');
    end
end
```

Esto **requiere tocar dos archivos fuera de las apps**:

- [STRIKE_GOLDD.m:26](STRIKE-GOLDD/STRIKE_GOLDD.m#L26): admitir una estructura en el caso de un argumento. No sirve reutilizar el caso de cuatro argumentos, porque está reservado para `AutoRepar` y recalcula las rutas.
- [AutoRepar.m:36](STRIKE-GOLDD/functions/AutoRepar.m#L36): lee `current_options` directamente, así que debe recibir la configuración de otra forma.

`finishRun` debe comprobar `isvalid(app)`, por si se cierra la app durante el análisis.

### 4.4. Crear modelos sin generar código

```matlab
x = parseSymList(app.StatesEditField.Value);
p = parseSymList(app.ParametersEditField.Value);
f = parseExprLines(app.eqns.Value);
h = parseExprList(app.OutputEditField.Value);

if numel(f) ~= numel(x)
    error('There must be one equation per state.');
end
noDeclarados = setdiff(symvar([f(:); h(:)]), [x; p; u(:); w(:)]);
if ~isempty(noDeclarados)
    error('Undeclared symbols: %s', char(noDeclarados));
end
save(fullfile(modelsDir,nombre + ".mat"),'x','p','u','w','h','f','ics','known_ics');
```

Funciones auxiliares que usa:

```matlab
function v = parseSymList(txt)
    names = split(strtrim(regexprep(string(txt),'[\[\],;]',' ')));
    names = names(strlength(names) > 0);
    if ~all(arrayfun(@isvarname,names))
        error('Invalid symbol name.');
    end
    v = sym(names);
end

function v = parseExprList(txt)
    parts = strtrim(split(string(txt),","));
    v = str2sym(parts(strlength(parts) > 0));
end

function v = parseExprLines(lines)
    parts = strtrim(erase(string(lines),";"));
    v = str2sym(parts(strlength(parts) > 0));
end
```

Corrige 1.1, 1.2 y 2.16; 1.3 y 2.17 se resuelven leyendo las condiciones iniciales con el mismo criterio. La comprobación de símbolos además detecta erratas en las ecuaciones. Dos cuidados:

- `str2sym` acepta bien `gamma`, `beta` o `lambda`, pero convierte **`pi` e `i` en constantes sin avisar**. Conviene rechazar esos nombres como símbolos. Ningún modelo actual los usa como variable.
- Separar por comas rompería expresiones como `atan2(x1,x2)`. Es mejor pedir **una salida por línea**, como en las ecuaciones.

### 4.5. Un único listado de modelos

Hay tres versiones distintas (SG L275, SG L2209, NL L77) y de ahí vienen 2.11 y 2.15. Conviene una sola función compartida en `functions/aux_APP`, que además:

- rellene la lista al arrancar, en lugar de en `DropDownOpeningFcn`;
- oculte los modelos generados por el propio toolbox (38 `affine_*` y 7 `*_NExp`).

### 4.6. Quitar los callbacks que solo copian valores

Hay 44 callbacks `ValueChanged` (27 en SG, 7 en NL, 9 en NM, 1 en PP). Prácticamente todos copian un valor a una propiedad que Run vuelve a leer después. Se pueden eliminar junto con sus propiedades, y una sola función `collectOptions(app)` leería los componentes directamente.

**Estimación:** `STRIKE_GOLDD_APP` pasaría de unas 2250 líneas escritas a mano a unas 500.

## 5. Optimización

Medido en R2026b en modo batch, así que son valores orientativos.

| Operación | Medida | Valoración |
|---|---|---|
| Crear un modelo, con `clear all` | 627 ms | Único ahorro claro: quitar `clear all` (NM L95) |
| Crear un modelo, sin `clear all` | 34 ms | **18 veces menos** |
| Cambiar de algoritmo | 590 ms la primera vez, unos 30 ms después | Aceptable |
| Abrir `STRIKE_GOLDD_APP` | 4,6 s | Aceptable |

Otras mejoras menores, sin medir:

- Ejecutar `install` una vez al arrancar, no en cada Run (SG L364).
- En Run, leer solo lo necesario del modelo en lugar de cargarlo entero (SG L385).
- Eliminar los `clear b bb j ...` dentro de métodos, que desaparecen al extraer el parser.
- Usar `uiprogressdlg` en lugar de `uialert` para indicar progreso.

No se ha analizado el coste del cálculo simbólico, que está fuera de las apps y es lo que domina el tiempo de un análisis.

**Code Analyzer** da 117 mensajes, ninguno grave: 71 son argumentos sin usar, propios de App Designer, y 27 son variables que crecen en bucles. Los fallos de este documento no los detecta.

## 6. Limpieza

**Código muerto en `STRIKE_GOLDD_APP`:**

- La estructura `opts` de L973-1018, que se construye y no se usa (salvo que se aplique 4.3).
- Unas 190 líneas de código comentado (L163-170, L185-205, L450-485, L647-710, L718-780).
- Callbacks sin componente asociado: L1750, L1830, L1835 y L1842.
- `aaa = app.derU` (L867), restos de depuración.
- `maxTimAl` y `pars`: propiedades que nunca se leen. `VectorX`: variable sin usar (L386).

**En las otras apps:**

- MA: `comparar` y `optsMes` sin usar, y las comprobaciones L68-70 y L80-82, redundantes tras el `genpath`.
- NM: `startupFcn` vacío y `message1` sin usar.

**Repositorio:**

- `CreateNewModel.m` y `prevIdPars.m` son archivos generados, pero están en git. El primero contiene una **ruta personal absoluta** (línea 15, OneDrive), y usar las apps ensucia el repositorio.
- No hay `.gitignore`. Conviene ignorar al menos `results/`, `options_aux.m`, `current_options.m` y los modelos generados.
- `PathEditField` trae por defecto `id_results_C2M_11-May-2022`, un archivo que no está en el repositorio.

**Erratas:**

| Dónde | Texto | Correcto |
|---|---|---|
| MA L90, NL L92, SG L1565 | `wanto to` | `want to` |
| MA L7, L118-123 (nombre interno; `startupFcn` corrige el texto visible en L57) | `CONTROLLABILLITY` | `CONTROLLABILITY` |
| SG L540 | `woth` | `with` |
| NL L214 | `Accesibility` | `Accessibility` |
| NL L263 | `simbolically` | `symbolically` |
| NL L272 | `for the each test` | `for each test` |
| SG L2648 | falta `)` al final | `...and later)` |
| MA L106, NL L190, NM L356 | Título `MATLAB App` | Nombre real de cada ventana |

## 7. Comprobado y descartado

Se sospechó de estos puntos, pero **no son fallos**:

- **Funciones regeneradas obsoletas en memoria.** No se reproduce en R2026b sobre disco local, ni con 0 ni con 2 segundos de pausa. Podría ocurrir en carpetas de red u OneDrive, que no se han probado. Con 4.3 deja de importar.
- **`delete(app.SGoldd)` con la ventana sin abrir** (MA L25): `delete([])` no da error.
- **La matriz multiexperimento escrita como vector** (L175-177): el orden por columnas coincide con el de [ME_analysis.m:47](STRIKE-GOLDD/functions/ME_analysis.m#L47).
- **Listas separadas por comas** en derivadas: `2,3` y `2, 3` funcionan bien.

## 8. Plan propuesto

**Fase 1: correcciones pequeñas y aisladas.** Bajo riesgo, se pueden hacer en App Designer. Incluye primero los resultados incorrectos que admiten parche inmediato.
1.1, 1.2, 1.3, 1.5, 1.7, 2.4, 2.7, 2.9, 2.10, 2.12, 2.13, 2.14, 2.18, 2.19 y las erratas.

**Fase 2: funciones comunes.** Riesgo medio, sin tocar el motor.
Parser (4.1), visibilidad (4.2), listado de modelos (4.5), rutas (2.20, 2.21), carga de modelos (2.2, 2.3) y opciones ocultas (1.6). Esto resuelve también 1.4, 2.5, 2.6, 2.8, 2.11, 2.15 y 2.23. Es buen momento para añadir pruebas con `matlab.unittest`, porque el parser ya se podrá probar por separado.

**Fase 3: quitar los archivos generados.** Mayor alcance, toca `STRIKE_GOLDD.m` y `AutoRepar.m`.
Secciones 4.3, 4.4 y 4.6. Corrige 2.16, 2.17 y 2.22, sustituye los parches de 1.1 y 1.2, y permite rehacer *Save options file* (2.1).

## 9. Decisiones pendientes

1. **Linux y macOS:** ¿son un objetivo? Si lo son, 2.20 pasa a la fase 1.
2. **Calculate rank numerically:** propongo quitarla, porque la opción está obsoleta. La alternativa es reactivarla en el motor.
3. **Save options file:** ¿se mantiene? Si sigue, debe guardar un archivo autónomo, con los parámetros conocidos dentro.
4. **Archivos generados:** propongo sacar `CreateNewModel.m` y `prevIdPars.m` de git.
5. **Modelos duplicados:** propongo dejar una sola copia de los 12. Para `HIV` y `HIV_known_u` hay que elegir entre `delta` y `delta1`.
6. **Motor:** ¿se pueden modificar `STRIKE_GOLDD.m` y `AutoRepar.m`? La fase 3 depende de ello.
7. **Cómo aplicar los cambios:** el código de un `.mlapp` se puede editar reescribiendo el XML interno, pero conviene comprobar después cada app abriéndola en App Designer. Los cambios en componentes visuales son más seguros desde App Designer.
