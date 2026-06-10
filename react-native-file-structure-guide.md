# React Native File Structure & Architecture Guide

A comprehensive guide to organizing React Native projects, covering initial setup, directory structures, architectural patterns, and different repository types.

---

## Table of Contents

1. [Initial Setup Files](#initial-setup-files)
2. [Basic Project Structure](#basic-project-structure)
3. [Architectural Patterns](#architectural-patterns)
4. [Mono Repo Structure](#mono-repo-structure)
5. [Multi Repo Structure](#multi-repo-structure)
6. [File-by-File Breakdown](#file-by-file-breakdown)
7. [Best Practices](#best-practices)

---

## Initial Setup Files

When you initialize a new React Native project, several configuration files are created. Here's what each does:

### Root Level Configuration Files

```
project-root/
├── .babelrc                 # Babel transpiler configuration
├── .eslintrc.js            # ESLint linting rules
├── .eslintignore           # Files/dirs ESLint should ignore
├── .prettierrc              # Prettier code formatter config
├── .prettierignore          # Files/dirs Prettier should ignore
├── .gitignore              # Git ignore patterns
├── .git/                   # Git repository metadata
├── .watchmanconfig         # Watchman (file watcher) config
├── .editorconfig           # Cross-editor coding style config
├── .env.example            # Environment variables template
├── .env.local              # Local env variables (not committed)
├── .ruby-version           # Ruby version for CocoaPods
├── .node-version           # Node version specification
├── app.json                # Expo app config (if using Expo)
├── package.json            # Project dependencies & scripts
├── package-lock.json       # Locked dependency versions (npm)
├── yarn.lock               # Locked dependency versions (yarn)
├── pnpm-lock.yaml          # Locked dependency versions (pnpm)
├── tsconfig.json           # TypeScript configuration
├── jest.config.js          # Jest testing framework config
├── metro.config.js         # Metro bundler configuration
├── eas.json                # Expo Application Services config
└── README.md               # Project documentation
```

### Android-Specific Setup Files

```
android/
├── app/
│   ├── build.gradle        # App-level build configuration
│   ├── proguard-rules.pro  # ProGuard/R8 obfuscation rules
│   └── src/
│       └── AndroidManifest.xml
├── build.gradle            # Project-level build config
├── gradle.properties       # Gradle properties
├── local.properties        # Local SDK paths (not committed)
└── settings.gradle         # Gradle module settings
```

### iOS-Specific Setup Files

```
ios/
├── Podfile                 # CocoaPods dependency file
├── Podfile.lock            # Locked Pod versions
├── project-name.xcodeproj/
│   ├── project.pbxproj     # Xcode project file
│   └── xcshareddata/
│       └── xcschemes/
├── project-name.xcworkspace/
│   └── contents.xcworkspace
└── Info.plist              # App info (some configs moved to Xcode)
```

---

## Basic Project Structure

### Standard Single Repository Structure

```
project-root/
│
├── __tests__/                    # Unit & integration tests
│   ├── components/
│   │   ├── Button.test.tsx
│   │   └── Header.test.tsx
│   ├── screens/
│   │   └── HomeScreen.test.tsx
│   ├── utils/
│   │   └── helpers.test.ts
│   └── setup.ts                  # Test setup & mocks
│
├── src/                          # Source code
│   ├── api/                      # API calls & services
│   │   ├── client.ts             # HTTP client setup
│   │   ├── auth.ts               # Auth API endpoints
│   │   ├── user.ts               # User API endpoints
│   │   └── index.ts              # Export API functions
│   │
│   ├── components/               # Reusable UI components
│   │   ├── Button/
│   │   │   ├── Button.tsx        # Component
│   │   │   ├── Button.styles.ts  # Styles
│   │   │   ├── Button.props.ts   # TypeScript types
│   │   │   └── index.ts          # Export
│   │   ├── Header/
│   │   │   ├── Header.tsx
│   │   │   ├── Header.styles.ts
│   │   │   └── index.ts
│   │   ├── Card/
│   │   └── ...
│   │
│   ├── screens/                  # Screen components (pages)
│   │   ├── HomeScreen.tsx
│   │   ├── ProfileScreen.tsx
│   │   ├── SettingsScreen.tsx
│   │   └── AuthScreen.tsx
│   │
│   ├── navigation/               # Navigation config
│   │   ├── RootNavigator.tsx     # Main navigation structure
│   │   ├── AuthNavigator.tsx     # Auth stack
│   │   ├── AppNavigator.tsx      # App stack
│   │   ├── types.ts              # Navigation type definitions
│   │   └── linking.ts            # Deep linking config
│   │
│   ├── store/                    # State management
│   │   ├── store.ts              # Redux store setup
│   │   ├── slices/               # Redux slices (if using Redux Toolkit)
│   │   │   ├── authSlice.ts
│   │   │   ├── userSlice.ts
│   │   │   └── uiSlice.ts
│   │   ├── reducers/             # Reducers (if not using Toolkit)
│   │   ├── actions/              # Actions
│   │   ├── selectors/            # Selectors
│   │   ├── middleware/           # Custom middleware
│   │   └── hooks.ts              # Redux hooks (useAppDispatch, etc.)
│   │
│   ├── hooks/                    # Custom React hooks
│   │   ├── useAuth.ts
│   │   ├── useFetch.ts
│   │   ├── useLocalStorage.ts
│   │   └── useForm.ts
│   │
│   ├── utils/                    # Utility functions
│   │   ├── date.ts               # Date formatting
│   │   ├── string.ts             # String utilities
│   │   ├── validation.ts         # Form validation
│   │   ├── logger.ts             # Logging utility
│   │   ├── constants.ts          # App constants
│   │   ├── colors.ts             # Color palette
│   │   └── formatting.ts         # Data formatting
│   │
│   ├── types/                    # TypeScript type definitions
│   │   ├── index.ts              # Re-export all types
│   │   ├── user.ts               # User types
│   │   ├── api.ts                # API types
│   │   ├── navigation.ts         # Navigation types
│   │   └── common.ts             # Common shared types
│   │
│   ├── services/                 # Business logic services
│   │   ├── authService.ts        # Auth logic
│   │   ├── userService.ts        # User logic
│   │   ├── storageService.ts     # Local storage logic
│   │   └── analyticsService.ts   # Analytics logic
│   │
│   ├── context/                  # React Context (if used)
│   │   ├── AuthContext.tsx
│   │   ├── ThemeContext.tsx
│   │   └── UserContext.tsx
│   │
│   ├── config/                   # Configuration files
│   │   ├── env.ts                # Environment variables
│   │   ├── api.ts                # API configuration
│   │   └── app.ts                # App configuration
│   │
│   ├── assets/                   # Static assets
│   │   ├── images/
│   │   │   ├── logo.png
│   │   │   ├── icons/
│   │   │   └── illustrations/
│   │   ├── fonts/
│   │   │   ├── Roboto-Regular.ttf
│   │   │   └── Roboto-Bold.ttf
│   │   └── data/
│   │       └── mock-data.json
│   │
│   ├── theme/                    # Theme & styling
│   │   ├── colors.ts
│   │   ├── typography.ts
│   │   ├── spacing.ts
│   │   ├── shadows.ts
│   │   └── theme.ts              # Theme object
│   │
│   ├── App.tsx                   # Root component
│   └── index.js                  # Entry point
│
├── android/                      # Android native code
├── ios/                          # iOS native code
│
├── docs/                         # Documentation
│   ├── ARCHITECTURE.md           # Architecture decisions
│   ├── SETUP.md                  # Setup instructions
│   ├── API.md                    # API documentation
│   └── CONTRIBUTING.md           # Contribution guidelines
│
├── scripts/                      # Build & utility scripts
│   ├── prebuild.js
│   ├── postbuild.js
│   └── generate-types.js
│
├── .babelrc
├── .eslintrc.js
├── .prettierrc
├── .gitignore
├── .watchmanconfig
├── app.json
├── package.json
├── tsconfig.json
├── jest.config.js
├── metro.config.js
├── eas.json
└── README.md
```

---

## Architectural Patterns

### 1. Clean Architecture (Recommended)

Separates concerns into layers:

```
src/
├── presentation/          # UI & screens
│   ├── screens/
│   ├── components/
│   ├── navigation/
│   └── theme/
│
├── domain/               # Business logic & entities
│   ├── entities/
│   ├── repositories/     # Repository interfaces
│   └── usecases/         # Use cases
│
└── data/                 # Data access & external services
    ├── datasources/      # API, local storage, etc.
    ├── repositories/     # Repository implementations
    └── models/           # Data models (DTOs)
```

### 2. Feature-Based Architecture

Organizes by features with co-located files:

```
src/
├── features/
│   ├── auth/
│   │   ├── screens/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   ├── types.ts
│   │   ├── useAuth.ts
│   │   └── index.ts
│   │
│   ├── user/
│   │   ├── screens/
│   │   ├── components/
│   │   ├── services/
│   │   └── index.ts
│   │
│   └── dashboard/
│       └── ...
│
├── shared/               # Shared across features
│   ├── components/
│   ├── hooks/
│   ├── utils/
│   ├── types/
│   └── constants/
│
└── core/                 # App core (navigation, store, etc.)
    ├── navigation/
    ├── store/
    ├── services/
    └── config/
```

### 3. MVC Architecture

```
src/
├── models/               # Data structures
│   ├── User.ts
│   └── Post.ts
│
├── views/                # UI components
│   ├── screens/
│   └── components/
│
├── controllers/          # Business logic
│   ├── AuthController.ts
│   └── UserController.ts
│
└── services/
    └── api.ts
```

### 4. Redux-based Architecture

```
src/
├── store/
│   ├── index.ts          # Store creation
│   ├── slices/           # Feature slices
│   │   ├── auth/
│   │   ├── user/
│   │   └── ui/
│   ├── middleware/
│   ├── selectors/
│   └── hooks.ts
│
├── components/
├── screens/
└── ...
```

---

## Mono Repo Structure

### Yarn Workspaces Setup

```
monorepo-root/
│
├── packages/
│   ├── app/              # React Native app
│   │   ├── src/
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── shared/           # Shared code
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── utils/
│   │   │   ├── types/
│   │   │   └── hooks/
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── api/              # Shared API client
│   │   ├── src/
│   │   │   ├── client.ts
│   │   │   ├── endpoints/
│   │   │   └── types.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   ├── ui-kit/           # Design system
│   │   ├── src/
│   │   │   ├── Button/
│   │   │   ├── Card/
│   │   │   ├── Input/
│   │   │   ├── theme.ts
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   │
│   └── web/              # Web React app (optional)
│       ├── src/
│       ├── package.json
│       └── tsconfig.json
│
├── tools/                # Build tools & scripts
│   ├── eslint-config/
│   ├── typescript-config/
│   └── jest-config/
│
├── docs/
│   ├── ARCHITECTURE.md
│   ├── SETUP.md
│   └── CONTRIBUTING.md
│
├── package.json          # Root workspace config
├── yarn.lock
├── tsconfig.json         # Root TypeScript config
├── .eslintrc.js          # Root ESLint config
├── .prettierrc
├── jest.config.js        # Root Jest config
├── README.md
└── .gitignore
```

### Root package.json for Yarn Workspaces

```json
{
  "name": "monorepo-root",
  "private": true,
  "workspaces": [
    "packages/*",
    "tools/*"
  ],
  "devDependencies": {
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "typescript": "^5.0.0",
    "jest": "^29.0.0"
  },
  "scripts": {
    "dev": "yarn workspace @monorepo/app start",
    "build": "yarn workspaces foreach -A run build",
    "lint": "yarn workspaces foreach -A run lint",
    "test": "yarn workspaces foreach -A run test",
    "type-check": "yarn workspaces foreach -A run type-check"
  }
}
```

### Individual Package package.json

```json
{
  "name": "@monorepo/app",
  "version": "1.0.0",
  "description": "React Native app",
  "dependencies": {
    "@monorepo/shared": "*",
    "@monorepo/api": "*",
    "@monorepo/ui-kit": "*",
    "react-native": "^0.72.0",
    "react": "^18.0.0"
  },
  "devDependencies": {
    "@monorepo/typescript-config": "*",
    "@monorepo/eslint-config": "*"
  },
  "scripts": {
    "start": "react-native start",
    "android": "react-native run-android",
    "ios": "react-native run-ios",
    "build": "tsc --noEmit",
    "lint": "eslint src/",
    "test": "jest"
  }
}
```

### Lerna Configuration (Alternative)

```
monorepo-root/
├── lerna.json
├── packages/
│   ├── app/
│   ├── shared/
│   └── api/
└── package.json
```

**lerna.json:**
```json
{
  "version": "0.0.0",
  "npmClient": "yarn",
  "useWorkspaces": true,
  "packages": [
    "packages/*"
  ]
}
```

---

## Multi Repo Structure

### Separate App Repository

```
react-native-app/
├── src/
│   ├── screens/
│   ├── components/
│   ├── navigation/
│   ├── store/
│   ├── hooks/
│   ├── utils/
│   ├── types/
│   ├── services/
│   ├── App.tsx
│   └── index.js
├── android/
├── ios/
├── __tests__/
├── docs/
├── package.json
├── tsconfig.json
└── .eslintrc.js
```

### Separate API Client Repository

```
react-native-api-client/
├── src/
│   ├── client.ts          # HTTP client
│   ├── endpoints/
│   │   ├── auth.ts
│   │   ├── user.ts
│   │   └── posts.ts
│   ├── types/
│   │   ├── api.ts
│   │   ├── responses.ts
│   │   └── errors.ts
│   ├── interceptors.ts
│   ├── config.ts
│   └── index.ts
├── __tests__/
├── package.json
├── tsconfig.json
└── README.md
```

### Separate UI Component Library

```
react-native-ui-kit/
├── src/
│   ├── components/
│   │   ├── Button/
│   │   ├── Input/
│   │   ├── Card/
│   │   ├── Modal/
│   │   └── index.ts
│   ├── theme/
│   │   ├── colors.ts
│   │   ├── typography.ts
│   │   ├── spacing.ts
│   │   └── index.ts
│   ├── hooks/
│   └── index.ts
├── __tests__/
├── package.json
├── tsconfig.json
└── README.md
```

---

## File-by-File Breakdown

### .babelrc / babel.config.js

Configures Babel transpiler for JavaScript/TypeScript:

```javascript
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: [
    '@babel/plugin-transform-export-namespace-from',
    '@babel/plugin-proposal-decorators',
    'react-native-reanimated/plugin',
  ],
};
```

### .eslintrc.js

Defines code quality rules:

```javascript
module.exports = {
  root: true,
  extends: [
    '@react-native-community',
    'prettier',
  ],
  parser: '@typescript-eslint/parser',
  plugins: ['@typescript-eslint', 'react-hooks'],
  rules: {
    'no-console': 'warn',
    'react-hooks/rules-of-hooks': 'error',
    '@typescript-eslint/no-unused-vars': 'warn',
  },
};
```

### .prettierrc

Code formatting configuration:

```json
{
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "singleQuote": true,
  "trailingComma": "es5",
  "bracketSpacing": true,
  "arrowParens": "always"
}
```

### .watchmanconfig

File watcher configuration for Metro:

```json
{
  "ignore_dirs": ["node_modules", ".git", "ios", "android"]
}
```

### .editorconfig

Cross-editor style consistency:

```
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false
```

### .gitignore

Git ignore patterns:

```
# Dependencies
node_modules/
npm-debug.log
yarn-error.log
pnpm-debug.log

# React Native
.metro-bundler-cache/
.rn-metadata.json
/ios/Pods/
/ios/Podfile.lock
android/app/debug.keystore
android/.gradle/
android/local.properties

# IDE
.vscode/
.idea/
*.swp
*.swo
*~
.DS_Store

# Environment
.env
.env.local
.env.*.local

# Build outputs
build/
dist/
.expo/
.expo-shared/
```

### app.json (Expo)

Expo app configuration:

```json
{
  "expo": {
    "name": "MyApp",
    "slug": "my-app",
    "version": "1.0.0",
    "assetBundlePatterns": [
      "**/*"
    ],
    "ios": {
      "supportsTabletMode": true,
      "bundleIdentifier": "com.example.myapp"
    },
    "android": {
      "package": "com.example.myapp",
      "versionCode": 1
    },
    "web": {
      "favicon": "./assets/favicon.png"
    }
  }
}
```

### package.json

Project metadata and dependencies:

```json
{
  "name": "@example/react-native-app",
  "version": "1.0.0",
  "description": "A React Native application",
  "main": "index.js",
  "scripts": {
    "start": "react-native start",
    "android": "react-native run-android",
    "ios": "react-native run-ios",
    "build:android": "cd android && ./gradlew assembleRelease",
    "build:ios": "cd ios && xcodebuild -workspace MyApp.xcworkspace -scheme MyApp -configuration Release",
    "lint": "eslint src/ --ext .ts,.tsx",
    "format": "prettier --write \"src/**/*.{ts,tsx,json,css}\"",
    "test": "jest",
    "test:watch": "jest --watch",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-native": "^0.72.0",
    "@react-navigation/native": "^6.1.0",
    "@react-navigation/bottom-tabs": "^6.5.0",
    "@react-navigation/stack": "^6.3.0",
    "@reduxjs/toolkit": "^1.9.0",
    "react-redux": "^8.1.0",
    "axios": "^1.4.0",
    "zustand": "^4.3.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.0",
    "@types/react-native": "^0.72.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "prettier": "^3.0.0",
    "typescript": "^5.1.0",
    "jest": "^29.5.0",
    "@testing-library/react-native": "^12.0.0"
  }
}
```

### tsconfig.json

TypeScript configuration:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020"],
    "jsx": "react-native",
    "module": "commonjs",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "resolveJsonModule": true,
    "baseUrl": "./src",
    "paths": {
      "@/*": ["./*"],
      "@components/*": ["./components/*"],
      "@screens/*": ["./screens/*"],
      "@hooks/*": ["./hooks/*"],
      "@utils/*": ["./utils/*"],
      "@types/*": ["./types/*"],
      "@services/*": ["./services/*"],
      "@store/*": ["./store/*"],
      "@api/*": ["./api/*"],
      "@config/*": ["./config/*"],
      "@assets/*": ["./assets/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

### jest.config.js

Testing framework configuration:

```javascript
module.exports = {
  preset: 'react-native',
  moduleFileExtensions: ['ts', 'tsx', 'js', 'jsx', 'json', 'node'],
  setupFilesAfterEnv: ['<rootDir>/__tests__/setup.ts'],
  testPathIgnorePatterns: ['/node_modules/', '/android/', '/ios/'],
  transformIgnorePatterns: [
    'node_modules/(?!(react-native|@react-navigation)/)',
  ],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '^@components/(.*)$': '<rootDir>/src/components/$1',
    '^@screens/(.*)$': '<rootDir>/src/screens/$1',
    '^@hooks/(.*)$': '<rootDir>/src/hooks/$1',
    '^@utils/(.*)$': '<rootDir>/src/utils/$1',
    '^@types/(.*)$': '<rootDir>/src/types/$1',
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/index.ts',
  ],
};
```

### metro.config.js

Metro bundler configuration:

```javascript
const { getDefaultConfig } = require('@react-native/metro-config');

const config = getDefaultConfig(__dirname);

config.resolver.extraNodeModules = {
  // Alias modules for monorepo
};

module.exports = config;
```

### eas.json (Expo Application Services)

Build & deployment configuration:

```json
{
  "build": {
    "preview": {
      "android": {
        "buildType": "apk"
      },
      "ios": {
        "buildType": "simulator"
      }
    },
    "production": {
      "android": {
        "buildType": "aab"
      },
      "ios": {
        "buildType": "archive"
      }
    }
  },
  "submit": {
    "production": {
      "ios": {
        "ascAppId": "1234567890"
      },
      "android": {
        "serviceAccountKeyPath": "./service-account.json"
      }
    }
  }
}
```

### Podfile (iOS CocoaPods)

iOS dependency management:

```ruby
# Podfile
platform :ios, '13.0'

post_install do |installer|
  react_native_post_install(
    installer,
    config[:react_native_path],
    :mac_catalyst_enabled => false
  )
end

target 'MyApp' do
  config = use_native_modules!
  use_react_native!
end
```

### build.gradle (Android)

Android project-level configuration:

```gradle
// build.gradle (Project)
buildscript {
    ext {
        buildToolsVersion = "33.0.0"
        minSdkVersion = 21
        compileSdkVersion = 33
        targetSdkVersion = 33
    }
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath("com.android.tools.build:gradle:7.3.1")
    }
}
```

### android/app/build.gradle

Android app-level configuration:

```gradle
// android/app/build.gradle
apply plugin: "com.android.application"

android {
    compileSdkVersion rootProject.ext.compileSdkVersion

    defaultConfig {
        applicationId "com.example.myapp"
        minSdkVersion rootProject.ext.minSdkVersion
        targetSdkVersion rootProject.ext.targetSdkVersion
        versionCode 1
        versionName "1.0.0"
    }

    signingConfigs {
        release {
            keyAlias keystoreProperties['keyAlias']
            keyPassword keystoreProperties['keyPassword']
            storeFile file(keystoreProperties['storeFile'])
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled enableProguardInReleaseBuilds
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro"
        }
    }
}

dependencies {
    implementation project(':react-native-gesture-handler')
    // ... other dependencies
}
```

### src/App.tsx

Root application component:

```typescript
import React, { useEffect } from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { useAppSelector } from '@store/hooks';
import RootNavigator from '@navigation/RootNavigator';

const App: React.FC = () => {
  useEffect(() => {
    // App initialization logic
  }, []);

  return (
    <NavigationContainer>
      <RootNavigator />
    </NavigationContainer>
  );
};

export default App;
```

### src/index.js

Entry point for the app:

```javascript
import { AppRegistry } from 'react-native';
import App from './App';
import { name as appName } from '../app.json';

AppRegistry.registerComponent(appName, () => App);
```

---

## Best Practices

### 1. File Organization

- **Co-locate related files**: Keep component, styles, and tests together
- **Use index.ts for exports**: Simplify imports
- **One component per file**: Easier to find and test
- **Group by feature**: Organize screens and services by domain

### 2. Naming Conventions

```typescript
// Components (PascalCase)
LoginScreen.tsx
UserProfile.tsx
CustomButton.tsx

// Hooks (camelCase with 'use' prefix)
useAuth.ts
useFetch.ts
useForm.ts

// Utils & helpers (camelCase)
stringUtils.ts
dateFormatter.ts
validation.ts

// Types/interfaces (PascalCase)
User.ts
ApiResponse.ts
NavigationProps.ts

// Constants (UPPER_CASE)
API_BASE_URL
DEFAULT_TIMEOUT
MAX_RETRIES

// Directories (kebab-case)
api-client/
ui-components/
shared-hooks/
```

### 3. Path Aliases

Use tsconfig paths to avoid deep imports:

```typescript
// Instead of:
import Button from '../../../components/Button';

// Use:
import Button from '@components/Button';
```

### 4. Environment Configuration

```typescript
// src/config/env.ts
const ENV = {
  API_URL: process.env.REACT_APP_API_URL || 'https://api.example.com',
  ENV: process.env.NODE_ENV || 'development',
  DEBUG: process.env.DEBUG === 'true',
};

export default ENV;
```

### 5. API Organization

```typescript
// src/api/client.ts
import axios from 'axios';
import ENV from '@config/env';

const client = axios.create({
  baseURL: ENV.API_URL,
  timeout: 10000,
});

export default client;

// src/api/user.ts
import client from './client';

export const fetchUser = (id: string) =>
  client.get(`/users/${id}`);

export const updateUser = (id: string, data: any) =>
  client.put(`/users/${id}`, data);
```

### 6. Type Safety

Create separate type definition files:

```typescript
// src/types/user.ts
export interface User {
  id: string;
  email: string;
  name: string;
  avatar?: string;
}

export type UserRole = 'admin' | 'user' | 'guest';

// src/types/api.ts
export interface ApiResponse<T> {
  success: boolean;
  data: T;
  error?: string;
}
```

### 7. Component Structure

```typescript
// src/components/Button/Button.tsx
import React from 'react';
import { TouchableOpacity, Text } from 'react-native';
import styles from './Button.styles';
import { ButtonProps } from './Button.props';

const Button: React.FC<ButtonProps> = ({
  title,
  onPress,
  variant = 'primary',
}) => {
  return (
    <TouchableOpacity
      style={[styles.button, styles[variant]]}
      onPress={onPress}
    >
      <Text style={styles.text}>{title}</Text>
    </TouchableOpacity>
  );
};

export default Button;

// src/components/Button/Button.props.ts
export interface ButtonProps {
  title: string;
  onPress: () => void;
  variant?: 'primary' | 'secondary';
}

// src/components/Button/Button.styles.ts
import { StyleSheet } from 'react-native';
import { theme } from '@theme/theme';

export default StyleSheet.create({
  button: {
    paddingVertical: 12,
    paddingHorizontal: 16,
    borderRadius: 8,
    alignItems: 'center',
  },
  primary: {
    backgroundColor: theme.colors.primary,
  },
  secondary: {
    backgroundColor: theme.colors.secondary,
  },
  text: {
    color: theme.colors.white,
    fontSize: 16,
    fontWeight: '600',
  },
});

// src/components/Button/index.ts
export { default } from './Button';
export type { ButtonProps } from './Button.props';
```

### 8. Custom Hooks Pattern

```typescript
// src/hooks/useAuth.ts
import { useCallback } from 'react';
import { useAppDispatch, useAppSelector } from '@store/hooks';
import { login, logout } from '@store/slices/authSlice';

export const useAuth = () => {
  const dispatch = useAppDispatch();
  const { user, isLoading } = useAppSelector(state => state.auth);

  const handleLogin = useCallback(
    async (email: string, password: string) => {
      try {
        await dispatch(login({ email, password })).unwrap();
      } catch (error) {
        console.error('Login failed:', error);
      }
    },
    [dispatch],
  );

  const handleLogout = useCallback(() => {
    dispatch(logout());
  }, [dispatch]);

  return {
    user,
    isLoading,
    login: handleLogin,
    logout: handleLogout,
  };
};
```

### 9. State Management Pattern (Redux Toolkit)

```typescript
// src/store/slices/authSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { authAPI } from '@api/auth';

interface AuthState {
  user: User | null;
  token: string | null;
  isLoading: boolean;
  error: string | null;
}

export const login = createAsyncThunk(
  'auth/login',
  async ({ email, password }: LoginPayload) => {
    const response = await authAPI.login(email, password);
    return response.data;
  },
);

const authSlice = createSlice({
  name: 'auth',
  initialState: { user: null, token: null, isLoading: false, error: null },
  reducers: {
    logout: (state) => {
      state.user = null;
      state.token = null;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(login.pending, (state) => {
        state.isLoading = true;
      })
      .addCase(login.fulfilled, (state, action) => {
        state.isLoading = false;
        state.user = action.payload.user;
        state.token = action.payload.token;
      })
      .addCase(login.rejected, (state, action) => {
        state.isLoading = false;
        state.error = action.error.message || 'Login failed';
      });
  },
});

export const { logout } = authSlice.actions;
export default authSlice.reducer;
```

### 10. Navigation Structure

```typescript
// src/navigation/types.ts
import { NavigatorScreenParams } from '@react-navigation/native';

export type RootStackParamList = {
  Auth: NavigatorScreenParams<AuthStackParamList>;
  App: NavigatorScreenParams<AppStackParamList>;
  Splash: undefined;
};

export type AuthStackParamList = {
  Login: undefined;
  SignUp: undefined;
  ForgotPassword: undefined;
};

export type AppStackParamList = {
  Home: undefined;
  Profile: { userId: string };
  Settings: undefined;
};

// src/navigation/RootNavigator.tsx
import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { useAppSelector } from '@store/hooks';
import AuthNavigator from './AuthNavigator';
import AppNavigator from './AppNavigator';

const Stack = createNativeStackNavigator<RootStackParamList>();

const RootNavigator: React.FC = () => {
  const { token } = useAppSelector(state => state.auth);

  return (
    <Stack.Navigator screenOptions={{ headerShown: false }}>
      {token ? (
        <Stack.Screen name="App" component={AppNavigator} />
      ) : (
        <Stack.Screen name="Auth" component={AuthNavigator} />
      )}
    </Stack.Navigator>
  );
};

export default RootNavigator;
```

### 11. Testing Structure

```typescript
// __tests__/components/Button.test.tsx
import React from 'react';
import { render, fireEvent } from '@testing-library/react-native';
import Button from '@components/Button';

describe('Button Component', () => {
  it('should render with correct title', () => {
    const { getByText } = render(
      <Button title="Press Me" onPress={() => {}} />,
    );
    expect(getByText('Press Me')).toBeTruthy();
  });

  it('should call onPress when pressed', () => {
    const onPress = jest.fn();
    const { getByText } = render(
      <Button title="Press" onPress={onPress} />,
    );
    fireEvent.press(getByText('Press'));
    expect(onPress).toHaveBeenCalled();
  });
});
```

### 12. CI/CD Configuration

**.github/workflows/test.yml:**
```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
      - run: npm install
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test
```

### 13. Documentation

Create clear documentation:

```markdown
# ARCHITECTURE.md

## Overview
This document describes the architecture of the React Native application.

## Directory Structure
...

## Key Patterns
- Feature-based architecture
- Redux Toolkit for state management
- Custom hooks for logic reuse

## Adding New Features
1. Create feature directory in `src/features/`
2. Add screens, components, and services
3. Update navigation if needed
4. Export from feature index

## State Management
We use Redux Toolkit with the following patterns...
```

---

## Summary

A well-organized React Native project structure:

✅ **Scales** with your team
✅ **Maintains** consistency
✅ **Enables** code reuse
✅ **Facilitates** testing
✅ **Supports** both single and mono repos
✅ **Provides** clear separation of concerns

Choose the architecture that fits your project needs—Clean Architecture for large enterprise apps, Feature-Based for medium projects, or a simpler structure for MVPs. The key is consistency and maintainability.
