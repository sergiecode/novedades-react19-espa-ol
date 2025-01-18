# React v19

**Extraído del blog publicado el 5 de diciembre de 2024 por el equipo de React**
https://react.dev/blog/2024/12/05/react-19

### Novedades en React 19

#### Acciones (Actions)

Un caso común en aplicaciones React es realizar una mutación de datos y luego actualizar el estado en respuesta. Por ejemplo, cuando un usuario envía un formulario para cambiar su nombre, realizas una solicitud a una API y luego manejas la respuesta. Antes, tenías que manejar estados pendientes, errores, actualizaciones optimistas y solicitudes secuenciales manualmente.

Por ejemplo, así se manejaban los estados pendientes y de error con `useState`:

```jsx
// Antes de Actions
function UpdateName() {
  const [name, setName] = useState("");
  const [error, setError] = useState(null);
  const [isPending, setIsPending] = useState(false);

  const handleSubmit = async () => {
    setIsPending(true);
    const error = await updateName(name);
    setIsPending(false);
    if (error) {
      setError(error);
      return;
    } 
    redirect("/path");
  };

  return (
    <div>
      <input value={name} onChange={(event) => setName(event.target.value)} />
      <button onClick={handleSubmit} disabled={isPending}>
        Actualizar
      </button>
      {error && <p>{error}</p>}
    </div>
  );
}
```

En React 19, se añade soporte para usar funciones asíncronas en transiciones, lo que permite manejar automáticamente los estados pendientes, errores, formularios y actualizaciones optimistas.

Por ejemplo, puedes usar `useTransition` para manejar automáticamente el estado pendiente:

```jsx
// Usando estado pendiente con Actions
function UpdateName() {
  const [name, setName] = useState("");
  const [error, setError] = useState(null);
  const [isPending, startTransition] = useTransition();

  const handleSubmit = () => {
    startTransition(async () => {
      const error = await updateName(name);
      if (error) {
        setError(error);
        return;
      } 
      redirect("/path");
    });
  };

  return (
    <div>
      <input value={name} onChange={(event) => setName(event.target.value)} />
      <button onClick={handleSubmit} disabled={isPending}>
        Actualizar
      </button>
      {error && <p>{error}</p>}
    </div>
  );
}
```

La transición asíncrona establece inmediatamente `isPending` como `true`, realiza la solicitud asíncrona y cambia `isPending` a `false` cuando la transición finaliza. Esto permite mantener la interfaz actual interactiva mientras los datos se actualizan.

----------

#### Nota

Por convención, las funciones que utilizan transiciones asíncronas se llaman **"Actions" (Acciones)**.

**Actions** gestionan automáticamente el envío de datos:

-   **Estado pendiente**: Proporcionan un estado que comienza al inicio de una solicitud y se restablece automáticamente cuando se confirma la actualización final del estado.
-   **Actualizaciones optimistas**: Soportan el nuevo hook `useOptimistic` para dar retroalimentación instantánea al usuario mientras las solicitudes se procesan.
-   **Manejo de errores**: Muestran errores automáticamente en un límite de error y revierten las actualizaciones optimistas si ocurre un fallo.
-   **Formularios**: Los elementos `<form>` ahora permiten pasar funciones a las propiedades `action` y `formAction`, usando **Actions** por defecto y restableciendo el formulario automáticamente tras el envío.

Sobre esta base, React 19 introduce `useOptimistic` para gestionar actualizaciones optimistas y el nuevo hook `React.useActionState` para manejar casos comunes. Además, en `react-dom` se añaden **Actions** en formularios y `useFormStatus` para casos comunes en formularios.

El ejemplo anterior se puede simplificar aún más con estas nuevas características:

```jsx
// Usando <form> Actions y useActionState
function ChangeName({ name, setName }) {
  const [error, submitAction, isPending] = useActionState(
    async (previousState, formData) => {
      const error = await updateName(formData.get("name"));
      if (error) {
        return error;
      }
      redirect("/path");
      return null;
    },
    null,
  );

  return (
    <form action={submitAction}>
      <input type="text" name="name" />
      <button type="submit" disabled={isPending}>Actualizar</button>
      {error && <p>{error}</p>}
    </form>
  );
}
```

----------

### Nuevo Hook: useActionState

Para facilitar los casos comunes con **Actions**, se introduce el nuevo hook `useActionState`:

```jsx
const [error, submitAction, isPending] = useActionState(
  async (previousState, newName) => {
    const error = await updateName(newName);
    if (error) {
      // Puedes devolver cualquier resultado de la acción.
      return error;
    }
    // manejar éxito
    return null;
  },
  null,
);
```

`useActionState` acepta una función (la **Action**) y devuelve una acción envuelta para llamar. Cuando se llama a esta acción, `useActionState` devuelve el último resultado como datos y el estado pendiente como `isPending`.

#### Nota

`React.useActionState` se llamaba anteriormente `ReactDOM.useFormState` en las versiones Canary, pero ha sido renombrado.

-----------


### React DOM: Acciones en `<form>`

Las "Actions" (acciones) están integradas con las nuevas características de `<form>` en React 19 para `react-dom`. Ahora es posible pasar funciones a las propiedades `action` y `formAction` de los elementos `<form>`, `<input>` y `<button>`, para enviar formularios automáticamente con acciones:

```jsx
<form action={funcionDeAccion}>
```

Cuando una acción en un `<form>` se ejecuta con éxito, React restablecerá automáticamente el formulario para componentes no controlados. Si necesitas reiniciar el formulario manualmente, puedes llamar a la nueva API de React DOM `requestFormReset`.

**Más información:** consulta la documentación de `react-dom` sobre `<form>`, `<input>` y `<button>`.

----------

### React DOM: Nuevo hook `useFormStatus`

En sistemas de diseño, es común crear componentes que necesitan información sobre el `<form>` en el que están, sin necesidad de pasar props hacia abajo. Esto puede lograrse usando Context, pero para simplificar el caso común, React 19 incluye el nuevo hook `useFormStatus`:

```jsx
import { useFormStatus } from 'react-dom';

function BotonDeDiseño() {
  const { pending } = useFormStatus();
  return <button type="submit" disabled={pending}>Enviar</button>;
}
```

`useFormStatus` lee el estado del `<form>` padre como si este fuera un proveedor de Context.

**Más información:** consulta la documentación de `react-dom` sobre `useFormStatus`.

----------

### Nuevo hook: `useOptimistic`

Otro patrón común en la interfaz de usuario al realizar mutaciones de datos es mostrar el estado final de forma optimista mientras la solicitud asíncrona se procesa. Con React 19, ahora puedes usar el nuevo hook `useOptimistic` para manejar esto fácilmente:

```jsx
function CambiarNombre({ nombreActual, onActualizarNombre }) {
  const [nombreOptimista, setNombreOptimista] = useOptimistic(nombreActual);

  const accionEnvio = async (formData) => {
    const nuevoNombre = formData.get("name");
    setNombreOptimista(nuevoNombre);
    const nombreActualizado = await updateName(nuevoNombre);
    onActualizarNombre(nombreActualizado);
  };

  return (
    <form action={accionEnvio}>
      <p>Tu nombre es: {nombreOptimista}</p>
      <p>
        <label>Cambiar nombre:</label>
        <input
          type="text"
          name="name"
          disabled={nombreActual !== nombreOptimista}
        />
      </p>
    </form>
  );
}
```

El hook `useOptimistic` renderizará de inmediato el valor de `nombreOptimista` mientras la solicitud `updateName` está en progreso. Cuando la solicitud termine o falle, React cambiará automáticamente al valor actual (`nombreActual`).

**Más información:** consulta la documentación sobre `useOptimistic`.

-------

### Nueva API: `use`

En React 19, se introduce una nueva API llamada `use`, que permite leer recursos durante el renderizado.

#### Leer una promesa con `use`

Por ejemplo, puedes usar `use` para leer una promesa, y React suspenderá el renderizado hasta que la promesa se resuelva:

```jsx
import { use } from 'react';

function Comentarios({ promesaComentarios }) {
  // `use` suspenderá hasta que la promesa se resuelva.
  const comentarios = use(promesaComentarios);
  return comentarios.map(comentario => <p key={comentario.id}>{comentario}</p>);
}

function Pagina({ promesaComentarios }) {
  // Cuando `use` suspenda en Comentarios,
  // este límite de Suspense será mostrado.
  return (
    <Suspense fallback={<div>Cargando...</div>}>
      <Comentarios promesaComentarios={promesaComentarios} />
    </Suspense>
  );
}
```

#### Nota importante:

-   **`use` no admite promesas creadas durante el renderizado.** Si intentas usar una promesa creada en el renderizado con `use`, React mostrará una advertencia:
    
    ![Error de Consola](https://raw.githubusercontent.com/sergiecode/novedades-react19-espanol/refs/heads/main/consola1.png)
    
    Para solucionarlo, debes usar una promesa de una biblioteca o framework compatible con Suspense que admita el almacenamiento en caché de promesas. En el futuro, React planea facilitar el almacenamiento en caché de promesas en el renderizado.
    

#### Leer Context con `use`

También puedes usar `use` para leer Context de forma condicional, incluso después de retornos tempranos:

```jsx
import { use } from 'react';
import ThemeContext from './ThemeContext';

function Encabezado({ children }) {
  if (children == null) {
    return null;
  }
  
  // Esto no funcionaría con `useContext` debido al retorno temprano.
  const tema = use(ThemeContext);
  return (
    <h1 style={{ color: tema.color }}>
      {children}
    </h1>
  );
}
```

La API `use` solo puede llamarse durante el renderizado, similar a los hooks. Sin embargo, a diferencia de los hooks, `use` se puede llamar de manera condicional.

**Más información:** consulta la documentación de la API `use`.

----------

### Nuevas APIs estáticas de React DOM

Se agregaron dos nuevas APIs en `react-dom/static` para la generación de sitios estáticos:

-   **`prerender`**
-   **`prerenderToNodeStream`**

Estas nuevas APIs mejoran `renderToString`, ya que esperan que los datos se carguen antes de generar HTML estático. Están diseñadas para entornos de streaming, como los Streams de Node.js y los Web Streams.

#### Ejemplo con `prerender` en un entorno de Web Stream:

```jsx
import { prerender } from 'react-dom/static';

async function manejador(request) {
  const { prelude } = await prerender(<App />, {
    bootstrapScripts: ['/main.js']
  });
  return new Response(prelude, {
    headers: { 'content-type': 'text/html' },
  });
}
```

Las APIs `prerender` esperan que todos los datos se carguen antes de devolver el stream de HTML estático. Los streams pueden convertirse a cadenas o enviarse como una respuesta de streaming. Estas APIs **no admiten la transmisión de contenido mientras se carga**, algo que sí es compatible con las APIs existentes de renderizado en el servidor de React DOM.

**Más información:** consulta la documentación sobre React DOM Static APIs.

----------


### **React Server Components**

1.  **Qué son:**
    
    -   Permiten renderizar componentes en un entorno separado del cliente, antes de empacarlos.
    -   Pueden ejecutarse en el momento de la construcción (CI server) o en cada solicitud (web server).
2.  **Características clave:**
    
    -   Estabilidad en React 19, aunque las APIs subyacentes pueden cambiar entre versiones menores.
    -   Útil para bibliotecas y frameworks que implementan la arquitectura Full-stack React.
3.  **Recomendaciones:**
    
    -   Fijar la versión de React al usar Server Components en frameworks o usar la versión Canary para evitar rupturas.
    -   Documentación: [React Server Components](https://react.dev).

----------

### **Server Actions**

1.  **Qué son:**
    
    -   Funciones asíncronas ejecutadas en el servidor, accesibles desde componentes cliente.
2.  **Uso:**
    
    -   Declaradas con la directiva `"use server"`.
    -   Se referencian automáticamente y se envían al cliente para su ejecución.
3.  **Nota importante:**
    
    -   No se necesita directiva para Server Components.
    -   La directiva `"use server"` es exclusiva para Server Actions.
4.  **Más información:** [Server Actions](https://react.dev).
    

----------

### **Mejoras en React 19**

1.  **`ref` como prop:**
    
    -   Ahora puedes pasar `ref` directamente como una prop en componentes funcionales, sin necesidad de usar `forwardRef`.

```jsx       
// Definición del componente funcional con ref como prop
function MiInput({ placeholder, ref }) {
  return <input placeholder={placeholder} ref={ref} />;
}

//...
// Uso del componente pasando una ref como prop
<MiInput ref={ref} />;
``` 

2.  **Errores de hidratación con diffs:**
    
    -   Mejoras en los mensajes de error: incluyen un diff claro entre el contenido del servidor y el cliente.
    -   Documentación mejorada para solucionar problemas comunes como datos externos o diferencias de entorno.

  ![Error de Consola](https://raw.githubusercontent.com/sergiecode/novedades-react19-espanol/refs/heads/main/consola2.png)

  ![Error de Consola](https://raw.githubusercontent.com/sergiecode/novedades-react19-espanol/refs/heads/main/consola3.png)
      
3.  **`<Context>` como proveedor:**
    
    -   Puedes usar `<Context>` directamente como proveedor, eliminando la necesidad de `<Context.Provider>`. Ejemplo:
        
```jsx       
const ThemeContext = createContext(''); // Crear el contexto con un valor predeterminado vacío

function Aplicacion({children}) {
  return (
    <ThemeContext value="dark">
      {children} {/* Renderizar los hijos directamente dentro del contexto */}
    </ThemeContext>
  );  
}
```
        
4.  **Funciones de limpieza en `ref`:**
    
    -   Los callbacks de refs ahora pueden devolver funciones de limpieza que se ejecutan cuando el componente se desmonta.
    -   **Nota:** React dejará de llamar a los callbacks de ref con `null` al desmontar en futuras versiones.
    
    Ejemplo:
    
```jsx

<input
  ref={(ref) => {
    // ref creada

    // NUEVO: devuelve una función de limpieza (cleanup)
    // para resetear la ref cuando el elemento sea eliminado del DOM.
    return () => {
      // limpieza de la ref
    };
  }}
/>

   ```
    
5.  **Soporte mejorado para TypeScript:**
    
    -   TypeScript rechazará callbacks de refs que devuelvan valores implícitos (excepto funciones de limpieza).

----------

### **Otras notas clave**

-   React planea deprecar APIs como `forwardRef`, `<Context.Provider>`, y callbacks de refs que no usen funciones de limpieza.
-   La transición a estas nuevas características puede ser automatizada con herramientas como **codemods**.



-----------


#### `useDeferredValue` con valor inicial

React 19 introduce una mejora en el hook `useDeferredValue`, permitiendo un valor inicial mediante la opción `initialValue`:

```jsx

function Search({ deferredValue }) {
  const value = useDeferredValue(deferredValue, ''); // Valor inicial como ''
  return <Results query={value} />;
}
```

Al usar `initialValue`, el valor devuelto en el primer render será el especificado, y luego se actualizará en segundo plano con `deferredValue`. Esto es útil para optimizar renders iniciales con datos aún no diferidos.

#### Soporte para metadatos en el documento

Ahora React soporta nativamente etiquetas como `<title>`, `<link>`, y `<meta>` directamente en los componentes. Estas etiquetas serán automáticamente movidas al `<head>` del documento.


```jsx

function BlogPost({ post }) {
  return (
    <article>
      <h1>{post.title}</h1>
      <title>{post.title}</title>
      <meta name="author" content="Josh" />
      <meta name="keywords" content={post.keywords} />
    </article>
  );
}
```

Esto elimina la necesidad de bibliotecas externas como `react-helmet` en casos simples, mejorando la experiencia en SSR y componentes del servidor.

#### Soporte para hojas de estilo

React 19 agrega manejo nativo de hojas de estilo con la opción de indicar la **precedencia**:

```jsx

<Suspense fallback="loading...">
  <link rel="stylesheet" href="foo.css" precedence="default" />
  <link rel="stylesheet" href="bar.css" precedence="high" />
</Suspense>

```

**Ventajas:**

1.  Control automático del orden de las hojas de estilo.
2.  Garantiza que los estilos se carguen antes de mostrar el contenido.
3.  Evita duplicación en el DOM incluso si se renderiza desde múltiples componentes.

#### Soporte para scripts asíncronos

Los scripts asíncronos (`<script async>`) ahora son compatibles y se manejan automáticamente en React:

```jsx

function MyComponent() {
  return <script async src="..."></script>;
}

```

**Características:**

-   Evita duplicación de scripts.
-   Optimización en SSR: se priorizan los scripts en el `<head>`.
-   Mejora la integración con actualizaciones concurrentes y del lado del cliente.

#### API para precarga de recursos

React 19 incluye nuevas APIs para optimizar la carga de recursos:

```jsx
import { prefetchDNS, preconnect, preload, preinit } from 'react-dom';

function MiComponente() {
  preinit('https://.../ruta/a/algun/script.js', { as: 'script' }); // Carga y ejecuta este script de forma anticipada
  preload('https://.../ruta/a/fuente.woff', { as: 'font' }); // Precarga esta fuente
  preload('https://.../ruta/a/hoja-de-estilo.css', { as: 'style' }); // Precarga esta hoja de estilo
  prefetchDNS('https://...'); // Para cuando es posible que no solicites nada de este host
  preconnect('https://...'); // Para cuando sabes que harás una solicitud, pero no estás seguro de qué
}
```

**Beneficios:**

-   Mejora el rendimiento al informar al navegador sobre recursos críticos.
-   Permite predecir y cargar recursos de manera eficiente durante actualizaciones en cliente.

React continúa mejorando la experiencia del desarrollador con capacidades que integran estilos, scripts y recursos en un flujo unificado, simplificando aplicaciones complejas.

-------


**Compatibilidad con scripts y extensiones de terceros**

Hemos mejorado la hidratación para tener en cuenta los scripts de terceros y las extensiones del navegador.

Cuando React realiza la hidratación, si un elemento que se renderiza en el cliente no coincide con el elemento encontrado en el HTML del servidor, React obligará a una nueva renderización en el cliente para corregir el contenido. Anteriormente, si un elemento era insertado por scripts de terceros o extensiones del navegador, se generaba un error de desajuste y una nueva renderización en el cliente.

En React 19, las etiquetas inesperadas en `<head>` y `<body>` se omitirán, evitando los errores de desajuste. Si React necesita volver a renderizar todo el documento debido a un desajuste de hidratación no relacionado, dejará intactas las hojas de estilo insertadas por scripts de terceros y extensiones del navegador.

**Mejora en el reporte de errores**

Mejoramos el manejo de errores en React 19 para eliminar duplicados y proporcionar opciones para manejar errores capturados y no capturados. Por ejemplo, cuando hay un error en el renderizado capturado por un "Error Boundary", antes React lanzaba el error dos veces (una para el error original y otra al no poder recuperarse automáticamente), y luego mostraba un mensaje en la consola con información sobre el error.

Esto resultaba en tres errores por cada error capturado:

![Error de Consola](https://raw.githubusercontent.com/sergiecode/novedades-react19-espanol/refs/heads/main/consola4.png)

**React 19**: Ahora se registra un solo error con toda la información relevante:

![Error de Consola](https://raw.githubusercontent.com/sergiecode/novedades-react19-espanol/refs/heads/main/consola5.png)

Además, se han agregado dos nuevas opciones a los "root" para complementar `onRecoverableError`:

-   **onCaughtError**: se llama cuando React captura un error en un "Error Boundary".
-   **onUncaughtError**: se llama cuando un error es lanzado y no es capturado por un "Error Boundary".
-   **onRecoverableError**: se llama cuando un error es lanzado y se recupera automáticamente.

Para más información y ejemplos, consulta la documentación de `createRoot` y `hydrateRoot`.

**Soporte para Elementos Personalizados**

React 19 agrega soporte completo para elementos personalizados y pasa todas las pruebas de _Custom Elements Everywhere_.

En versiones anteriores, usar _Custom Elements_ en React era difícil porque React trataba las propiedades no reconocidas como atributos en lugar de propiedades. En React 19, hemos agregado soporte para propiedades que funcionan tanto en el cliente como durante la SSR con la siguiente estrategia:

-   **Renderizado del lado del servidor (SSR)**: las propiedades pasadas a un elemento personalizado se renderizarán como atributos si su tipo es un valor primitivo, como una cadena, número o si el valor es `true`. Las propiedades con tipos no primitivos, como objeto, símbolo, función o `false`, serán omitidas.
-   **Renderizado del lado del cliente (CSR)**: las propiedades que coincidan con una propiedad en la instancia del Elemento Personalizado se asignarán como propiedades, de lo contrario, se asignarán como atributos.

## ¿Cómo actualizar versión?

https://react.dev/blog/2024/04/25/react-19-upgrade-guide
