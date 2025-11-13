# Click-to-Time Calibration Implementation Notes

## Overview
This document provides technical details about the click-to-time calibration feature implementation.

## Problem Statement
Two persistent day-view problems were addressed:
1. Dropping a dragged event often returns it to the original time instead of the cursor's time
2. Creating an event at 12:00 AM appears at 4:30 AM (+4:30/+5:30 shift)

## Root Causes Identified
1. **Inaccurate Y→time calculations**: Math that ignores the inner scrollable grid, actual slot height, and potential UI drift from CSS/zoom
2. **Timezone handling issues**: Sending UTC timestamps or applying manual offsets when writing to Google Calendar

## Solution Architecture

### 1. Calibration State Management
```javascript
const [calibrationMode, setCalibrationMode] = useState(false);
const [calibrationOffsetPx, setCalibrationOffsetPx] = useState(0);
```

- `calibrationMode`: Boolean flag indicating if user is in calibration mode
- `calibrationOffsetPx`: Pixel offset to apply to all Y→time conversions
- Persisted to localStorage with key: `click_time_calibration_px`

### 2. Grid Context Helper
```javascript
getGridContextForPoint(x, y) {
  // Returns: { targetColumn, gridRect, scrollTop, hourHeight }
}
```

This helper provides accurate measurements for any point on the time grid:
- **targetColumn**: The DOM element for the calendar column under the cursor
- **gridRect**: Bounding rectangle of the scrollable time grid
- **scrollTop**: Current scroll position of the grid
- **hourHeight**: Measured height of one hour slot (from data-hour-slot attribute)

### 3. Calibration Click Handler
```javascript
handleCalibrationClick(e) {
  // 1. Get grid context for click position
  const gridContext = getGridContextForPoint(e.clientX, e.clientY);
  
  // 2. Calculate what time we think was clicked (without calibration)
  const relativeY = (e.clientY - gridRect.top) + scrollTop;
  const hourIndex = Math.floor(relativeY / hourHeight);
  
  // 3. Calculate expected Y for 12:00 AM
  const expectedHourIndex = 0 - visibleHourStart;
  const expectedY = expectedHourIndex * hourHeight;
  
  // 4. Compute and save offset
  const offsetPx = expectedY - relativeY;
  setCalibrationOffsetPx(offsetPx);
  localStorage.setItem('click_time_calibration_px', offsetPx.toString());
}
```

**Key Formula**: `calibrationOffsetPx = expectedY - computedY`

This offset compensates for any drift between what the user sees and what the code calculates.

### 4. Applying Calibration Offset

The calibration offset is applied in all Y→time conversions:

#### A) Event Creation (handleMouseDown)
```javascript
const relativeY = e.clientY - rect.top + calibrationOffsetPx;
const totalMinutes = (relativeY / rect.height) * 60;
const snappedMinutes = Math.round(totalMinutes / 15) * 15;
```

#### B) Drag Preview (getDropPreviewPosition)
```javascript
const relativeY = dragPosition.y - gridRect.top + scrollTop + calibrationOffsetPx;
const hourIndex = Math.floor(relativeY / hourHeight);
const minuteInHour = ((relativeY % hourHeight) / hourHeight) * 60;
```

#### C) Event Drop (handleEventDragEnd)
```javascript
const relativeY = dragPosition.y - gridRect.top + scrollTop + calibrationOffsetPx;
const hourIndex = Math.floor(relativeY / hourHeight);
const snappedMinutes = Math.round(minuteInHour / 15) * 15;
```

#### D) Selection Drag (handleMouseMove)
```javascript
const relativeY = e.clientY - rect.top + calibrationOffsetPx;
const totalMinutes = (relativeY / rect.height) * 60;
```

### 5. Timezone-Safe Writes

All Google Calendar API writes use local datetime strings with explicit timezone:

```javascript
// Helper function
const toLocalDateTimeString = (date) => {
  const pad = (n) => String(n).padStart(2, '0');
  return `${date.getFullYear()}-${pad(date.getMonth()+1)}-${pad(date.getDate())}` +
         `T${pad(date.getHours())}:${pad(date.getMinutes())}:${pad(date.getSeconds())}`;
};

// Usage in API calls
const tz = Intl.DateTimeFormat().resolvedOptions().timeZone;
const eventData = {
  summary: title,
  start: { 
    dateTime: toLocalDateTimeString(startTime),
    timeZone: tz
  },
  end: { 
    dateTime: toLocalDateTimeString(endTime),
    timeZone: tz
  }
};
```

**Key Points**:
- No use of `toISOString()` which converts to UTC
- No manual `timeZoneOffset` math applied to times
- Browser's IANA timezone used (e.g., "America/New_York")
- Google Calendar API handles timezone conversion

### 6. Safe Cross-Calendar Moves

When dragging events between calendars:
```javascript
// 1. Create event on target calendar first
const createResponse = await fetch(...targetCalendar/events, {
  method: 'POST',
  body: JSON.stringify(eventData)
});

// 2. Only delete from source if creation succeeded
if (createResponse.ok) {
  await fetch(...sourceCalendar/events/${eventId}, {
    method: 'DELETE'
  });
}
```

This prevents accidental event loss if the network fails during the move.

## UI Components

### Settings Button
Located in Settings dialog under "Click-to-Time Calibration":
- Blue button: "Calibrate Click-to-Time"
- Shows current offset if not zero
- Reset button to clear calibration

### Calibration Banner
Yellow banner that appears when calibrationMode is active:
- Instructions: "Click exactly on the 12:00 AM line..."
- Cancel button to exit calibration mode
- Automatically dismisses after successful calibration

## Data Flow

### Calibration Flow
1. User clicks "Calibrate Click-to-Time" button
2. `setCalibrationMode(true)`, Settings dialog closes
3. Yellow banner appears with instructions
4. User clicks on 12:00 AM line in time grid
5. `handleCalibrationClick()` calculates offset
6. Offset saved to state and localStorage
7. `setCalibrationMode(false)`, banner disappears

### Event Creation Flow (with Calibration)
1. User clicks on time grid at position Y
2. `handleMouseDown()` calculates: `relativeY = Y + calibrationOffsetPx`
3. Converts relativeY to time (hour, minute)
4. Creates event at calculated time
5. Sends to Google Calendar API with timezone-safe format
6. Event appears at correct time in UI and Google Calendar

### Event Drag Flow (with Calibration)
1. User drags event to new position
2. `getDropPreviewPosition()` shows shadow at: `Y + calibrationOffsetPx`
3. User releases mouse
4. `handleEventDragEnd()` calculates new time using calibrated Y
5. Updates event via Google Calendar API
6. Event appears at new time

## Edge Cases Handled

### 1. Midnight Not Visible
If `visibleHourStart` is 6 (6 AM), and user tries to calibrate to 12:00 AM:
- `expectedHourIndex = 0 - 6 = -6`
- `expectedY = -6 * 64 = -384px`
- Negative offset is valid and will be applied correctly

### 2. Browser Zoom Changes
- Calibration is stored per-device in localStorage
- User should recalibrate if zoom level changes significantly
- Offset compensates for zoom-induced pixel drift

### 3. Scroll Position
- `getGridContextForPoint()` always includes current `scrollTop`
- Calculations work correctly regardless of scroll position
- Formula: `relativeY = (clientY - gridRect.top) + scrollTop + calibrationOffsetPx`

### 4. Multiple Columns
- Each column is a separate `.calendar-column` element
- `getGridContextForPoint()` finds the column under the cursor
- Calibration applies uniformly across all columns

### 5. Cross-Calendar Moves
- Create event on target first
- Delete from source only if create succeeds
- Preserves event data if operation fails

## Testing Considerations

### Unit Test Scenarios
1. Calculate offset when clicking at various Y positions
2. Apply offset in Y→time conversions
3. Handle negative offsets (midnight before visible range)
4. Persist/restore offset from localStorage
5. Reset offset to zero

### Integration Test Scenarios
1. Complete calibration flow
2. Create event after calibration
3. Drag event after calibration
4. Cross-calendar drag with calibration
5. Zoom change requiring recalibration

### Manual Test Checklist
See TESTING.md for comprehensive manual testing guide.

## Performance Considerations

### Minimal Impact
- State updates: 2 new state variables (boolean + number)
- Calculations: Simple arithmetic (addition) applied to existing logic
- Storage: Single localStorage key with small numeric value
- No new network requests or heavy computations

### Optimization Opportunities
- Calibration offset cached in memory (state)
- Grid measurements reused in calculations
- No unnecessary re-renders triggered

## Browser Compatibility

### Requirements
- Modern browser with localStorage support
- CSS Grid and Flexbox support
- ES6+ JavaScript features (arrow functions, template literals)
- getBoundingClientRect() API

### Tested Browsers
- Chrome/Edge (Chromium-based)
- Firefox
- Safari
- All modern browsers with React 18 support

## Future Enhancements

### Potential Improvements
1. **Multi-point calibration**: Allow calibrating to multiple hour lines for better accuracy
2. **Auto-calibration**: Detect misalignment and suggest calibration
3. **Per-zoom calibration**: Store separate offsets for common zoom levels
4. **Visual calibration guide**: Overlay showing expected vs actual click positions
5. **Calibration validation**: Test accuracy after calibration and report results

### Known Limitations
1. Requires manual recalibration when zoom changes
2. Assumes uniform hour slot heights
3. Single calibration point (12:00 AM)
4. No automatic drift detection

## Debugging

### Console Commands
```javascript
// Check calibration offset
localStorage.getItem('click_time_calibration_px')

// Reset calibration
localStorage.setItem('click_time_calibration_px', '0')

// Check timezone
Intl.DateTimeFormat().resolvedOptions().timeZone

// Inspect grid measurements
document.querySelector('[data-hour-slot]').getBoundingClientRect().height
```

### Common Issues
1. **Events still misaligned**: Recalibrate at current zoom level
2. **Negative offset seems wrong**: Normal if midnight before visible range
3. **Calibration not persisting**: Check localStorage quota/permissions
4. **Different results on reload**: Verify offset is loaded from localStorage

## Code Maintenance

### Key Files
- `index.html`: All implementation in single file (React app)
- Lines 131-132: State definitions
- Lines 175-205: Calibration handler
- Lines 686-703: Event creation with calibration
- Lines 730-798: Event drag with calibration
- Lines 869-894: Drag preview with calibration
- Lines 1725-1740: Calibration banner UI
- Lines 1869-1897: Settings calibration section

### Code Review Points
- Verify calibrationOffsetPx is applied in all Y→time conversions
- Check localStorage persistence on mount and change
- Ensure timezone-safe writes don't use toISOString()
- Validate grid measurements use data-hour-slot and .time-grid
- Confirm safe cross-calendar move pattern (create first, then delete)

## References
- Problem Statement: Original issue describing the timing problems
- TESTING.md: Comprehensive manual testing guide
- index.html: Complete implementation source code
