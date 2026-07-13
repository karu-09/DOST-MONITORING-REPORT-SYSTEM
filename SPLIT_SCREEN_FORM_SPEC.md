# Split-Screen Form Redesign Specification

## Overview
Redesign pass slip and attendance forms with split-screen layout: form fields on left, live preview on right.

## Requirements

### Layout
- **Split screen**: 50/50 left-right
- **Form (Left)**: Compact, no scrolling, fits viewport
- **Preview (Right)**: Live-updating preview of PDF format
- **Static height**: Fits in viewport without scrolling

### Functionality
1. **Live preview**: Updates as user types
2. **Preview button**: Opens full-size preview modal
3. **Print button**: Generates and opens PDF for printing
4. **Save behavior**: 
   - Pass slips: Save to database
   - Attendance: No database save (preview/print only)

### PDF Format (Pass Slip)
Based on provided PDF, the format includes:
- Two copies per page (ORD's COPY and GUARD's COPY)
- Control number in top-right corner
- Checkbox for Official/Personal
- Destination and Purpose only show for Official
- Signature lines for employee, guard, supervisor
- Note at bottom about two copies

## Implementation Files

### 1. Pass Slip Form (`app/(app)/pass-slips/new/page.tsx`)
```
- Split layout container
- Left: Compact form fields
- Right: Preview component showing PDF layout
- Buttons: Preview (modal) and Save & Print
```

### 2. Attendance Form (`app/(app)/attendance/new/page.tsx`)
```
- Same split layout
- Buttons: Preview and Print (no database save)
```

### 3. Preview Component (`src/components/form-preview.tsx`)
```
- Renders HTML template as live preview
- Scales to fit preview pane
- Updates on form change
```

### 4. Preview Modal (`src/components/preview-modal.tsx`)
```
- Full-screen modal
- Shows complete PDF layout
- Print button triggers PDF generation
```

### 5. PDF Templates
- ✅ Pass slip: `/src/templates/form7.ts` (UPDATED)
- Attendance: `/src/templates/attendance-form.ts` (needs update)

## Status
- ✅ PDF template updated to match official format
- ⏳ Split-screen form layout (NOT STARTED)
- ⏳ Preview component (NOT STARTED)
- ⏳ Preview modal (NOT STARTED)

## Next Steps
1. Create split-screen layout for pass slip form
2. Build live preview component
3. Create preview/print modal
4. Apply same pattern to attendance form
5. Test on different screen sizes
