# Technical Deep Dive: Oman Air SSIM Processing Flow

This document details the end-to-end process of how the **GalleyXHelper** application handles and synchronizes flight data from Oman Air. The process is initiated when a request is made to the REST API to trigger an update for the **"Oman"** client.

---

## 1. The Entry Point: `fProcessUpdateMessages`

The entire workflow for Oman Air begins in **`502_ClientOman.ts`** within the `CClientSpecificFunctionalityOA` class.  
The main orchestration method is **`fProcessUpdateMessages`**.

**Execution Steps:**

1. **Establish FTP Connection**  
   An FTP connection is established using the credentials specified for Oman Air in `gxh-client-config.cjs`.

2. **Fetch File List**  
   Retrieves all files from the configured FTP directory (`/ASM/`).

3. **Filter for Relevant Files**  
   Filters for `.dat` or `.txt` files modified within the last 24 hours.

4. **Iterate and Process Files**  
   Each filtered file is processed one by one.

---

## 2. File Parsing and Message Identification

Within the file loop, the application reads each file line by line. Based on the prefix of the line, the appropriate parser is chosen:

- **`foParseAsmFile(sLine)`** → If line starts with `ASM`
- **`foParseMvtFile(sLine)`** → If line starts with `MVT`
- **`foParseSimFile(sLine)` → `foParseSsimFileRecord(sLine)`** → For all other lines (SSIM schedule records)

This enables handling of mixed-content files, though typically files contain a single type of message.

---

## 3. Deep Dive into SSIM Message Parsers

Each parser updates the in-memory data store **`ooMasterData`** (instance of `CMasterData` from `100_Data.ts`).

### 3.1 `foParseAsmFile` (Ad-hoc Schedule Message)

**Purpose:** Process real-time changes to flight schedules.

**Logic:**

- Extracts fields (Flight No., Departure/Arrival Airports, Departure/Arrival Times).
- Generates a **unique flight key**.
- Uses `ooMasterData.foFlightGetOrCreate` to fetch/create a `CFlight`.
- Calls `oFlight.fUpdateFromASM` to update with new data.

---

### 3.2 `foParseMvtFile` (Movement Message)

**Purpose:** Update operational flight status (e.g., departure/arrival).

**Logic:**

- Extracts flight details from MVT line.
- Uses `ooMasterData.foFlightGetOrCreate`.
- Calls `oFlight.fUpdateFromMVT` to update status, actual departure, and arrival.

---

### 3.3 `foParseSimFile` (Schedule Information Message)

**Purpose:** Process master flight schedule (multi-line, complex).

**Logic via `foParseSsimFileRecord`:**

- **Record Type '3'** → Main flight leg record (Flight number, airports, timings).  
  Creates a new `CFlight` object.

- **Record Types '4' & '5'** → Continuation records with additional flight details.  
  Appended to current `CFlight`.

- **End of Record** → On new Type 3 or EOF, completed flight object is added to `ooMasterData`.

---

## 4. Database Synchronization with FileMaker

Once parsing is complete and `ooMasterData` is populated, synchronization begins.

- **`CFMFlightO.fUpdateFromFlight` (in 502_ClientOman.ts):**  
  Maps internal `CFlight` objects to FileMaker schema.  
  Handles type conversion, flags, and metadata.

- **`101_FMDB.ts`:**  
  Data access layer for FileMaker using `fm-odata-client`.  
  Provides `foGetFlights`, `foCreateFlight`, `foUpdateFlight`.

**Process:**

1. Fetch all existing FileMaker flights.
2. Compare with `ooMasterData`.
3. Determine **create/update/delete** actions.
4. Execute via **`Promise.all`** for concurrent DB operations.

---

## Summary

The Oman Air SSIM processing flow involves:

1. **File Retrieval** → FTP fetch & filtering
2. **Parsing** → ASM, MVT, and SIM file handling
3. **In-memory Aggregation** → `ooMasterData` store
4. **Database Sync** → FileMaker update via `fm-odata-client`

This ensures Oman Air’s flight data is **up-to-date, accurate, and synchronized** between FTP input and the FileMaker system.
