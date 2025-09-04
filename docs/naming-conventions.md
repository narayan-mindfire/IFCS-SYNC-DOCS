# IFCS GalleyXHelper – Naming Conventions

This document describes the naming conventions used across the IFCS GalleyXHelper codebase.  
The goal is to provide clarity on how variables, functions, classes, and files are named, so new developers can quickly navigate and understand the structure.

---

## 1. Prefixes in Identifiers

Throughout the project, different prefixes are used to indicate the type or purpose of an identifier.  
This follows a _Hungarian Notation_-like style.

| Prefix | Meaning                            | Example                                               | Notes                                                      |
| ------ | ---------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------- |
| **i**  | Integer / Number                   | `iIndentLevel`, `iPortExpress`                        | Used for counters, indexes, numeric values.                |
| **b**  | Boolean (true/false)               | `bSkipFeatures`, `bIsDTFM`                            | Clear indicator of flag-like variables.                    |
| **s**  | String                             | `sHostFM`, `sFMSDataPath`                             | Used for paths, hostnames, labels, etc.                    |
| **o**  | Object (single instance)           | `oClientConfig`, `oConfigFMSHost`                     | Represents an object with structured data.                 |
| **oo** | Object of Objects (map/dictionary) | `ooClientConfigs`                                     | Typically key-value lookup objects.                        |
| **a**  | Array                              | `asHomeDestinations`, `aoLogLocations`                | Collections of strings, objects, or other values.          |
| **as** | Array of Strings                   | `asLocalFilePaths`, `asRemoteHosts`                   | Makes array contents explicit.                             |
| **ao** | Array of Objects                   | `aoStackEntries`, `aoLogLocations`                    | Indicates collection of structured objects.                |
| **e**  | Enum Value                         | `eIFCSClient`, `ePositionInPairOrGroup`               | Refers to specific enum members.                           |
| **E**  | Enum Type                          | `EIFCSClient`, `EAppProcess`                          | Denotes an enum definition (PascalCase).                   |
| **C**  | Class                              | `CLog`, `CFTPHost`, `CMOAirports`                     | Core classes (PascalCase).                                 |
| **S**  | Static Utility / Singleton         | `SSingleton`, `SGeneral`, `STime`                     | Classes with only static members or singleton enforcement. |
| **T**  | Type Alias                         | `TPath`, `TClientConfig`                              | Used with TypeScript `type` keyword.                       |
| **f**  | Function                           | `fLog`, `foClientConfigFromFileAndArg`                | General-purpose function or method.                        |
| **fb** | Function returning Boolean         | `fbDateIsInConfigRange`                               | Starts with `f` but signals a boolean return.              |
| **fo** | Function returning Object          | `foFTPHostFromCommon`, `foClientConfigFromFileAndArg` | Indicates the function returns an object.                  |
| **fs** | Function returning String          | `fsDefaultLogFilePath`                                | Indicates the function returns a string.                   |
| **fi** | Function returning Integer         | `fiMsLogStart`                                        | Indicates numeric return value.                            |
| **fd** | Function returning Date            | `fdNow`, `fdStartFrom`                                | For Date/time values.                                      |
| **m**  | Map (object-as-dictionary)         | `moAirports`, `moAircrafts`                           | Maps of structured objects, often keyed by code.           |
| **g**  | Global (shared singleton)          | `goConfig`, `goLog`                                   | Exported global object instances.                          |

---

## 2. File Naming Conventions

Files are named with a **3-digit prefix** to indicate their relative order or category.  
This ensures logical grouping and predictable load order.

| Prefix      | Purpose                 | Example Files                                                            | Description                                                          |
| ----------- | ----------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| **001–004** | Startup / Configuration | `001_ConfigEntry.ts`, `002_Config.ts`, `003_App.ts`, `004_RESTServer.ts` | Core entry flow: settings, config loading, entry point, REST server. |
| **100–199** | Data Handling           | `100_Data.ts`, `101_FMDB.ts`                                             | Business entities and DB communication.                              |
| **500–599** | Client Logic            | `500_Client.ts`, `501_ClientTransat.ts`, `502_ClientOman.ts`             | Airline-specific syncing, pairing, rules.                            |
| **700–799** | Industry Data           | `702_IndustryData.ts`                                                    | Data about airports/aircrafts from GitHub/JSON.                      |
| **800–899** | Utilities               | `800_DateTime.ts`, `801_Files.ts`                                        | Shared utilities for date/time and FTP.                              |
| **998–999** | Infrastructure          | `998_Logging.ts`, `999_Common.ts`                                        | Logging, singleton helpers, stack parsing, misc utilities.           |

This structure makes it easy to trace where a function/class should belong.

---

## 3. Examples in Practice

### Example: Variable

```ts
const iDaysAgoToStartFrom = 30; // integer
const sHostFM = "uat-at.ifcs.ca"; // string
const bSkipFeatures = false; // boolean
const asHomeDestinations = ["MCT", "YYZ"]; // array of strings
```

### Example: Function

```ts
function fbDateIsInConfigRange(dDate: Date): boolean { ... }
// "fb" → function returning boolean
```

### Example: Objects and Globals

```ts
export const goConfig = { ... }   // "g" → global shared config object
export const goLog = new CLog();  // global logger instance
```

### Example: Class

```ts
export class CFTPHost { ... }     // "C" → class
```

## 3. Summary

The naming system is systematic, not random:

- Prefixes encode type or intent.

- File numbers encode category and order.

- Global exports (goSomething) act as central singletons.

Once the prefixes are understood, it is easy to predict what a variable or function is just by its name.
