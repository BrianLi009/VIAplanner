<template>
    <v-row @mouseover="hovered = true" @mouseleave="hovered = false">
        <v-tooltip left v-if="hovered">
            <template v-slot:activator="{ on, attrs }">
                <v-btn
                    v-if="hovered"
                    fab
                    dark
                    small
                    color="green"
                    @click="exportTimetables"
                    v-bind="attrs"
                    v-on="on"
                >
                    <v-icon>mdi-download</v-icon>
                </v-btn>
            </template>
            <span>Export Timetables</span>
        </v-tooltip>
        <v-btn v-else fab dark small color="green">
            <v-icon>mdi-download</v-icon>
        </v-btn>
    </v-row>
</template>

<script>
import axios from "axios";
import { mapGetters, mapMutations } from "vuex";

export default {
    data() {
        return {
            hovered: true,
            exportURL: null,
        };
    },
    computed: {
        ...mapGetters(["getSemesterStatus"]),
    },
    methods: {
        ...mapMutations(["setSemesterStatus", "setExportOverlay"]),

        // timer function for making the system sleep
        sleep(ms) {
            return new Promise((resolve) => setTimeout(resolve, ms));
        },

        // switch timetable download, then repeat
        async exportTimetables() {
            
            this.setExportOverlay(true)

            let fileName = "";

            if (this.getSemesterStatus === "F") {
                this.setSemesterStatus("S");
                fileName = "Winter-Timetable.png";
            } else {
                this.setSemesterStatus("F");
                fileName = "Fall-Timetable.png";
            }

            // wait until the page is full loaded
            await this.sleep(50);
            this.exportCurrTimetable(fileName);

            if (this.getSemesterStatus === "F") {
                this.setSemesterStatus("S");
                fileName = "Winter-Timetable.png";
            } else {
                this.setSemesterStatus("F");
                fileName = "Fall-Timetable.png";
            }

            await this.sleep(50);
            this.exportCurrTimetable(fileName);

            this.setExportOverlay(false)

        },

        async exportCurrTimetable(fileName) {
            // grab the entire screen element
            const el = document.querySelector("#exportMe");

            const options = {
                type: "dataURL",
            };

            try {
                this.exportURL = await this.$html2canvas(el, options);
            } catch (error) {
                console.log("conversion error");
            }

            this.downloadWithAxios(fileName);
        },

        // magic
        forceFileDownload(response, fileName) {
            const url = window.URL.createObjectURL(new Blob([response.data]));
            const link = document.createElement("a");
            link.href = url;
            link.setAttribute("download", fileName);
            // add a new link in the html for download
            document.body.appendChild(link);
            link.click();
            // remove that child
            document.body.removeChild(document.body.lastChild);
        },

        // more magic
        async downloadWithAxios(fileName) {
            let response = null;

            try {
                response = await axios({
                    method: "get",
                    url: this.exportURL,
                    responseType: "arraybuffer",
                });
                this.forceFileDownload(response, fileName);
            } catch (error) {
                console.log("Download Failed");
            }
        },
    },
};
</script>
