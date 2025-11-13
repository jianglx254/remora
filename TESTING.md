# Testing Guide for Click-to-Time Calibration

## Overview
This document describes how to test the click-to-time calibration feature that fixes event timing issues in the day view.

## Prerequisites
- A Google account with calendar access
- A modern web browser (Chrome, Firefox, Safari, or Edge)
- The Remora Calendar application loaded in the browser

## Test Scenarios

### 1. Calibration Setup
**Steps:**
1. Sign in to the application with your Google account
2. Navigate to Day view (press 'd' or click Day button)
3. Click the Settings button (gear icon)
4. Locate the "Click-to-Time Calibration" section
5. Click "Calibrate Click-to-Time" button
6. Observe the yellow banner appears with instructions

**Expected Result:**
- Yellow banner displays: "Calibration Mode Active: Click exactly on the 12:00 AM line in the day view time grid to calibrate."
- Cancel button is present in the banner

### 2. Perform Calibration
**Steps:**
1. After entering calibration mode (from Test 1)
2. Scroll the time grid to make 12:00 AM (midnight) visible
3. Click precisely on the 12:00 AM line in the time grid
4. Observe calibration completes automatically

**Expected Result:**
- Yellow banner disappears
- Calibration offset is calculated and saved
- Can verify by opening Settings again and seeing "Current offset: X.Xpx" displayed

### 3. Create Event at Midnight
**Steps:**
1. Ensure calibration is complete (from Test 2)
2. Navigate to Day view
3. Scroll to make 12:00 AM visible
4. Click on the 12:00 AM line to create a new event
5. Enter event details (title, etc.)
6. Save the event
7. Check the event's time in Google Calendar

**Expected Result:**
- Event is created at exactly 12:00 AM (midnight)
- No time offset (+4:30 or +5:30)
- Time matches what you clicked in the UI

### 4. Drag and Drop Event
**Steps:**
1. Create an event at any time (e.g., 9:00 AM)
2. Click and hold on the event
3. Drag the event to a different time (e.g., 2:00 PM)
4. Observe the preview shadow shows the target time
5. Release to drop the event
6. Verify the event time updated correctly

**Expected Result:**
- Preview shadow appears directly under cursor while dragging
- Preview snaps to 15-minute intervals
- Event drops at the time shown in preview
- Event duration is preserved
- Final time matches the drop location (15-min snap)

### 5. Create Event by Dragging
**Steps:**
1. Click and hold at 3:00 PM
2. Drag down to 4:30 PM
3. Release to create a new event
4. Enter event title
5. Verify the start and end times in the modal

**Expected Result:**
- Blue selection box appears while dragging
- Start time: 3:00 PM
- End time: 4:30 PM
- Times are accurate (no offset)

### 6. Cross-Calendar Drag
**Steps:**
1. Set up multiple calendars in columns (Settings > Column Configuration)
2. Create an event in Column 1
3. Drag the event to Column 2 (different calendar)
4. Drop the event
5. Verify event appears in target calendar

**Expected Result:**
- Event is copied to target calendar first
- Original event is deleted only after copy succeeds
- No data loss if operation fails
- Event time is preserved correctly

### 7. Reset Calibration
**Steps:**
1. Open Settings
2. Locate "Click-to-Time Calibration" section
3. Click "Reset" button next to current offset
4. Close Settings
5. Try creating an event

**Expected Result:**
- Calibration offset is reset to 0
- Events may be slightly misaligned if zoom/scaling is active
- User can re-calibrate if needed

### 8. Zoom/Scale Testing
**Steps:**
1. Perform calibration at 100% browser zoom
2. Change browser zoom to 150%
3. Try clicking on a specific time (e.g., 1:00 PM)
4. Check if event is created at the correct time
5. If misaligned, recalibrate at the new zoom level

**Expected Result:**
- With calibration, clicks should map to correct times
- May need to recalibrate when zoom changes
- Calibration compensates for zoom/scaling effects

## Debugging

### Check Calibration Offset
1. Open browser developer console (F12)
2. Type: `localStorage.getItem('click_time_calibration_px')`
3. Press Enter
4. Observe the stored offset value

### Check Timezone
1. Open browser developer console (F12)
2. Type: `Intl.DateTimeFormat().resolvedOptions().timeZone`
3. Press Enter
4. Verify your system timezone is detected correctly

### Verify Event Times
1. Open Google Calendar in a separate tab
2. Check event times there match what you see in Remora
3. Times should be consistent across both interfaces

## Known Limitations
- Calibration is specific to current browser zoom level
- Recalibration needed if zoom changes significantly
- Calibration assumes 12:00 AM line is clicked precisely
- Works best in Day view (calibration target)

## Troubleshooting

### Events Still Have Wrong Times
- Re-run calibration process
- Ensure clicking exactly on hour line
- Check browser zoom level is stable
- Verify timezone settings are correct

### Calibration Mode Won't Exit
- Click Cancel button in yellow banner
- Refresh page if needed
- Check browser console for errors

### Drag Preview Misaligned
- Perform calibration again
- Check that time-grid has data-hour-slot attributes
- Verify scrollTop is being measured correctly
