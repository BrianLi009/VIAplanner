<template>
  <div>
    <div v-if="event.start > 0">
      <v-dialog v-model="dialog" scrollable width="825px" @input="atInput">
        <template v-slot:activator="{ on }">
          <div
            @mouseover="hovered = true"
            @mouseleave="hovered = false"
            v-on="on"
            class="event"
            :class="durationClass(event.start, event.end)"
            :style="{ background: getCourseColor(event.code) }"
          >
            <h4>{{ event.code }}</h4>

            <div class="lock-button">
              <v-btn small dark @click.stop="lockToggle" v-if="locked" icon>
                <v-icon>mdi-lock</v-icon>
              </v-btn>
              <v-btn small dark @click.stop="lockToggle" v-if="!locked && hovered" icon>
                <v-icon>mdi-lock-open</v-icon>
              </v-btn>
            </div>

            <v-row class="px-3">
              <div>{{ event.sectionCode }} ({{deliveryMethod}})</div>
              <v-spacer />
              <div v-if="locations.length === 2 && durationClass === 'one-hour'">{{locations[0]}}</div>
            </v-row>

            <v-row class="px-3">
              <div>{{ getFormattedTime(event.start, event.end) }}</div>
              <v-spacer />
              <div v-if="locations.length === 1 || durationClass != 'one-hour'">{{locations[0]}}</div>
              <div
                v-else-if="locations.length === 2 && durationClass === 'one-hour'"
              >{{locations[1]}}</div>
            </v-row>
            <v-row class="px-3" v-if="locations.length === 2 && durationClass != 'one-hour'">
              <v-spacer />
              <div>{{locations[1]}}</div>
            </v-row>
          </div>
        </template>
        <course-section-picker v-on:done="dialog = false" :code="event.code" ref="popUp" />,
      </v-dialog>
    </div>
    <div
      v-else-if="checkHalfHour(event.currStart, event.currEnd)"
      class="event empty-event half-hour"
    />
    <div
      v-else
      v-ripple
      class="event empty-event one-hour"
      :style="dynamicColor"
      @mouseover="hovered = true"
      @mouseleave="hovered = false"
      @click.stop="lockedSectionToggle"
    >
      <div v-if="hovered">
        <v-row>
          <v-col>
            <p class="center unselectable">{{ dynamicText }}</p>
          </v-col>
        </v-row>
      </div>
    </div>
  </div>
</template>

<script>
import CourseSectionPicker from "../Popup/CourseSectionPicker";
import { mapGetters, mapActions, mapMutations } from "vuex";
const convertSecondsToHours = seconds => {
  return seconds / 3600;
};
export default {
  name: "timetable-event",
  props: {
    event: {
      type: Object,
      default: () => {}
    },
    currDay: {
      type: String,
      default: ""
    }
  },
  components: {
    CourseSectionPicker
  },
  data() {
    return {
      hovered: false,
      dialog: false
    };
  },
  computed: {
    ...mapGetters(["getCourseColor", "getLockedSections"]),
    deliveryMethod() {
      if (this.event.sectionCode[1] === "0") {
        return "In Person";
      } else if (this.event.sectionCode[1] === "8") {
        return "Rotate";
      } else {
        return "Sync";
      }
    },
    locations() {
      return this.event.location.split("; ");
    },
    dynamicText() {
      return !this.locked ? "Block This Time" : "Unblock This Time";
    },
    // change the color in the event so it correct based on hovering or locked
    dynamicColor() {
      if (this.locked) {
        return { background: "#d9d9d9", cursor: "pointer" };
      } else {
        return this.hovered
          ? { background: "#d9d9d9", cursor: "pointer" }
          : { background: "white", cursor: "pointer" };
      }
    },
    // stores the info of the current section
    currSecData() {
      return {
        name: `Locked Section`,
        courseCode: `Lock${this.currDay}${this.event.currStart}`,
        meeting_sections: [
          {
            sectionCode: "L0001",
            instructors: ["NA"],
            times: [
              {
                day: this.currDay,
                start: this.event.currStart,
                end: this.event.currStart + 3600,
                location: "NA"
              }
            ]
          }
        ]
      };
    },
    // lock the status of the current section
    locked() {
      for (var section of this.getLockedSections) {
        if (
          section === `${this.event.code}${this.event.sectionCode}` ||
          section ===
            `${this.currSecData.courseCode}${this.currSecData.meeting_sections[0].sectionCode}`
        ) {
          return true;
        }
      }
      return false;
    }
  },
  methods: {
    ...mapActions(["deleteCourse"]),
    ...mapMutations(["lockSection", "unlockSection", "addCourse"]),
    atInput() {
      var courseSectionPicker = this.$refs.popUp;
      if (typeof courseSectionPicker != "undefined") {
        courseSectionPicker.resetSelectedMeetingSections();
      }
    },
    checkHalfHour(currStart, end) {
      // if one hour
      if (Number.isInteger((end - currStart) / 3600)) {
        return false;
      }
      // half hour
      return true;
    },
    //Toggle locked status of this TimetableEvent when it is not empty (lock/unlock this section)
    lockToggle() {
      // modifies vuex based on the current section's lock status
      !this.locked
        ? this.lockSection(`${this.event.code}${this.event.sectionCode}`)
        : this.unlockSection(`${this.event.code}${this.event.sectionCode}`);
    },
    durationClass(start, end) {
      const duration = convertSecondsToHours(end - start);
      if (duration === 1) {
        return "one-hour";
      } else if (duration === 1.5) {
        return "one-hour-half";
      } else if (duration === 2) {
        return "two-hours";
      } else if (duration === 2.5) {
        return "two-hours-half";
      } else if (duration === 3) {
        return "three-hours";
      } else if (duration === 4) {
        return "four-hours";
      }
    },
    getFormattedTime(start, end) {
      var s = (start / 3600) % 12;
      if (s == 0) {
        s = 12;
      }
      var e = (end / 3600) % 12;
      if (e == 0) {
        e = 12;
      }
      let startHalf = Number.isInteger(s) ? "00" : "30";
      let endHalf = Number.isInteger(e) ? "00" : "30";
      return `${s - startHalf / 6 / 10}:${startHalf} - ${e -
        endHalf / 6 / 10}:${endHalf}`;
    },
    //Toggle locked status of this TimetableEvent when it is empty (block/unblock this hour)
    lockedSectionToggle() {
      if (!this.locked) {
        // if the user clicks on an empty timeslot, it will be added as a course in vuex
        this.addCourse({ course: this.currSecData });
        this.lockSection(
          `${this.currSecData.courseCode}${this.currSecData.meeting_sections[0].sectionCode}`
        );
      } else {
        // if the user clicks on a lock timeslot, it will be removed
        this.deleteCourse({ code: this.currSecData.courseCode });
      }
    }
  }
};
</script>

<style scoped>
@import url("https://fonts.googleapis.com/css?family=Montserrat&display=swap");
* {
  font-family: "Montserrat", sans-serif;
}
.unselectable {
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}
.center {
  color: black;
  text-align: center;
  padding-top: 8px !important;
}
.event {
  border: 1px solid gray;
  color: white;
  padding: 8px;
  padding-top: 3px;
  position: relative;
  cursor: pointer;
  font-size: small;
}
.empty-event {
  background: white;
  border: 0.2px solid gray;
  cursor: default;
}
.half-hour {
  height: 32px;
}
.one-hour {
  height: 64px;
}
.one-hour-half {
  height: 96px;
}
.two-hours {
  height: 128px;
}
.two-hours-half {
  height: 160px;
}
.three-hours {
  height: 192px;
}
.four-hours {
  height: 336px;
}
.course-code {
  margin-left: 3px;
}
.align-left {
  position: absolute;
  left: 0px;
}
.align-right {
  position: absolute;
  right: 0px;
}
.lock-button {
  position: absolute;
  right: 3px;
  top: 3px;
  z-index: 1;
}
</style>
