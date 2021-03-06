<template>
    <div>
        <v-container class="background" style="padding-right: 50px !important">
            <v-row>
                <NoTimetablePopup></NoTimetablePopup>
                <v-col class="time-axis">
                    <div class="top-margin"></div>
                    <v-row
                        v-for="(time, index) in timeRange"
                        :key="index"
                        class="time-axis-number"
                    >
                        <hour-switch
                            v-if="index != timeRange.length - 1"
                            :time="time"
                            :last="false"
                        ></hour-switch>
                        <hour-switch v-else :time="time" :last="true"></hour-switch>
                    </v-row>
                </v-col>
                <v-col cols="11">
                    <v-row name="week-days-axis">
                        <v-col v-for="weekday in weekdays" :key="weekday">
                            <weekday-switch :weekday="weekday"></weekday-switch>
                        </v-col>
                    </v-row>
                    <v-row name="timetable-content">
                        <v-col
                            v-for="(meetingSections, day) in timetable"
                            :key="day"
                        >
                            <div
                                v-for="event in getEventsForDay(meetingSections)"
                                :key="event.start"
                            >
                                <timetable-event
                                    :event="event"
                                    v-if="event.start > 0"
                                />
                                <timetable-event
                                    :event="event"
                                    v-else
                                    :currDay="day"
                                />
                            </div>
                        </v-col>
                    </v-row>
                </v-col>
            </v-row>
        </v-container>
        <div v-if="getExportOverlay">
            <timetable-course-card
                class="my-4 mr-7 ml-11"
                v-for="(course, code) in getSelectedCourses"
                :key="code"
                :course="course"
            />
        </div>
    </div>
</template>

<script>
import TimetableEvent from "./TimetableEvent";
import NoTimetablePopup from "../Popup/NoTimetablePopup";
import HourSwitch from "./HourSwitch";
import WeekdaySwitch from "./WeekdaySwitch";
import TimetableCourseCard from "./TimetableCourseCard";
import { mapMutations, mapGetters } from "vuex";

const convertSecondsToHours = (seconds) => {
    return seconds / 3600;
};

export default {
    name: "Timetable",
    components: {
        TimetableEvent,
        WeekdaySwitch,
        HourSwitch,
        NoTimetablePopup,
        TimetableCourseCard,
    },
    props: {
        timetable: {
            type: Object,
        },
    },
    computed: {
        ...mapGetters(["getLockedSections", "selectedCourses", "getExportOverlay"]),
        timetableStart() {
            var earliest = 9;
            for (let day in this.timetable) {
                const dayEvents = this.timetable[day];
                for (let event of dayEvents) {
                    const start = convertSecondsToHours(event.start);
                    if (start < earliest && !event.code.includes("Lock")) {
                        earliest = start;
                    }
                }
            }
            return earliest;
        },
        timetableEnd() {
            var latest = 18;
            for (let day in this.timetable) {
                const dayEvents = this.timetable[day];
                for (let event of dayEvents) {
                    const end = convertSecondsToHours(event.end);
                    if (end > latest && !event.code.includes("Lock")) {
                        latest = end;
                    }
                }
            }
            return latest;
        },
        timeRange() {
            const result = [];
            for (let i = this.timetableStart; i <= this.timetableEnd; i++) {
                if (i > 12) {
                    result.push(`${i % 12} PM`);
                } else if (i == 12) {
                    result.push(`${12} PM`);
                } else {
                    result.push(`${i % 12} AM`);
                }
            }
            return result;
        },
        // filters user lock timeslots
        getSelectedCourses() {
            this.timetable; //force re-render the selected courses
            const filteredCourses = {};
            for (var code in this.selectedCourses) {
                if (!code.includes("Lock")) {
                    filteredCourses[code] = this.selectedCourses[code];
                }
            }
            return filteredCourses;
        },
    },
    data() {
        return {
            colors: ["#FBB347", "#83CC77", "#4C91F9", "#F26B83", "#5CD1EB"],
            weekdays: ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
        };
    },
    methods: {
        ...mapMutations(["lockSection", "unlockSection"]),
        getEventsForDay(meetingSections) {
            const result = [];
            let currTime = this.timetableStart;
            let invalidStart = -1;
            if (meetingSections.length === 0) {
                for (let j = 0; j < this.timetableEnd - this.timetableStart; j++) {
                    result.push({
                        start: invalidStart,
                        currStart: (currTime + j) * 3600,
                        currEnd: (currTime + j + 1) * 3600,
                    });
                    invalidStart--;
                }
                return result;
            }
            for (let i = 0; i < meetingSections.length; i++) {
                const event = meetingSections[i];
                let eventStart = convertSecondsToHours(event.start);
                let eventEnd = convertSecondsToHours(event.end);

                // if the current locked event starts before the timetable start time
                if (eventStart < this.timetableStart) {
                    continue;
                } else if (eventStart >= this.timetableEnd) {
                    break;
                }

                //Pad empty hour or half hours before the event
                //If event starts at whole hour
                if (Number.isInteger(eventStart - currTime)) {
                    for (let j = 0; j < eventStart - currTime; j++) {
                        result.push({
                            start: invalidStart,
                            currStart: (currTime + j) * 3600,
                            currEnd: (currTime + j + 1) * 3600,
                        });
                        invalidStart--;
                    }
                }
                //If event starts at half hour
                else {
                    //there is half hour exist
                    //previous end time is one hour
                    if (Number.isInteger(currTime)) {
                        for (let j = 0; j < eventStart - currTime - 1; j++) {
                            result.push({
                                start: invalidStart,
                                currStart: (currTime + j) * 3600,
                                currEnd: (currTime + j + 1) * 3600,
                            });
                            invalidStart--;
                        } //pushing in half hour
                        result.push({
                            start: invalidStart,
                            currStart: (eventStart - 0.5) * 3600,
                            currEnd: eventStart * 3600,
                        });
                        invalidStart--;
                        // previous end time is full hour
                    } else {
                        result.push({
                            start: invalidStart,
                            currStart: currTime * 3600,
                            currEnd: (currTime + 0.5) * 3600,
                        });
                        invalidStart--;
                        for (let j = 0; j < eventStart - (currTime + 0.5); j++) {
                            result.push({
                                start: invalidStart,
                                currStart: (currTime + 0.5 + j) * 3600,
                                currEnd: (currTime + 0.5 + j + 1) * 3600,
                            });
                            invalidStart--;
                        }
                    }
                }

                //Make a block for the current event
                // if the section is a user locked section, pass it in as a locked event
                if (event.code.includes("Lock")) {
                    result.push({
                        start: invalidStart,
                        currStart: event.start,
                        currEnd: event.start + 3600,
                    });
                    invalidStart--;
                } else {
                    event["currStart"] = event.start;
                    result.push(event);
                }

                currTime = eventEnd;

                //If last event, pad empty events after it
                if (
                    i === meetingSections.length - 1 ||
                    convertSecondsToHours(meetingSections[i + 1].start) >=
                        this.timetableEnd
                ) {
                    //half hour
                    if (!Number.isInteger(currTime)) {
                        result.push({
                            start: invalidStart,
                            currStart: currTime * 3600,
                            currEnd: (currTime + 0.5) * 3600,
                        });
                        invalidStart--;
                        currTime = currTime + 0.5;
                    }
                    for (let k = 0; k < this.timetableEnd - currTime; k++) {
                        result.push({
                            start: invalidStart,
                            currStart: (currTime + k) * 3600,
                            currEnd: (currTime + k + 1) * 3600,
                        });
                        invalidStart--;
                    }
                }
            }
            return result;
        },
    },
};
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css?family=Montserrat&display=swap");
* {
    font-family: "Montserrat", sans-serif;
}
.col {
    padding: 0px !important;
}

.background {
    background-color: white;
}

.container {
    padding-left: 24px !important;
    padding-right: 70px !important;
    padding-top: 20px !important;
    padding-bottom: 0px !important;
}

.time-axis-number {
    height: 64px;
    text-align: right;
}
.top-margin {
    margin-bottom: 25px;
}

.time-axis {
    margin-right: 20px;
}

.time-label {
    text-align: right;
}
</style>
