## 1. Overview

**Product Name**: GoTime  
**Primary Goal**: Help users build and follow a structured morning routine, avoiding “brain-rot” and staying on track.

**Objectives**:
1. Let users define routines: a list of activities with associated durations.  
2. Use **Live Activities** to show real-time progress of a currently running routine on the Lock Screen (and Dynamic Island where supported).  
3. Save all data locally, with potential for Apple ID/iCloud sync in the future.

**Assumptions**:
- We will target iOS 16+ devices capable of running ActivityKit.  
- Routines and usage data are primarily stored locally (e.g., Core Data).  
- Future extension to iCloud or CloudKit for multi-device sync is possible.

**Key Technical Dependencies**:
- **Swift / SwiftUI** for the main app UI.  
- **ActivityKit (Live Activities)** for iOS 16+ real-time Lock Screen updates.  
- **Core Data** (or SQLite) for local persistence.  
- **Optional CloudKit** for Apple ID-based sync (future).  

---

## 2. Features

1. **Routine Management**  
   - Users can create and configure “Routines,” which are sequences of labeled activities with estimated durations.  

2. **Live Activities Integration**  
   - When a user starts a routine, a Live Activity appears on the Lock Screen (and Dynamic Island on newer iPhones).  
   - This Live Activity provides a continuously updating progress indicator: current activity, time remaining, and next activity.

3. **Local & (Future) Cloud Sync**  
   - Data (routines, sessions) are stored locally.  
   - Potential future extension: enable Apple ID/iCloud sync.

---

## 3. Requirements for Each Feature

### 3.1 Routine Management

1. **Create & Configure Routine**  
   - **Req-1.1**: User can create a new routine with a custom name (e.g., “Morning Routine”).  
   - **Req-1.2**: User can add multiple activities to that routine; each activity has:  
     - `activityName` (String)  
     - `durationMinutes` (Int)  
   - **Req-1.3**: User can reorder activities.  
   - **Req-1.4**: User can update (edit) the routine’s name, activities, and their durations at any time.

2. **Run Routine**  
   - **Req-2.1**: User can start a routine at any time from the app’s UI.  
   - **Req-2.2**: The app calculates start times for each activity by summing the durations of previous activities.  
   - **Req-2.3**: The app keeps track of which activity is “current” based on the routine’s start time and activity durations.

3. **Progress Tracking**  
   - **Req-3.1**: The app must handle partial or delayed progress. If the user starts an activity late, we still show them “behind schedule” vs. “on track.”  
   - **Req-3.2**: The user may mark an activity “Done” early, which might adjust the timeline if they finish sooner.

### 3.2 Live Activities (Real-time Notifications)

1. **Activity Start**  
   - **Req-4.1**: When the user taps “Start Routine,” the app initiates a Live Activity using **ActivityKit**.  
   - **Req-4.2**: The Live Activity includes:  
     - Current activity name  
     - Time elapsed or time remaining for the current activity  
     - Next activity and approximate start time  
     - Overall routine progress (e.g., “1 of 8 tasks completed”)

2. **Updates & Notifications**  
   - **Req-5.1**: The Live Activity must update periodically (e.g., every minute or at each transition between activities).  
   - **Req-5.2**: If the user completes an activity ahead of schedule or falls behind, the Live Activity updates to reflect the new status.  
   - **Req-5.3**: The user may dismiss the Live Activity at any time; the app’s internal routine tracking continues regardless.

3. **Completion & Dismissal**  
   - **Req-6.1**: When the routine finishes, the Live Activity ends automatically.  
   - **Req-6.2**: If the user cancels or dismisses the routine before completion, the Live Activity ends.  
   - **Req-6.3**: The user can re-start a routine from scratch if desired.

### 3.3 Local Persistence & Future Sync

1. **Local Data Persistence**  
   - **Req-7.1**: All routine configurations are stored on the device using Core Data (or a similar local storage solution).  
   - **Req-7.2**: Routine sessions (runs) are also persisted for history/tracking.

2. **Apple ID Sync (Future)**  
   - **Req-8.1**: The app architecture should allow optional syncing with iCloud/CloudKit once the feature is implemented.  
   - **Req-8.2**: Keep data model flexible for cloud-based merges or conflict resolution.

---

## 4. Data Model

An example **Core Data** schema (or equivalent approach):

```swift
// Core Entities

Entity: Routine
- routineId: UUID (primary key)
- routineName: String
- createdAt: Date
- updatedAt: Date
- activities: [Activity] (to-many relationship)

Entity: Activity
- activityId: UUID (primary key)
- activityName: String
- durationMinutes: Int16
- sortIndex: Int16   // for ordering
- parentRoutine: Routine (to-one relationship)

Entity: RoutineSession
- sessionId: UUID (primary key)
- routineId: UUID  // foreign key to Routine
- startTime: Date
- endTime: Date?   // nil if not completed
- currentActivityIndex: Int16 // or a separate entity for each activity’s progress
```

### Data Flow

1. **Routine Creation**  
   - User creates a `Routine` with multiple `Activity` entries. The system persists it locally.

2. **Starting a Routine**  
   - The app creates a `RoutineSession` with a `startTime`.  
   - A Live Activity is created via `ActivityKit`.  
   - The session tracks the current activity as time progresses.

3. **Updating Live Activities**  
   - As each activity completes or transitions, the app updates the `RoutineSession` in local storage.  
   - The app also calls `Activity.update(using:)` to reflect the new state in the Live Activity.

4. **Completing a Routine**  
   - When the final activity is done, the `endTime` is set in `RoutineSession`.  
   - The Live Activity is ended.

---

## 5. API Contract

Since much of the data will be local, this contract describes internal Swift interfaces:

### 5.1 Routines

```swift
protocol RoutineRepository {
    func createRoutine(
        name: String, 
        activities: [(name: String, durationMinutes: Int)]
    ) -> Routine
    
    func updateRoutine(
        routineId: UUID,
        name: String?,
        activities: [(activityId: UUID?, name: String, durationMinutes: Int)]?
    )
    
    func fetchRoutine(routineId: UUID) -> Routine?
    func fetchAllRoutines() -> [Routine]
    func deleteRoutine(routineId: UUID)
}
```

### 5.2 Routine Sessions

```swift
protocol RoutineSessionRepository {
    func startSession(routineId: UUID, startTime: Date) -> RoutineSession
    func updateSession(sessionId: UUID, currentActivityIndex: Int?)
    func endSession(sessionId: UUID, endTime: Date)
    func fetchActiveSession(for routineId: UUID) -> RoutineSession?
    func fetchSession(sessionId: UUID) -> RoutineSession?
}
```

### 5.3 Live Activities

```swift
// Example attributes and state for ActivityKit
struct GoTimeAttributes: ActivityAttributes {
    public struct ContentState: Codable, Hashable {
        let routineId: UUID
        let currentActivityName: String
        let currentActivityIndex: Int
        let totalActivities: Int
        let timeRemainingForCurrentActivity: Int // e.g. in seconds
    }

    // Could include static data shared across all states
    let routineName: String
}

// Interface for launching/updating Live Activities
protocol LiveActivityManager {
    func startLiveActivity(
        attributes: GoTimeAttributes, 
        initialContentState: GoTimeAttributes.ContentState
    ) -> Activity<GoTimeAttributes>?
    
    func updateLiveActivity(
        for activityId: String, 
        newState: GoTimeAttributes.ContentState
    ) 
    
    func endLiveActivity(for activityId: String)
}
```

- **Implementation Notes**:  
  - Each Live Activity is identified by an `Activity<GoTimeAttributes>`.  
  - `updateLiveActivity` might require storing the `Activity` reference or its `id`.

---

## 6. Implementation Details

1. **ActivityKit Integration**  
   - When the user taps “Start Routine,” create a `RoutineSession` in local storage.  
   - Construct `GoTimeAttributes` with the routine’s name.  
   - Construct the initial `ContentState`, including the first activity name, index=0, timeRemaining=duration.  
   - Call `Activity.request(...)` to start the Live Activity.  
   - Keep track of the `Activity` instance (or `id`) in memory.

2. **Periodic or Event-Based Updates**  
   - You can either schedule local timers that update the Live Activity’s content state (e.g., every minute), or  
   - Update on major events (activity start/end, user manually skipping activity).  
   - In each update, recalculate the “current activity,” “time remaining,” and then call `myActivity.update(using: contentState)`.

3. **Ending the Live Activity**  
   - When the final activity finishes or the user cancels, call `myActivity.end(dismissalPolicy: .immediate)` to remove the Live Activity from the Lock Screen.

4. **Edge Cases**  
   - If the user force-closes the app, the Live Activity can still display, but you’ll need background tasks or notifications to continue updating it.  
   - If a user dismisses the Live Activity, the routine continues in-app, but no real-time lock screen display is shown.

5. **Backward Compatibility**  
   - Only iOS 16+ devices support Live Activities. For older iOS versions, you might fallback to local notifications or do not offer real-time tracking.

---

## 7. Edge Cases & Questions

- **User Dismisses Live Activity**: Do we prompt them or allow them to continue silently? The routine session still runs in-app.  
- **Overrun Activities**: If the user takes longer than the allocated time, do we automatically shift subsequent start times? Or do we simply indicate “Behind schedule”?  
- **Multiple Routines**: If the user tries to start two routines simultaneously, do we allow multiple Live Activities or limit to one?

---

## 8. Summary & Next Steps

1. **Data Model Finalization**: Confirm all attributes needed for routine, activity, and session tracking.  
2. **Implementation**:  
   - Build the local storage (Core Data or equivalent).  
   - Implement routine creation/editing screens.  
   - Implement session logic (start/stop/track progress).  
   - Integrate ActivityKit for Live Activity creation and updates.  
3. **UI/UX Testing**: Ensure the Live Activity looks correct on the Lock Screen and in the Dynamic Island (for newer iPhones).  
4. **Future Enhancements**: Apple ID/iCloud sync, more robust reminders, or statistical insights about routine adherence.
