# VisionPark Backend Design (Firebase/Firestore)

## 1. System Overview
VisionPark is an AI-powered parking management ecosystem designed to streamline parking discovery, reservation, and infrastructure oversight. The system leverages YOLOv8 computer vision for real-time vehicle identification and occupancy monitoring.

**Core User Roles & Flows:**
- **Drivers:** Discover spots via a real-time interactive map, perform digital check-ins, and pay deposits/fees via Telebirr or Chapa.
- **Owners:** Manage physical assets (Branches, Zones, Spots) and configure dynamic pricing hierarchies based on vehicle categories.
- **Attendants:** Stationed at specific branches to monitor the "Live Grid," handle walk-up POS transactions, and manage enforcement (e.g., squatters and overstays).
- **System Admins:** Oversee platform-wide health, provision owner accounts, and tune global AI model parameters.

## 2. Collections

### Collection: `users`
*Represents all human actors in the system.*

| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `uid` | string | Yes | Unique identifier from Firebase Auth. |
| `fullName` | string | Yes | User's legal name. |
| `email` | string | Yes | Verified email address. |
| `role` | string | Yes | Enum: `"admin" | "owner" | "attendant" | "driver"`. |
| `phoneNumber` | string | No | International format contact. |
| `assignedBranchId` | reference | No | Link to `branches` (primarily for attendants). |
| `vehicleProfiles` | array<map> | No | Driver vehicles: `{ plate: string, type: string }`. |
| `paymentPreferences` | map | No | e.g., `{ defaultMethod: "Telebirr" }`. |
| `createdAt` | timestamp | Yes | Account initialization date. |

### Collection: `branches`
*Represents a physical parking facility.*

| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | Unique Branch ID. |
| `ownerId` | reference | Yes | Reference to the owner in the `users` collection. |
| `name` | string | Yes | Trading name (e.g., "Megenagna SMART"). |
| `region` | string | Yes | e.g., "Addis Ababa", "Oromia Region". |
| `city` | string | Yes | e.g., "Adama", "Bahir Dar". |
| `location` | geopoint | Yes | Lat/Lon for the map interface. |
| `address` | string | Yes | Physical street address. |
| `serviceType` | string | Yes | `"24 Hour Service" | "Day Service Only"`. |
| `operatingHours` | map | No | `{ open: "06:00 AM", close: "06:00 PM" }`. |
| `overstayMultiplier` | number | Yes | Penalty rate multiplier (e.g., 1.75). |

### Collection: `zones`
*Logical subdivisions within a branch.*

| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | Unique Zone ID. |
| `branchId` | reference | Yes | Parent branch. |
| `name` | string | Yes | e.g., "Zone B (Buses)", "Level 1". |
| `allowedCategories` | array<string> | Yes | List of vehicle types allowed in this zone. |

### Collection: `parkingSlots`
*The individual parking spots.*

| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | Spot identifier (e.g., "A-01"). |
| `zoneId` | reference | Yes | Parent zone. |
| `branchId` | reference | Yes | Denormalized parent branch ID for performance. |
| `status` | string | Yes | Enum: `"Free" | "Reserved" | "Occupied" | "Maintenance"`. |
| `currentVehiclePlate` | string | No | Plate detected by AI or entered by POS. |
| `isConflict` | boolean | Yes | AI flag for unauthorized vehicle detection. |
| `sensorId` | string | No | ID of the YOLOv8 edge node monitoring the spot. |
| `overrideCategories` | array<string> | No | Specific allowed types if different from zone. |

### Collection: `sessions`
*Active and historical parking/reservation records.*

| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | Unique Session ID. |
| `driverId` | reference | No | UID of the driver (null for walk-up POS). |
| `slotId` | reference | Yes | Reference to the `parkingSlot`. |
| `branchId` | reference | Yes | Denormalized for owner reporting. |
| `status` | string | Yes | Enum: `"Reserved" | "Active" | "Completed" | "Cancelled"`. |
| `startTime` | timestamp | Yes | Reservation or check-in timestamp. |
| `endTime` | timestamp | No | Checkout timestamp. |
| `pricing` | map | Yes | `{ baseRate: number, multiplierAtStart: number }`. |
| `payment` | map | Yes | `{ deposit: number, totalPaid: number, method: string, transactionId: string }`. |

### Collection: `pricingConfigs`
*Rate tables for branches.*

| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `branchId` | reference | Yes | Parent branch. |
| `rates` | map<string, number> | Yes | Key-value pairs of `{ vehicleCategory: ratePerHour }`. |

### Collection: `incidents`
*Reports of disputes, damage, or flee events.*

| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | Incident ID. |
| `branchId` | reference | Yes | Location of event. |
| `type` | string | Yes | Enum: `"Squatter" | "Overstay" | "Damage" | "Fled"`. |
| `description` | string | Yes | Textual report from attendant. |
| `reportedBy` | reference | Yes | UID of the attendant. |
| `timestamp` | timestamp | Yes | Event time. |

### Collection: `auditLogs`
*Immutable technical logs.*

| Field | Type | Required | Description |
| :--- | :--- | :---: | :--- |
| `id` | string | Yes | Log ID. |
| `timestamp` | timestamp | Yes | Event time. |
| `actor` | map | Yes | `{ uid: string, name: string, role: string }`. |
| `category` | string | Yes | `"Auth" | "Config" | "Financial" | "Vehicle" | "Node"`. |
| `description` | string | Yes | Human-readable action description. |
| `metadata` | map | No | JSON payload of Before/After state data. |

## 3. Relationships

- **Hierarchy (1:N):** `branches` -> `zones` -> `parkingSlots`. Optimized for physical site management.
- **Ownership (1:N):** `users` (Owner) -> `branches`. Allows multi-site operations.
- **Deployment (N:1):** `users` (Attendant) -> `branches`. Pins staff to a specific live dashboard.
- **Activity (N:1):** `sessions` -> `parkingSlots` & `users` (Driver). Tracks historical usage of assets and users.

## 4. Notes & Best Practices

- **Flattened Hierarchy:** Collections are kept at the root level rather than nested sub-collections (e.g., `branches/{id}/zones`) to support efficient global queries and easier indexing.
- **Query Optimization:** `branchId` is denormalized into `parkingSlots` and `sessions` to facilitate instant dashboard filtering without performing multi-collection joins.
- **AI Integration:** The `isConflict` flag and `status` in `parkingSlots` are intended to be updated by a FastAPI backend receiving telemetry from YOLOv8 edge nodes.
- **Financial Integrity:** Checkout calculations (Base Rate + Penalties) must be processed on the backend (Cloud Functions) to prevent client-side manipulation.
- **Auto-Maintenance:** A scheduled task should automatically expire stale reservations (`Reserved` status) if the driver doesn't check in within a defined grace period.
