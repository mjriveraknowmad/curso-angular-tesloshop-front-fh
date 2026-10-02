# Análisis de Cambios Staged - Proyecto Angular Teslo Shop

## Resumen General

Se han implementado cambios significativos relacionados con **autenticación** y **gestión de tokens**. Los cambios incluyen:
- ✅ Nuevo módulo de autenticación con rutas, guards e interceptores
- ✅ Implementación de interceptor de autorización para inyectar token
- ✅ Implementación de guard para proteger rutas de autenticación
- ✅ Servicio de autenticación con gestión de estado
- ✅ Componentes de login y registro
- ✅ Actualización de la navbar con UI condicionada al estado de autenticación
- ✅ Interceptor de logging para monitorear respuestas HTTP

---

## 1. Cambios en Configuración Global

### 📝 `src/app/app.config.ts`

**Descripción**: Configuración de la aplicación - Se agregaron interceptores HTTP.

**Cambios principales**:
- ✅ Importación de `withInterceptors` desde `@angular/common/http`
- ✅ Importación de los interceptores:
  - `loggingInterceptor` de `@shared/interceptors/logging.interceptor`
  - `authInterceptor` de `@auth/interceptors/auth.interceptor`
- ✅ Configuración de `withInterceptors()` con ambos interceptores
- ✅ `loggingInterceptor` comentado (desactivado)
- ✅ `authInterceptor` activado

**Código agregado**:
```typescript
withInterceptors([
  // loggingInterceptor,  // Comentado
  authInterceptor,         // Activo
])
```

---

## 2. Cambios en Rutas Principales

### 📝 `src/app/app.routes.ts`

**Descripción**: Rutas principales de la aplicación - Se agregó ruta de autenticación con guard.

**Cambios principales**:
- ✅ Importación del guard: `NotAuthenticatedGuard`
- ✅ Nueva ruta `/auth` con:
  - `loadChildren`: Carga las rutas del módulo auth
  - `canMatch`: Protegida con `NotAuthenticatedGuard`
  - Se usa `canMatch` (no `canActivate`)
- ✅ Hay código comentado que sugiere futuras implementaciones

**Estructura de rutas**:
```typescript
{
  path: 'auth',
  loadChildren: () => import('./auth/auth.routes'),
  canMatch: [NotAuthenticatedGuard],
}
```

---

## 3. Módulo de Autenticación

### 📝 `src/app/auth/auth.routes.ts` (NUEVO)

**Descripción**: Define las rutas específicas del módulo de autenticación.

**Estructura**:
- Componente padre: `AuthLayoutComponent` (con router-outlet)
- Rutas secundarias:
  - `/auth/login` → `LoginPageComponent`
  - `/auth/register` → `RegisterPageComponent`
  - `**` → Redirige a `/auth/login` (fallback)

**Exportación**: `export default authRoutes` (para lazy loading)

---

### 🔐 Guardia de Autenticación

#### `src/app/auth/guards/not-authenticated.guard.ts` (NUEVO)

**Tipo**: `CanMatchFn` (guard asincrónico)

**Funcionalidad**:
- ✅ Verifica si el usuario está autenticado usando `AuthService.checkStatus()`
- ✅ Si está autenticado → redirecciona a `/` (home) y rechaza acceso
- ✅ Si NO está autenticado → permite acceso a rutas de auth

**Flujo**:
```
1. Inyecta AuthService y Router
2. Llama a checkStatus() y espera respuesta (asincrónica)
3. Si isAuthenticated = true → router.navigateByUrl('/') y return false
4. Si isAuthenticated = false → return true
```

---

### 🔑 Interceptores HTTP

#### `src/app/auth/interceptors/auth.interceptor.ts` (NUEVO)

**Tipo**: Interceptor funcional de autorización

**Funcionalidad**:
- ✅ Inyecta el token del `AuthService`
- ✅ Clona la request HTTP
- ✅ Agrega header `Authorization: Bearer {token}` a todas las requests
- ✅ Continúa con la request modificada

**Implementación**:
```typescript
export function authInterceptor(req, next) {
  const token = inject(AuthService).token();
  const newReq = req.clone({
    headers: req.headers.append('Authorization', `Bearer ${token}`)
  });
  return next(newReq);
}
```

**Nota**: Este interceptor se ejecuta ANTES de cada petición HTTP, añadiendo automáticamente el token de autorización.

---

#### `src/app/shared/interceptors/logging.interceptor.ts` (NUEVO)

**Tipo**: Interceptor funcional de logging

**Funcionalidad**:
- ✅ Captura las respuestas HTTP (evento `HttpEventType.Response`)
- ✅ Registra en consola: URL, código de estado
- ✅ No modifica las requests/responses
- ✅ Actualmente desactivado en `app.config.ts`

**Ejemplo de log**:
```
"https://api.example.com/auth/login" returned a response with status 200
```

---

### 📋 Interfaces

#### `src/app/auth/interfaces/auth-response.interface.ts` (NUEVO)

```typescript
interface AuthResponse {
  user: User;
  token: string;
}
```

**Uso**: Tipado de respuestas del servidor en operaciones de autenticación.

---

#### `src/app/auth/interfaces/user.interface.ts` (NUEVO - referenciado)

Se espera que contenga:
```typescript
interface User {
  id: string;
  fullName: string;
  email: string;
  // ... otros campos
}
```

---

### 🔐 Servicio de Autenticación

#### `src/app/auth/services/auth.service.ts` (NUEVO)

**Tipo**: `Injectable` (providedIn: 'root')

**Señales (Signals)**:
- `_authStatus`: Signal<'checking' | 'authenticated' | 'not-authenticated'>
- `_user`: Signal<User | null>
- `_token`: Signal<string | null> (inicializada desde localStorage)

**Métodos principales**:

##### 1. `login(email, password): Observable<boolean>`
- POST a `${baseUrl}/auth/login`
- Recibe `AuthResponse` con usuario y token
- Llama a `handleAuthSuccess()` si es exitoso
- Maneja errores con `handleAuthError()`

##### 2. `checkStatus(): Observable<boolean>`
- GET a `${baseUrl}/auth/check-status`
- Verifica si el usuario aún tiene sesión válida
- Obtiene el token de localStorage
- Si no hay token → logout automático
- Se usa en el guard para validar acceso

##### 3. `logout()`
- Limpia todas las señales de usuario
- Remueve token de localStorage
- Establece estado como 'not-authenticated'

**Computed Properties** (propiedades derivadas):
- `authStatus()`: Retorna estado actual con lógica
- `user()`: Retorna usuario actual
- `token()`: Retorna token actual

**Almacenamiento**:
- Token guardado en `localStorage` con clave `'token'`
- Se recupera al inicializar la aplicación

---

### 🎨 Layout de Autenticación

#### `src/app/auth/layout/auth-layout/` (NUEVO)

**TypeScript** (`auth-layout.component.ts`):
- Componente simple que encapsula el contenedor de auth
- Importa `RouterOutlet` para renderizar componentes secundarios

**HTML** (`auth-layout.component.html`):
- Contenedor centrado: flexbox con altura mínima de pantalla
- Tarjeta blanca con sombra y bordes redondeados
- Título: "Autenticación"
- `<router-outlet />`: Punto de entrada para login/register

**Estilos**:
- Usa Tailwind CSS + DaisyUI
- Diseño responsive
- Clase `font-montserrat` para tipografía

---

### 📄 Página de Login

#### `src/app/auth/pages/login-page/` (NUEVO)

**TypeScript** (`login-page.component.ts`):
- Signals:
  - `hasError`: Bandera de error de validación
  - `isPosting`: Bandera para desactivar botón durante POST (definida pero no utilizada)
  
- FormGroup con validación:
  - `email`: Requerido, debe ser email válido
  - `password`: Requerido, mínimo 6 caracteres

- Método `onSubmit()`:
  - Valida el formulario
  - Si hay error → muestra alerta 2 segundos
  - Llama a `authService.login(email, password)`
  - Si es exitoso → redirecciona a `/`
  - Si falla → muestra alerta de error

**HTML** (`login-page.component.html`):
- Formulario reactivo con `formGroup`
- Dos inputs: email y password con iconos SVG
- Botón submit: "Login"
- Link a registro: "¿No tienes cuenta? Crea una aquí"
- Alerta de error animada (fija en esquina inferior derecha)
- Usa componentes de DaisyUI (input bordered, btn, alert)

---

### 📄 Página de Registro

#### `src/app/auth/pages/register-page/` (NUEVO)

**Estado**: Mínimo (placeholder)
- Componente básico con placeholder
- HTML: `<p>register-page works!</p>`
- TypeScript: Solo la declaración del componente
- **Nota**: Esta página aún requiere implementación completa

---

## 4. Cambios en Componentes Existentes

### 📝 `src/app/store-front/components/front-navbar/` (MODIFICADO)

**Descripción**: Navbar de la tienda con UI dinámica según estado de autenticación.

#### TypeScript (`front-navbar.component.ts`):
**Cambios**:
- ✅ Importación de `inject` desde `@angular/core`
- ✅ Importación del `AuthService`
- ✅ Inyección: `authService = inject(AuthService)`

#### HTML (`front-navbar.component.html`):
**Cambios en navbar-end**:
- ✅ Se agregó `gap-4` para espaciado entre elementos
- ✅ Estructura condicional con `@if/@else if/@else`:

**Condición 1**: Si `authService.authStatus() === 'authenticated'`
- Botón con nombre completo del usuario: `{{ authService.user()?.fullName }}`
- Botón "Salir" (rojo) que ejecuta `authService.logout()`

**Condición 2**: Si `authService.authStatus() === 'not-authenticated'`
- Link "Login" que redirecciona a `/auth/login`
- Estilo: botón secundario

**Condición 3**: Estado "checking" (cargando)
- Muestra `...` como placeholder de carga

**Nota**: Se usa sintaxis Angular 17+ con `@if/@else if/@else`

---

## 📊 Resumen de Archivos

### Nuevos Archivos (13)
| Archivo | Descripción |
|---------|-------------|
| `auth/auth.routes.ts` | Rutas del módulo auth |
| `auth/guards/not-authenticated.guard.ts` | Guard para proteger rutas de auth |
| `auth/interceptors/auth.interceptor.ts` | Inyecta token en requests |
| `auth/interfaces/auth-response.interface.ts` | Interfaz de respuesta de autenticación |
| `auth/interfaces/user.interface.ts` | Interfaz de usuario (referenciada) |
| `auth/layout/auth-layout/auth-layout.component.html` | Template del layout |
| `auth/layout/auth-layout/auth-layout.component.ts` | Componente del layout |
| `auth/pages/login-page/login-page.component.html` | Template de login |
| `auth/pages/login-page/login-page.component.ts` | Lógica de login |
| `auth/pages/register-page/register-page.component.html` | Template de registro |
| `auth/pages/register-page/register-page.component.ts` | Lógica de registro |
| `auth/services/auth.service.ts` | Servicio de autenticación |
| `shared/interceptors/logging.interceptor.ts` | Interceptor de logging |

### Archivos Modificados (3)
| Archivo | Cambios |
|---------|---------|
| `app.config.ts` | Configuración de interceptores |
| `app.routes.ts` | Agregada ruta `/auth` con guard |
| `store-front/components/front-navbar/` | UI dinámica según autenticación |

---

## 🔄 Flujo de Autenticación

```
1. Usuario llega a /auth
   ↓
2. NotAuthenticatedGuard verifica checkStatus()
   ├─ Si autenticado → redirecciona a /
   └─ Si no autenticado → permite acceso a /auth

3. Usuario ve formulario de login
   ↓
4. Ingresa email y password → onSubmit()
   ↓
5. AuthService.login() → POST /auth/login
   ├─ authInterceptor añade Authorization header
   └─ Respuesta: { user, token }

6. AuthService.handleAuthSuccess()
   ├─ Guarda usuario en signal
   ├─ Guarda token en signal y localStorage
   └─ Retorna true

7. Router redirecciona a /
   ↓
8. Navbar renderiza nombre de usuario + botón "Salir"
   ↓
9. En cada request HTTP:
   └─ authInterceptor inyecta "Authorization: Bearer {token}"

10. Usuario hace click en "Salir"
    ├─ AuthService.logout()
    ├─ Limpia signals y localStorage
    └─ Redirecciona a /auth/login
```

---

## ⚙️ Configuración de Ambiente

**Uso**: Todas las rutas API usan `environment.baseUrl`

```typescript
const baseUrl = environment.baseUrl;
// Ej: https://api.example.com
```

**Variables utilizadas**:
- `${baseUrl}/auth/login`
- `${baseUrl}/auth/check-status`

---

## ✅ Pendientes / Observaciones

- [ ] Interfaz `user.interface.ts` - contenido no visible (referenciado en código)
- [ ] Página de registro aún requiere implementación completa
- [ ] `isPosting` signal en LoginPageComponent definida pero no utilizada
- [ ] `loggingInterceptor` comentado en `app.config.ts` (puede activarse si necesario)
- [ ] Comentario en `checkStatus()` sugiere futura modificación en headers

---

## 🎯 Variables de Entorno Requeridas

Para que funcione correctamente, el archivo `environment.ts` debe incluir:

```typescript
export const environment = {
  baseUrl: 'https://api.example.com'  // URL del servidor backend
};
```

---

*Documento generado: Análisis completo de cambios staged en el proyecto Angular Teslo Shop*
