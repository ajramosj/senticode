# Senticode

## Scaffold

Estos comandos crean el scaffold de **React Native + Expo + Firebase** dentro del repositorio actual. Ejecútalos por bloques y en orden; todavía no implementan las funcionalidades del producto.

Usaremos npm, TypeScript, Expo Router y Cloud Functions. Para esta primera etapa, cada proyecto tendrá sus propias dependencias y su `package-lock.json`.

**1. Preparar el entorno**

Necesitas:

- Node.js 22 actualizado, como mínimo 22.13.
- Java JDK 21 para los emuladores de Firebase.
- Xcode para compilar iOS.
- Android Studio y un dispositivo virtual para compilar Android.

La plantilla actual de Expo utiliza SDK 58. [Documentación de Expo](https://expo.dev/changelog/sdk-58).

Si ya tienes **nvm** instalado:

```bash
cd /Users/alejandro2/Documents/GitHub/senticode

nvm install 22
nvm use 22

node --version > .nvmrc

node --version
npm --version
java -version
```

Si falta Node o Java, instálalos antes de continuar.

**2. Inicializar el repositorio y las herramientas**

```bash
cd /Users/alejandro2/Documents/GitHub/senticode

npm init -y
npm pkg set name="codesense"
npm pkg set private=true --json
npm pkg set 'engines.node=22.x'

npm install --save-dev --save-exact \
  firebase-tools \
  prettier \
  vitest \
  typescript \
  @firebase/rules-unit-testing \
  firebase
```

Crear la configuración básica:

```bash
cat > .editorconfig <<'EOF'
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 2
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
EOF

cat > .prettierrc.json <<'EOF'
{
  "singleQuote": true,
  "semi": true,
  "trailingComma": "all"
}
EOF

cat > .prettierignore <<'EOF'
node_modules/
.expo/
dist/
build/
coverage/
functions/lib/
apps/mobile/ios/
apps/mobile/android/
package-lock.json
**/package-lock.json
.firebase/
.emulator-data/
EOF

cat >> .gitignore <<'EOF'

# Dependencies
node_modules/

# Generated output
.expo/
dist/
build/
coverage/
functions/lib/
apps/mobile/ios/
apps/mobile/android/

# Local configuration
.env
.env.*
!.env.example
**/.env
**/.env.*
!**/.env.example
*.local

# Firebase local files
.firebase/
.emulator-data/
*-debug.log*
functions/.secret.local
*service-account*.json

# OS
.DS_Store
EOF
```

**3. Crear la aplicación Expo**

La plantilla `default` incluye TypeScript y Expo Router. [Referencia de create-expo-app](https://docs.expo.dev/more/create-expo/).

```bash
mkdir -p apps

npx create-expo-app@latest apps/mobile \
  --template default@sdk-58

cd apps/mobile

npm pkg set name="@codesense/mobile"
npm pkg set private=true --json

npx expo install \
  firebase \
  @react-native-async-storage/async-storage \
  expo-dev-client

npm install --save-exact zod

npm pkg set 'scripts.typecheck=tsc --noEmit'

cd ../..
```

Usaremos el **Firebase JavaScript SDK**. Expo documenta esta integración para Authentication y Firestore; no necesitamos añadir React Native Firebase al scaffold. [Firebase en Expo](https://docs.expo.dev/guides/using-firebase/).

**4. Crear las carpetas del producto**

Las carpetas de funcionalidades van dentro de `src`. Conserva las rutas que genere la plantilla de Expo Router.

```bash
mkdir -p \
  apps/mobile/src/features/auth \
  apps/mobile/src/features/onboarding \
  apps/mobile/src/features/dashboard \
  apps/mobile/src/features/learning \
  apps/mobile/src/features/lessons \
  apps/mobile/src/features/review \
  apps/mobile/src/features/profile \
  apps/mobile/src/components/ui \
  apps/mobile/src/components/exercises \
  apps/mobile/src/components/learning \
  apps/mobile/src/components/gamification \
  apps/mobile/src/lib/firebase \
  apps/mobile/src/lib/analytics \
  apps/mobile/src/lib/date \
  apps/mobile/src/theme \
  apps/mobile/src/hooks \
  packages/contracts/src \
  content/seed \
  scripts/seed \
  scripts/validate-content \
  tests/integration \
  tests/rules \
  docs/decisions
```

Para que Git conserve las carpetas vacías:

```bash
find apps/mobile/src packages content scripts tests docs \
  -type d -empty -exec touch {}/.gitkeep \;
```

`packages/contracts` queda reservado para los contratos públicos compartidos. Conectaremos ese paquete cuando definamos los modelos; las soluciones privadas permanecerán en el backend.

**5. Inicializar Firebase**

Desde la raíz:

```bash
npx firebase login

npx firebase projects:list

npx firebase init
```

Responde al asistente así:

| Pregunta | Selección |
|---|---|
| Proyecto | Tu proyecto Firebase de desarrollo; créalo si todavía no existe |
| Archivo de reglas | `firestore.rules` |
| Archivo de índices | `firestore.indexes.json` |
| Lenguaje de Functions | TypeScript |
| ESLint | Sí |
| Instalar dependencias | Sí |
| Emuladores | Authentication, Functions y Firestore |
| Emulator UI | Sí |
| Descargar emuladores | Sí |

Si pregunta por la ubicación de Firestore, escoge la región donde alojarás los datos; para este proyecto en España puedes valorar Madrid si está disponible.

Después:

```bash
npm --prefix functions pkg set 'engines.node=22'

mkdir -p \
  functions/src/handlers \
  functions/src/application \
  functions/src/domain/exercises \
  functions/src/domain/progression \
  functions/src/domain/scoring \
  functions/src/domain/streaks \
  functions/src/domain/review \
  functions/src/infrastructure \
  functions/src/validation \
  functions/tests/unit

find functions/src functions/tests \
  -type d -empty -exec touch {}/.gitkeep \;
```

Firebase genera el proyecto TypeScript, la compilación y la configuración del backend. [Inicio de Cloud Functions](https://firebase.google.com/docs/functions/get-started).

**6. Establecer reglas iniciales de Firestore**

Hasta implementar autenticación y autorización, deja todas las operaciones de cliente bloqueadas:

```bash
cat > firestore.rules <<'EOF'
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
EOF
```

Estas reglas son el punto de partida. Las sustituiremos por permisos específicos y pruebas de aislamiento cuando implementemos persistencia.

**7. Preparar las variables de entorno**

```bash
cat > apps/mobile/.env.example <<'EOF'
EXPO_PUBLIC_FIREBASE_API_KEY=
EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN=
EXPO_PUBLIC_FIREBASE_PROJECT_ID=
EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET=
EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
EXPO_PUBLIC_FIREBASE_APP_ID=

EXPO_PUBLIC_FIREBASE_FUNCTIONS_REGION=europe-southwest1

EXPO_PUBLIC_USE_FIREBASE_EMULATORS=true
EXPO_PUBLIC_FIREBASE_EMULATOR_HOST=localhost
EOF

cp apps/mobile/.env.example apps/mobile/.env.local
```

La configuración se obtiene registrando una **aplicación web** en tu proyecto Firebase para utilizar el SDK JavaScript.

Los valores `EXPO_PUBLIC_*` se incluyen en la app. No pongas ahí credenciales administrativas ni secretos. La autorización dependerá de Authentication, Security Rules y las validaciones del backend.

Cuando conectemos los emuladores, el host será:

- Simulador iOS: `localhost`.
- Emulador Android: `10.0.2.2`.
- Dispositivo físico: la IP local de tu Mac.

Por ahora este archivo es una plantilla; todavía no inicializa Firebase en la app.

**8. Añadir comandos desde la raíz**

```bash
npm pkg set 'scripts.mobile=npm --prefix apps/mobile run start -- --dev-client'
npm pkg set 'scripts.ios=npm --prefix apps/mobile exec -- expo run:ios'
npm pkg set 'scripts.android=npm --prefix apps/mobile exec -- expo run:android'

npm pkg set 'scripts.lint:mobile=npm --prefix apps/mobile run lint'
npm pkg set 'scripts.lint:functions=npm --prefix functions run lint'
npm pkg set 'scripts.lint=npm run lint:mobile && npm run lint:functions'

npm pkg set 'scripts.typecheck=npm --prefix apps/mobile run typecheck'
npm pkg set 'scripts.build:functions=npm --prefix functions run build'

npm pkg set 'scripts.emulators=npm run build:functions && firebase emulators:start --only auth,firestore,functions --project demo-codesense'

npm pkg set 'scripts.test=vitest run'
npm pkg set 'scripts.test:watch=vitest'
npm pkg set 'scripts.format=prettier --write .'
npm pkg set 'scripts.format:check=prettier --check .'
```

El proyecto `demo-codesense` permite ejecutar los emuladores con un identificador local de demostración. Cuando conectemos la app a ellos, deberá utilizar el mismo identificador. [Emulator Suite](https://firebase.google.com/docs/emulator-suite/install_and_configure).

cd /Users/alejandro2/Documents/GitHub/senticode/apps/mobile
npx expo start --web

He realizado estos cambios:
1. En [app-tabs.web.tsx](/Users/alejandro2/Documents/GitHub/senticode/apps/mobile/src/components/app-tabs.web.tsx), añadí:
   import type { ComponentProps } from 'react';
2. Creé un tipo que conserva las propiedades de Expo Router y utiliza el tipo de ref de Pressable:
   type TabButtonProps = Omit<TabTriggerSlotProps, 'ref'> &
     Pick<ComponentProps<typeof Pressable>, 'ref'>;
3. Cambié el parámetro de TabButton de TabTriggerSlotProps a TabButtonProps. Esto corrige el error sin cambiar el comportamiento visual.
Además, al ejecutar expo lint, Expo añadió automáticamente estas dependencias a [package.json](/Users/alejandro2/Documents/GitHub/senticode/apps/mobile/package.json):
"eslint": "^9.0.0",
"eslint-config-expo": "~58.0.4"
Su instalación quedó incompleta y no se generó la configuración de ESLint.

cd apps/mobile
npm install

**9. Verificar el scaffold**

```bash
npm run typecheck
npm run lint
npm run build:functions

cd apps/mobile
npx expo install --check
npx expo-doctor@latest
cd ../..
```

Todavía no ejecutes `npm test` como comprobación funcional: no hemos escrito pruebas.

Arranca Firebase en una terminal:

```bash
cd /Users/alejandro2/Documents/GitHub/senticode

npm run emulators
```

Para compilar e iniciar la aplicación en otra terminal, elige una plataforma:

```bash
cd /Users/alejandro2/Documents/GitHub/senticode

# Requiere Xcode y un simulador iOS instalado.
npm run ios
```

O:

```bash
cd /Users/alejandro2/Documents/GitHub/senticode

# Requiere Android SDK y un emulador o dispositivo conectado.
npm run android
```

Después de instalar la primera compilación de desarrollo, puedes iniciar Metro con:

```bash
npm run mobile
```

El resultado será la aplicación de ejemplo de Expo ejecutable, el backend TypeScript compilable y los emuladores de Firebase configurados. **Autenticación, pantallas de CodeSense, conexión a Firebase y reglas educativas se implementarán en los siguientes pasos.**

## Otros

```bash
nvm install 22
nvm use 22

# Guardar la versión exacta para este proyecto
node --version > .nvmrc

# En futuras sesiones, desde el repositorio
nvm use
```

```
cd /Users/alejandro2/Documents/GitHub/senticode/apps/mobile

npx expo start --web

npm pkg set 'scripts.web=npm --prefix apps/mobile run web'

npm run web
```

## Firebase

*OPENJDK*

```bash
npx firebase login
npx firebase login:list
npx firebase init
npx firebase projects:list
```

```bash
Error: Request to https\://serviceusage.googleapis.com/v1/projects/senticode-project-dev/services/firestore.googleapis.com had HTTP Error: 403, Caller does not have required permission to use project senticode-project-dev. Grant the caller the roles/serviceusage.serviceUsageConsumer role, or a custom role with the serviceusage.services.use permission, by visiting https\://console.developers.google.com/iam-admin/iam?project=senticode-project-dev and then retry. Propagation of the new permission may take a few minutes.
```

```
• Enable
Gemini in Firebase features? No
i Firebase optionally collects CLI and Emulator Suite usage and error reporting information to help improve our products. Data is collected in accordance with Google's privacy policy (https://policies google,com/privacy) and is not used to identify you
• Allow Firebase to collect CLI and Emulator Suite usage and error reporting information? No
```

## Android

*ANDROID STUDIO*

```
export ANDROID_HOME="$HOME/Library/Android/sdk"
```

```bash
alejandro2@MacBook-Air-de-Alejandro2 senticode % cd /Users/alejandro2/Documents/GitHub/senticode

# Requiere Android SDK y un emulador o dispositivo conectado.
npm run android
zsh: command not found: #

> codesense@1.0.0 android
> cd apps/mobile && npx expo run:android

env: load .env.local
env: export EXPO_PUBLIC_FIREBASE_API_KEY EXPO_PUBLIC_FIREBASE_APP_ID EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN EXPO_PUBLIC_FIREBASE_EMULATOR_HOST EXPO_PUBLIC_FIREBASE_FUNCTIONS_REGION EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID EXPO_PUBLIC_FIREBASE_PROJECT_ID EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET EXPO_PUBLIC_USE_FIREBASE_EMULATORS
› Building app...
Configuration on demand is an incubating feature.

> Configure project :
[ExpoRootProject] The Kotlin Gradle plugin used by this build is 2.2.10, but the version catalog declares 2.2.0. Using the plugin version.
[ExpoRootProject] Using the following versions:
  - buildTools:  37.0.0
  - minSdk:      24
  - compileSdk:  37
  - targetSdk:   36
  - ndk:         27.1.12297006
  - kotlin:      2.2.10
  - ksp:         2.2.10-2.0.2

> Configure project :app
 ℹ️  Applying gradle plugin 'expo-max-sdk-override-plugin'
  [expo-max-sdk-override-plugin] This plugin will find all permissions declared with `android:maxSdkVersion`. If there exists a declaration with the `android:maxSdkVersion` annotation and another one without, the plugin will remove the annotation from the final merged manifest. In order to see a log with the changes run a clean build of the app.
 ℹ️  Applying gradle plugin 'expo-dev-launcher-gradle-plugin'
Checking the license for package NDK (Side by side) 27.1.12297006 in /Users/alejandro2/Library/Android/sdk/licenses
License for package NDK (Side by side) 27.1.12297006 accepted.
Preparing "Install NDK (Side by side) 27.1.12297006 v.27.1.12297006".
"Install NDK (Side by side) 27.1.12297006 v.27.1.12297006" ready.
Installing NDK (Side by side) 27.1.12297006 in /Users/alejandro2/Library/Android/sdk/ndk/27.1.12297006
"Install NDK (Side by side) 27.1.12297006 v.27.1.12297006" complete.
"Install NDK (Side by side) 27.1.12297006 v.27.1.12297006" finished.
Checking the license for package Android SDK Build-Tools 37 in /Users/alejandro2/Library/Android/sdk/licenses
License for package Android SDK Build-Tools 37 accepted.
Preparing "Install Android SDK Build-Tools 37 v.37.0.0".
"Install Android SDK Build-Tools 37 v.37.0.0" ready.
Installing Android SDK Build-Tools 37 in /Users/alejandro2/Library/Android/sdk/build-tools/37.0.0
"Install Android SDK Build-Tools 37 v.37.0.0" complete.
"Install Android SDK Build-Tools 37 v.37.0.0" finished.
[=========                              ] 25%                                   
> Configure project :expo
WARNING: API 'libraryVariants' is obsolete and has been replaced with 'AndroidComponentsExtension'.
It will be removed in version 10.0 of the Android Gradle plugin.
The legacy variant API is disabled by default in AGP 9.0, but can be re-enabled by adding 
    android.newDsl=false
to this project's gradle.properties file.
For more information, see http://developer.android.com/build/r/new-dsl.

To determine what is calling libraryVariants, use -Pandroid.debug.obsoleteApi=true on the command line to display more information.

Using expo modules
  - expo-dom-webview (58.0.2)
  - expo-log-box (58.0.15)
  - expo-constants (58.0.16)
  - expo-dev-client (58.0.20)
  - expo-dev-launcher (58.0.20)
  - expo-dev-menu (58.0.13)
  - expo-dev-menu-interface (58.0.2)
  - expo-json-utils (58.0.1)
  - expo-manifests (58.0.1)

> Configure project :react-native-worklets
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-worklets/android/build.gradle.kts:162:1: 'fun Project.android(configure: Action<LibraryExtension>): Unit' is deprecated. Replaced by com.android.build.api.dsl.LibraryExtension.
This class is not used for the public extensions in AGP when android.newDsl=true, which is the default in AGP 9.0, and will be removed in AGP 10.0.

> Configure project :expo-modules-core
Linking react-native-worklets native libs into expo-modules-core build tasks
task ':react-native-worklets:mergeDebugNativeLibs'
task ':react-native-worklets:mergeReleaseNativeLibs'

> Configure project :expo
  - expo-modules-core (58.1.0)
  - expo-updates-interface (58.0.1)1m 9s]
  - [📦] expo-ui (58.0.28)
  - [📦] expo-asset (58.0.21)
  - [📦] expo-device (58.0.5)
  - [📦] expo-file-system (58.0.13)
  - [📦] expo-font (58.0.12)
  - [📦] expo-image (58.0.18)
  - [📦] expo-keep-awake (58.0.4)
  - [📦] expo-linking (58.0.17)
  - [📦] expo-router (58.0.31)
  - [📦] expo-splash-screen (58.0.14)
  - [📦] expo-status-bar (58.0.5)
  - [📦] expo-system-ui (58.0.7)
  - [📦] expo-web-browser (58.0.8)


> Configure project :react-native-reanimated
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/build.gradle.kts:168:1: 'fun Project.android(configure: Action<LibraryExtension>): Unit' is deprecated. Replaced by com.android.build.api.dsl.LibraryExtension.
This class is not used for the public extensions in AGP when android.newDsl=true, which is the default in AGP 9.0, and will be removed in AGP 10.0.

> Task :expo-constants:createDebugExpoConfig
env: load .env.local
env: export EXPO_PUBLIC_FIREBASE_API_KEY EXPO_PUBLIC_FIREBASE_APP_ID EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN EXPO_PUBLIC_FIREBASE_EMULATOR_HOST EXPO_PUBLIC_FIREBASE_FUNCTIONS_REGION EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID EXPO_PUBLIC_FIREBASE_PROJECT_ID EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET EXPO_PUBLIC_USE_FIREBASE_EMULATORS

> Task :expo-modules-core:processDebugManifest
/Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/AndroidManifest.xml:8:9-11:45 Warning:
        meta-data#com.facebook.soloader.enabled@android:value was tagged at AndroidManifest.xml:8 to replace other declarations but no other declaration present

> Task :react-native-async-storage_async-storage:compileDebugJavaWithJavac
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.
Note: /Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/@react-native-async-storage/async-storage/android/src/javaPackage/java/com/reactnativecommunity/asyncstorage/AsyncStoragePackage.java uses unchecked or unsafe operations.
Note: Recompile with -Xlint:unchecked for details.

> Task :react-native-masked-view_masked-view:compileDebugJavaWithJavac
Note: /Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/@react-native-masked-view/masked-view/android/src/main/java/org/reactnative/maskedview/RNCMaskedViewPackage.java uses or overrides a deprecated API.
Note: Recompile with -Xlint:deprecation for details.

> Task :react-native-async-storage_async-storage:processDebugManifest
package="com.reactnativecommunity.asyncstorage" found in source AndroidManifest.xml: /Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/@react-native-async-storage/async-storage/android/src/main/AndroidManifest.xml.
Setting the namespace via the package attribute in the source AndroidManifest.xml is no longer supported, and the value is ignored.
Recommendation: remove package="com.reactnativecommunity.asyncstorage" from the source AndroidManifest.xml: /Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/@react-native-async-storage/async-storage/android/src/main/AndroidManifest.xml.

> Task :react-native-safe-area-context:processDebugManifest
package="com.th3rdwave.safeareacontext" found in source AndroidManifest.xml: /Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-safe-area-context/android/src/main/AndroidManifest.xml.
Setting the namespace via the package attribute in the source AndroidManifest.xml is no longer supported, and the value is ignored.
Recommendation: remove package="com.th3rdwave.safeareacontext" from the source AndroidManifest.xml: /Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-safe-area-context/android/src/main/AndroidManifest.xml.

> Task :react-native-worklets:configureCMakeDebug[arm64-v8a]
Checking the license for package CMake 3.22.1 in /Users/alejandro2/Library/Android/sdk/licenses
License for package CMake 3.22.1 accepted.
Preparing "Install CMake 3.22.1 v.3.22.1".
"Install CMake 3.22.1 v.3.22.1" ready.
Installing CMake 3.22.1 in /Users/alejandro2/Library/Android/sdk/cmake/3.22.1
"Install CMake 3.22.1 v.3.22.1" complete.
"Install CMake 3.22.1 v.3.22.1" finished.

> Task :react-native-safe-area-context:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-safe-area-context/android/src/main/java/com/th3rdwave/safeareacontext/SafeAreaProviderManager.kt:46:19 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-safe-area-context/android/src/main/java/com/th3rdwave/safeareacontext/SafeAreaView.kt:12:8 'class UIManagerModule : ReactContextBaseJavaModule, LifecycleEventListener, UIManager' is deprecated. This class is part of Legacy Architecture and will be removed in a future release.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-safe-area-context/android/src/main/java/com/th3rdwave/safeareacontext/SafeAreaView.kt:55:54 'class UIManagerModule : ReactContextBaseJavaModule, LifecycleEventListener, UIManager' is deprecated. This class is part of Legacy Architecture and will be removed in a future release.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-safe-area-context/android/src/main/java/com/th3rdwave/safeareacontext/SafeAreaViewShadowNode.kt:9:32 'class LayoutShadowNode : ReactShadowNodeImpl' is deprecated. This class is part of Legacy Architecture and will be removed in a future release.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-safe-area-context/android/src/main/java/com/th3rdwave/safeareacontext/SafeAreaViewShadowNode.kt:110:61 'class NativeViewHierarchyOptimizer : Any' is deprecated. This class is part of Legacy Architecture and will be removed in a future release.

> Task :react-native-screens:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/RNScreensPackage.kt:70:9 The corresponding parameter in the supertype 'BaseReactPackage' is named 'name'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/RNScreensPackage.kt:71:9 The corresponding parameter in the supertype 'BaseReactPackage' is named 'reactContext'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/RNScreensPackage.kt:83:17 'constructor(name: String, className: String, canOverrideExistingModule: Boolean, needsEagerInit: Boolean, hasConstants: Boolean, isCxxModule: Boolean, isTurboModule: Boolean): ReactModuleInfo' is deprecated. This constructor is deprecated and will be removed in the future. Use ReactModuleInfo(String, String, boolean, boolean, boolean, boolean)].
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/Screen.kt:54:77 Unchecked cast of '(CoordinatorLayout.Behavior<View!>?..CoordinatorLayout.Behavior<*>?)' to 'BottomSheetBehavior<Screen>'.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/Screen.kt:59:33 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/Screen.kt:534:14 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenContainer.kt:73:74 'static field HIDE_NOT_ALWAYS: Int' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenContainer.kt:252:18 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenContainer.kt:262:18 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenFragment.kt:225:37 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenFragment.kt:242:14 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenFragment.kt:258:22 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenFragment.kt:327:22 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStack.kt:120:14 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStackFragment.kt:179:18 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStackFragment.kt:236:31 'var targetElevation: Float' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStackFragment.kt:239:13 'fun setHasOptionsMenu(p0: Boolean): Unit' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStackFragment.kt:450:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStackFragment.kt:457:22 'fun onPrepareOptionsMenu(p0: Menu): Unit' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStackFragment.kt:460:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStackFragment.kt:465:22 'fun onCreateOptionsMenu(p0: Menu, p1: MenuInflater): Unit' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStackHeaderConfig.kt:199:14 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenStackHeaderConfig.kt:209:14 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenWindowTraits.kt:48:42 'fun replaceSystemWindowInsets(p0: Int, p1: Int, p2: Int, p3: Int): @NonNull() WindowInsetsCompat' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenWindowTraits.kt:49:39 'val systemWindowInsetLeft: Int' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenWindowTraits.kt:51:39 'val systemWindowInsetRight: Int' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreenWindowTraits.kt:52:39 'val systemWindowInsetBottom: Int' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/ScreensModule.kt:104:14 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/SearchBarView.kt:137:29 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/bottomsheet/DimmingView.kt:66:9 The corresponding parameter in the supertype 'ReactCompoundView' is named 'touchX'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/bottomsheet/DimmingView.kt:67:9 The corresponding parameter in the supertype 'ReactCompoundView' is named 'touchY'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/bottomsheet/DimmingView.kt:71:9 The corresponding parameter in the supertype 'ReactCompoundViewGroup' is named 'touchX'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/bottomsheet/DimmingView.kt:72:9 The corresponding parameter in the supertype 'ReactCompoundViewGroup' is named 'touchY'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/common/event/BaseEventEmitter.kt:12:38 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/modals/dimmingview/DimmingView.kt:64:9 The corresponding parameter in the supertype 'ReactCompoundView' is named 'touchX'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/modals/dimmingview/DimmingView.kt:65:9 The corresponding parameter in the supertype 'ReactCompoundView' is named 'touchY'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/modals/dimmingview/DimmingView.kt:69:9 The corresponding parameter in the supertype 'ReactCompoundViewGroup' is named 'touchX'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/modals/dimmingview/DimmingView.kt:70:9 The corresponding parameter in the supertype 'ReactCompoundViewGroup' is named 'touchY'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/modals/formsheet/FormSheetContentView.kt:11:8 'object ReactFeatureFlags : Any' is deprecated. Use com.facebook.react.internal.featureflags.ReactNativeFeatureFlags instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/modals/formsheet/FormSheetContentView.kt:39:33 'fun getEventDispatcher(context: ReactContext, uiManagerType: Int): EventDispatcher?' is deprecated. UIManagerType is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/modals/formsheet/FormSheetContentView.kt:44:13 'object ReactFeatureFlags : Any' is deprecated. Use com.facebook.react.internal.featureflags.ReactNativeFeatureFlags instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/modals/formsheet/FormSheetHostViewManager.kt:38:9 The corresponding parameter in the supertype 'ViewGroupManager' is named 'view'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/gamma/tabs/host/TabsHostViewManager.kt:44:9 The corresponding parameter in the supertype 'ViewGroupManager' is named 'view'. This may cause problems when calling this function with named arguments.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-screens/android/src/main/java/com/swmansion/rnscreens/safearea/SafeAreaView.kt:143:45 'fun consumeDisplayCutout(): @NonNull() WindowInsetsCompat' is deprecated. Deprecated in Java.

> Task :app:processDebugMainManifest
/Users/alejandro2/Documents/GitHub/senticode/apps/mobile/android/app/src/debug/AndroidManifest.xml:6:5-162 Warning:
        application@android:usesCleartextTraffic was tagged at AndroidManifest.xml:6 to replace other declarations but no other declaration present
/Users/alejandro2/Documents/GitHub/senticode/apps/mobile/android/app/src/debug/AndroidManifest.xml Warning:
        provider#expo.modules.filesystem.FileSystemFileProvider@android:authorities was tagged at AndroidManifest.xml:0 to replace other declarations but no other declaration present

> Task :app:expoDebugOverrideMaxSdkConflicts
---------- Expo Max Sdk Override Plugin ----------
>>> No 'android:maxSdkVersion' conflicts found
--------------------------------------------------

> Task :react-native-reanimated:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/NodesManager.kt:13:8 'class UIManagerModule : ReactContextBaseJavaModule, LifecycleEventListener, UIManager' is deprecated. This class is part of Legacy Architecture and will be removed in a future release.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/NodesManager.kt:58:95 'fun resolveCustomDirectEventName(eventName: String): String?' is deprecated.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/NodesManager.kt:232:29 'fun receiveEvent(surfaceId: Int, targetTag: Int, eventName: String, canCoalesceEvent: Boolean, customCoalesceKey: Int, params: WritableMap?, category: Int): Unit' is deprecated. Use the overload with eventTimestamp parameter instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/ReanimatedPackage.kt:5:8 'class ReactInstanceManager : Any' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/ReanimatedPackage.kt:60:73 'class ReactInstanceManager : Any' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/ReanimatedPackage.kt:61:63 'val reactNativeHost: ReactNativeHost' is deprecated. You should not use ReactNativeHost directly in the New Architecture. Use ReactHost instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/nativeProxy/EventHandler.kt:7:8 'class UIManagerModule : ReactContextBaseJavaModule, LifecycleEventListener, UIManager' is deprecated. This class is part of Legacy Architecture and will be removed in a future release.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/nativeProxy/EventHandler.kt:26:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/nativeProxy/EventHandler.kt:39:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/nativeProxy/EventHandler.kt:49:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/nativeProxy/EventHandler.kt:58:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/nativeProxy/NoopEventHandler.kt:8:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/nativeProxy/NoopEventHandler.kt:20:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/nativeProxy/NoopEventHandler.kt:29:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/nativeProxy/NoopEventHandler.kt:37:18 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-reanimated/android/src/main/java/com/swmansion/reanimated/sensor/ReanimatedSensor.kt:28:26 'val defaultDisplay: Display!' is deprecated. Deprecated in Java.

> Task :expo-modules-core:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/CppType.kt:42:23 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/JavaCallback.kt:38:10 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/JavaCallback.kt:96:45 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:49:15 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:64:30 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:66:14 'constructor(byteBuffer: ByteBuffer): NativeArrayBuffer' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:74:39 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:75:7 'constructor(byteBuffer: ByteBuffer): NativeArrayBuffer' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:80:37 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:86:47 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:92:23 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:92:43 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:95:41 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/jni/NativeArrayBuffer.kt:105:14 'constructor(byteBuffer: ByteBuffer): NativeArrayBuffer' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/types/ColorCompat.kt:115:5 This annotation is currently applied to the value parameter only, but in the future it will also be applied to field.
- To opt in to applying to both value parameter and field, add '-Xannotation-default-target=param-property' to your compiler arguments.
- To keep applying to the value parameter only, use the '@param:' annotation target.

See https://youtrack.jetbrains.com/issue/KT-73255 for more details.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/types/JSTypeConverter.kt:7:8 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/types/JSTypeConverter.kt:245:55 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/types/JSTypeConverter.kt:247:19 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/types/ReturnType.kt:6:8 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/types/ReturnType.kt:45:5 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/types/TypeConverterProvider.kt:19:8 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-modules-core/android/src/main/java/expo/modules/kotlin/types/TypeConverterProvider.kt:254:7 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.

> Task :expo-modules-core:compileDebugJavaWithJavac
Note: Some input files use or override a deprecated API.
Note: Recompile with -Xlint:deprecation for details.

> Task :expo-manifests:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-manifests/android/src/main/java/expo/modules/manifests/core/EmbeddedManifest.kt:19:16 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-manifests/android/src/main/java/expo/modules/manifests/core/EmbeddedManifest.kt:19:86 'fun getLegacyID(): String' is deprecated. Prefer scopeKey or projectId depending on use case.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-manifests/android/src/main/java/expo/modules/manifests/core/ExpoUpdatesManifest.kt:16:16 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-manifests/android/src/main/java/expo/modules/manifests/core/Manifest.kt:13:3 Deprecations and opt-ins on a method overridden from 'Any' may not be reported.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-manifests/android/src/main/java/expo/modules/manifests/core/Manifest.kt:15:12 'fun getRawJson(): JSONObject' is deprecated. Prefer to use specific field getters.

> Task :expo-dom-webview:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/@expo/dom-webview/android/src/main/java/expo/modules/webview/DomWebView.kt:200:16 'var allowFileAccessFromFileURLs: Boolean' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/@expo/dom-webview/android/src/main/java/expo/modules/webview/DomWebView.kt:254:18 'fun executeOnJavaScriptThread(runnable: Runnable): Unit' is deprecated. Use RuntimeContext.schedule instead.

> Task :react-native-gesture-handler:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-gesture-handler/android/src/main/java/com/swmansion/gesturehandler/react/RNGestureHandlerButtonViewManager.kt:799:45 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-gesture-handler/android/src/main/java/com/swmansion/gesturehandler/react/RNGestureHandlerButtonViewManager.kt:1187:45 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/react-native-gesture-handler/android/src/main/java/com/swmansion/gesturehandler/react/RNGestureHandlerDetectorView.kt:240:43 'fun getEventDispatcherForReactTag(context: ReactContext, reactTag: Int): EventDispatcher?' is deprecated. reactTag is no longer needed.

> Task :expo-dev-menu:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-menu/android/src/debug/java/com/facebook/react/devsupport/DevMenuSettingsBase.kt:6:8 'class PreferenceManager : Any' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-menu/android/src/debug/java/com/facebook/react/devsupport/DevMenuSettingsBase.kt:18:51 'class PreferenceManager : Any' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-menu/android/src/debug/java/com/facebook/react/devsupport/DevMenuSettingsBase.kt:18:69 'static fun getDefaultSharedPreferences(p0: Context!): SharedPreferences!' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-menu/android/src/debug/java/com/facebook/react/devsupport/DevMenuSettingsBase.kt:51:13 This code uses error suppression for 'NOTHING_TO_OVERRIDE'. While it might compile and work, the compiler behavior is UNSPECIFIED and WILL NOT BE PRESERVED. Please report your use case to the Kotlin issue tracker instead: https://kotl.in/issue
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-menu/android/src/debug/java/com/facebook/react/devsupport/DevMenuSettingsBase.kt:58:13 This code uses error suppression for 'NOTHING_TO_OVERRIDE'. While it might compile and work, the compiler behavior is UNSPECIFIED and WILL NOT BE PRESERVED. Please report your use case to the Kotlin issue tracker instead: https://kotl.in/issue

> Task :expo-dev-launcher:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-launcher/android/src/debug/java/expo/modules/devlauncher/DevLauncherController.kt:519:12 Call uses reflection API which is not found in compilation classpath. Make sure you have kotlin-reflect.jar in the classpath.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-launcher/android/src/debug/java/expo/modules/devlauncher/launcher/DevLauncherRecentlyOpenedAppsRegistry.kt:33:47 Unchecked cast of 'MutableMap<Any?, Any?>' to 'MutableMap<String, Any>'.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-launcher/android/src/debug/java/expo/modules/devlauncher/launcher/DevLauncherRecentlyOpenedAppsRegistry.kt:51:27 'fun getRawJson(): JSONObject' is deprecated. Prefer to use specific field getters.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-launcher/android/src/debug/java/expo/modules/devlauncher/modules/DevLauncherModule.kt:14:29 'fun getRawJson(): JSONObject' is deprecated. Prefer to use specific field getters.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-launcher/android/src/debug/java/expo/modules/devlauncher/react/DevLauncherBridgelessDevSupportManager.kt:48:38 This code uses error suppression for 'NOTHING_TO_OVERRIDE'. While it might compile and work, the compiler behavior is UNSPECIFIED and WILL NOT BE PRESERVED. Please report your use case to the Kotlin issue tracker instead: https://kotl.in/issue
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-launcher/android/src/debug/java/expo/modules/devlauncher/react/DevLauncherDevSupportManagerFactory.kt:18:16 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo-dev-launcher/android/src/main/java/expo/modules/devlauncher/DevLauncherPackage.kt:15:16 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.

> Task :expo:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/ExpoModulesPackage.kt:34:16 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/ReactActivityDelegateWrapper.kt:22:8 'class ReactInstanceManager : Any' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/ReactActivityDelegateWrapper.kt:103:16 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/ReactActivityDelegateWrapper.kt:103:43 'class ReactInstanceManager : Any' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/ReactActivityDelegateWrapper.kt:104:21 'val reactInstanceManager: ReactInstanceManager' is deprecated. Deprecated in Java.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/fetch/ExpoFetchModule.kt:15:8 'class NativeArrayBuffer : Destructible' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/fetch/ExpoFetchModule.kt:33:39 'constructor(reactContext: ReactContext): ForwardingCookieHandler' is deprecated. Use the default constructor.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/fetch/ExpoFetchModule.kt:111:27 'constructor(byteBuffer: ByteBuffer): NativeArrayBuffer' is deprecated. Use ArrayBuffer instead.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/fetch/NativeResponse.kt:44:16 This declaration overrides a deprecated member but is not marked as deprecated itself. Add the '@Deprecated' annotation or suppress the diagnostic.
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/node_modules/expo/android/src/main/java/expo/modules/fetch/NativeResponse.kt:46:11 'fun deallocate(): Unit' is deprecated. Use sharedObjectDidRelease() instead.

> Task :app:compileDebugKotlin
w: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/android/app/src/main/java/com/anonymous/mobile/MainActivity.kt:40:20 'constructor(activity: ReactActivity, mainComponentName: String, fabricEnabled: Boolean): DefaultReactActivityDelegate' is deprecated. Creating DefaultReactActivityDelegate with flags is deprecated.

[Incubating] Problems report is available at: file:///Users/alejandro2/Documents/GitHub/senticode/apps/mobile/android/build/reports/problems/problems-report.html

Deprecated Gradle features were used in this build, making it incompatible with Gradle 10.

You can use '--warning-mode all' to show the individual deprecation warnings and determine if they come from your own scripts or plugins.

For more on this, please refer to https://docs.gradle.org/9.4.1/userguide/command_line_interface.html#sec:command_line_warnings in the Gradle documentation.

BUILD SUCCESSFUL in 4m 24s
483 actionable tasks: 359 executed, 92 from cache, 32 up-to-date
Using src/app as the root directory for Expo Router.
React Compiler enabled
Starting Metro Bundler

▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
█ ▄▄▄▄▄ █▀ █▀▀██▄▄▀█▄▀  ▄▀█ ▄▄▄▄▄ █
█ █   █ █▀ ▄ ██▄█▄▀ █▀▄▀███ █   █ █
█ █▄▄▄█ █▀█ █▄   ▀▄▀▄█ ▄▄▀█ █▄▄▄█ █
█▄▄▄▄▄▄▄█▄█▄█ ▀▄▀▄█ ▀ ▀▄█ █▄▄▄▄▄▄▄█
█▄ ▄  █▄▄  ▄█▄█▄▄▄   █▀█ ▄ █▄█ █▄▀█
██▀█ ▀▄▄   ▀ ▄█▀ █▄▀▀█  ██  █    ▀█
█ ▄ ██▀▄▄█ ▀▄▀▀▀█▄▄▄▀▄▀▀ ▀▄▀██▀██ █
█ █▀█▀ ▄▀▀▀██▀ ▄ ██▄█▄▀▀█▄ ██  ▀███
██▀ ▄▄▀▄█▄█▄█▄█▄▄▄   ▄ ▄ ▄ ▀▀▄█▀▀▀█
█ ███▄▄▄▄█▀█ ▄█▀ █▄▀█▄ ▀▀ █▄█ ▀▀▄██
█   ██▀▄▄  █▄▀▀▀█▄▄▄▀▄▀█▀█ ▀ ▄██▀ █
█ █▀█ ▀▄▄█ ██▀ ▄ ▄▄ ▀▀  █████    ██
█▄██▄▄▄▄▄ ▄ █▄█▄ ▀▄▄█▀▀▀▀ ▄▄▄ ▀█▄▀█
█ ▄▄▄▄▄ █▄▄  ▄█▀▄▄█▄▀█▀█  █▄█ ▀ ▄▀█
█ █   █ █ ▄▄▄▀▀▀▄▄▄▄▀▄ ▄▀▄ ▄  ▀▀▀ █
█ █▄▄▄█ █  ██▀ ▄  █ █ ▀▄ ▀▀  ▀▄ ███
█▄▄▄▄▄▄▄█▄███▄█▄▄█▄▄████▄▄▄▄▄▄▄█▄██

› Scan the QR code above to open in a development build. (Learn more)
› Metro: exp+mobile://?__expo_url=http%3A%2F%2F192.168.1.210%3A8081
› Web: http://localhost:8081

› Using development build
› Press s │ switch to Expo Go

› Press a │ open Android
› Press i │ open iOS simulator
› Press w │ open web

› Press j │ open debugger
› Press r │ reload app
› Press m │ toggle menu
› shift+m │ more tools
› Press o │ open project code in your editor

› Press ? │ show all commands

› Installing /Users/alejandro2/Documents/GitHub/senticode/apps/mobile/android/app/build/outputs/apk/debug/app-debug.apk
› Opening exp+mobile://?__expo_url=http%3A%2F%2F192.168.1.210%3A8081 on Medium_Phone_API_37.0

› Logs for your project will appear below. Press Ctrl+C to exit.
Android Bundled 5266ms node_modules/expo-router/entry.js (1797 modules)
```

La primera vez, o después de añadir dependencias nativas:
npm run android
Para cambios habituales de TypeScript, pantallas y estilos, con la app ya instalada:
npm run mobile
Después pulsa a en la terminal para abrirla en Android.

npm run mobile no compila ni instala la aplicación.