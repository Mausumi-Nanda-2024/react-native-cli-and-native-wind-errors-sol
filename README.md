# react-native-cli with nativewind configuration error solutions
This repository contains a canonical, single-file reference that documents the common error arised when NativeWind v4 used with a React Native CLI project . It collects research, exact configs, reproducible examples, common errors, and nativewind v2 & nativewind v4 migration checklist.

## Common errors and diagnostic steps

 ### 1) .plugins is not a valid Plugin property
 
In NativeWind v4, the Babel plugin is configured as a preset, not as a plugin. The official installation guide explicitly shows adding ```nativewind/babel``` inside the presets array of ```babel.config.js```

#### Correct format 
<img width="995" height="321" alt="Screenshot 2025-10-22 213547" src="https://github.com/user-attachments/assets/172d8f15-a770-4f77-a3ad-6204ebd461eb" />

#### Wrong format
<img width="748" height="371" alt="Screenshot 2025-10-25 221631" src="https://github.com/user-attachments/assets/df2191e4-0866-4c76-b7e6-d891f57fc511" />


If you instead put it under plugins, Babel will complain (for example, users report the error ```.plugins is not a valid Plugin property```

<img width="1483" height="128" alt="Screenshot 2025-10-22 212917" src="https://github.com/user-attachments/assets/dfb4783c-3316-40c4-9fb8-9f2579d5043e" />

This difference arises because NativeWind split its documentation: earlier v2 docs used the plugin approach, whereas v4 docs use a preset.

### 2) Nativewind styles are not being applied in v4

#### Clearing Common Misconceptions

Before deep diving into the solution it’s important to clear a common misconception among beginners setting up NativeWind with React Native CLI.

When many developers first configure NativeWind, they often skip reading the official documentation thoroughly. As a result, when their NativeWind styles don’t apply correctly, they assume something is wrong with the Babel configuration.

#### What Beginners Commonly Do

In NativeWind v2, the Babel configuration required adding the following inside the plugins array:
```
module.exports = {
presets: ['module:metro-react-native-babel-preset'],
plugins: ['nativewind/babel'],
};

```
 
However, in NativeWind v4, the internal architecture has changed. The configuration must now look like this:
```
module.exports = {
presets: ['module:@react-native/babel-preset', 'nativewind/babel'],
};
```
#### The Misconception
Many beginners including myself when I first started encountered a situation where NativeWind styles weren’t applying. Thinking the issue was with Babel, I moved ```nativewind/babel``` from the plugins array to the presets array in v2-style configuration. However, this caused a new error:
``` Error: .plugin is not a valid Plugin property ```
This happens because in NativeWind v4, nativewind/babel is designed to function as a preset, not a plugin. Using it as a plugin leads to incorrect parsing by Babel, which expects a preset configuration.

#### The Real Cause of Styles Not Applying

The real reason styles don’t apply in NativeWind v4 is not the Babel configuration, but rather a missing import of the global.css file.

Create a file named global.css in your root project directory containing code like this:
```
@tailwind base;
@tailwind components;
@tailwind utilities;

```
Because the new architecture in v4 relies on CSS interop powered by react-native-css-interop, and this requires importing your generated global.css file inside your root file (e.g., App.tsx).
Example:
```
// App.tsx
import "./global.css";


export default function App() {
return <YourApp />;
}
```
Without this import, NativeWind cannot inject Tailwind styles into your React Native app, even if your Babel configuration is correct.

### 3) "Missing semicolon" in node_modules/react-native/index.js

Cause: Babel did not apply the required transforms for React Native source code. This happens when the Babel preset is wrong, missing, or not resolved.

Why changing preset fixed it: Using ```module:@react-native/babel-preset``` or ```@react-native/babel-preset``` supplies necessary transforms. If ```module:metro-react-native-babel-preset``` is not installed or mismatched, Babel leaves modern syntax untransformed. Metro then fails parsing with syntax errors.

Fix:
```
npm install -D @react-native/babel-preset
# or
yarn add -D @react-native/babel-preset
```
Use in babel.config.js:
```
module.exports = {
  presets: ['module:@react-native/babel-preset', 'nativewind/babel'],
}
```
Then reset Metro cache:
``` npx react-native start --reset-cache ```

### 4) Cannot find module 'react-native-worklets/plugin'

Cause: react-native-worklets is a peer dependency. The package referencing it expects you to install it.

Fix:
you install as per the latest version published
``` npm install react-native-worklets@^0.6.1 ```
Ensure versions of 'react-native-reanimated' and RN are compatible.

### 5)  Metro works but Android fails (or vice versa)

Cause: Version mismatch between JS-side plugins and native JSI/native bindings. 

Examples:

i) Installing ```react-native-worklets``` fixed Metro, but Android lacked compatible native modules.

ii) Removing ```react-native-worklets``` allowed Android to build but broke Metro.

Fix: Align versions. Update the whole toolchain to compatible releases:

```
npm install nativewind@latest react-native-reanimated@latest react-native-safe-area-context@5.4.0
npm install -D tailwindcss@^3.4.17 prettier-plugin-tailwindcss@^0.5.11
npm install react-native-worklets@^0.6.1
```
Then clean build artifacts and reinstall:
```
rm -rf node_modules android/.gradle android/app/build
npm install
npx react-native run-android
npx react-native start --reset-cache
```

## Worklets: what they are and why NativeWind uses them

Worklet:

i) A function compiled to run on the UI thread or alternate JS runtime for low-latency tasks.

ii) Common in ```react-native-reanimated ``` and related animation libs.

iii) Marked by a ``` 'worklet'``` directive or compiled by a Babel plugin.

```react-native-worklets``` plugin:

i) Babel plugin: ```react-native-worklets/plugin``` transforms functions for the worklet runtime.

ii) Native runtime: ships JSI/native bindings that must match React Native and Reanimated versions.

#### Why NativeWind v4 references worklets

i) Internal modules like ```react-native-css-interop``` use worklets to update styles ideally on the UI thread.

iii) The NativeWind toolchain expects ```react-native-worklets``` as a peer dependency. If it is missing, Babel errors arise.

#### Symptoms when missing

i) Metro error: ```Cannot find module 'react-native-worklets/plugin'```

ii) Installing ```react-native-worklets``` fixes the Babel error but may expose native ABI mismatches on Android if Reanimated or RN versions differ.

## High-level difference: v2 vs v4

#### V2

Architecture: Babel plugin driven.

Setup: ```nativewind/babel``` in ```plugins``` and optional ```postcss.config.js``` for web builds.

Behavior: Babel plugin transforms ```className``` strings into RN style objects (or helper calls) during JS transform.

Pros: Works without explicit CSS import in many setups.

Cons: More runtime overhead, less type safety, relies on Babel plugin configuration.

#### V4

Architecture: Compiler + Tailwind extraction pipeline.

Setup: ```nativewind/babel``` is typically used as a ```preset``` and the tool expects a CSS entry point such as ```global.css``` imported in ```App.tsx```.

Behavior: NativeWind compiles Tailwind directives into an RN JS style registry. Importing ```global.css``` triggers the compiler to emit and register styles for runtime lookup.

Pros: Faster startup, smaller bundle, stronger type safety, build-time extraction.

Cons: Must import ```global.css```. Misconfiguring Babel or omitting the CSS import leads to missing styles.

#### In short:

v2: Used ```plugins``` and ```postcss.config.js```.

v4: Uses ```presets``` and requires ```global.css``` import.

## Some important configuration file detail breakdown ( why this file matters )

```global.css (v4)```:
```
@tailwind base;
@tailwind components;
@tailwind utilities;
```
i) Contains Tailwind directives that the NativeWind compiler processes.

ii) Must be imported once in the app entry (e.g. ```App.tsx```). The import triggers Metro to include the generated stylesheet JS module and registers the style registry.

iii) Without this import, ```className``` has no mapped RN style objects.

```postcss.config.js```:

v2: Often required because the project-level PostCSS pipeline runs ```tailwindcss``` and ```autoprefixer```.

v4: Not required for pure React Native CLI mobile apps because the NativeWind compiler includes an internal PostCSS/Tailwind runner. Keep it only if:

  i) You target web (React Native Web, Expo Web) and need real CSS output.

  ii) You need custom PostCSS plugins (e.g., ```postcss-import```, ```cssnano```).

Example (web):

```
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

```babel.config.js```:

v2: ```nativewind/babel``` often lived in ```plugins```.

v4: Official docs show ```nativewind/babel``` in ```presets```. Putting it in ```plugins``` can lead to errors.

```tailwind.config.js```:

Include ```content``` globs for all RN source files so the compiler extracts class names.

Use NativeWind preset when recommended.

Example:
```
import { nativewind } from 'nativewind/tailwind'

export default {
  content: ['./App.{js,ts,tsx,jsx}', './src/**/*.{js,ts,tsx,jsx}'],
  presets: [nativewind],
}
```


## 🧩 Credits

This research and documentation are based on official references from:  
- [NativeWind Documentation](https://www.nativewind.dev/)  
- [React Native Documentation](https://reactnative.dev/)

Thanks to the NativeWind maintainers for their detailed migration guide and continuous support for React Native developers.
