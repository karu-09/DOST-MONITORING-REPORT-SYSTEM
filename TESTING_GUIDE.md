# Testing Guide - Recent Improvements

## Server Status
✅ Development server is running at: http://localhost:3000

## What to Test

### 1. Pass Slip Creation with Success Modal

**Steps:**
1. Open browser to http://localhost:3000
2. Navigate to "Pass Slips" → "New Pass Slip" (or go directly to http://localhost:3000/pass-slips/new)
3. Fill out the form:
   - Select an employee
   - Fill in Date
   - Fill in Time Out
   - Choose type (Official or Personal)
   - Fill in Destination
   - Fill in Purpose
4. Click "Save & Create"
5. **EXPECTED:** You should see a modal popup with:
   - Green checkmark icon
   - "Pass Slip Created!" heading
   - "Your pass slip has been successfully created. Redirecting to home page..." message
6. **EXPECTED:** After 2 seconds, you should be redirected to http://localhost:3000/

**If it doesn't work:**
- Check browser console (F12) for errors
- Make sure the form actually submits successfully (no validation errors)

---

### 2. Attendance Sheet Creation with Success Modal

**Steps:**
1. Open browser to http://localhost:3000
2. Navigate to "Attendance" → "Record Attendance" (or go directly to http://localhost:3000/attendance/new)
3. Fill out the form:
   - Enter Activity
   - Enter Venue
   - Select Inclusive Date (from/to)
   - Enter Facilitated By
   - Optionally select Noted By
4. Click "Proceed"
5. **EXPECTED:** You should see a modal popup with:
   - Green checkmark icon
   - "Attendance Sheet Created!" heading
   - "Your attendance sheet has been successfully created. Redirecting to home page..." message
6. **EXPECTED:** After 2 seconds, you should be redirected to http://localhost:3000/

**If it doesn't work:**
- Check browser console (F12) for errors
- The attendance form might have backend issues - check the Network tab in dev tools

---

### 3. Sign Out Redirects to Home

**Steps:**
1. Log in as an admin user
2. Navigate to any admin page (e.g., http://localhost:3000/pass-slips or http://localhost:3000/attendance)
3. In the sidebar, click "Sign out"
4. **EXPECTED:** You should be redirected to http://localhost:3000/ (home page)
5. **EXPECTED:** You should be logged out (session destroyed)

**If it doesn't work:**
- Check browser console for errors
- Check Network tab to see if the logout request succeeds
- Verify you're actually redirected to "/" and not "/login"

---

### 4. Admin Can View New Records Immediately

**Steps:**
1. Create a new pass slip (following steps in Test #1)
2. After being redirected to home, log in as admin
3. Navigate to http://localhost:3000/pass-slips
4. **EXPECTED:** The newly created pass slip should appear in the listing
5. Click "Print PDF" button next to the record
6. **EXPECTED:** PDF should open in new tab

**Repeat for Attendance:**
1. Create a new attendance sheet (following steps in Test #2)
2. After being redirected to home, log in as admin
3. Navigate to http://localhost:3000/attendance
4. **EXPECTED:** The newly created attendance record should appear in the listing
5. Click "Print PDF" button in the Actions column
6. **EXPECTED:** PDF should open in new tab

---

## Troubleshooting

### Pass Slip Modal Not Showing
1. Open the pass slip creation page
2. Open browser developer tools (F12)
3. Go to Console tab
4. Submit the form
5. Look for any errors
6. If you see "showSuccessModal" errors, the code might not have been saved properly

### Attendance Modal Not Showing  
1. Same steps as above but for attendance page
2. The attendance API might have different behavior - check Network tab

### Sign Out Not Redirecting
1. Open browser developer tools (F12)
2. Go to Network tab
3. Click "Sign out"
4. Look for the POST request to `/api/auth/logout`
5. Check the response - it should indicate a redirect
6. If not redirecting, check console for JavaScript errors

### Records Not Showing Up
1. This should work automatically if the forms submit successfully
2. Check the admin listing pages:
   - http://localhost:3000/pass-slips (for pass slips)
   - http://localhost:3000/attendance (for attendance)
3. Make sure you're logged in as admin
4. Try refreshing the page

---

## Quick Test Checklist

- [ ] Pass slip form shows success modal
- [ ] Pass slip redirects to home after 2 seconds
- [ ] Attendance form shows success modal
- [ ] Attendance redirects to home after 2 seconds
- [ ] Sign out redirects to home page
- [ ] New pass slip appears in admin listing
- [ ] New attendance appears in admin listing
- [ ] Print PDF works for pass slips
- [ ] Print PDF works for attendance

---

## Files Modified

1. **Pass Slip Form:** `/app/(app)/pass-slips/new/page.tsx`
   - Added success modal state and component
   - Updated submit handler to show modal and redirect

2. **Attendance Form:** `/app/(app)/attendance/new/page.tsx`
   - Changed inline success to modal
   - Added redirect after success

3. **Logout Button:** `/src/components/sidebar-client.tsx`
   - Changed from form POST to JavaScript button
   - Explicitly handles redirect to home

4. **Logout API:** `/app/api/auth/logout/route.ts`
   - Added redirectTo: "/" parameter

---

## If Nothing Works

1. **Stop the dev server:**
   - Press Ctrl+C in the terminal running `npm run dev`

2. **Clear Next.js cache:**
   ```bash
   cd dost-records
   rm -rf .next
   ```

3. **Restart the dev server:**
   ```bash
   npm run dev
   ```

4. **Clear browser cache:**
   - Open dev tools (F12)
   - Right-click the refresh button
   - Choose "Empty Cache and Hard Reload"

5. **Try again with all tests**

---

## Expected Behavior Summary

**User Journey:**
1. User creates form → Success modal appears → Auto-redirect to home after 2s
2. Admin logs in → Views new record in listing → Prints PDF if needed
3. Admin signs out → Redirected to home page → Ready for next user

**The Complete Flow Should Be:**
```
Kiosk User creates pass slip/attendance
    ↓
[✓ Success Modal with green checkmark]
    ↓
[Waiting 2 seconds...]
    ↓
[Redirect to home page]
    ↓
Admin logs in
    ↓
Views record in /pass-slips or /attendance
    ↓
Clicks "Print PDF"
    ↓
PDF opens in new tab
    ↓
Admin clicks "Sign out"
    ↓
[Redirect to home page]
    ↓
Ready for next user
```
