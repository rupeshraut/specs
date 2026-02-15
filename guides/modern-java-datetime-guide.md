# Effective Ways of Using Modern Java Date/Time API

A comprehensive guide to mastering the java.time API (Java 8+) for date, time, and duration handling.

---

## Table of Contents

1. [Why java.time API?](#why-javatime-api)
2. [Core Types Overview](#core-types-overview)
3. [LocalDate, LocalTime, LocalDateTime](#localdate-localtime-localdatetime)
4. [Instant and Machine Time](#instant-and-machine-time)
5. [ZonedDateTime and Time Zones](#zoneddatetime-and-time-zones)
6. [Duration and Period](#duration-and-period)
7. [Formatting and Parsing](#formatting-and-parsing)
8. [Date Arithmetic](#date-arithmetic)
9. [Working with Legacy Code](#working-with-legacy-code)
10. [Database Integration](#database-integration)
11. [REST API Best Practices](#rest-api-best-practices)
12. [Testing with Time](#testing-with-time)
13. [Common Patterns](#common-patterns)
14. [Pitfalls to Avoid](#pitfalls-to-avoid)
15. [Decision Tree](#decision-tree)

---

## Why java.time API?

### Problems with Old Date API (java.util.Date)

```java
public class OldDateProblems {
    
    // ❌ OLD API - Multiple issues
    public void oldDateProblems() {
        // 1. Mutable - not thread-safe
        Date date = new Date();
        date.setTime(System.currentTimeMillis());  // Can be modified!
        
        // 2. Poor API design
        Date d = new Date(2024, 0, 15);  // Year is 1900-based, month is 0-based!
        // Actually creates: 3924-01-15 (2024 + 1900)
        
        // 3. No time zone support
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        // Uses system default time zone - error-prone
        
        // 4. Not thread-safe
        // SimpleDateFormat instances can't be shared between threads
        
        // 5. Confusing naming
        // java.util.Date actually includes time
        // java.sql.Date, java.sql.Time, java.sql.Timestamp are different
    }
    
    // ✅ NEW API - Solutions to all problems
    public void newDateSolutions() {
        // 1. Immutable - thread-safe
        LocalDate date = LocalDate.of(2024, 1, 15);
        LocalDate modified = date.plusDays(1);  // Returns new instance
        
        // 2. Clear, intuitive API
        LocalDate d = LocalDate.of(2024, Month.JANUARY, 15);
        
        // 3. Excellent time zone support
        ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("America/New_York"));
        
        // 4. Thread-safe
        DateTimeFormatter formatter = DateTimeFormatter.ISO_LOCAL_DATE;
        // Can be safely shared
        
        // 5. Clear types for different needs
        LocalDate.now();      // Date only
        LocalTime.now();      // Time only
        LocalDateTime.now();  // Date and time, no timezone
        ZonedDateTime.now();  // Date, time, and timezone
        Instant.now();        // Machine timestamp
    }
}
```

---

## Core Types Overview

```java
public class CoreTypesOverview {
    
    /**
     * Type Selection Guide:
     * 
     * LocalDate - Date without time or timezone
     * Use: Birthdays, holidays, business dates
     * Example: 2024-01-15
     * 
     * LocalTime - Time without date or timezone
     * Use: Business hours, schedules
     * Example: 14:30:00
     * 
     * LocalDateTime - Date and time without timezone
     * Use: Appointment times, timestamps in single timezone context
     * Example: 2024-01-15T14:30:00
     * 
     * ZonedDateTime - Date, time, and timezone
     * Use: Events across timezones, international scheduling
     * Example: 2024-01-15T14:30:00-05:00[America/New_York]
     * 
     * OffsetDateTime - Date, time, and UTC offset (no timezone rules)
     * Use: Storing in database, serialization
     * Example: 2024-01-15T14:30:00-05:00
     * 
     * Instant - Machine timestamp (nanoseconds from epoch)
     * Use: Event timestamps, audit logs, measuring duration
     * Example: 1705340400.123456789 (epoch seconds.nanos)
     * 
     * Duration - Time-based amount (hours, minutes, seconds)
     * Use: Time elapsed, timeouts, expiry
     * Example: PT2H30M (2 hours 30 minutes)
     * 
     * Period - Date-based amount (years, months, days)
     * Use: Age calculation, date ranges
     * Example: P1Y2M3D (1 year, 2 months, 3 days)
     */
}
```

---

## LocalDate, LocalTime, LocalDateTime

### 1. **LocalDate (Date without Time)**

```java
public class LocalDateExamples {
    
    // Creating LocalDate instances
    public void creation() {
        // Current date
        LocalDate today = LocalDate.now();
        
        // Specific date
        LocalDate date1 = LocalDate.of(2024, 1, 15);
        LocalDate date2 = LocalDate.of(2024, Month.JANUARY, 15);
        
        // From string
        LocalDate date3 = LocalDate.parse("2024-01-15");
        LocalDate date4 = LocalDate.parse("15/01/2024", 
            DateTimeFormatter.ofPattern("dd/MM/yyyy"));
        
        // From epoch day
        LocalDate date5 = LocalDate.ofEpochDay(19736);  // Days since 1970-01-01
        
        // From year and day of year
        LocalDate date6 = LocalDate.ofYearDay(2024, 46);  // 46th day of 2024
    }
    
    // Getting components
    public void extraction() {
        LocalDate date = LocalDate.of(2024, 1, 15);
        
        int year = date.getYear();           // 2024
        Month month = date.getMonth();       // JANUARY
        int monthValue = date.getMonthValue(); // 1
        int day = date.getDayOfMonth();      // 15
        DayOfWeek dayOfWeek = date.getDayOfWeek(); // MONDAY
        int dayOfYear = date.getDayOfYear(); // 15
        
        boolean isLeapYear = date.isLeapYear();
        int lengthOfMonth = date.lengthOfMonth(); // 31
        int lengthOfYear = date.lengthOfYear();   // 366 (leap year)
    }
    
    // Modification (returns new instance - immutable!)
    public void modification() {
        LocalDate date = LocalDate.of(2024, 1, 15);
        
        // Add/subtract
        LocalDate tomorrow = date.plusDays(1);
        LocalDate nextWeek = date.plusWeeks(1);
        LocalDate nextMonth = date.plusMonths(1);
        LocalDate nextYear = date.plusYears(1);
        
        LocalDate yesterday = date.minusDays(1);
        
        // With (replace component)
        LocalDate newDate = date.withYear(2025);
        LocalDate firstDayOfMonth = date.withDayOfMonth(1);
        LocalDate lastDayOfMonth = date.withDayOfMonth(date.lengthOfMonth());
        
        // Temporal adjusters (advanced)
        LocalDate nextMonday = date.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
        LocalDate firstDayOfNextMonth = date.with(TemporalAdjusters.firstDayOfNextMonth());
        LocalDate lastDayOfYear = date.with(TemporalAdjusters.lastDayOfYear());
    }
    
    // Comparison
    public void comparison() {
        LocalDate date1 = LocalDate.of(2024, 1, 15);
        LocalDate date2 = LocalDate.of(2024, 2, 20);
        
        boolean isBefore = date1.isBefore(date2);  // true
        boolean isAfter = date1.isAfter(date2);    // false
        boolean isEqual = date1.isEqual(date2);    // false
        
        int comparison = date1.compareTo(date2);   // -1 (date1 < date2)
    }
    
    // Real-world examples
    public boolean isHoliday(LocalDate date) {
        // Check if date is a specific holiday
        return date.equals(LocalDate.of(date.getYear(), Month.DECEMBER, 25)) // Christmas
            || date.equals(LocalDate.of(date.getYear(), Month.JANUARY, 1));  // New Year
    }
    
    public boolean isWeekend(LocalDate date) {
        DayOfWeek day = date.getDayOfWeek();
        return day == DayOfWeek.SATURDAY || day == DayOfWeek.SUNDAY;
    }
    
    public LocalDate getNextBusinessDay(LocalDate date) {
        LocalDate nextDay = date.plusDays(1);
        while (isWeekend(nextDay) || isHoliday(nextDay)) {
            nextDay = nextDay.plusDays(1);
        }
        return nextDay;
    }
    
    public int calculateAge(LocalDate birthDate) {
        return Period.between(birthDate, LocalDate.now()).getYears();
    }
}
```

### 2. **LocalTime (Time without Date)**

```java
public class LocalTimeExamples {
    
    // Creating LocalTime instances
    public void creation() {
        // Current time
        LocalTime now = LocalTime.now();
        
        // Specific time
        LocalTime time1 = LocalTime.of(14, 30);           // 14:30:00
        LocalTime time2 = LocalTime.of(14, 30, 45);       // 14:30:45
        LocalTime time3 = LocalTime.of(14, 30, 45, 123456789); // 14:30:45.123456789
        
        // From string
        LocalTime time4 = LocalTime.parse("14:30:45");
        
        // Special times
        LocalTime midnight = LocalTime.MIDNIGHT;  // 00:00:00
        LocalTime noon = LocalTime.NOON;          // 12:00:00
        LocalTime min = LocalTime.MIN;            // 00:00:00
        LocalTime max = LocalTime.MAX;            // 23:59:59.999999999
    }
    
    // Getting components
    public void extraction() {
        LocalTime time = LocalTime.of(14, 30, 45, 123456789);
        
        int hour = time.getHour();     // 14
        int minute = time.getMinute(); // 30
        int second = time.getSecond(); // 45
        int nano = time.getNano();     // 123456789
    }
    
    // Modification
    public void modification() {
        LocalTime time = LocalTime.of(14, 30);
        
        LocalTime plusHours = time.plusHours(2);      // 16:30:00
        LocalTime plusMinutes = time.plusMinutes(15); // 14:45:00
        LocalTime minusHours = time.minusHours(1);    // 13:30:00
        
        LocalTime newTime = time.withHour(16);        // 16:30:00
        LocalTime rounded = time.withSecond(0).withNano(0); // 14:30:00.000000000
    }
    
    // Real-world examples
    public boolean isWithinBusinessHours(LocalTime time) {
        LocalTime businessStart = LocalTime.of(9, 0);
        LocalTime businessEnd = LocalTime.of(17, 0);
        
        return !time.isBefore(businessStart) && !time.isAfter(businessEnd);
    }
    
    public LocalTime roundToNearestQuarterHour(LocalTime time) {
        int minute = time.getMinute();
        int roundedMinute = ((minute + 7) / 15) * 15;  // Round to nearest 15
        
        if (roundedMinute == 60) {
            return time.plusHours(1).withMinute(0).withSecond(0).withNano(0);
        }
        
        return time.withMinute(roundedMinute).withSecond(0).withNano(0);
    }
}
```

### 3. **LocalDateTime (Date and Time, no Timezone)**

```java
public class LocalDateTimeExamples {
    
    // Creating LocalDateTime instances
    public void creation() {
        // Current date-time
        LocalDateTime now = LocalDateTime.now();
        
        // Specific date-time
        LocalDateTime dt1 = LocalDateTime.of(2024, 1, 15, 14, 30);
        LocalDateTime dt2 = LocalDateTime.of(2024, Month.JANUARY, 15, 14, 30, 45);
        
        // From LocalDate and LocalTime
        LocalDate date = LocalDate.of(2024, 1, 15);
        LocalTime time = LocalTime.of(14, 30);
        LocalDateTime dt3 = LocalDateTime.of(date, time);
        LocalDateTime dt4 = date.atTime(time);
        LocalDateTime dt5 = time.atDate(date);
        
        // From string
        LocalDateTime dt6 = LocalDateTime.parse("2024-01-15T14:30:45");
    }
    
    // Extract date or time
    public void extraction() {
        LocalDateTime dateTime = LocalDateTime.of(2024, 1, 15, 14, 30, 45);
        
        LocalDate date = dateTime.toLocalDate();  // 2024-01-15
        LocalTime time = dateTime.toLocalTime();  // 14:30:45
        
        // All LocalDate methods work
        int year = dateTime.getYear();
        Month month = dateTime.getMonth();
        
        // All LocalTime methods work
        int hour = dateTime.getHour();
        int minute = dateTime.getMinute();
    }
    
    // Real-world examples
    public class AppointmentService {
        
        // Schedule appointment
        public Appointment scheduleAppointment(
                LocalDateTime requestedTime, 
                Duration duration) {
            
            // Check if within business hours
            if (!isWithinBusinessHours(requestedTime)) {
                throw new IllegalArgumentException("Outside business hours");
            }
            
            // Check if slot available
            LocalDateTime endTime = requestedTime.plus(duration);
            if (hasConflict(requestedTime, endTime)) {
                throw new IllegalArgumentException("Time slot not available");
            }
            
            return new Appointment(requestedTime, endTime);
        }
        
        private boolean isWithinBusinessHours(LocalDateTime dateTime) {
            LocalTime time = dateTime.toLocalTime();
            return !time.isBefore(LocalTime.of(9, 0)) 
                && !time.isAfter(LocalTime.of(17, 0));
        }
        
        // Find available slots
        public List<LocalDateTime> findAvailableSlots(
                LocalDate date, 
                Duration slotDuration,
                Duration gap) {
            
            List<LocalDateTime> slots = new ArrayList<>();
            LocalDateTime current = date.atTime(9, 0);  // Business start
            LocalDateTime end = date.atTime(17, 0);      // Business end
            
            while (current.plus(slotDuration).isBefore(end) 
                   || current.plus(slotDuration).equals(end)) {
                
                if (!hasConflict(current, current.plus(slotDuration))) {
                    slots.add(current);
                }
                
                current = current.plus(slotDuration).plus(gap);
            }
            
            return slots;
        }
        
        private boolean hasConflict(LocalDateTime start, LocalDateTime end) {
            // Check against existing appointments
            return false;  // Simplified
        }
    }
}
```

---

## Instant and Machine Time

### Understanding Instant

```java
public class InstantExamples {
    
    /**
     * Instant represents a point on the timeline (machine time)
     * - Stored as seconds + nanoseconds from epoch (1970-01-01T00:00:00Z)
     * - Always in UTC
     * - No timezone or calendar system
     * - Best for: timestamps, measuring durations, event ordering
     */
    
    // Creating Instant instances
    public void creation() {
        // Current instant (now)
        Instant now = Instant.now();
        
        // From epoch seconds
        Instant instant1 = Instant.ofEpochSecond(1705340400L);
        
        // From epoch milliseconds
        Instant instant2 = Instant.ofEpochMilli(1705340400000L);
        
        // From epoch seconds + nanosecond adjustment
        Instant instant3 = Instant.ofEpochSecond(1705340400L, 123456789L);
        
        // From string (ISO-8601)
        Instant instant4 = Instant.parse("2024-01-15T14:30:00Z");
        
        // Special instants
        Instant epoch = Instant.EPOCH;  // 1970-01-01T00:00:00Z
        Instant min = Instant.MIN;      // -1000000000-01-01T00:00:00Z
        Instant max = Instant.MAX;      // +1000000000-12-31T23:59:59.999999999Z
    }
    
    // Getting components
    public void extraction() {
        Instant instant = Instant.now();
        
        long epochSecond = instant.getEpochSecond();  // Seconds from epoch
        int nano = instant.getNano();                 // Nanosecond within second
        long epochMilli = instant.toEpochMilli();     // Milliseconds from epoch
    }
    
    // Modification
    public void modification() {
        Instant instant = Instant.now();
        
        Instant plus5Sec = instant.plusSeconds(5);
        Instant plus100Millis = instant.plusMillis(100);
        Instant plus1000Nanos = instant.plusNanos(1000);
        
        Instant minus1Hour = instant.minus(Duration.ofHours(1));
        Instant plus2Days = instant.plus(Duration.ofDays(2));
    }
    
    // Comparison
    public void comparison() {
        Instant instant1 = Instant.now();
        Instant instant2 = instant1.plusSeconds(10);
        
        boolean isBefore = instant1.isBefore(instant2);  // true
        boolean isAfter = instant1.isAfter(instant2);    // false
        
        int comparison = instant1.compareTo(instant2);   // -1
    }
    
    // Real-world examples
    public class EventLogger {
        
        // Log event with timestamp
        public void logEvent(String event) {
            Instant timestamp = Instant.now();
            logger.info("Event: {} at {}", event, timestamp);
            
            // Store in database
            eventRepository.save(new Event(event, timestamp));
        }
        
        // Measure execution time
        public void measureExecutionTime() {
            Instant start = Instant.now();
            
            performOperation();
            
            Instant end = Instant.now();
            Duration duration = Duration.between(start, end);
            
            logger.info("Operation took {} ms", duration.toMillis());
        }
        
        // Check if event is recent
        public boolean isRecent(Instant eventTime, Duration threshold) {
            Instant now = Instant.now();
            Duration elapsed = Duration.between(eventTime, now);
            
            return elapsed.compareTo(threshold) < 0;
        }
        
        // Find events in time range
        public List<Event> findEventsInRange(Instant start, Instant end) {
            return eventRepository.findByTimestampBetween(start, end);
        }
    }
    
    // Convert to/from other types
    public void conversion() {
        Instant instant = Instant.now();
        
        // To ZonedDateTime (add timezone)
        ZonedDateTime zdt = instant.atZone(ZoneId.of("America/New_York"));
        
        // To OffsetDateTime (add offset)
        OffsetDateTime odt = instant.atOffset(ZoneOffset.UTC);
        
        // From ZonedDateTime
        ZonedDateTime zonedDateTime = ZonedDateTime.now();
        Instant fromZoned = zonedDateTime.toInstant();
        
        // From LocalDateTime (requires timezone)
        LocalDateTime localDateTime = LocalDateTime.now();
        Instant fromLocal = localDateTime.toInstant(ZoneOffset.UTC);
    }
}
```

---

## ZonedDateTime and Time Zones

### 1. **ZonedDateTime Basics**

```java
public class ZonedDateTimeExamples {
    
    /**
     * ZonedDateTime = LocalDateTime + ZoneId
     * - Includes timezone rules (DST, etc.)
     * - Best for: scheduling across timezones, user-facing times
     */
    
    // Creating ZonedDateTime instances
    public void creation() {
        // Current time in system timezone
        ZonedDateTime now = ZonedDateTime.now();
        
        // Current time in specific timezone
        ZonedDateTime nyTime = ZonedDateTime.now(ZoneId.of("America/New_York"));
        ZonedDateTime tokyoTime = ZonedDateTime.now(ZoneId.of("Asia/Tokyo"));
        
        // Specific date-time in timezone
        ZonedDateTime zdt1 = ZonedDateTime.of(
            2024, 1, 15, 14, 30, 0, 0,
            ZoneId.of("America/New_York")
        );
        
        // From LocalDateTime
        LocalDateTime ldt = LocalDateTime.of(2024, 1, 15, 14, 30);
        ZonedDateTime zdt2 = ldt.atZone(ZoneId.of("America/New_York"));
        
        // From Instant
        Instant instant = Instant.now();
        ZonedDateTime zdt3 = instant.atZone(ZoneId.of("America/New_York"));
        
        // From string
        ZonedDateTime zdt4 = ZonedDateTime.parse("2024-01-15T14:30:00-05:00[America/New_York]");
    }
    
    // Getting components
    public void extraction() {
        ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("America/New_York"));
        
        // All LocalDateTime methods work
        LocalDate date = zdt.toLocalDate();
        LocalTime time = zdt.toLocalTime();
        LocalDateTime localDateTime = zdt.toLocalDateTime();
        
        // Timezone information
        ZoneId zone = zdt.getZone();        // America/New_York
        ZoneOffset offset = zdt.getOffset(); // -05:00 or -04:00 (depends on DST)
        
        // Convert to Instant (UTC)
        Instant instant = zdt.toInstant();
    }
    
    // Timezone conversion
    public void conversion() {
        // Meeting at 2 PM EST
        ZonedDateTime nyTime = ZonedDateTime.of(
            2024, 1, 15, 14, 0, 0, 0,
            ZoneId.of("America/New_York")
        );
        
        // What time is it in Tokyo?
        ZonedDateTime tokyoTime = nyTime.withZoneSameInstant(
            ZoneId.of("Asia/Tokyo")
        );  // 2024-01-16T04:00+09:00[Asia/Tokyo]
        
        // Same local time, different timezone (usually wrong!)
        ZonedDateTime wrongConversion = nyTime.withZoneSameLocal(
            ZoneId.of("Asia/Tokyo")
        );  // 2024-01-15T14:00+09:00[Asia/Tokyo] - WRONG!
    }
    
    // Real-world examples
    public class MeetingScheduler {
        
        // Schedule meeting across timezones
        public Meeting scheduleMeeting(
                LocalDateTime localTime,
                ZoneId organizerZone,
                List<ZoneId> participantZones) {
            
            // Create meeting in organizer's timezone
            ZonedDateTime meetingTime = localTime.atZone(organizerZone);
            
            // Calculate time for each participant
            Map<ZoneId, ZonedDateTime> participantTimes = participantZones.stream()
                .collect(Collectors.toMap(
                    zone -> zone,
                    zone -> meetingTime.withZoneSameInstant(zone)
                ));
            
            return new Meeting(meetingTime, participantTimes);
        }
        
        // Check if time is within business hours for a timezone
        public boolean isWithinBusinessHours(ZonedDateTime time, ZoneId timezone) {
            ZonedDateTime localTime = time.withZoneSameInstant(timezone);
            LocalTime businessStart = LocalTime.of(9, 0);
            LocalTime businessEnd = LocalTime.of(17, 0);
            
            LocalTime checkTime = localTime.toLocalTime();
            return !checkTime.isBefore(businessStart) && !checkTime.isAfter(businessEnd);
        }
        
        // Find overlapping business hours
        public Optional<TimeRange> findOverlappingBusinessHours(
                ZoneId zone1, 
                ZoneId zone2,
                LocalDate date) {
            
            // Business hours in zone1: 9 AM - 5 PM
            ZonedDateTime start1 = date.atTime(9, 0).atZone(zone1);
            ZonedDateTime end1 = date.atTime(17, 0).atZone(zone1);
            
            // Convert to zone2
            ZonedDateTime start1InZone2 = start1.withZoneSameInstant(zone2);
            ZonedDateTime end1InZone2 = end1.withZoneSameInstant(zone2);
            
            // Business hours in zone2: 9 AM - 5 PM
            ZonedDateTime start2 = date.atTime(9, 0).atZone(zone2);
            ZonedDateTime end2 = date.atTime(17, 0).atZone(zone2);
            
            // Find overlap
            ZonedDateTime overlapStart = start1InZone2.isAfter(start2) 
                ? start1InZone2 : start2;
            ZonedDateTime overlapEnd = end1InZone2.isBefore(end2) 
                ? end1InZone2 : end2;
            
            if (overlapStart.isBefore(overlapEnd)) {
                return Optional.of(new TimeRange(overlapStart, overlapEnd));
            }
            
            return Optional.empty();
        }
    }
}
```

### 2. **Daylight Saving Time (DST)**

```java
public class DSTExamples {
    
    // DST transition handling
    public void dstTransition() {
        ZoneId newYork = ZoneId.of("America/New_York");
        
        // Spring forward (DST starts) - 2 AM becomes 3 AM
        // March 10, 2024 at 2:00 AM
        LocalDateTime springForward = LocalDateTime.of(2024, 3, 10, 2, 30);
        
        try {
            ZonedDateTime invalid = springForward.atZone(newYork);
            // This time doesn't exist! It's automatically adjusted to 3:30 AM
            System.out.println(invalid);  // 2024-03-10T03:30-04:00[America/New_York]
        } catch (DateTimeException e) {
            // Won't throw - automatically adjusted
        }
        
        // Fall back (DST ends) - 2 AM happens twice!
        // November 3, 2024 at 1:30 AM
        LocalDateTime fallBack = LocalDateTime.of(2024, 11, 3, 1, 30);
        
        // First occurrence (DST still in effect)
        ZonedDateTime first = fallBack.atZone(newYork);
        System.out.println(first);  // 2024-11-03T01:30-04:00[America/New_York]
        
        // After fallback (DST ended)
        ZonedDateTime second = first.plusHours(1);
        System.out.println(second);  // 2024-11-03T01:30-05:00[America/New_York]
    }
    
    // Safe DST handling
    public class SafeDSTHandling {
        
        // Add days (safe across DST)
        public ZonedDateTime addDays(ZonedDateTime dateTime, int days) {
            // plusDays handles DST correctly
            return dateTime.plusDays(days);
        }
        
        // Add exact duration (may cross DST)
        public ZonedDateTime addExactDuration(ZonedDateTime dateTime, Duration duration) {
            // Use Instant for exact duration
            Instant instant = dateTime.toInstant();
            Instant newInstant = instant.plus(duration);
            return newInstant.atZone(dateTime.getZone());
        }
        
        // Check if DST is in effect
        public boolean isDST(ZonedDateTime dateTime) {
            ZoneRules rules = dateTime.getZone().getRules();
            return rules.isDaylightSavings(dateTime.toInstant());
        }
        
        // Get next DST transition
        public Optional<ZonedDateTime> getNextDSTTransition(ZonedDateTime from) {
            ZoneRules rules = from.getZone().getRules();
            ZoneOffsetTransition transition = rules.nextTransition(from.toInstant());
            
            if (transition != null) {
                return Optional.of(transition.getInstant().atZone(from.getZone()));
            }
            
            return Optional.empty();
        }
    }
}
```

---

## Duration and Period

### 1. **Duration (Time-based)**

```java
public class DurationExamples {
    
    /**
     * Duration - Time-based amount (hours, minutes, seconds, nanos)
     * - Can be negative
     * - Works with Instant, LocalTime, LocalDateTime, ZonedDateTime
     */
    
    // Creating Duration instances
    public void creation() {
        // From amount
        Duration d1 = Duration.ofDays(2);
        Duration d2 = Duration.ofHours(5);
        Duration d3 = Duration.ofMinutes(30);
        Duration d4 = Duration.ofSeconds(45);
        Duration d5 = Duration.ofMillis(1000);
        Duration d6 = Duration.ofNanos(1000000);
        
        // From multiple units
        Duration d7 = Duration.ofHours(2).plusMinutes(30);  // 2h 30m
        
        // Between two temporals
        Instant start = Instant.now();
        Instant end = start.plusSeconds(3600);
        Duration d8 = Duration.between(start, end);  // 1 hour
        
        LocalTime time1 = LocalTime.of(9, 0);
        LocalTime time2 = LocalTime.of(17, 30);
        Duration d9 = Duration.between(time1, time2);  // 8h 30m
        
        // From string (ISO-8601)
        Duration d10 = Duration.parse("PT2H30M");  // 2 hours 30 minutes
        Duration d11 = Duration.parse("PT15M");     // 15 minutes
        Duration d12 = Duration.parse("P2DT3H4M"); // 2 days 3 hours 4 minutes
        
        // Special durations
        Duration zero = Duration.ZERO;
    }
    
    // Getting components
    public void extraction() {
        Duration duration = Duration.ofHours(2).plusMinutes(30).plusSeconds(45);
        
        long seconds = duration.getSeconds();      // Total seconds: 9045
        int nanos = duration.getNano();            // Nanoseconds part: 0
        
        // Convert to units
        long days = duration.toDays();             // 0
        long hours = duration.toHours();           // 2
        long minutes = duration.toMinutes();       // 150
        long totalSeconds = duration.getSeconds(); // 9045
        long millis = duration.toMillis();         // 9045000
        long totalNanos = duration.toNanos();      // 9045000000000
        
        // Get specific parts
        long hoursPart = duration.toHoursPart();   // 2
        int minutesPart = duration.toMinutesPart(); // 30
        int secondsPart = duration.toSecondsPart(); // 45
    }
    
    // Arithmetic
    public void arithmetic() {
        Duration d1 = Duration.ofHours(2);
        Duration d2 = Duration.ofMinutes(30);
        
        Duration sum = d1.plus(d2);           // 2h 30m
        Duration diff = d1.minus(d2);         // 1h 30m
        Duration doubled = d1.multipliedBy(2); // 4h
        Duration halved = d1.dividedBy(2);    // 1h
        Duration negated = d1.negated();      // -2h
        Duration absolute = d1.abs();         // 2h
    }
    
    // Comparison
    public void comparison() {
        Duration d1 = Duration.ofHours(2);
        Duration d2 = Duration.ofMinutes(150);
        
        boolean isZero = d1.isZero();
        boolean isNegative = d1.isNegative();
        
        int comparison = d1.compareTo(d2);  // -1 (2h < 2.5h)
    }
    
    // Real-world examples
    public class RateLimiter {
        
        private final Map<String, Instant> lastRequestTime = new ConcurrentHashMap<>();
        private final Duration cooldown = Duration.ofSeconds(1);
        
        public boolean canMakeRequest(String userId) {
            Instant now = Instant.now();
            Instant lastRequest = lastRequestTime.get(userId);
            
            if (lastRequest == null) {
                lastRequestTime.put(userId, now);
                return true;
            }
            
            Duration timeSinceLastRequest = Duration.between(lastRequest, now);
            
            if (timeSinceLastRequest.compareTo(cooldown) >= 0) {
                lastRequestTime.put(userId, now);
                return true;
            }
            
            return false;
        }
    }
    
    public class SessionManager {
        
        private static final Duration SESSION_TIMEOUT = Duration.ofMinutes(30);
        
        public boolean isSessionValid(Instant lastActivity) {
            Instant now = Instant.now();
            Duration inactive = Duration.between(lastActivity, now);
            
            return inactive.compareTo(SESSION_TIMEOUT) < 0;
        }
        
        public Duration getRemainingTime(Instant lastActivity) {
            Instant now = Instant.now();
            Duration inactive = Duration.between(lastActivity, now);
            Duration remaining = SESSION_TIMEOUT.minus(inactive);
            
            return remaining.isNegative() ? Duration.ZERO : remaining;
        }
    }
}
```

### 2. **Period (Date-based)**

```java
public class PeriodExamples {
    
    /**
     * Period - Date-based amount (years, months, days)
     * - Does NOT include time components
     * - Works with LocalDate
     */
    
    // Creating Period instances
    public void creation() {
        // From amount
        Period p1 = Period.ofYears(2);
        Period p2 = Period.ofMonths(6);
        Period p3 = Period.ofWeeks(3);  // Converted to days (21)
        Period p4 = Period.ofDays(10);
        
        // Combined
        Period p5 = Period.of(2, 6, 10);  // 2 years, 6 months, 10 days
        
        // Between two dates
        LocalDate start = LocalDate.of(2020, 1, 15);
        LocalDate end = LocalDate.of(2024, 3, 20);
        Period p6 = Period.between(start, end);  // 4 years, 2 months, 5 days
        
        // From string (ISO-8601)
        Period p7 = Period.parse("P2Y6M10D");  // 2 years, 6 months, 10 days
        Period p8 = Period.parse("P3M");        // 3 months
        
        // Special periods
        Period zero = Period.ZERO;
    }
    
    // Getting components
    public void extraction() {
        Period period = Period.of(2, 6, 10);
        
        int years = period.getYears();   // 2
        int months = period.getMonths(); // 6
        int days = period.getDays();     // 10
        
        // Total months (years * 12 + months)
        long totalMonths = period.toTotalMonths();  // 30
    }
    
    // Arithmetic
    public void arithmetic() {
        Period p1 = Period.ofYears(2);
        Period p2 = Period.ofMonths(6);
        
        Period sum = p1.plus(p2);           // 2 years, 6 months
        Period diff = p1.minus(p2);         // 1 year, 6 months
        Period doubled = p1.multipliedBy(2); // 4 years
        Period negated = p1.negated();      // -2 years
    }
    
    // Normalization
    public void normalization() {
        // 14 months normalized to 1 year 2 months
        Period period = Period.ofMonths(14);
        Period normalized = period.normalized();  // 1 year, 2 months
        
        // WARNING: Days are NOT normalized (months have variable days)
        Period p2 = Period.ofDays(40);
        Period notNormalized = p2.normalized();  // Still 40 days, not 1 month 10 days
    }
    
    // Real-world examples
    public int calculateAge(LocalDate birthDate) {
        return Period.between(birthDate, LocalDate.now()).getYears();
    }
    
    public String formatAge(LocalDate birthDate) {
        Period age = Period.between(birthDate, LocalDate.now());
        return String.format("%d years, %d months, %d days",
            age.getYears(), age.getMonths(), age.getDays());
    }
    
    public boolean isEligibleForDiscount(LocalDate birthDate) {
        int age = calculateAge(birthDate);
        return age >= 65;  // Senior discount
    }
    
    public LocalDate calculateMaturityDate(LocalDate startDate, Period tenure) {
        return startDate.plus(tenure);
    }
    
    // Subscription example
    public class SubscriptionService {
        
        public LocalDate calculateRenewalDate(LocalDate startDate, Period billingPeriod) {
            return startDate.plus(billingPeriod);
        }
        
        public List<LocalDate> generateBillingDates(
                LocalDate startDate,
                LocalDate endDate,
                Period billingPeriod) {
            
            List<LocalDate> dates = new ArrayList<>();
            LocalDate current = startDate;
            
            while (current.isBefore(endDate) || current.isEqual(endDate)) {
                dates.add(current);
                current = current.plus(billingPeriod);
            }
            
            return dates;
        }
        
        public Period getRemainingPeriod(LocalDate startDate, LocalDate endDate) {
            return Period.between(LocalDate.now(), endDate);
        }
    }
}
```

---

## Formatting and Parsing

### 1. **DateTimeFormatter**

```java
public class FormattingExamples {
    
    // Predefined formatters
    public void predefinedFormatters() {
        LocalDateTime dateTime = LocalDateTime.of(2024, 1, 15, 14, 30, 45);
        
        // ISO formatters
        String iso = dateTime.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        // 2024-01-15T14:30:45
        
        String isoDate = dateTime.format(DateTimeFormatter.ISO_LOCAL_DATE);
        // 2024-01-15
        
        String isoTime = dateTime.format(DateTimeFormatter.ISO_LOCAL_TIME);
        // 14:30:45
        
        // Basic ISO
        String basic = dateTime.format(DateTimeFormatter.BASIC_ISO_DATE);
        // 20240115
        
        // RFC-1123 (for HTTP headers)
        ZonedDateTime zdt = dateTime.atZone(ZoneId.of("GMT"));
        String rfc = zdt.format(DateTimeFormatter.RFC_1123_DATE_TIME);
        // Mon, 15 Jan 2024 14:30:45 GMT
    }
    
    // Custom patterns
    public void customPatterns() {
        LocalDateTime dateTime = LocalDateTime.of(2024, 1, 15, 14, 30, 45);
        
        // Common patterns
        DateTimeFormatter formatter1 = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        String s1 = dateTime.format(formatter1);  // 2024-01-15 14:30:45
        
        DateTimeFormatter formatter2 = DateTimeFormatter.ofPattern("dd/MM/yyyy");
        String s2 = dateTime.format(formatter2);  // 15/01/2024
        
        DateTimeFormatter formatter3 = DateTimeFormatter.ofPattern("MMM dd, yyyy");
        String s3 = dateTime.format(formatter3);  // Jan 15, 2024
        
        DateTimeFormatter formatter4 = DateTimeFormatter.ofPattern("EEEE, MMMM dd, yyyy");
        String s4 = dateTime.format(formatter4);  // Monday, January 15, 2024
        
        DateTimeFormatter formatter5 = DateTimeFormatter.ofPattern("hh:mm a");
        String s5 = dateTime.format(formatter5);  // 02:30 PM
        
        // With timezone
        ZonedDateTime zdt = dateTime.atZone(ZoneId.of("America/New_York"));
        DateTimeFormatter formatter6 = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss z");
        String s6 = zdt.format(formatter6);  // 2024-01-15 14:30:45 EST
    }
    
    /**
     * Pattern symbols:
     * 
     * y - Year (2024, 24)
     * M - Month (1, 01, Jan, January)
     * d - Day of month (1, 01)
     * H - Hour 0-23 (14)
     * h - Hour 1-12 (2)
     * m - Minute (30)
     * s - Second (45)
     * S - Millisecond (123)
     * a - AM/PM (PM)
     * E - Day of week (Mon, Monday)
     * z - Time zone (EST, America/New_York)
     * Z - Time zone offset (-0500)
     */
    
    // Localized formatting
    public void localizedFormatting() {
        LocalDateTime dateTime = LocalDateTime.of(2024, 1, 15, 14, 30);
        
        // US format
        DateTimeFormatter usFormatter = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM)
            .withLocale(Locale.US);
        String us = dateTime.format(usFormatter);  // Jan 15, 2024, 2:30:00 PM
        
        // French format
        DateTimeFormatter frFormatter = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM)
            .withLocale(Locale.FRANCE);
        String fr = dateTime.format(frFormatter);  // 15 janv. 2024, 14:30:00
        
        // German format
        DateTimeFormatter deFormatter = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM)
            .withLocale(Locale.GERMANY);
        String de = dateTime.format(deFormatter);  // 15.01.2024, 14:30:00
    }
    
    // Parsing
    public void parsing() {
        // With predefined formatter
        LocalDate date1 = LocalDate.parse("2024-01-15", DateTimeFormatter.ISO_LOCAL_DATE);
        
        // With custom formatter
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
        LocalDate date2 = LocalDate.parse("15/01/2024", formatter);
        
        // Lenient parsing
        DateTimeFormatter lenient = DateTimeFormatter.ofPattern("d/M/yyyy")
            .withResolverStyle(ResolverStyle.LENIENT);
        LocalDate date3 = LocalDate.parse("5/3/2024", lenient);  // Single digit ok
        
        // Parse with default values
        DateTimeFormatter withDefaults = new DateTimeFormatterBuilder()
            .appendPattern("dd/MM/yyyy")
            .parseDefaulting(ChronoField.HOUR_OF_DAY, 12)
            .parseDefaulting(ChronoField.MINUTE_OF_HOUR, 0)
            .toFormatter();
        
        LocalDateTime dateTime = LocalDate.parse("15/01/2024", withDefaults)
            .atTime(12, 0);
    }
    
    // Thread-safe formatting
    public class DateFormatter {
        
        // DateTimeFormatter is immutable and thread-safe
        private static final DateTimeFormatter FORMATTER = 
            DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        
        public String format(LocalDateTime dateTime) {
            return dateTime.format(FORMATTER);  // Safe to share
        }
        
        // Can be used concurrently
        public List<String> formatDates(List<LocalDateTime> dates) {
            return dates.parallelStream()
                .map(this::format)
                .collect(Collectors.toList());
        }
    }
}
```

---

## Date Arithmetic

```java
public class DateArithmetic {
    
    // Adding/subtracting units
    public void basicArithmetic() {
        LocalDate date = LocalDate.of(2024, 1, 15);
        
        LocalDate tomorrow = date.plusDays(1);
        LocalDate nextWeek = date.plusWeeks(1);
        LocalDate nextMonth = date.plusMonths(1);
        LocalDate nextYear = date.plusYears(1);
        
        LocalDate yesterday = date.minusDays(1);
        
        // Can chain
        LocalDate future = date.plusYears(2).plusMonths(3).plusDays(10);
    }
    
    // Using Period/Duration
    public void usingPeriodDuration() {
        LocalDate date = LocalDate.of(2024, 1, 15);
        Period period = Period.ofMonths(3);
        LocalDate future = date.plus(period);  // 2024-04-15
        
        LocalDateTime dateTime = LocalDateTime.of(2024, 1, 15, 14, 30);
        Duration duration = Duration.ofHours(2);
        LocalDateTime later = dateTime.plus(duration);  // 2024-01-15T16:30
    }
    
    // Temporal adjusters
    public void temporalAdjusters() {
        LocalDate date = LocalDate.of(2024, 1, 15);  // Monday
        
        // Next/Previous day of week
        LocalDate nextMonday = date.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
        LocalDate previousFriday = date.with(TemporalAdjusters.previous(DayOfWeek.FRIDAY));
        LocalDate nextOrSameMonday = date.with(TemporalAdjusters.nextOrSame(DayOfWeek.MONDAY));
        
        // First/Last day of month
        LocalDate firstDay = date.with(TemporalAdjusters.firstDayOfMonth());
        LocalDate lastDay = date.with(TemporalAdjusters.lastDayOfMonth());
        
        // First/Last day of year
        LocalDate firstDayOfYear = date.with(TemporalAdjusters.firstDayOfYear());
        LocalDate lastDayOfYear = date.with(TemporalAdjusters.lastDayOfYear());
        
        // First/Last day of next month
        LocalDate firstDayOfNextMonth = date.with(TemporalAdjusters.firstDayOfNextMonth());
        LocalDate lastDayOfNextMonth = date.with(TemporalAdjusters.lastDayOfNextMonth());
        
        // Nth day of week in month
        LocalDate secondTuesday = date.with(
            TemporalAdjusters.dayOfWeekInMonth(2, DayOfWeek.TUESDAY)
        );
        
        // Last occurrence of day in month
        LocalDate lastFriday = date.with(
            TemporalAdjusters.lastInMonth(DayOfWeek.FRIDAY)
        );
    }
    
    // Custom temporal adjuster
    public void customAdjuster() {
        // Next business day adjuster
        TemporalAdjuster nextBusinessDay = temporal -> {
            LocalDate date = LocalDate.from(temporal);
            DayOfWeek dayOfWeek = date.getDayOfWeek();
            
            int daysToAdd = switch (dayOfWeek) {
                case FRIDAY -> 3;  // Friday -> Monday
                case SATURDAY -> 2; // Saturday -> Monday
                default -> 1;      // Any other day -> next day
            };
            
            return date.plusDays(daysToAdd);
        };
        
        LocalDate date = LocalDate.of(2024, 1, 19);  // Friday
        LocalDate nextBizDay = date.with(nextBusinessDay);  // Monday, Jan 22
    }
    
    // Business day calculator
    public class BusinessDayCalculator {
        
        private final Set<LocalDate> holidays;
        
        public BusinessDayCalculator(Set<LocalDate> holidays) {
            this.holidays = holidays;
        }
        
        public boolean isBusinessDay(LocalDate date) {
            DayOfWeek day = date.getDayOfWeek();
            return day != DayOfWeek.SATURDAY 
                && day != DayOfWeek.SUNDAY
                && !holidays.contains(date);
        }
        
        public LocalDate addBusinessDays(LocalDate startDate, int businessDays) {
            LocalDate result = startDate;
            int daysAdded = 0;
            
            while (daysAdded < businessDays) {
                result = result.plusDays(1);
                if (isBusinessDay(result)) {
                    daysAdded++;
                }
            }
            
            return result;
        }
        
        public int countBusinessDays(LocalDate start, LocalDate end) {
            int count = 0;
            LocalDate current = start;
            
            while (current.isBefore(end) || current.isEqual(end)) {
                if (isBusinessDay(current)) {
                    count++;
                }
                current = current.plusDays(1);
            }
            
            return count;
        }
    }
}
```

---

## Working with Legacy Code

```java
public class LegacyConversion {
    
    // java.util.Date ↔ Instant
    public void dateToInstant() {
        // Date → Instant
        Date legacyDate = new Date();
        Instant instant = legacyDate.toInstant();
        
        // Instant → Date
        Instant instant2 = Instant.now();
        Date date = Date.from(instant2);
    }
    
    // java.util.Calendar ↔ ZonedDateTime
    public void calendarToZonedDateTime() {
        // Calendar → ZonedDateTime
        Calendar calendar = Calendar.getInstance();
        ZonedDateTime zdt = ZonedDateTime.ofInstant(
            calendar.toInstant(),
            calendar.getTimeZone().toZoneId()
        );
        
        // ZonedDateTime → Calendar
        ZonedDateTime zonedDateTime = ZonedDateTime.now();
        Calendar cal = GregorianCalendar.from(zonedDateTime);
    }
    
    // java.sql.Date ↔ LocalDate
    public void sqlDateToLocalDate() {
        // SQL Date → LocalDate
        java.sql.Date sqlDate = java.sql.Date.valueOf("2024-01-15");
        LocalDate localDate = sqlDate.toLocalDate();
        
        // LocalDate → SQL Date
        LocalDate date = LocalDate.now();
        java.sql.Date sqlDate2 = java.sql.Date.valueOf(date);
    }
    
    // java.sql.Timestamp ↔ Instant/LocalDateTime
    public void timestampConversion() {
        // Timestamp → Instant
        Timestamp timestamp = new Timestamp(System.currentTimeMillis());
        Instant instant = timestamp.toInstant();
        
        // Timestamp → LocalDateTime
        LocalDateTime localDateTime = timestamp.toLocalDateTime();
        
        // Instant → Timestamp
        Instant instant2 = Instant.now();
        Timestamp ts = Timestamp.from(instant2);
        
        // LocalDateTime → Timestamp
        LocalDateTime ldt = LocalDateTime.now();
        Timestamp ts2 = Timestamp.valueOf(ldt);
    }
    
    // TimeZone ↔ ZoneId
    public void timeZoneConversion() {
        // TimeZone → ZoneId
        TimeZone timeZone = TimeZone.getDefault();
        ZoneId zoneId = timeZone.toZoneId();
        
        // ZoneId → TimeZone
        ZoneId zoneId2 = ZoneId.of("America/New_York");
        TimeZone tz = TimeZone.getTimeZone(zoneId2);
    }
}
```

---

## Database Integration

```java
public class DatabaseIntegration {
    
    // JPA/Hibernate entities
    @Entity
    public class Event {
        
        @Id
        private Long id;
        
        // Store as TIMESTAMP in database
        @Column(name = "created_at")
        private Instant createdAt;
        
        // Store as DATE in database
        @Column(name = "event_date")
        private LocalDate eventDate;
        
        // Store as TIME in database
        @Column(name = "event_time")
        private LocalTime eventTime;
        
        // Store as TIMESTAMP in database
        @Column(name = "event_datetime")
        private LocalDateTime eventDateTime;
        
        // Store as TIMESTAMP WITH TIME ZONE (PostgreSQL)
        @Column(name = "scheduled_at")
        private ZonedDateTime scheduledAt;
        
        // Store as TIMESTAMP WITH TIME ZONE
        @Column(name = "scheduled_offset")
        private OffsetDateTime scheduledOffset;
    }
    
    // Spring Data JPA queries
    @Repository
    public interface EventRepository extends JpaRepository<Event, Long> {
        
        // Find events after specific instant
        List<Event> findByCreatedAtAfter(Instant instant);
        
        // Find events on specific date
        List<Event> findByEventDate(LocalDate date);
        
        // Find events between dates
        List<Event> findByEventDateBetween(LocalDate start, LocalDate end);
        
        // Find events in date range
        @Query("SELECT e FROM Event e WHERE e.eventDate >= :start AND e.eventDate <= :end")
        List<Event> findEventsInRange(
            @Param("start") LocalDate start,
            @Param("end") LocalDate end
        );
    }
    
    // JDBC handling
    public class JDBCDateHandling {
        
        public void saveEvent(Event event) throws SQLException {
            String sql = "INSERT INTO events (created_at, event_date, event_time) VALUES (?, ?, ?)";
            
            try (PreparedStatement stmt = connection.prepareStatement(sql)) {
                // Instant → Timestamp
                stmt.setTimestamp(1, Timestamp.from(event.getCreatedAt()));
                
                // LocalDate → java.sql.Date
                stmt.setDate(2, java.sql.Date.valueOf(event.getEventDate()));
                
                // LocalTime → java.sql.Time
                stmt.setTime(3, java.sql.Time.valueOf(event.getEventTime()));
                
                stmt.executeUpdate();
            }
        }
        
        public Event loadEvent(Long id) throws SQLException {
            String sql = "SELECT created_at, event_date, event_time FROM events WHERE id = ?";
            
            try (PreparedStatement stmt = connection.prepareStatement(sql)) {
                stmt.setLong(1, id);
                
                try (ResultSet rs = stmt.executeQuery()) {
                    if (rs.next()) {
                        Event event = new Event();
                        
                        // Timestamp → Instant
                        event.setCreatedAt(rs.getTimestamp("created_at").toInstant());
                        
                        // java.sql.Date → LocalDate
                        event.setEventDate(rs.getDate("event_date").toLocalDate());
                        
                        // java.sql.Time → LocalTime
                        event.setEventTime(rs.getTime("event_time").toLocalTime());
                        
                        return event;
                    }
                }
            }
            
            return null;
        }
    }
}
```

---

## REST API Best Practices

```java
public class RESTAPIBestPractices {
    
    /**
     * Best Practices:
     * 1. Always use ISO-8601 format
     * 2. Always include timezone information
     * 3. Use Instant for timestamps (UTC)
     * 4. Use ZonedDateTime for user-facing times
     * 5. Never use LocalDateTime in APIs (ambiguous!)
     */
    
    // Jackson configuration
    @Configuration
    public class JacksonConfig {
        
        @Bean
        public ObjectMapper objectMapper() {
            ObjectMapper mapper = new ObjectMapper();
            
            // Register JavaTimeModule for java.time support
            mapper.registerModule(new JavaTimeModule());
            
            // Write dates as ISO-8601 strings (not timestamps)
            mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
            
            return mapper;
        }
    }
    
    // DTO with proper date/time fields
    public record EventDTO(
        String id,
        String name,
        Instant createdAt,          // Machine timestamp (UTC)
        ZonedDateTime scheduledAt,  // User-facing time with timezone
        LocalDate eventDate         // Date only (no time)
    ) {}
    
    // REST Controller
    @RestController
    @RequestMapping("/api/events")
    public class EventController {
        
        // ✅ GOOD - Returns Instant (UTC timestamp)
        @GetMapping("/{id}")
        public ResponseEntity<EventDTO> getEvent(@PathVariable String id) {
            Event event = eventService.findById(id);
            
            EventDTO dto = new EventDTO(
                event.getId(),
                event.getName(),
                event.getCreatedAt(),      // Instant
                event.getScheduledAt(),    // ZonedDateTime
                event.getEventDate()       // LocalDate
            );
            
            return ResponseEntity.ok(dto);
        }
        
        // ✅ GOOD - Accepts ISO-8601 format automatically
        @PostMapping
        public ResponseEntity<EventDTO> createEvent(@RequestBody CreateEventRequest request) {
            Event event = eventService.create(request);
            return ResponseEntity.ok(toDTO(event));
        }
        
        // ✅ GOOD - Query parameter with date
        @GetMapping
        public ResponseEntity<List<EventDTO>> getEventsByDate(
                @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate date) {
            
            List<Event> events = eventService.findByDate(date);
            return ResponseEntity.ok(events.stream()
                .map(this::toDTO)
                .collect(Collectors.toList()));
        }
        
        // ✅ GOOD - Range query with proper date handling
        @GetMapping("/range")
        public ResponseEntity<List<EventDTO>> getEventsInRange(
                @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) 
                ZonedDateTime start,
                @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE_TIME) 
                ZonedDateTime end) {
            
            // Convert to UTC for database query
            Instant startInstant = start.toInstant();
            Instant endInstant = end.toInstant();
            
            List<Event> events = eventService.findByRange(startInstant, endInstant);
            return ResponseEntity.ok(events.stream()
                .map(this::toDTO)
                .collect(Collectors.toList()));
        }
    }
    
    // Example JSON serialization
    public void exampleJSON() {
        EventDTO dto = new EventDTO(
            "123",
            "Team Meeting",
            Instant.parse("2024-01-15T19:30:00Z"),
            ZonedDateTime.parse("2024-01-15T14:30:00-05:00[America/New_York]"),
            LocalDate.parse("2024-01-15")
        );
        
        /**
         * JSON output:
         * {
         *   "id": "123",
         *   "name": "Team Meeting",
         *   "createdAt": "2024-01-15T19:30:00Z",
         *   "scheduledAt": "2024-01-15T14:30:00-05:00[America/New_York]",
         *   "eventDate": "2024-01-15"
         * }
         */
    }
}
```

---

## Testing with Time

```java
public class TestingWithTime {
    
    // Problem: Tests that depend on current time are non-deterministic
    
    // ❌ BAD - Hard to test
    public class OrderService {
        public boolean isExpired(Order order) {
            Instant now = Instant.now();  // Current time - hard to control!
            return order.getExpiresAt().isBefore(now);
        }
    }
    
    // ✅ GOOD - Inject Clock
    public class OrderServiceTestable {
        
        private final Clock clock;
        
        public OrderServiceTestable(Clock clock) {
            this.clock = clock;
        }
        
        public boolean isExpired(Order order) {
            Instant now = Instant.now(clock);  // Controllable!
            return order.getExpiresAt().isBefore(now);
        }
    }
    
    // Unit tests
    @Test
    void shouldDetectExpiredOrder() {
        // Arrange - Fix time to specific instant
        Instant fixedTime = Instant.parse("2024-01-15T14:30:00Z");
        Clock fixedClock = Clock.fixed(fixedTime, ZoneId.of("UTC"));
        
        OrderServiceTestable service = new OrderServiceTestable(fixedClock);
        
        Order order = new Order();
        order.setExpiresAt(Instant.parse("2024-01-15T14:00:00Z"));  // Expired
        
        // Act & Assert
        assertTrue(service.isExpired(order));
    }
    
    @Test
    void shouldDetectValidOrder() {
        // Arrange
        Instant fixedTime = Instant.parse("2024-01-15T14:30:00Z");
        Clock fixedClock = Clock.fixed(fixedTime, ZoneId.of("UTC"));
        
        OrderServiceTestable service = new OrderServiceTestable(fixedClock);
        
        Order order = new Order();
        order.setExpiresAt(Instant.parse("2024-01-15T15:00:00Z"));  // Not expired
        
        // Act & Assert
        assertFalse(service.isExpired(order));
    }
    
    // Testing with different timezones
    @Test
    void shouldHandleTimezones() {
        ZoneId newYork = ZoneId.of("America/New_York");
        Clock nyClock = Clock.system(newYork);
        
        OrderServiceTestable service = new OrderServiceTestable(nyClock);
        
        // Test logic
    }
    
    // Clock types
    public void clockTypes() {
        // System clock (default)
        Clock systemClock = Clock.systemDefaultZone();
        Clock utcClock = Clock.systemUTC();
        Clock nyClock = Clock.system(ZoneId.of("America/New_York"));
        
        // Fixed clock (for testing)
        Instant fixedInstant = Instant.parse("2024-01-15T14:30:00Z");
        Clock fixedClock = Clock.fixed(fixedInstant, ZoneId.of("UTC"));
        
        // Offset clock (shift time)
        Clock offsetClock = Clock.offset(systemClock, Duration.ofHours(2));
        
        // Tick clock (truncate to specific precision)
        Clock tickClock = Clock.tick(systemClock, Duration.ofSeconds(1));
    }
    
    // Spring Boot configuration
    @Configuration
    public class ClockConfig {
        
        @Bean
        @Profile("!test")
        public Clock clock() {
            return Clock.systemDefaultZone();
        }
        
        @Bean
        @Profile("test")
        public Clock testClock() {
            // Can be overridden in tests
            return Clock.systemDefaultZone();
        }
    }
}
```

---

## Common Patterns

```java
public class CommonPatterns {
    
    // 1. Date range validation
    public boolean isWithinRange(LocalDate date, LocalDate start, LocalDate end) {
        return !date.isBefore(start) && !date.isAfter(end);
    }
    
    // 2. Overlap detection
    public boolean hasOverlap(
            LocalDateTime start1, LocalDateTime end1,
            LocalDateTime start2, LocalDateTime end2) {
        
        return start1.isBefore(end2) && start2.isBefore(end1);
    }
    
    // 3. Start/end of day
    public Instant startOfDay(LocalDate date, ZoneId zone) {
        return date.atStartOfDay(zone).toInstant();
    }
    
    public Instant endOfDay(LocalDate date, ZoneId zone) {
        return date.atTime(LocalTime.MAX).atZone(zone).toInstant();
    }
    
    // 4. Month boundaries
    public LocalDate firstDayOfMonth(YearMonth yearMonth) {
        return yearMonth.atDay(1);
    }
    
    public LocalDate lastDayOfMonth(YearMonth yearMonth) {
        return yearMonth.atEndOfMonth();
    }
    
    // 5. Quarter calculations
    public YearMonth getQuarterStart(YearMonth month) {
        int quarter = (month.getMonthValue() - 1) / 3;
        int startMonth = quarter * 3 + 1;
        return YearMonth.of(month.getYear(), startMonth);
    }
    
    // 6. Fiscal year
    public int getFiscalYear(LocalDate date, Month fiscalYearStart) {
        int year = date.getYear();
        if (date.getMonth().getValue() < fiscalYearStart.getValue()) {
            return year;
        }
        return year + 1;
    }
    
    // 7. Cache with expiry
    public class ExpiringCache<K, V> {
        
        private final Map<K, CacheEntry<V>> cache = new ConcurrentHashMap<>();
        private final Duration ttl;
        private final Clock clock;
        
        public ExpiringCache(Duration ttl, Clock clock) {
            this.ttl = ttl;
            this.clock = clock;
        }
        
        public void put(K key, V value) {
            Instant expiresAt = Instant.now(clock).plus(ttl);
            cache.put(key, new CacheEntry<>(value, expiresAt));
        }
        
        public Optional<V> get(K key) {
            CacheEntry<V> entry = cache.get(key);
            
            if (entry == null) {
                return Optional.empty();
            }
            
            Instant now = Instant.now(clock);
            if (now.isAfter(entry.expiresAt())) {
                cache.remove(key);
                return Optional.empty();
            }
            
            return Optional.of(entry.value());
        }
        
        record CacheEntry<V>(V value, Instant expiresAt) {}
    }
    
    // 8. Retry with exponential backoff
    public class RetryWithBackoff {
        
        public <T> T executeWithRetry(
                Supplier<T> operation,
                int maxRetries,
                Duration initialDelay) {
            
            int attempt = 0;
            Duration delay = initialDelay;
            
            while (true) {
                try {
                    return operation.get();
                } catch (Exception e) {
                    attempt++;
                    
                    if (attempt >= maxRetries) {
                        throw e;
                    }
                    
                    sleep(delay);
                    delay = delay.multipliedBy(2);  // Exponential backoff
                }
            }
        }
        
        private void sleep(Duration duration) {
            try {
                Thread.sleep(duration.toMillis());
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

---

## Pitfalls to Avoid

```java
public class Pitfalls {
    
    // ❌ PITFALL 1: Using LocalDateTime for API timestamps
    public class WrongAPI {
        // WRONG - Ambiguous! What timezone?
        public record Event(String id, LocalDateTime timestamp) {}
    }
    
    public class CorrectAPI {
        // RIGHT - Unambiguous UTC timestamp
        public record Event(String id, Instant timestamp) {}
        
        // Or with timezone
        public record Event2(String id, ZonedDateTime timestamp) {}
    }
    
    // ❌ PITFALL 2: Comparing LocalDateTime across timezones
    public void wrongComparison() {
        LocalDateTime ny = LocalDateTime.of(2024, 1, 15, 14, 0);
        LocalDateTime tokyo = LocalDateTime.of(2024, 1, 15, 14, 0);
        
        boolean same = ny.equals(tokyo);  // true, but they're 14 hours apart!
    }
    
    public void correctComparison() {
        ZonedDateTime ny = ZonedDateTime.of(
            LocalDateTime.of(2024, 1, 15, 14, 0),
            ZoneId.of("America/New_York")
        );
        
        ZonedDateTime tokyo = ZonedDateTime.of(
            LocalDateTime.of(2024, 1, 15, 14, 0),
            ZoneId.of("Asia/Tokyo")
        );
        
        // Compare as Instant
        boolean same = ny.toInstant().equals(tokyo.toInstant());  // false - correct!
    }
    
    // ❌ PITFALL 3: Forgetting that types are immutable
    public void wrongModification() {
        LocalDate date = LocalDate.of(2024, 1, 15);
        date.plusDays(1);  // Does nothing! Result is ignored
        System.out.println(date);  // Still 2024-01-15
    }
    
    public void correctModification() {
        LocalDate date = LocalDate.of(2024, 1, 15);
        date = date.plusDays(1);  // Reassign
        System.out.println(date);  // 2024-01-16
    }
    
    // ❌ PITFALL 4: Using wrong type for duration
    public void wrongDuration() {
        // WRONG - Period is date-based, doesn't work with time
        LocalTime time = LocalTime.of(14, 30);
        // time.plus(Period.ofDays(1));  // Runtime error!
    }
    
    public void correctDuration() {
        LocalTime time = LocalTime.of(14, 30);
        time = time.plus(Duration.ofHours(2));  // Correct
        
        LocalDate date = LocalDate.of(2024, 1, 15);
        date = date.plus(Period.ofDays(1));  // Correct
    }
    
    // ❌ PITFALL 5: Hardcoding timezones
    public void hardcodedTimezone() {
        // WRONG - Assumes server timezone
        ZonedDateTime now = ZonedDateTime.now();  // Uses system default
    }
    
    public void explicitTimezone() {
        // RIGHT - Explicit timezone
        ZonedDateTime now = ZonedDateTime.now(ZoneId.of("UTC"));
    }
    
    // ❌ PITFALL 6: Not handling DST transitions
    public void dstProblem() {
        ZoneId newYork = ZoneId.of("America/New_York");
        
        // Spring forward - 2 AM doesn't exist!
        LocalDateTime invalid = LocalDateTime.of(2024, 3, 10, 2, 30);
        ZonedDateTime zdt = invalid.atZone(newYork);
        
        // Automatically adjusted to 3:30 AM - may not be what you want!
        System.out.println(zdt);
    }
    
    public void dstHandling() {
        // Be aware of DST transitions
        // Use Instant for absolute time
        // Use ZonedDateTime carefully
        // Test around DST boundaries!
    }
}
```

---

## Decision Tree

```
Date/Time Type Selection:

START: What do you need to represent?

├─ Just a DATE (no time)?
│  └─► Use LocalDate
│      Examples: Birthday, holiday, report date
│      Format: 2024-01-15
│
├─ Just a TIME (no date)?
│  └─► Use LocalTime
│      Examples: Business hours, meeting time
│      Format: 14:30:00
│
├─ DATE and TIME together?
│  │
│  ├─ Need TIMEZONE information?
│  │  │
│  │  ├─ For API/storage/logging?
│  │  │  └─► Use Instant
│  │  │      Always UTC, best for timestamps
│  │  │      Format: 2024-01-15T19:30:00Z
│  │  │
│  │  └─ For user display/scheduling?
│  │     └─► Use ZonedDateTime
│  │         Includes timezone rules (DST)
│  │         Format: 2024-01-15T14:30:00-05:00[America/New_York]
│  │
│  └─ NO timezone needed (single timezone context)?
│     └─► Use LocalDateTime
│         Examples: Appointment in office, local event
│         Format: 2024-01-15T14:30:00
│         ⚠️  Never use in APIs!
│
├─ Need to measure TIME DURATION?
│  │
│  ├─ Time-based (hours/minutes/seconds)?
│  │  └─► Use Duration
│  │      Examples: Timeout, execution time, video length
│  │      Format: PT2H30M (2 hours 30 minutes)
│  │
│  └─ Date-based (years/months/days)?
│     └─► Use Period
│         Examples: Age, subscription period, loan tenure
│         Format: P1Y2M10D (1 year 2 months 10 days)
│
└─ Need YEAR-MONTH only?
   └─► Use YearMonth
       Examples: Credit card expiry, billing period
       Format: 2024-01
```

### Quick Reference

| Use Case | Type | Example |
|----------|------|---------|
| Database timestamp | `Instant` | Event logs, audit trails |
| API request/response time | `Instant` or `ZonedDateTime` | Created/modified timestamps |
| User birthday | `LocalDate` | 1990-05-15 |
| Business hours | `LocalTime` | 09:00 - 17:00 |
| Meeting in single timezone | `LocalDateTime` | Office appointment |
| Global meeting | `ZonedDateTime` | Webinar across timezones |
| Session timeout | `Duration` | 30 minutes |
| Age calculation | `Period` | Years since birth |
| Credit card expiry | `YearMonth` | 2025-12 |

---

## Best Practices Summary

### ✅ DO

1. **Use Instant for timestamps** - Always UTC, unambiguous
2. **Use ZonedDateTime for user-facing times** - Includes timezone
3. **Never use LocalDateTime in APIs** - Ambiguous timezone
4. **Always specify timezone explicitly** - Don't rely on system default
5. **Use DateTimeFormatter (thread-safe)** - Not SimpleDateFormat
6. **Store as Instant in database** - Convert to user timezone for display
7. **Use Clock for testing** - Inject Clock, fix time in tests
8. **Use ISO-8601 format** - Standard, unambiguous
9. **Remember immutability** - Assign result of operations
10. **Handle DST transitions** - Test around spring/fall changes

### ❌ DON'T

1. **Don't use java.util.Date** - Use java.time API
2. **Don't use SimpleDateFormat** - Not thread-safe
3. **Don't use LocalDateTime for timestamps** - Use Instant
4. **Don't hardcode timezones** - Use ZoneId.of() explicitly
5. **Don't compare LocalDateTime across zones** - Convert to Instant
6. **Don't ignore return values** - Types are immutable
7. **Don't use Period for time** - Use Duration
8. **Don't use Duration for dates** - Use Period
9. **Don't assume 24-hour days** - DST transitions exist
10. **Don't parse without try-catch** - Parsing can fail

---

## Conclusion

The java.time API (JSR-310) is a vast improvement over the legacy Date API:

**Key Principles:**
1. **Immutable** - Thread-safe by design
2. **Clear naming** - LocalDate, Instant, ZonedDateTime are self-explanatory
3. **Fluent API** - Method chaining for readability
4. **Type-safe** - Different types for different needs
5. **ISO-8601** - Standard format by default

**Remember:**
- **Instant** = Machine time (timestamps, logs, storage)
- **ZonedDateTime** = Human time (scheduling, display)
- **LocalDate/Time/DateTime** = Local context only
- **Duration** = Time amount (hours, minutes)
- **Period** = Date amount (years, months, days)

Master these types and you'll handle 99% of date/time scenarios correctly!
