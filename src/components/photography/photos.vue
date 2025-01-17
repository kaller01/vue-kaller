<template>
  <div>
    <v-container>
      <v-overlay :value="overlay" opacity="0.9">
        <overlay
          :overlay-prop="overlayProp"
          :window-size="windowSize"
          @close="closeOverlay"
          @next="nextOverlay"
          @prev="prevOverlay"
          @fullscreen="toggleFullscreen"
        ></overlay>
      </v-overlay>
      <v-row align="center" justify="center"></v-row>
      <v-row justify="space-between">
        <v-col :class="`d-flex justify-start pb-1 px-${text ? 5 : 4} `">
          <v-tooltip bottom>
            <template v-slot:activator="{ on }">
              <v-btn @click="flipPhotos()" height="48px" class="mr-2 pa-0" v-on="on">
                <v-icon class="pa-0">{{order.icon}}</v-icon>
              </v-btn>
            </template>
            <span>Sort by date</span>
          </v-tooltip>

          <v-btn large height="48px">
            <v-icon class="pa-0">mdi-filter-menu</v-icon>
          </v-btn>
        </v-col>
        <v-col :class="`d-flex justify-end pb-1 px-${text ? 5 : 4}  `">
          <v-btn-toggle v-model="columns.mode">
            <v-tooltip bottom v-for="(size,index) in columns.allowedSizes" v-bind:key="index">
              <template v-slot:activator="{ on }">
                <v-btn @click="splitPhotosV2(size)" v-on="on">
                  <v-icon>{{columns.icons[size]}}</v-icon>
                </v-btn>
              </template>
              <span>Show {{size}} photos per row</span>
            </v-tooltip>
          </v-btn-toggle>
        </v-col>
      </v-row>
      <v-row no-gutters v-show="show" class="pt-0 mt-0">
        <v-col
          v-for="(array,index) in splitPhotos"
          v-bind:key="index"
          :class="'d-flex flex-column'+order.reverse+' justify-'+order.justify"
        >
          <div v-for="photo in array" :class="`d-flex pa-${text ? 2 : 1}`" v-bind:key="photo.id">
            <v-hover v-slot:default="{ hover }">
              <photoCard
                :class="`elevation-${hover ? 24 : 0}`"
                :photo="photo"
                :click="showOverlay"
                :text="text"
                :hover="hover"
              ></photoCard>
            </v-hover>
          </div>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script>
import Overlay from "./PhotoOverlay";
import photoCard from "./photoCard";
import { mapState } from "vuex";
import "../../services/findKey.js";

export default {
  name: "Photos",
  data() {
    return {
      hover: false,
      show: false,
      overlay: false,
      overlayProp: {
        path: "path",
        width: "",
        height: "",
        index: 0,
        fullscreen: false
      },
      paddingSize: 2,
      splitPhotos: {},
      order: {
        reverse: "",
        justify: "start",
        icon: "mdi-sort-descending"
      },
      columns: {
        amount: 6,
        size: 4,
        allowedSizes: [],
        mode: 0,
        icons: [
          "none",
          "mdi-numeric-1-box-multiple-outline",
          "mdi-numeric-2-box-multiple-outline",
          "mdi-numeric-3-box-multiple-outline",
          "mdi-numeric-4-box-multiple-outline",
          "mdi-numeric-5-box-multiple-outline",
          "mdi-numeric-6-box-multiple-outline",
          "mdi-numeric-7-box-multiple-outline",
          "mdi-numeric-8-box-multiple-outline",
          "mdi-numeric-9-box-multiple-outline",
          "mdi-numeric-9-plus-box-multiple-outline",
          "mdi-numeric-9-plus-box-multiple-outline",
          "mdi-numeric-9-plus-box-multiple-outline"
        ]
      },
      text: true
    };
  },
  props: {
    photo: Object
  },
  computed: {
    ...mapState(["photos", "windowSize"])
  },
  methods: {
    showOverlay(photo = this.overlayProp.photo, active = true, router = true) {
      if (active) {
        let index = this.photos.indexOf(photo);
        this.overlayProp.photo = photo;
        this.overlayProp.index = index;
        this.overlayProp.path = photo.path;
        this.overlayProp.height = photo.height;
        let width = photo.width / (photo.height / (this.windowSize.y * 0.9));
        if (width > this.windowSize.x * 0.9) {
          this.overlayProp.height = "auto";
          if (
            this.overlayProp.fullscreen &&
            !this.$vuetify.breakpoint.smAndDown
          ) {
            this.overlayProp.width = this.windowSize.x * 0.98;
          } else if (this.windowSize.x < 960) {
            this.overlayProp.fullscreen = true;
            this.overlayProp.width = this.windowSize.x * 0.95;
          } else this.overlayProp.width = this.windowSize.x * 0.8;
        } else {
          this.overlayProp.width = width;
          this.overlayProp.height = this.windowSize.y * 0.9;
        }
        this.overlay = true;
        if (this.$route.params.photo !== photo.filename)
          this.$router.push(
            "/" + this.$i18n.locale + "/photography/" + photo.filename
          );
        if (this.$vuetify.breakpoint.smAndDown)
          this.overlayProp.fullscreen = true;
      }
    },
    onResize() {
      if (this.overlay) this.showOverlay(this.overlayProp.photo);
      this.$store.dispatch("getWindowSize");

      this.showOverlay(this.overlayProp.photo, this.overlay);

      //recalculates the amount of allowedcolumns, uses px because breakpoints werent fast enough
      if (this.windowSize.x > 1904) {
        this.columns.allowedSizes = [4, 6, 9];
      } else if (this.windowSize.x > 1264) {
        this.columns.allowedSizes = [3, 4, 6];
      } else if (this.windowSize.x > 960) {
        this.columns.allowedSizes = [2, 3, 4];
        this.text = true;
      } else {
        this.columns.allowedSizes = [1, 2];
      }

      //changes the current amount of columns so it doesnt overide the max limit
      if (this.columns.amount > this.columns.allowedSizes.slice(-1)[0])
        this.splitPhotosV2(this.columns.allowedSizes.slice(-1)[0]);
      if (this.columns.amount < this.columns.allowedSizes[0])
        this.splitPhotosV2(this.columns.allowedSizes[0]);
      if (Math.max(...this.columns.allowedSizes) === this.columns.amount) {
        this.text = false;
      } else this.text = true;
    },
    closeOverlay(value) {
      this.overlay = value;
      this.$router.push("/" + this.$i18n.locale + "/photography");
    },
    nextOverlay() {
      let index = this.overlayProp.index;
      this.$router.push(
        "/" +
          this.$i18n.locale +
          "/photography/" +
          this.photos[index + 1].filename
      );
      console.log(this.windowSize.y);
    },
    prevOverlay() {
      let index = this.overlayProp.index;
      this.$router.push(
        "/" +
          this.$i18n.locale +
          "/photography/" +
          this.photos[index - 1].filename
      );
    },
    toggleFullscreen(value, reset = false) {
      this.overlayProp.fullscreen = value;
      if (!reset) this.showOverlay();
    },
    //checks if theres an image as param
    photoSelected() {
      if (this.$route.params.photo) {
        this.photos
          .findKey(this.$route.params.photo, "filename")
          .then(photo => {
            this.showOverlay(photo, true, false);
          });
      } else this.overlay = false;
    },
    //Splits the array of photos in cols in the right order
    splitPhotosV2(cols) {
      this.columns.amount = cols;
      this.columns.mode = this.columns.allowedSizes.indexOf(cols);
      //Grid size
      this.columns.size = 12 / cols;
      console.log(this.columns.size);

      const photos = this.photos;
      let splitPhotos = [];
      let heights = [];

      //adds one photo in each cols amount of arryas
      for (let i = 0; i < cols; i++) {
        splitPhotos[i] = [photos[i]];
        heights[i] = photos[i].height / photos[i].width;
      }

      //checks which column contains the images with the least amount of height and ads one more imahe to that array
      this.photos.forEach(async (photo, index) => {
        if (index > cols - 1) {
          const temp = Math.min(...heights);
          const key = heights.indexOf(temp);
          splitPhotos[key].push(photo);
          heights[key] += photo.height / photo.width;
        }
      });
      console.log("Split photos", splitPhotos);
      this.splitPhotos = splitPhotos;
      if (Math.max(...this.columns.allowedSizes) === this.columns.amount) {
        this.text = false;
      } else this.text = true;
    },
    flipPhotos() {
      //flips the order of the columns with flex
      this.order.reverse !== "-reverse"
        ? (this.order.reverse = "-reverse")
        : (this.order.reverse = "");
      this.order.justify !== "start"
        ? (this.order.justify = "start")
        : (this.order.justify = "end");
      this.order.icon !== "mdi-sort-descending"
        ? (this.order.icon = "mdi-sort-descending")
        : (this.order.icon = "mdi-sort-ascending");
    },
    //Adjusts the amount of columns needed depending on screensize.
    initiatePhotos() {
      switch (this.$vuetify.breakpoint.name) {
        case "xl":
          this.splitPhotosV2(6);
          break;
        case "lg":
          this.splitPhotosV2(4);
          break;
        case "md":
          this.splitPhotosV2(3);
          break;
        case "sm":
        case "xs":
          this.splitPhotosV2(2);
          break;
      }
    },
    shortcuts(e) {
      let key = parseInt(e.key);
      if (!isNaN(key)) {
        if (key === 0) key = 12;
        this.splitPhotosV2(key);
        if (!this.columns.allowedSizes.includes(key)) this.text = false;
      }
    },
    test(photo) {
      this.hover = true;
    }
  },
  created() {
    console.log(this.$vuetify.breakpoint.sm);
    setTimeout(() => (this.show = true), 1);
    window.addEventListener("resize", this.onResize);
    window.addEventListener("keydown", e => this.shortcuts(e));
  },
  mounted() {
    if (this.photos.length) {
      this.onResize();
      this.initiatePhotos();
    }
    this.show = true;
  },
  watch: {
    photos: {
      handler: function(val, oldVal) {
        this.photoSelected();
        this.onResize();
        this.initiatePhotos();
      }
    }
  },
  components: {
    Overlay,
    photoCard
  },
  beforeRouteUpdate(to, from, next) {
    //If browser presses backbutton overlay needs to be changed but since the component is still alive that didnt do it by itself
    next();
    this.photoSelected();
  },
  beforeRouteEnter(to, from, next) {
    //since the component isn't rendered at this stage you need to use the viewmodel
    next(vm => {
      vm.photoSelected();
      next();
    });
  }
};
</script>

<style scoped>
</style>
