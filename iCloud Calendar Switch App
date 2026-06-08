/**
 *  iCloud Calendar Switch
 *  Hubitat Elevation App
 *
 *  Fetches a public iCal/ICS URL, finds today's events, and schedules
 *  a virtual switch ON at event start and OFF at event end.
 *
 *  Runs daily at 9:00 AM. Also has a "Run Now" button for manual use.
 *
 *  SETUP:
 *  1. In Apple Calendar, make the Jackie calendar public and copy the URL
 *  2. Change webcal:// to https:// in that URL
 *  3. Install this app via Apps Code > New App, then Apps > Add User App
 *  4. Enter the ICS URL and select your virtual switch
 */

definition(
    name:        "iCloud Calendar Switch",
    namespace:   "jlslate",
    author:      "Hubitat User",
    description: "Turns a switch ON/OFF based on today's events in a public iCal feed.",
    category:    "Utility",
    iconUrl:     "",
    iconX2Url:   ""
)

preferences {
    page(name: "mainPage")
}

def mainPage() {
    dynamicPage(name: "mainPage", title: "iCloud Calendar Switch", install: true, uninstall: true) {

        section("App Name") {
            label title: "Name this app", required: false
        }

        section("Calendar") {
            input "icsUrl", "string", title: "Public ICS URL (https://...)", required: true
        }

        section("Switch") {
            input "triggerSwitch", "capability.switch",
                  title: "Virtual switch to turn ON/OFF", required: true, multiple: false
        }

        section("Schedule") {
            input "runHour",   "number", title: "Hour to run daily (0-23, 24hr format)", required: true, defaultValue: 9, range: "0..23"
            input "runMinute", "number", title: "Minute to run daily (0-59)",            required: true, defaultValue: 0, range: "0..59"
        }

        section("Logging") {
            input "logEnable", "bool", title: "Enable debug logging", defaultValue: false
        }

        section("Manual Run") {
            input "runNow", "button", title: "Run Now"
        }
    }
}

// ─── Lifecycle ────────────────────────────────────────────────────────────────

def installed()   { log.info "[iCloud Cal] Installed";  initialize() }
def updated()     { log.info "[iCloud Cal] Updated";    unschedule(); initialize() }
def uninstalled() { unschedule(); triggerSwitch?.off() }

def initialize() {
    def h = runHour   != null ? runHour.toInteger()   : 9
    def m = runMinute != null ? runMinute.toInteger() : 0
    schedule("0 ${m} ${h} * * ?", checkCalendar)
    log.info "[iCloud Cal] Scheduled daily at ${String.format('%02d', h)}:${String.format('%02d', m)}"
    checkCalendar()
}

def appButtonHandler(btn) {
    if (btn == "runNow") {
        log.info "[iCloud Cal] Manual run triggered"
        checkCalendar()
    }
}

// ─── Main Logic ───────────────────────────────────────────────────────────────

def checkCalendar() {
    log.info "[iCloud Cal] Checking calendar"
    triggerSwitch.off()
    unschedule(switchOn)
    unschedule(switchOff)

    try {
        def icsData = null
        httpGet([uri: icsUrl, textParser: true]) { resp ->
            if (resp.status == 200) {
                icsData = resp.data.text
                logDebug("ICS fetched, length: ${icsData?.length()}")
            } else {
                log.warn "[iCloud Cal] HTTP ${resp.status}"
            }
        }

        if (!icsData) { log.warn "[iCloud Cal] No data returned"; return }

        def tz     = location.timeZone
        def events = parseICS(icsData, tz)
        log.info "[iCloud Cal] Today's events: ${events.size()}"

        events.each { evt ->
            def startDate = new Date(evt.startMs)
            def endDate   = new Date(evt.endMs)
            log.info "[iCloud Cal] Event: '${evt.summary}' ${startDate} → ${endDate}"

            def nowMs   = now()
            def startMs = evt.startMs
            def endMs   = evt.endMs

            if (endMs <= nowMs) {
                log.info "[iCloud Cal] Already ended, skipping"
            } else if (startMs <= nowMs) {
                log.info "[iCloud Cal] In progress — switch ON now"
                triggerSwitch.on()
                runOnce(endDate, switchOff)
            } else {
                log.info "[iCloud Cal] Scheduling ON at ${startDate}, OFF at ${endDate}"
                runOnce(startDate, switchOn)
                runOnce(endDate,   switchOff)
            }
        }

        if (!events) log.info "[iCloud Cal] No events today"

    } catch (Exception e) {
        log.error "[iCloud Cal] Error: ${e.message}"
    }
}

def switchOn()  { log.info "[iCloud Cal] Switch ON";  triggerSwitch.on()  }
def switchOff() { log.info "[iCloud Cal] Switch OFF"; triggerSwitch.off() }

// ─── ICS Parser ───────────────────────────────────────────────────────────────

def parseICS(String ics, TimeZone tz) {
    def results = []
    if (!ics) return results

    // Unfold lines (RFC 5545 — continuation lines start with space or tab)
    def unfolded = ics.replaceAll(/\r?\n[ \t]/, "")

    def todayStr = new java.text.SimpleDateFormat("yyyyMMdd").with { it.setTimeZone(tz); it }.format(new Date())

    // Find all VEVENT blocks
    def vevents = []
    unfolded.findAll(/(?s)BEGIN:VEVENT(.*?)END:VEVENT/) { full, inner -> vevents << inner }

    // Separate base recurring events, overrides, and one-offs
    def baseEvents = [:]   // uid -> vevent text
    def overrides  = [:]   // uid:dateStr -> vevent text
    def oneOffs    = []

    vevents.each { ve ->
        def uid    = prop(ve, "UID")
        def recId  = prop(ve, "RECURRENCE-ID")
        def rrule  = prop(ve, "RRULE")

        if (recId) {
            def dt = parseDT(recId, tz)
            if (dt) overrides["${uid}:${fmtDate(dt, tz)}"] = ve
        } else if (rrule) {
            baseEvents[uid] = ve
        } else {
            oneOffs << ve
        }
    }

    // Process one-off events
    oneOffs.each { ve ->
        def startRaw = prop(ve, "DTSTART")
        def endRaw   = prop(ve, "DTEND")
        def startDt  = parseDT(startRaw, tz)
        def endDt    = parseDT(endRaw, tz)
        if (startDt && fmtDate(startDt, tz) == todayStr) {
            results << [summary: prop(ve, "SUMMARY") ?: "Event",
                        startMs: startDt.time,
                        endMs:   endDt ? endDt.time : startDt.time + 3600000]
        }
    }

    // Expand recurring events
    baseEvents.each { uid, ve ->
        def rruleRaw = prop(ve, "RRULE")
        def startRaw = prop(ve, "DTSTART")
        def endRaw   = prop(ve, "DTEND")
        def startDt  = parseDT(startRaw, tz)
        def endDt    = parseDT(endRaw, tz)
        if (!startDt || !rruleRaw) return

        def duration = endDt ? (endDt.time - startDt.time) : 3600000L

        // Parse RRULE
        def parts    = rruleRaw.split(";").collectEntries { p ->
            def kv = p.split("=", 2); [(kv[0]): (kv.size() > 1 ? kv[1] : "")]
        }
        def freq     = parts.FREQ ?: ""
        def interval = (parts.INTERVAL ?: "1").toInteger()
        def until    = parts.UNTIL ? parseDT(parts.UNTIL, tz) : null
        def count    = parts.COUNT ? parts.COUNT.toInteger() : null

        // Collect EXDATEs
        def exdates = [] as Set
        unfolded.findAll(/(?m)^EXDATE(?:;[^:]*)?:(.+)$/) { full, val ->
            def ex = parseDT(val.trim(), tz)
            if (ex) exdates << fmtDate(ex, tz)
        }

        if (freq == "WEEKLY") {
            def occurrence = new Date(startDt.time)
            def limit      = new Date(new Date().time + 366L * 24 * 60 * 60 * 1000)
            def occCount   = 0

            while (!occurrence.after(limit)) {
                def occStr = fmtDate(occurrence, tz)
                def withinUntil = (until == null || !occurrence.after(until))
                def withinCount = (count == null || occCount < count)

                if (withinUntil && withinCount && !exdates.contains(occStr)) {
                    if (occStr == todayStr) {
                        def overrideKey = "${uid}:${occStr}"
                        if (overrides.containsKey(overrideKey)) {
                            def ov       = overrides[overrideKey]
                            def ovStart  = parseDT(prop(ov, "DTSTART"), tz)
                            def ovEnd    = parseDT(prop(ov, "DTEND"), tz)
                            if (ovStart) {
                                results << [summary: prop(ov, "SUMMARY") ?: "Event",
                                            startMs: ovStart.time,
                                            endMs:   ovEnd ? ovEnd.time : ovStart.time + duration]
                            }
                        } else {
                            def occEnd = new Date(occurrence.time + duration)
                            results << [summary: prop(ve, "SUMMARY") ?: "Event",
                                        startMs: occurrence.time,
                                        endMs:   occEnd.time]
                        }
                    }
                    occCount++
                }
                // Advance by interval weeks
                occurrence = new Date(occurrence.time + (interval * 7L * 24 * 60 * 60 * 1000))
            }
        }
        // Additional FREQ types (DAILY, MONTHLY, YEARLY) can be added here if needed
    }

    return results
}

// ─── Helpers ──────────────────────────────────────────────────────────────────

/** Extract a property value from a VEVENT block, handling TZID parameters. */
def prop(String block, String name) {
    def m = block.find(~"(?m)^${name}(?:;[^:]*)?:(.+)\$") { full, val -> val?.trim() }
    return m ?: null
}

/**
 * Parse a date/time string into a Date.
 * Handles: 20260603T201700Z (UTC), 20260603T201700 (floating → hub tz), 20260603 (all-day)
 */
def parseDT(String val, TimeZone tz) {
    if (!val) return null
    val = val.trim()
    try {
        if (val.length() >= 15 && val.contains("T")) {
            if (val.endsWith("Z")) {
                def fmt = new java.text.SimpleDateFormat("yyyyMMdd'T'HHmmss'Z'")
                fmt.setTimeZone(TimeZone.getTimeZone("UTC"))
                return fmt.parse(val)
            } else {
                def fmt = new java.text.SimpleDateFormat("yyyyMMdd'T'HHmmss")
                fmt.setTimeZone(tz)
                return fmt.parse(val.substring(0, 15))
            }
        } else if (val.length() >= 8) {
            def fmt = new java.text.SimpleDateFormat("yyyyMMdd")
            fmt.setTimeZone(tz)
            return fmt.parse(val.substring(0, 8))
        }
    } catch (Exception e) { logDebug("parseDT failed '${val}': ${e.message}") }
    return null
}

/** Format a Date as yyyyMMdd in hub timezone for comparison. */
def fmtDate(Date d, TimeZone tz) {
    def fmt = new java.text.SimpleDateFormat("yyyyMMdd")
    fmt.setTimeZone(tz)
    return fmt.format(d)
}

def logDebug(String msg) { if (logEnable) log.debug "[iCloud Cal] ${msg}" }
