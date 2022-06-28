<template>
  <q-page class="extractor-page">
    <input
      ref="input_image"
      type="file"
      accept="image/*"
      multiple
      style="display: none"
      @change="updateImage"
    />
    <q-bar
      class="q-electron-drag"
      style="display: flex; flex-direction: row-reverse; width: 50vw"
    >
      <q-btn dense flat icon="close" @click="closeApp" />
      <q-btn dense flat icon="crop_square" @click="toggleMaximize" />
      <q-btn dense flat icon="minimize" @click="minimize" />
    </q-bar>
    <div class="header-content fullcover">
      <div class="header-item"></div>
      <div class="header-item tab-section">
        <div v-if="showHighlight" class="tab">
          <div class="btn-container">
            <i class="fa fa-sun-o" aria-hidden="true"></i>
            <label class="switch btn-color-mode-switch">
              <input
                type="checkbox"
                name="color_mode"
                id="color_mode"
                value="1"
                v-model="isShowMap"
              />
              <label
                for="color_mode"
                data-on="Map"
                data-off="Exif"
                class="btn-color-mode-switch-inner"
              ></label>
            </label>
          </div>
        </div>
      </div>
    </div>
    <div class="main-content fullcover">
      <div class="image-list-container">
        <div class="empty-image fullcover">
          <q-card
            v-if="imageFiles.length === 0"
            class="empty-image-card shadow-8"
            @click="onClickAddImage"
          >
            <q-img
              class="empty-image-image"
              src="../assets/empty_item.svg"
            ></q-img>
            <div class="text-h5 text-accent text-weight-bold text-center">
              Empty bucket, click to add an image...
            </div>
          </q-card>
          <div
            v-else
            class="fullcover"
            style="
              display: flex;
              flex-direction: column;
              padding-bottom: 0.3em;
              padding-left: 0.5em;
            "
          >
            <q-scroll-area class="images-container fullcover">
              <q-card
                v-for="image in displayImage"
                :key="image.name"
                class="image-card shadow-5"
                :class="isShowMap && image.gps == null ? 'no-gps-card' : ''"
                @click="onClickDisplayInfo(image)"
              >
                <div class="preview-text">
                  <div class="text-h6 text-white">
                    {{
                      image.name.length > 20
                        ? image.name.substring(0, 20) + "..."
                        : image.name
                    }}
                  </div>
                </div>
                <div
                  class="preview-image"
                  :style="{ 'background-image': 'url(' + image.url + ')' }"
                ></div>
              </q-card>
            </q-scroll-area>
            <q-btn @click="onClickAddImage" color="green">Add Image</q-btn>
          </div>
        </div>
      </div>
      <div class="image-info-container">
        <div
          v-if="showHighlight && !isShowMap"
          class="highlight-container fullcover"
        >
          <div class="highlight-image-container">
            <q-img class="highlight-image" :src="highlightImage.url"></q-img>
            <!-- <div class="highlight-image-loading"></div> -->
          </div>
          <div v-if="showExifData" class="highlight-exif">
            <q-scroll-area class="fullcover">
              <div
                v-for="(title, category) in highlightImage.exifdata"
                :key="category"
                class="exif-container"
              >
                <div class="exif-header">
                  <q-separator color="orange" size="2px" />
                  <div class="exif-title text-h6">
                    {{ category.toUpperCase() }}
                  </div>
                </div>
                <div
                  v-for="(value, titleKey) in title"
                  :key="titleKey"
                  class="exif-body"
                >
                  <span class="text-overline"> {{ titleKey }} : </span
                  ><span class="text-caption"> {{ value }}</span>
                </div>
              </div>
            </q-scroll-area>
          </div>
        </div>
        <div v-if="showHighlight && isShowMap">
          <l-map
            style="height: 80vh"
            :zoom="zoom"
            :center="center"
            @update:center="centerUpdated"
          >
            <l-tile-layer :url="url" :attribution="attribution"></l-tile-layer>
            <div v-for="image in displayImage" :key="image.name">
              <l-marker
                v-if="image.gps"
                :lat-lng="[image.gps?.lat, image.gps?.lng]"
                @click="openPopUp(image)"
              >
                <l-popup>
                  <div>
                    <img
                      @click="onClickPopUpImage(image)"
                      :src="popUpImageUrl"
                      style="width: 104px; height: 142px"
                    />
                    <div class="text-weight-bold">
                      {{ image.name }}
                    </div>
                  </div>
                </l-popup>
              </l-marker>
            </div>
          </l-map>
        </div>
      </div>
    </div>
  </q-page>
</template>

<script>
import "leaflet/dist/leaflet.css";
import { LMap, LTileLayer, LMarker, LPopup } from "@vue-leaflet/vue-leaflet";
import _ from "lodash";
import ExifReader from "exifreader";

export default {
  components: {
    LMap,
    LTileLayer,
    LMarker,

    LPopup,
  },
  name: "extractorPage",
  data() {
    return {
      url: "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
      attribution:
        '&copy; <a target="_blank" href="http://osm.org/copyright">OpenStreetMap</a> contributors',
      zoom: 16,
      center: [3.140853, 101.693207],
      markerLatLng: [3.140853, 101.693207],
      imageFiles: [],
      highlightReady: false,
      highlightImage: {},
      isShowMap: false,
      popUpImageUrl: "",
      total: {
        lng: [],
        lat: [],
      },
    };
  },
  computed: {
    displayImage() {
      let res = [];

      this.imageFiles.forEach((item) => {
        let imgObj = {};

        imgObj.name = item.name;
        imgObj.url = URL.createObjectURL(item);
        imgObj.gps = item.gps;

        res.push(imgObj);
      });

      return res;
    },
    showHighlight() {
      return !_.isEmpty(this.highlightImage);
    },
    showExifData() {
      return !_.isEmpty(_.get(this.highlightImage, "exifdata"));
    },
  },
  methods: {
    updateImage() {
      const files = this.$refs["input_image"].files;
      if (files.length === 0) {
        return;
      }

      let vm = this;

      let i = files.length;
      for (const file of files) {
        if (
          this.imageFiles.filter((item) => item.name === file.name).length > 0
        )
          continue;

        ExifReader.load(file)
          .then(function (tags) {
            delete tags["MakerNote"];
            let lat = tags["GPSLatitude"]?.description || null;
            let lng = tags["GPSLongitude"]?.description || null;

            if (lat != null && lng != null) {
              vm.total.lng.push(lng);
              vm.total.lat.push(lat);
              file.gps = { lat, lng };
            }

            vm.imageFiles.push(file);
            vm.onClickDisplayInfo(file);
          })
          .catch(function (error) {
            // Handle error.
          })
          .finally(() => {
            if (!--i) vm.updateCenter();
          });
      }
    },
    onClickAddImage() {
      this.$refs["input_image"].click();
    },
    centerUpdated(center) {
      this.center = center;
    },
    setMapCenter(gps) {
      if (gps == null) return;
      this.center = [gps.lat, gps.lng];
    },
    updateCenter() {
      const allLat = this.total.lat;
      const allLng = this.total.lng;

      const arrSum = (arr) => arr.reduce((a, b) => a + b, 0);

      if (allLat.size === 0 || allLng.size === 0) {
        this.center = [3.140853, 101.693207];
        return;
      }

      const totalLat = arrSum(allLat);
      const totalLng = arrSum(allLng);

      this.center = [totalLat / allLat.length, totalLng / allLng.length];
    },
    onClickDisplayInfo(image) {
      let vm = this;

      if (vm.isShowMap) {
        this.setMapCenter(image.gps);
        return;
      }

      let imageName = image.name;
      const imageFile = this.imageFiles.filter(
        (item) => item.name === imageName
      )[0];

      ExifReader.load(imageFile)
        .then(function (tags) {
          delete tags["MakerNote"];

          vm.highlightImage = {
            url: URL.createObjectURL(imageFile),
            exifdata: vm.extractExif(tags, imageFile),
          };
        })
        .catch(function (error) {
          // Handle error.
        });
    },
    openPopUp(image) {
      const imageFile = this.imageFiles.filter(
        (item) => item.name === image.name
      )[0];

      this.popUpImageUrl = URL.createObjectURL(imageFile);
    },
    onClickPopUpImage(image) {
      this.isShowMap = false;
      this.onClickDisplayInfo(image);
    },
    extractExif(tags, imageFile) {
      let camera = {
        "Camera Maker": "Make",
        "Camera Model": "Model",
        "ISO Speed": "ISOSpeedRatings",
        "Exposure Time": "ExposureTime",
        "Focal Length": "FocalLength",
        "Flash Mode": "Flash",
        "Max Aperture": "ApertureValue",
      };

      let file = {
        "File Name": "name",
        Format: "type",
        "Date created": "DateTimeOriginal",
        "Date modified": "DateTime",
        Size: "size",
      };

      let GPS = {
        Longitude: "GPSLongitude",
        Latitude: "GPSLatitude",
      };

      let origin = {
        "Date Taken": "DateTimeOriginal",
        Dimension: "ImageWidth",
        Width: "ImageWidth",
        Height: "ImageLength",
        "Bit depth": "Bits Per Sample",
        "Color Scheme": "ColorSpace",
      };
      console.log(tags);

      let software = {
        Software: "Software",
      };

      let exifType = {
        origin,
        GPS,
        file,
        camera,
        software,
      };

      _.forEach(exifType, (title, category) => {
        _.forEach(title, (exifKey, titleKey) => {
          let value = tags[exifKey]?.description || "N/A";
          exifType[category][titleKey] = value;
        });
      });

      exifType.file["File Name"] = imageFile.name;
      exifType.file["Format"] = imageFile.type;
      exifType.file["Size"] = this.formatBytes(parseInt(imageFile.size));

      return exifType;
    },
    formatBytes(bytes, decimals = 2) {
      if (bytes === 0) return "0 Bytes";

      const k = 1024;
      const dm = decimals < 0 ? 0 : decimals;
      const sizes = ["Bytes", "KB", "MB", "GB", "TB", "PB", "EB", "ZB", "YB"];

      const i = Math.floor(Math.log(bytes) / Math.log(k));

      return parseFloat((bytes / Math.pow(k, i)).toFixed(dm)) + " " + sizes[i];
    },
    closeApp() {
      if (process.env.MODE === "electron") {
        window.myWindowAPI.close();
      }
    },
    minimize() {
      if (process.env.MODE === "electron") {
        window.myWindowAPI.minimize();
      }
    },

    toggleMaximize() {
      if (process.env.MODE === "electron") {
        window.myWindowAPI.toggleMaximize();
      }
    },
  },
};
</script>
