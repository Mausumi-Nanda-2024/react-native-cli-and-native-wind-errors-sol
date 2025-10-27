# react-native-cli with nativewind configuration error solutions
This repository contains a canonical, single-file reference that documents the common error arised when NativeWind v4 used with a React Native CLI project . It collects research, exact configs, reproducible examples, common errors, and nativewind v2 & nativewind v4 migration checklist.

## Common errors and diagnostic steps

 ### 1) .plugins is not a valid Plugin property
 
In NativeWind v4, the Babel plugin is configured as a preset, not as a plugin. The official installation guide explicitly shows adding "nativewind/babel" inside the presets array of babel.config.js

#### Correct format 
<img width="995" height="321" alt="Screenshot 2025-10-22 213547" src="https://github.com/user-attachments/assets/172d8f15-a770-4f77-a3ad-6204ebd461eb" />

#### Wrong format
<img width="748" height="371" alt="Screenshot 2025-10-25 221631" src="https://github.com/user-attachments/assets/df2191e4-0866-4c76-b7e6-d891f57fc511" />


If you instead put it under plugins, Babel will complain (for example, users report the error “.plugins is not a valid Plugin property”

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
Many beginners including myself when I first started encountered a situation where NativeWind styles weren’t applying. Thinking the issue was with Babel, I moved 'nativewind/babel' from the plugins array to the presets array in v2-style configuration. However, this caused a new error:
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


