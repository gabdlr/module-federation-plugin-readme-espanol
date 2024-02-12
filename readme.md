# Tutorial: Iniciando con Webpack Module Federation y Angular

Este tutorial muestra cómo utilizar Webpack Module Federation en conjunto con el CLI de Angular y el plugin de `@angular-architects/module-federation`. La meta es hacer una shell capaz de **cargar un microfrontend compilado y desplegado por separado**

![Microfrontend Loaded into Shell](https://github.com/angular-architects/module-federation-plugin/raw/main/libs/mf/tutorial/result.png)

**Importante**: Este tutorial fue escrito para Angular y el **Angular CLI 14** y superior. Para conocer acerca de las pequeñas diferencias para versiones menores de Angular y para la migración desde tales versiones menores, por favor eche un vistazo a nuestra [guía de migración](https://github.com/angular-architects/module-federation-plugin/blob/main/migration-guide.md).

## Parte 1: Clone e Inspeccione el Kit de Inicio

En esta sección usted clonara el kit de inicio e inspeccionará sus proyectos.

1. Clone el el kit de inicio para este tutorial:

   ```
   git clone https://github.com/gabdlr/module-federation-plugin-example.git
   ```

2. Adéntrese en el directorio del proyecto e instale las dependencias **con npm**:

   ```
   cd module-federation-plugin-example
   npm i
   ```

3. Inicie la shell (`ng serve shell -o`) e inspeccionela un poco:

   1. Haga click en el enlace `flights`. Lo llevará a una ruta ficticia. Esta ruta será luego utilizada para cargar el Micro Frontend compilado separadamente.

   2. Eche un vistazo al código fuente de la shell.
   
   3. Detenga el CLI (`CTRL+C`).

4. Haga lo mismo para el Micro Frontend. En este proyecto, se llama `mfe1` (Micro Frontend 1) Puede iniciarlo con `ng serve mfe1 -o`.

## Parte 2: Active y Configure Module Federation

Ahora, vamos a activar y configurar module federation:

1. Instala `@angular-architects/module-federation` en la shell:
   
   ```
   ng add @angular-architects/module-federation@14.3.0 --project shell --type host --port 4200
   ```

   Además, instalalo en el micro frontend:

   ```
   ng add @angular-architects/module-federation@14.3.0 --project mfe1 --type remote --port 4201
   ```

   Esto activa module federation, asigna un puerto para el ng serve, y genera un esquema de la configuración de module federation.

2. Entre en el proyecto `mfe1` y abra el archivo de configuración generado `projects\mfe1\webpack.config.js`. Este contiene la configuración de module federation para `mfe1`. Ajustelo de la siguiente manera:

   ```javascript
   const {
     shareAll,
     withModuleFederationPlugin,
   } = require('@angular-architects/module-federation/webpack');

   module.exports = withModuleFederationPlugin({
     name: 'mfe1',

     exposes: {
       // Actualice toda esta línea (tanto la parte de la derecha como la izquierda):
       './Module': './projects/mfe1/src/app/flights/flights.module.ts',
     },

     shared: {
       ...shareAll({
         singleton: true,
         strictVersion: true,
         requiredVersion: 'auto',
       }),
     },
   });
   ```
   
   Esto expone el `FlightsModule` bajo el Nombre `./Module`. Por tanto, la shell puede usar esta ruta para cargarlo.

3. Entre en el proyecto `shell` y abra el archivo `projects\shell\webpack.config.js`. Asegurese que el mapeo en la sección remotes use el puerto `4201` (y por tanto, apunte al Micro Frontend):

   ```javascript
   const {
     shareAll,
     withModuleFederationPlugin,
   } = require('@angular-architects/module-federation/webpack');

   module.exports = withModuleFederationPlugin({
     remotes: {
       // Revise esta línea. Esta el puerto 4201 configurado?
       mfe1: 'http://localhost:4201/remoteEntry.js',
     },

     shared: {
       ...shareAll({
         singleton: true,
         strictVersion: true,
         requiredVersion: 'auto',
       }),
     },
   });
   ```

   Esto referencia al proyecto `mfe1` compilado y desplegado separadamente.

4. Abra la configuración del router de la `shell` (`projects\shell\src\app\app.routes.ts`) y agregue una ruta que cargue el Micro Frontend:

   ```javascript
   {
      path: 'flights',
      loadChildren: () => import('mfe1/Module')
        .then(m => m.FlightsModule)
   },

   {
      path: '**',
      component: NotFoundComponent
   }

   // NO inserte rutas después de esta.
   // { path:'**', ...} debe ser LA ULTIMA.
   ```

   Por favor note que la URL importada consiste en los nombres definidos en los archivos de configuración más arriba.

5. Como la URL `mfe1/Module` no existe en tiempo de compilación, ayude al compilador de TypeScript agregando la siguiente línea al archivo `projects\shell\src\decl.d.ts`:

   ```javascript
   declare module 'mfe1/Module';
   ```

## Parte 3: Pruebalo

Ahora, vamos a probarlo!

1. Inicie `shell` y `mfe1` lado a lado en dos terminales diferentes:

   ```
   ng serve shell -o
   ng serve mfe1 -o
   ```

   **Consejo:** Tal vez debas usar dos terminales para esto.

2. Luego de que abra una ventana del navegador con la shell (`http://localhost:4200`), haga click en `Flights`. Esto debería cargar el Micro Frontend dentro de la shell:

   ![Shell](https://github.com/angular-architects/module-federation-plugin/raw/main/libs/mf/tutorial/shell.png)

3. Además, comprueba que el Micro Frontend también corre de forma aislada en http://localhost:4201:

   ![Microfrontend](https://github.com/angular-architects/module-federation-plugin/raw/main/libs/mf/tutorial/mfe1.png)

**Consejo:** También puede llamar el siguiente script para iniciar todos los proyectos al mismo tiempo: `npm run run:all`. Este script es agregado por el plugin Module Federation.

**Felicidades!** Haz implementado tu primer proyecto de Module Federation con Angular!

## Parte 4: Cambiar a Dynamic Federation

Ahora, eliminemos la necesidad de registrar nuestros Micro Frontends por adelantado en la shell.

### Parte 4a: Uso Básico de la Dynamic Federation

1. Ve a tu aplicación `shell` y abre el archivo `projects\shell\webpack.config.js`. Aquí, elimina los remotos registrados:

   ```javascript
   remotes: {
      // Elimina esta linea:
      // "mfe1": "http://localhost:4201/remoteEntry.js",
   },
   ```

2. Abre el archivo `app.routes.ts` y usa la función `loadRemoteModule` en lugar de la declaración del `import` dinámico:

   ```typescript
   import { loadRemoteModule } from '@angular-architects/module-federation';

   [...]
   const routes: Routes = [
      [...]
      {
          path: 'flights',
          loadChildren: () =>
               loadRemoteModule({
                  type: 'module',
                  remoteEntry: 'http://localhost:4201/remoteEntry.js',
                  exposedModule: './Module'
              })
              .then(m => m.FlightsModule)
      },
      [...]
   ]
   ```

   _Observaciones:_ `type: 'module'` es necesario en Angular 13 o superior a partir de la versión 13, el CLI produce modulos EcmaScript en lugar de los "simples y viejos" archivos JavaScript.

3. **Reinicie ambos**, la `shell` y el micro frontend (`mfe1`).

4. La shell debería poder cargar el micro frontend. De cualquier forma, ahora es cargado dinámicamente.

### Parte 4b: Cargando la Meta Data por Adelantado

Eso fue fácil, no? Aun así, podemos mejorar esta solución un poco. Idealmente, cargamos los  **remoteEntry.js** de los Micro Frontend antes de que Angular inicie. Este archivo contiene meta data del Micro Frontend, especialmente acerca de sus dependencias compartidas. Conocerlas por adelantado ayuda a Module Federation a evitar conflictos de versiones.

1. Entra al proyecto de la `shell` y abre el archivo `main.ts`. Ajustalo de la siguiente manera:

  ```typescript
  import { loadRemoteEntry } from '@angular-architects/module-federation';

  Promise.all([
    loadRemoteEntry({
      type: 'module',
      remoteEntry: 'http://localhost:4201/remoteEntry.js',
    }),
  ])
    .catch((err) => console.error('Error loading remote entries', err))
    .then(() => import('./bootstrap'))
    .catch((err) => console.error(err));
  ```

2. Reinicia ambos, la `shell` y el micro frontend (`mfe1`).

3. La shell aun debería poder cargar el micro frontend.

### Parte 4c: Usa un Registro

Hasta ahora, hemos hardcodeado las URLs que apuntan a nuestros Micro Frontends. De cualquier forma, en un escenario del mundo real, preferiríamos recuperar esta información en tiempo de ejecución de un archivo de configuración o un servicio de registro. De esto es lo que trata esta parte del laboratorio.

1. Vaya a la shell y cree el archivo `mf.manifest.json` en el directorio `assets` (`projects\shell\src\assets\mf.manifest.json`):

  ```json
  {
    "mfe1": "http://localhost:4201/remoteEntry.js"
  }
  ```

2. Ajuste el archivo `main.ts` de la shell (`projects/shell/src/main.ts`) de la siguiente manera:

   ```typescript
   import { loadManifest } from '@angular-architects/module-federation';
   loadManifest('assets/mf.manifest.json')
    .catch((err) => console.error('Error loading remote entries', err))
    .then(() => import('./bootstrap'))
    .catch((err) => console.error(err));
   ```

La función importada `loadManifest` también carga los entry points de los remotos.

3. Ajuste la ruta lazy de la shell apuntando al Micro Frontend de la siguiente manera (`projects/shell/src/app/app.routes.ts`):

  ```typescript
  {
    path: 'flights',
    loadChildren: () =>
        loadRemoteModule({
            type: 'manifest',
            remoteName: 'mfe1',
            exposedModule: './Module'
        })
        .then(m => m.FlightsModule)
  },
  ```

4. Reinicie tanto la `shell` como el micro frontend (`mfe1`).

5. La shell aún debería poder cargar el micro frontend.

**Consejo:** El comando `ng add` usado inicialmente también provee una opción `--type dynamic-host`. Esto hace que `ng add` genere el `mf.manifest.json` como también la llamada a `loadManifest` en el `main.ts`.

## Parte 5: Comunicación Entre los Micro Frontends y Compartiendo Librerías Monorepo

1. Agrega una librería a tu monorepo

  ```
    ng g lib auth-lib
  ```

2. En tu `tsconfig.json` en la raíz de tu workspace, ajusta el mapeo de la ruta para `auth-lib` de tal forma que apunte al entry point de la librería:

  ```json
  "auth-lib": [
      "projects/auth-lib/src/public-api.ts"
  ]
  ```

3. Como la mayoría de los IDEs solo leen los archivos de configuración globales tales como el `tsconfig.json` una sola vez, reinicia tu IDE (Como opción, tu IDE también podría proveer una opción para recargar estas configuraciones).

4. Entra en tu proyecto `auth-lib` y abre el archivo `auth-lib.service.ts` (`projects\auth-lib\src\lib\auth-lib.service.ts`). Ajustalo de la siguiente manera:

  ```typescript
  import { Injectable } from '@angular/core';

  @Injectable({
    providedIn: 'root',
  })
  export class AuthLibService {
    private userName: string;

    public get user(): string {
      return this.userName;
    }

    constructor() {}

    public login(userName: string, password: string): void {
      // Authentication for **honest** users TM. (c) Manfred Steyer
      this.userName = userName;
    }
  }
  ```

5. Entra al proyecto de tu `shell` y abre su `app.component.ts` (`projects\shell\src\app\app.component.ts`). Usa el `AuthLibService` para loguear a un usuario:


  ```typescript
  [...]

  // IMPORTANTE: Asegurate de importar el servicio
  // de 'auth-lib'!
  import { AuthLibService } from 'auth-lib';

  [...]

  @Component({
    selector: 'app-root',
    templateUrl: './app.component.html',
  })
  export class AppComponent {
    title = 'shell';

    constructor(private service: AuthLibService) {
      this.service.login('Max', null);
    }
  }
  ```

6. Entra en tu proyecto `mfe1` y abre su `flights-search.component.ts` (`projects\mfe1\src\app\flights\flights-search\flights-search.component.ts`). Usa el servicio compartido para recuperar el nombre del usuario actual:

  ```typescript
  [...]

  // IMPORTANTE: Asegurate de importar el servicio
  // de 'auth-lib'!
  import { AuthLibService } from 'auth-lib';

  [...]

  export class FlightsSearchComponent {

      // Agrega esto:
      user = this.service.user;

      // Y esto otro:
      constructor(private service: AuthLibService) { }

      [...]
  }
  ```

7. Abre el template de este componente (`flights-search.component.html`) y haz data bind a la propiedad `user`:

```html
<div id="container">
  <!-- Agrega esta linea: -->
  <div>User: {{user}}</div>
  [...]
</div>
```

8. Reinicia tanto la `shell` como el micro frontend (`mfe1`).

9. En la shell, navega al micro frontend. Si muestra que el nombre de usuario usado es `Max`, la librería se está compartiendo.

**Observaciones:** Todas las librerías en tu Monorepo son compartidas por defecto. La siguiente sección muestra cómo seleccionar las librerías a compartir.

## Bono - Parte 6: Configurar Explicitamente las Dependencias Compartidas

Hasta ahora, todas las dependencias se han compartido. El uso de la función `shareAll` asegura que, todos los paquetes en la sección de `dependencies` de tu  `package.json` sean compartidas y por defecto, todas las librerías monorepo-internas como `auth-lib` también son compartidas.

Mientras que esto facilita el empezar con Module Federation, podemos lograr soluciones más eficientes al definir directamente que compartir. Esto es porque las dependencias compartidas no son tree-shakable y terminan en su propio bundle que debe ser cargado.

Para compartir las dependencias de forma explicita, podrías cambiar a la siguiente configuración:

1. En el `webpack.config.js` de la shell (`projects\shell\webpack.config.js`):

  ```javascript
  // Importa share en lugar de shareAll:
  const {
    share,
    withModuleFederationPlugin,
  } = require('@angular-architects/module-federation/webpack');

  module.exports = withModuleFederationPlugin({
    remotes: {
      // Revise esta linea. Esta el puerto 4201 configurado?
      // "mfe1": "http://localhost:4201/remoteEntry.js",
    },

    // Explicitamente comparta los paquetes:
    shared: share({
      '@angular/core': {
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
      },
      '@angular/common': {
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
      },
      '@angular/common/http': {
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
      },
      '@angular/router': {
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
      },
    }),

    // Explicitamente comparta las librerías del mono-repo:
    sharedMappings: ['auth-lib'],
  });
  ```

2. En el `webpack.config.js` del Micro Frontend (`projects\mfe1\webpack.config.js`):

  ```javascript
  // Importa share en lugar de shareAll:
  const {
    share,
    withModuleFederationPlugin,
  } = require('@angular-architects/module-federation/webpack');

  module.exports = withModuleFederationPlugin({
    name: 'mfe1',

    exposes: {
       // Actualice toda esta linea (tanto la parte de la derecha como la izquierda):
      './Module': './projects/mfe1/src/app/flights/flights.module.ts',
    },

    // Explicitamente comparta los paquetes:
    shared: share({
      '@angular/core': {
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
      },
      '@angular/common': {
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
      },
      '@angular/common/http': {
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
      },
      '@angular/router': {
        singleton: true,
        strictVersion: true,
        requiredVersion: 'auto',
      },
    }),

    // Explicitamente comparta las librerías del mono-repo:
    sharedMappings: ['auth-lib'],
  });
  ```

Luego de esto, reinicie la shell y el Micro Frontend.

## Más Detalles de Module Federation

Eche un vistazo en [serie de artículos acerca de Module Federation](https://www.angulararchitects.io/aktuelles/the-microfrontend-revolution-part-2-module-federation-with-angular/)

## Angular Trainings, Workshops, y Consultoría

- [Angular Trainings y Workshops](https://www.angulararchitects.io/en/angular-workshops/)
- [Angular Consulting](https://www.angulararchitects.io/en/consulting/)
