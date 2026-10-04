# CRM ORM/ODM Lab

API REST de un CRM básico que combina un ORM (Sequelize + PostgreSQL) y un ODM (Mongoose + MongoDB).

## Stack

- Node.js 22, Express 5, CommonJS
- Sequelize + PostgreSQL 16 (`User`, `Company`, `Contact`)
- Mongoose + MongoDB 7 (`Activity`)
- Jest + Supertest
- GitHub Codespaces, Dev Containers, Docker Compose
- Supervisor (`npm run dev`)

## Arquitectura

```text
GitHub Codespace
│
├── app       Node.js 22  ──┬── Sequelize ──> postgres (PostgreSQL)
│                           └── Mongoose  ──> mongo    (MongoDB)
├── postgres
└── mongo
```

La aplicación se conecta por nombre de servicio (`postgres`, `mongo`). Las credenciales de desarrollo llegan como variables de entorno definidas en `.devcontainer/docker-compose.yml` (ver `.env.example`).

## Iniciar el Codespace

1. En GitHub: **Code → Codespaces → Create codespace on main**.
2. Espera a que se levanten los tres servicios (`app`, `postgres`, `mongo`). `postCreateCommand` ejecuta `npm install`.

## Instalar dependencias

```bash
npm install
```

## Seed y reset

```bash
npm run seed    # inserta datos deterministas (3 users, 4 companies, 8 contacts, 10 activities)
npm run reset   # elimina y recrea tablas/base de datos y vuelve a sembrar
```

## Iniciar la API

```bash
npm start       # node ./bin/www
npm run dev     # supervisor ./bin/www
```

Servidor en el puerto `3000` (variable `PORT`).

## Pruebas

```bash
npm test
```

Cada suite restablece PostgreSQL y MongoDB antes de ejecutarse y cierra las conexiones al terminar.

## Endpoints

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Health check |
| GET | `/users` | Listar usuarios |
| GET | `/users/:id` | Obtener usuario |
| POST | `/users` | Crear usuario |
| PUT | `/users/:id` | Actualizar usuario |
| DELETE | `/users/:id` | Eliminar usuario |
| GET | `/companies` | Listar compañías (`?industry=`) |
| GET | `/companies/:id` | Obtener compañía |
| POST | `/companies` | Crear compañía |
| PUT | `/companies/:id` | Actualizar compañía |
| DELETE | `/companies/:id` | Eliminar compañía |
| GET | `/contacts` | Listar contactos |
| GET | `/contacts/:id` | Obtener contacto |
| POST | `/contacts` | Crear contacto |
| PUT | `/contacts/:id` | Actualizar contacto |
| DELETE | `/contacts/:id` | Eliminar contacto |
| GET | `/activities` | Listar actividades (`?type=`) |
| GET | `/activities/:id` | Obtener actividad |
| POST | `/activities` | Crear actividad |
| PUT | `/activities/:id` | Actualizar actividad |
| DELETE | `/activities/:id` | Eliminar actividad |

Los errores se devuelven como JSON: `{ "error": "Contact not found" }`.

## Respuestas

**1. Dos motores**

A las actividades les queda bien MongoDB porque su metadata no es igual para todas: una llamada guarda duración, un correo guarda el asunto, y cada tipo trae campos distintos. 

**2. ORM vs ODM**

Los dos te dejan trabajar con la base usando objetos en vez de escribir las consultas a mano. El ORM es para bases relacionales (aquí Sequelize con PostgreSQL) y el ODM para bases de documentos (aquí Mongoose con MongoDB). Lo que cambia es que uno trabaja con tablas de estructura fija y el otro con documentos que pueden variar.

**3. Configuración por variables de entorno**

Los usuarios y contraseñas de las bases no están en el código, sino en variables de entorno (el .env y el docker-compose.yml del .devcontainer). Si los dejara escritos en los .js podrían quedar expuestos en el repositorio, además de que no podría cambiarlos según el entorno. Los hosts son postgres y mongo, y no localhost, porque cada base vive en su propio contenedor y se llega a ella por el nombre del servicio.

**4. Asociaciones**

Una compañía puede tener varios contactos, y así está puesto en models/sequelize/index.js con Company.hasMany(Contact) y Contact.belongsTo(Company). La llave foránea es companyId y está en la tabla de contactos. El alias as: 'contacts' es el nombre utilizado para traer esos contactos junto con la compañía.

**5. Eager loading**

Si traigo la compañía por un lado y sus contactos por otro, son dos consultas; con include todo sale en una sola. Es preferible include porque hay menos ida y vuelta a la base y la respuesta ya viene completa. Se aplicó en getById de companies.js.

**6. Instancia vs consulta**

Cuando busco el contacto con findByPk y luego le hago .update(), me regresa el contacto ya actualizado y puedo responderlo directo. Si usara Model.update con un where, solo me diría cuántas filas cambiaron, no el contacto, y tendría que hacer otra consulta para devolverlo. Para este caso me sirvió más la instancia.

**7. Esquema flexible**

El campo metadata en models/mongoose/activity.js es de tipo Mixed, que acepta cualquier cosa adentro, y por eso cada tipo de actividad puede guardar campos diferentes. Lo malo es que al aceptar todo, no valida ni revisa el tipo de dato, así que se pierde ese control.

**8. Sin ref**

contactId y userId son números que vienen de PostgreSQL, no ids de Mongo, entonces no les puedo poner ref ni usar populate, que solo funcionan entre documentos de Mongo. El problema es que las dos bases no se enteran una de la otra: si borro un usuario en PostgreSQL, la actividad en Mongo se queda apuntando a un id que ya no existe.

**9. Documento actualizado**

Al principio el PUT del reto 08 me devolvía la actividad como estaba antes de cambiarla, porque findByIdAndUpdate por defecto regresa la versión vieja. Lo arreglé pasándole { new: true } para que devuelva la ya actualizada, y { runValidators: true } para que revise los datos contra el esquema.

**10. Pruebas de comportamiento**

Las pruebas revisan lo que responde la API y eso permite resolver cada reto a mi manera y, mientras la respuesta sea la correcta, pasa. 

**11. Repetibilidad**

tests/setup.js conecta las dos bases en beforeAll, vuelve a cargar los datos de prueba con reset() y al final cierra las conexiones. Gracias a eso cada suite parte de los mismos datos y las pruebas se pueden ejecutar nuevamente bajo las mismas condiciones.

**12. Mi experiencia**

Los retos de código no se me hicieron tan difíciles porque pude apoyarme en los controladores que ya estaban resueltos, como users.js. Lo que más se me complicó fue que, después de resolver los retos, al ejecutar npm test volvieron a fallar los retos 03 y 05 que ya había pasado. El mensaje de Jest del reto 03 me ayudó a identificar el problema, ya que mostraba Expected length: 2 / Received length: 4, indicando que todavía se estaban devolviendo las 4 compañías sin filtrar. Después me di cuenta de que no había guardado companies.js. Al guardarlo, las 9 suites pasaron correctamente.

## Evidencia

<img width="1920" height="1080" alt="Captura de pantalla (675)" src="https://github.com/user-attachments/assets/026adfeb-d165-49e3-b5b1-4271b88068fa" />

