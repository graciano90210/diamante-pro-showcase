# Mobile App — Flutter

## Overview

The Diamante Pro mobile app is built with **Flutter 3.x** for Android and targets field collectors (cobradores) who operate in areas with unreliable internet connectivity.

**Core design principle: Offline-First.** Every feature works without a network connection. Data syncs automatically when connectivity is restored.

---

## Architecture

```mermaid
flowchart TB
    subgraph UI["UI Layer (Flutter Widgets)"]
        HOME[Home Dashboard]
        RUTA[Route View\nClient List]
        COBRO[Payment Screen]
        SYNC[Sync Status]
    end

    subgraph State["State Management (Provider)"]
        AUTH_P[Auth Provider]
        RUTA_P[Route Provider]
        SYNC_P[Sync Provider]
    end

    subgraph Services["Services"]
        AUTH_S[Auth Service\nJWT Management]
        SYNC_S[SyncService\nsyncAll()]
        GPS[GPS Service\nLocation Tracking]
        CAM[Camera Service\nPhoto Capture]
    end

    subgraph Storage["Local Storage"]
        DB[(SQLCipher\nAES-256 SQLite)]
        QUEUE[Sync Queue\nPending Operations]
    end

    subgraph Backend["Backend API"]
        API[Flask REST API\n/api/v1/]
    end

    UI --> State
    State --> Services
    Services --> Storage
    SYNC_S -->|on reconnect| API
    GPS --> Storage
    CAM --> Storage
```

---

## Offline-First Sync Strategy

All mutations (new clients, payments, visits) are written to the local SQLCipher database first. `SyncService.syncAll()` flushes them to the backend when connectivity is available.

**Sync order is strict** to respect foreign key constraints:

```
1. Clientes   → create new clients registered in the field
2. Creditos   → new loans or loan updates
3. Pagos      → payment records (reference credito_id)
```

Conflict resolution: **server wins**. If the backend rejects a record (e.g., duplicate payment), the local queue marks it as failed and surfaces it to the collector for review.

---

## Security

### Local Database Encryption

All local data is encrypted with **SQLCipher AES-256**:

```dart
final db = await openDatabaseWithOptions(
  path,
  options: OpenDatabaseOptions(
    password: derivedKey,  // derived from user PIN + device ID
    version: 1,
  ),
);
```

The encryption key is derived at runtime — never stored on disk.

### JWT Handling

- Tokens are stored in Flutter Secure Storage (Android Keystore backed)
- Automatic refresh before expiry
- On token revocation, local DB is wiped and user is logged out

---

## GPS & Visit Verification

Each payment recording captures:

```dart
Position position = await Geolocator.getCurrentPosition(
  desiredAccuracy: LocationAccuracy.high,
);

Payment payment = Payment(
  prestamoId: prestamo.id,
  monto: amount,
  lat: position.latitude,
  lng: position.longitude,
  timestamp: DateTime.now().toUtc(),
  photoPath: capturedPhotoPath,
);
```

The backend cross-references these coordinates against the registered client address to flag anomalies (payments recorded far from the client's location).

---

## Key Screens

| Screen | Description |
|---|---|
| **Login** | JWT auth, PIN setup for offline sessions |
| **Route Dashboard** | Summary: clients to visit, collected vs pending |
| **Client List** | Sorted by priority (overdue first), search |
| **Client Detail** | Loan history, payment schedule, balance |
| **Payment Capture** | Amount entry, photo, GPS auto-capture |
| **Daily Summary** | End-of-day totals, sync status, pending items |
| **Sync Status** | Queue progress, failed records, retry |

---

## Tech Stack

| Component | Technology |
|---|---|
| Framework | Flutter 3.x (Dart) |
| State Management | Provider |
| Local DB | SQLCipher (AES-256 encrypted SQLite) |
| HTTP Client | Dio |
| Secure Storage | flutter_secure_storage (Android Keystore) |
| GPS | geolocator |
| Camera | image_picker |
| Distribution | Google Play (beta) |

---

## Distribution

- **Package**: `com.diamantepro.app`
- **Platform**: Android (iOS planned)
- **Track**: Closed beta on Google Play
- **Store**: [Google Play](https://play.google.com/store/apps/details?id=com.diamantepro.app)

---

## Local Development

```bash
cd mobile-app/

# Install dependencies
flutter pub get

# Run on connected device or emulator
flutter run

# Build release APK
flutter build apk --release
```

Requires Flutter 3.x and Android SDK. The app connects to `localhost:5001` in debug mode via a configured base URL constant.
