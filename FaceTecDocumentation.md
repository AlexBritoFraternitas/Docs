# 🚀 FaceTec SDK Migration Guide — From 9.x to 10.x  
### Full iOS + Backend Update Documentation

---

## 📌 Overview

FaceTec SDK 10.x introduces a new **secure blob-based architecture** that replaces the older flow where iOS handled raw biometrics (`faceScanBase64`, `auditTrail`, etc.).

This guide covers:

- All breaking changes  
- Required iOS updates  
- Required backend updates  
- New recommended architecture  
- Sample code for both sides  

---

# 📘 1. What Changed From SDK 9.x to 10.x?

| Area | SDK 9.x | SDK 10.x |
|------|---------|-----------|
| **Initialization** | `initializeInProductionMode` | `initializeWithSessionRequest` |
| **Data Flow** | iOS received `faceScanBase64` & audit images | iOS receives encrypted request/response blobs |
| **Backend Role** | Minimal | Full biometric verification |
| **Security** | Biometrics exposed on client | No biometrics exposed on client |
| **Client APIs** | Delegate-based (`FaceTecFaceScanProcessorDelegate`) | Processor-based (`FaceTecSessionRequestProcessor`) |
| **Outcome Handling** | iOS checks raw result | iOS checks only sessionStatus |

---

# 📙 2. New iOS Architecture

---

## 2.1 Initialization

### ❌ Old (9.x)
```swift
FaceTec.sdk.initializeInProductionMode(
    productionKeyText: FaceTecConfig.productionKeyText,
    deviceKeyIdentifier: FaceTecConfig.DeviceKeyIdentifier,
    faceScanEncryptionKey: FaceTecConfig.PublicFaceScanEncryptionKey
) { success in ... }
```

### ✅ New (10.x)
```swift
FaceTec.sdk.initializeWithSessionRequest(
    deviceKeyIdentifier: Config.DeviceKeyIdentifier,
    sessionRequestProcessor: EGSessionRequestProcessor(),
    completion: { initialized, status in
        ...
    }
)
```

---

## 2.2 Session Flow

### ❌ Old (9.x)
```swift
sessionResult.faceScanBase64
sessionResult.auditTrailCompressedBase64
```

### ❌ Old backend payload
```json
{
  "faceScan": "<base64>",
  "auditTrailImage": "<base64>"
}
```

---

## ✅ New (10.x) — Blob Architecture

### iOS Receives:
```swift
func onSessionRequest(
    sessionRequestBlob: String,
    sessionRequestCallback: FaceTecSessionRequestProcessorCallback
)
```

### iOS Sends to Backend:
```json
{
  "sessionRequestBlob": "<blob>",
  "userId": "12345"
}
```

### Backend Responds:
```json
{
  "sessionResponseBlob": "<blob>",
  "success": true,
  "livenessPassed": true
}
```

### iOS Responses to SDK:
```swift
sessionRequestCallback.processResponse(sessionResponseBlob)
```

### Final Result:
```swift
override func onFaceTecExit(sessionResult: FaceTecSessionResult) {
    switch sessionResult.sessionStatus {
    case .sessionCompleted:
        onSuccessSelfie?()
    case .userCancelled:
        onCloseSelfie?()
    default:
        onErrorSelfie?()
    }
}
```

---

# 📗 3. iOS Implementation (Recommended)

---

## 3.1 EGSessionRequestProcessor.swift

```swift
import FaceTecSDK
import UIKit

final class EGSessionRequestProcessor: SessionRequestProcessor {

    var onSuccessSelfie: (() -> Void)?
    var onErrorSelfie: (() -> Void)?
    var onCloseSelfie: (() -> Void)?

    override func onFaceTecExit(sessionResult: FaceTecSessionResult) {
        DispatchQueue.main.async {
            switch sessionResult.sessionStatus {
            case .sessionCompleted:
                self.onSuccessSelfie?()

            case .userCancelled:
                self.onCloseSelfie?()

            default:
                self.onErrorSelfie?()
            }
        }
    }
}
```

---

## 3.2 Presenting FaceTec from a ViewController

```swift
let processor = EGSessionRequestProcessor()

processor.onSuccessSelfie = { ... }
processor.onErrorSelfie = { ... }
processor.onCloseSelfie = { ... }

let faceTecVC = FaceTec.sdk.createSessionVC(
    sessionRequestProcessor: processor
)

present(faceTecVC, animated: true)
```

---

# 🖥️ 4. Backend Changes (Critical)

---

## ❌ Old Backend Flow

Backend received raw biometrics:

```json
{
  "faceScan": "<base64>",
  "auditTrailImage": "<base64>"
}
```

High compliance & security risk.

---

## ✅ New Backend Flow (SDK 10.x)

### Backend receives:
```json
{
  "sessionRequestBlob": "<blob>",
  "userId": "12345"
}
```

Backend calls **FaceTec Server SDK** and receives:
```json
{
  "sessionResponseBlob": "<blob>",
  "success": true,
  "wasLivenessPassed": true
}
```

Backend responds to iOS:
```json
{
  "sessionResponseBlob": "<blob>",
  "success": true,
  "livenessPassed": true
}
```

---

# 🔐 5. Why FaceTec Changed the Architecture

| Reason | Description |
|--------|-------------|
| **Security** | Removes biometric exposure on client apps |
| **Compliance** | Meets GDPR, CCPA, biometric laws |
| **Unified Architecture** | Android/iOS/Web share one flow |
| **Future Compatibility** | iOS tightening camera/biometric access |
| **Lower Risk** | Prevents leakage of biometric data |

---

# 📑 6. Backend API Specification

### POST `/facetec/session`

### Request Body:
```json
{
  "sessionRequestBlob": "<blob>",
  "userId": "12345",
  "flow": "cardRequest"
}
```

### Response Body:
```json
{
  "sessionResponseBlob": "<blob>",
  "success": true,
  "livenessPassed": true,
  "reason": "OK"
}
```

---

# 📝 7. Migration Checklist

### iOS
- [x] Remove deprecated initialization (`initializeInProductionMode`)
- [x] Remove:
  - `faceScanBase64`
  - `auditTrailCompressedBase64`
- [x] Implement `EGSessionRequestProcessor`
- [x] Use `sessionStatus` to determine success
- [x] Connect with backend endpoint

### Backend
- [ ] Implement `/facetec/session`
- [ ] Integrate FaceTec Server SDK
- [ ] Accept `sessionRequestBlob`
- [ ] Return valid `sessionResponseBlob`
- [ ] Determine liveness
- [ ] Handle secure logging & auditing
