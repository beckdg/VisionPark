# VisionPark Backend Design (Firebase/Firestore)

## 1. System Overview
VisionPark is an AI-powered parking management ecosystem designed to streamline parking discovery, reservation, and infrastructure oversight. The system leverages YOLOv8 computer vision for real-time vehicle identification and occupancy monitoring.

**Core User Roles & Flows:**
- **Drivers:** Discover spots via a real-time interactive map, perform digital check-ins, and pay deposits/fees.
- **Owners:** Manage physical assets (Branches, Zones, Spots) and configure dynamic pricing.
- **Attendants:** Stationed at specific branches to monitor the "Live Grid" and handle walk-up POS.
- **System Admins:** Oversee platform-wide health and owner account provisioning.

## 2. Collections (Role-Specific)

### Collection: `drivers`
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `uid` | string | Yes | Firebase Auth UID. |
| `fullName` | string | Yes | |
| `email` | string | Yes | |
| `phoneNumber` | string | No | |
| `vehicleProfiles` | array<map> | No | `{ plate: string, type: string }`. |
| `paymentPreferences` | map | No | e.g., `{ defaultMethod: "Telebirr" }`. |
| `createdAt` | timestamp | Yes | |

### Collection: `attendants`
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `uid` | string | Yes | Firebase Auth UID. |
| `fullName` | string | Yes | |
| `email` | string | Yes | |
| `assignedBranchId` | reference | Yes | Link to the specific `branch` they manage. |
| `shiftStatus` | string | Yes | `"on-duty" | "off-duty"`. |
| `createdAt` | timestamp | Yes | |

### Collection: `owners`
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `uid` | string | Yes | Firebase Auth UID. |
| `fullName` | string | Yes | |
| `email` | string | Yes | |
| `companyName` | string | No | |
| `createdAt` | timestamp | Yes | |

### Collection: `admins`
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `uid` | string | Yes | Firebase Auth UID. |
| `fullName` | string | Yes | |
| `email` | string | Yes | |
| `createdAt` | timestamp | Yes | |

## 3. Infrastructure Collections

### Collection: `branches`
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | |
| `ownerId` | reference | Yes | Reference to the `owners` collection. |
| `name` | string | Yes | |
| `region` | string | Yes | |
| `location` | geopoint | Yes | Lat/Lon for map. |
| `serviceType` | string | Yes | `"24 Hour Service" | "Day Service Only"`. |
| `overstayMultiplier` | number | Yes | Penalty multiplier. |

### Collection: `zones`
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | |
| `branchId` | reference | Yes | |
| `name` | string | Yes | |
| `allowedCategories` | array<string> | Yes | |

### Collection: `parkingSlots`
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | |
| `zoneId` | reference | Yes | |
| `branchId` | reference | Yes | Denormalized for speed. |
| `status` | string | Yes | `"Free" | "Reserved" | "Occupied" | "Maintenance"`. |
| `isConflict` | boolean | Yes | AI flag. |

## 4. Operational Collections

### Collection: `sessions`
| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | |
| `driverId` | reference | No | Reference to `drivers` (null for walk-ups). |
| `slotId` | reference | Yes | |
| `branchId` | reference | Yes | Denormalized. |
| `status` | string | Yes | `"Reserved" | "Active" | "Completed"`. |
| `startTime` | timestamp | Yes | |
| `payment` | map | Yes | `{ totalPaid: number, method: string }`. |

### Collection: `pricingConfigs`
*Linked to `branches` via `branchId`.*

### Collection: `incidents`
*Reported by `attendants`.*

### Collection: `auditLogs`
*Actor UID references one of the four role-specific collections.*

## 5. Relationships
- **Owner-to-Asset:** `owners.uid` (1) -> `branches` (many).
- **Staffing:** `branches.id` (1) -> `attendants` (many via `assignedBranchId`).
- **Activity:** `drivers.uid` (1) -> `sessions` (many via `driverId`).
- **Infrastructure:** `branches` -> `zones` -> `parkingSlots` (Strict Hierarchy).

## 6. Optimization & Logic
- **Security:** Use custom claims in Firebase Auth to identify user roles and gate access via Firestore Security Rules.
- **Denormalization:** `branchId` is present in `parkingSlots` and `sessions` to prevent deep-tree traversals.
- **Validation:** All checkout and pricing logic must reside in Firebase Cloud Functions.
